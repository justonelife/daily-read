# README — Hệ thống Daily-Read Autonomous

## 1. Giới thiệu

**Daily-Read Autonomous** là một hệ thống tự động hóa được kích hoạt bởi thẻ **Hermes Kanban**. Ý tưởng cốt lõi: bạn chỉ cần **tạo một thẻ Kanban**, và toàn bộ quy trình sẽ tự chạy từ việc phân tích yêu cầu, thiết kế kiến trúc, viết mã, review, tạo PR, tải artifact lên Google Drive, cập nhật Asana, đến khi đóng thẻ **Done** — mà **không cần bạn can thiệp ở giữa**.

> **Lưu ý pháp lý:** Hiện tại hệ thống đang ở bản thiết kế **v0.1**. Bạn cần hoàn tất các Phase trong `workflow-spec.md` (đặc biệt là Phase 1 — Local Orchestrator) trước khi vận hành thử.

Hệ thống sử dụng **Hermes Kanban làm bộ điều khiển trung tâm (control-plane)**, kết hợp với các công cụ:

- **ChatGPT** + **Google Gemini** — phân tích & thiết kế  
- **Claude Code** + **Codex CLI** — viết mã & review  
- **GitHub** + **Bitbucket** — quản lý repo & PR  
- **Google Drive** — lưu trữ artifact  
- **Asana** — theo dõi task  
- **Hermes Agent** — điều phối (orchestrator)

---

## 2. Luồng hoạt động tổng quan

```
Tạo thẻ Kanban
    ↓
[0] Khởi động (Hermes Agent)
    ↓
[1] Phân tích yêu cầu (ChatGPT)
    ↓
[2] Thiết kế kiến trúc (Google Gemini)
    ↓
[3] Viết mã (Claude Code)
    ↓
[4] Review & Test (Codex CLI)
    ↓ (nếu pass)
[5] Tạo PR & CI (GitHub + Bitbucket)
    ↓ (sau khi CI pass)
[6] Đẩy artifact (Google Drive)
    ↓
[7] Cập nhật Asana
    ↓
[8] Hoàn tất (Hermes Agent)
```

Mỗi bước lưu kết quả vào thư mục `runs/<mã_chạy>/` và cập nhật trạng thái dưới dạng comment vào thẻ Kanban (tag `[orch://<run_id>]`). Nếu có lỗi, hệ thống tự thử lại tối đa 3 lần.

---

## 3. Những gì bạn cần chuẩn bị

### 3.1 Tài khoản & API Keys

| Dịch vụ | Cần chuẩn bị | Ghi chú |
|---------|--------------|---------|
| **OpenAI (ChatGPT)** | API Key | Tạo tại [platform.openai.com](https://platform.openai.com) |
| **Google Gemini** | API Key | Tạo tại [ai.google.dev](https://ai.google.dev) |
| **Google Drive** | Service Account JSON (scope: drive file read/write) | Tạo tại [Google Cloud Console](https://console.cloud.google.com) |
| **GitHub** | `gh` CLI đã xác thực **hoặc** PAT | Chạy `gh auth login` |
| **Bitbucket** | App Password hoặc PAT | Có quyền repo read/write, pull-request write |
| **Asana** | Personal Access Token (PAT) | Tạo trong Asana Developer Console |
| **Hermes Agent** | Tích hợp Kanban đã bật | Kiểm tra [Hermes docs](https://hermes-agent.nousresearch.com/docs) |

### 3.2 Công cụ dòng lệnh (CLIs) cần cài đặt

Hệ thống gọi các CLI sau như tiến trình con:

```bash
# macOS (Homebrew)
brew install anthropic/claude/claude-code    # Claude Code
brew install openai/tap/codex               # Codex CLI
brew install gh                             # GitHub CLI
brew install atlassian/tap/bitbucket-cli    # Bitbucket CLI (tùy chọn)
brew install rclone                          # Hoặc gdrive CLI cho Google Drive
```

**Yêu cầu chung:** `Node.js >= 18`, `Python >= 3.11`, `git`.

### 3.3 Quyền hạn hệ thống

- **Hệ thống file:** quyền đọc/ghi vào `/Users/hys/twohearts/daily-read/runs/`.
- **Git:** quyền push lên repo GitHub/Bitbucket của bạn.
- **Mạng:** cho phép outbound HTTPS đến API của OpenAI, Google, GitHub, Bitbucket, Asana.
- **Hermes Agent:** có thể chạy script/plugin dưới profile của bạn.

### 3.4 Nơi lưu secrets (Rất quan trọng)

Đừng commit secrets vào git. Hệ thống sẽ đọc secrets từ:

```
/Users/hys/.config/twohearts/secrets/
```

Hoặc macOS **Keychain** (khuyến nghị). Bạn phải đảm bảo các entry sau tồn tại trước khi chạy:

- `openai.key` — API key ChatGPT  
- `gemini.key` — API key Google Gemini  
- `google-drive-service-account.json` — Service account cho Drive  
- `asana.pat` — Asana Personal Access Token  
- `github.token` — GitHub PAT (hoặc dùng `gh auth login`)  
- `bitbucket.app-password` — Bitbucket app password  
- `kanban-webhook-secret` — (tùy chọn) secret để xác thực webhook

> **Mẹo:** Dùng lệnh `security add-generic-password` trên macOS để lưu vào Keychain, sau đó script sẽ đọc qua `security find-generic-password`.

---

## 4. Cài đặt & Cấu hình

### Bước 1: Hoàn tất các Phase trong `workflow-spec.md`

Trước tiên, bạn cần code orchestrator thực tế:

1. **Phase 1** — Tạo plugin `plugins/hermes-kanban` để nhận sự kiện từ Kanban (webhook/cron).
2. **Phase 2** — Tạo các wrapper `tools/chatgpt.sh`, `tools/gemini.sh` gọi API với structured outputs.
3. **Phase 3** — Tạo `tools/claude-code.sh` và `tools/codex.sh` để spawn các CLI trong worktree riêng.
4. **Phase 4** — Cấu hình auth GitHub + Bitbucket, tạo PR mirror.
5. **Phase 5** — Tạo `tools/drive.sh` và `tools/asana.sh` cho artifact + tracking.
6. **Phase 6** — Thêm dashboard, audit logs, report.

### Bước 2: Cấu hình Hermes Plugin

- Mở Hermes profile (profile mặc định).
- Kích hoạt plugin `hermes-kanban`.
- Cấu hình board: cột **Ready for Orchestrator** → **In Progress** → **Review** → **Done**.
- Cấu hình delivery endpoint về `http://localhost:<port>` hoặc bật polling lokal.

### Bước 3: Khai báo Custom Fields cho thẻ Kanban

Trên mỗi thẻ, các trường sau phải có giá trị:

- `project_key`
- `repo_owner`
- `repo_name`
- `branch_spec` (ví dụ: `feature/orch-<run_id>`)
- `output_target_drive` (ID thư mục Drive)
- `asana_project_gid`
- `priority`
- `require_human_approval` (**đặt là `false` nếu bạn muốn chạy hoàn toàn tự động**)

---

## 5. Cách vận hành hàng ngày

### Mỗi đầu việc:

1. **Tạo thẻ Hermes Kanban** trong cột **Ready for Orchestrator**.
   - Điền tiêu đề, mô tả chi tiết.
   - Set `require_human_approval = false` để đảm bảo không có người can thiệp ở giữa.
2. **Chạy orchestrator** (nếu chưa chạy background):
   ```bash
   # Ví dụ (script thực tế sẽ có ở Phase 1)
   hermes-agent --plugin hermes-kanban --run-orchestrator
   ```
3. **Ngồi chờ.** Hệ thống sẽ:
   - Đọc thẻ mới
   - Chuyển thẻ sang `In Progress`
   - Tự động gọi từng agent theo thứ tự 0 → 8
   - Lưu log chi tiết vào `runs/<mã_chạy>/`
   - Ghi chú kết quả dưới dạng comment vào thẻ Kanban

### Theo dõi tiến độ

- **Thẻ Kanban:** xem comment có tag `[orch://<run_id>]`. Mỗi bước sẽ cập nhật trạng thái thẻ.
- **Thư mục `runs/<run_id>/`:** xem chi tiết từng bước.
- **`logs/`:** log từng agent (requirements, architecture, implementation, review...).
- Nếu có lỗi, thẻ sẽ được cập nhật và hệ thống tự thử lại tối đa 3 lần trước khi chuyển sang trạng thái **Failed** / **Needs Engineer**.

### Kết quả cuối cùng

Bạn sẽ nhận được:

- ✅ **Mã nguồn** đã viết, review, merge vào `main` trên GitHub/Bitbucket.
- ✅ **Artifact** (zip, diagram, báo cáo) được lưu trên Google Drive.
- ✅ **Task Asana** chuyển sang Done, có link PR + Drive + log milestone.
- ✅ **Thẻ Kanban** chuyển sang cột `Done`, kèm link PR, artifact và tóm tắt.
- ✅ Thông báo kết quả trên terminal (không cần mở email hay ứng dụng khác).

---

## 6. Lưu ý bảo mật

- **Không commit secrets** vào git. Thư mục `~/.config/twohearts/secrets/` phải nằm trong `.gitignore`.
- Mỗi dịch vụ dùng **PAT/app password riêng** scoped tối thiểu (least privilege).
- Nếu dùng webhook, phải ký bằng **HMAC shared secret** để chống spoofing.
- File `audit.log` (`runs/<run>/audit.log`) ghi lại mọi hành động để bạn có thể kiểm tra lại sau này.
- Mặc định, artifact trên Drive chỉ chia sẻ với thành viên project.
- Repo branch-protection: yêu cầu CI pass trước khi merge; nếu bật auto-merge, hãy chắc chắn đã bật `require_human_approval = false` những vẫn bật auto-merge ở phía GH/Bitbucket.

---

## 7. Xử lý sự cố

| Lỗi thường | Cách xử lý |
|-----------|-----------|
| Orchestrator không nhận thẻ mới | Kiểm tra Hermes Kanban đã bật cron/webhook chưa; xem log Hermes Agent. |
| Claude Code / Codex CLI fail | Kiểm tra `runs/<run>/logs/step-03-*.log`; hệ thống tự thử lại tối đa 3 lần rồi chuyển thẻ sang **Needs Engineer**. |
| GitHub / Bitbucket báo lỗi push | Kiểm tra PAT/app password còn hiệu lực; đảm bảo remote URL đúng. |
| Drive không upload được | Kiểm tra service account có quyền Writer vào folder đích; artifact sẽ được lưu tạm cục bộ và tự upload lại sau. |
| Asana không cập nhật | Kiểm tra PAT Asana; xem `runs/<run>/tracking/asana-pending.json`. |

Nếu bạn gặp lỗi chưa có trong bảng, hãy kiểm tra `runs/<run>/run-state.json` và `runs/<run>/failures.json` để xác định bước nào bị lỗi, sau đó sửa theo `workflow-spec.md`.
