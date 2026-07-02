# Autonomous AI Orchestration Workflow Specification
## Trigger: Create Hermes Kanban Card

**Status:** Draft / v0.1  
**Constraint:** All orchestration logic is implemented under `/Users/hys/twohearts/daily-read`.  
**Goal:** Create one Hermes Kanban card and automatically coordinate Claude Code, Codex CLI, Hermes Agent, ChatGPT, Google Gemini, Google Drive, Asana, Bitbucket, and GitHub with no human intervention until final delivery.

---

## 1. Trigger and Event Schema (Hermes Kanban)

### 1.1 Trigger
A new Hermes Kanban card is created (or moved into a designated "Ready for Orcherator" column/queue).

### 1.2 Event Payload
When the card is created, Hermes emits an event with the following JSON schema:

```json
{
  "$schema": "https://twohearts.ai/schemas/kanban-trigger-v1",
  "event_type": "kanban.card.created",
  "event_id": "evt_<uuid>",
  "timestamp": "<ISO8601>",
  "source": "hermes.kanban",
  "payload": {
    "card_id": "kban_<uuid>",
    "board_id": "board_<uuid>",
    "column": "Ready for Orcherator",
    "title": "<string>",
    "description": "<markdown>",
    "labels": ["<label>", "...", "feature", "bug", "urgent"],
    "assignee_ids": ["<user_id>"],
    "custom_fields": {
      "project_key": "<string>",
      "repo_owner": "<string>",
      "repo_name": "<string>",
      "branch_spec": "<string>",
      "output_target_drive": "<driveFolderId>",
      "asana_project_gid": "<gid>",
      "priority": "high|medium|low",
      "require_human_approval": true|false
    },
    "attachments": [{"filename": "<string>", "url": "<https://...>"}],
    "source_task_id": "asana_task_<gid>"
  },
  "metadata": {
    "orchestration_run_id": "orch_<uuid>",
    "initiated_by": "user_id|<special_value>",
    "retry_policy": {
      "max_attempts": 3,
      "backoff_ms": 1000
    }
  }
}
```

### 1.3 Delivery Mechanism
- **Immediate:** Hermes posts the event to a local orchestrator endpoint running under `/Users/hys/twohearts/daily-read`.
- **Webhook alternative:** If hosted, a webhook delivers the payload to the orchestrator HTTP entrypoint.
- **Polling fallback:** A cron poller (e.g., every 30 seconds) checks for new cards in "Ready for Orcherator" when push is unavailable.

---

## 2. Step Ownership: Agent / Tool Assignment

| Step | Owner | Tool / Agent | Responsibility |
|------|-------|--------------|----------------|
| 0 | Orchestrator | `Hermes Agent` (lightweight supervisor) | Consume trigger, bootstrap run state, schedule next step |
| 1 | Requirements | `ChatGPT` | Parse Kanban card into structured requirements + acceptance criteria |
| 2 | Architecture | `Google Gemini` | Draft system design, component interactions, data model |
| 3 | Implementation | `Claude Code` | Write code, tests, and commit to feature branch |
| 4 | Review & Tests | `Codex CLI` | Run tests/linters/security reviews and post feedback |
| 5 | Repository | `Bitbucket` + `GitHub` | Mirror code, manage PRs and CI/CD trigger |
| 6 | Artifacts | `Google Drive` | Store spec, diagrams, generated reports, signed artifacts |
| 7 | Task Tracking | `Asana` | Update task status, link commits, log issues |
| 8 | Completion | `Hermes Agent` | Finalize run, mark Kanban card, notify user |

---

## 3. Detailed Workflow Steps, Handoffs, and Data Contracts

### Step 0 — Bootstrap (Hermes Agent)
**Input:** `kanban.card.created` event  
**Actions:**
- Create orchestration run directory:  
  `/Users/hys/twohearts/daily-read/runs/<orch_run_id>/`
- Initialize `run-state.json` with event payload and `current_step=1`.
- Emit local event `step.1.trigger`.

**Output / Handoff:**
- `runs/<orch_run_id>/run-state.json`
- Event: `step.1.trigger`

---

### Step 1 — Requirements Breakdown (ChatGPT)
**Input:** `step.1.trigger` + `runs/<orch_run_id>/run-state.json`

**Actions:**
- Call ChatGPT with `card.title`, `card.description`, `custom_fields.project_key`.
- Receive structured JSON:
  ```json
  {
    "requirements": ["...", "..."],
    "acceptance_criteria": ["Given/When/Then", "..."],
    "non_functional": ["performance", "security", "..."],
    "clarifications": []
  }
  ```
- Write to `runs/<orch_run_id>/requirements.json`.

**Output / Handoff:**
- `runs/<orch_run_id>/requirements.json`
- Event: `step.2.trigger`

---

### Step 2 — Architecture & Design (Google Gemini)
**Input:** `step.2.trigger` + `requirements.json`

**Actions:**
- Send requirements to Google Gemini.
- Receive:
  ```json
  {
    "architecture": {"components": [...], "data_model": "...", "sequence": "..."},
    "api_contracts": [{"path": "...", "method": "...", "request": {...}, "response": {...}}],
    "diagram_ascii": "...",
    "risk_register": [{"risk": "...", "mitigation": "..."}]
  }
  ```
- Save design documents to `runs/<orch_run_id>/design/`.

**Output / Handoff:**
- `runs/<orch_run_id>/design/architecture.json`
- `runs/<orch_run_id>/design/diagram.{md,png}`
- Event: `step.3.trigger`

---

### Step 3 — Implementation (Claude Code)
**Input:** `step.3.trigger` + `design/` files

**Actions:**
- Claude Code opens a coding session scoped to:
  `/Users/hys/twohearts/daily-read/<project_key>-worktree/`
- Implements source code, unit tests, README update.
- Commits to local feature branch: `feature/orch-<orch_run_id>`.
- Runs local pre-commit hooks and `pyright`.
- Pushes branch to upstream mirror (see Step 5).

**Output / Handoff:**
- Git branch: `feature/orch-<orch_run_id>`
- `runs/<orch_run_id>/code/manifest.json` (files changed, test results)

**Data contract (manifest.json):**
```json
{
  "branch": "feature/orch-<orch_run_id>",
  "commit_sha": "...",
  "files_changed": ["src/x.py", "tests/y.py"],
  "test_summary": {"passed": N, "failed": N},
  "pyright_status": "ok|errors",
  "coverage_pct": N
}
```

**Event:** `step.4.trigger`

---

### Step 4 — Review & Quality Gate (Codex CLI)
**Input:** `step.4.trigger` + manifest.json

**Actions:**
- Codex CLI runs:
  - Test suite
  - Lint/type checks
  - Security scan (e.g., `bandit`, `npm audit`)
  - Performance benchmark (if applicable)
- Generates review report.

**Output / Handoff:**
- `runs/<orch_run_id>/reviews/codex-review.json`

**Data contract:**
```json
{
  "overall_status": "pass|warn|fail",
  "tests": {"passed": N, "failed": N},
  "lint": {"errors": 0, "warnings": N},
  "security": {"vulnerabilities": []},
  "recommendations": ["..."],
  "approved": true|false
}
```

**Decision gate:**
- If `approved=true` → Event: `step.5.trigger`
- Else if auto-remediation possible → Event: `step.3.fix_required` + update Asana task.
- Else → Event: `step.5.escalate` with review report attached.

---

### Step 5 — Repository Integration (Bitbucket / GitHub)
**Input:** `step.5.trigger` + manifest.json + review report

**Actions:**
- Open PR on **primary** host (`custom_fields.code_host`) from `feature/orch-<orch_run_id>` to `main`.
- Mirror PR to secondary host if dual-host strategy is configured.
- Trigger CI/CD jobs on both hosts.
- Update branch protection rules (require passing CI + 1 approval).

**Output / Handoff:**
- PR URL(s): `https://github.com/<owner>/<repo>/pull/<number>` and/or Bitbucket equivalent.
- `runs/<orch_run_id>/pr-meta.json`

**Data contract:**
```json
{
  "primary_host": "github|bitbucket",
  "primary_pr_url": "<url>",
  "mirror_pr_url": "<url>",
  "ci_status": "pending|success|failure",
  "merge_commit_sha": "<sha>"
}
```

**Event:** `step.6.trigger` (after CI success)

---

### Step 6 — Artifact Publication (Google Drive)
**Input:** `step.6.trigger` + design + code + review artifacts

**Actions:**
- Hermes Agent orchestrates Google Drive upload.
- Upload artifacts:
  - `/Users/hys/twohearts/daily-read/runs/<orch_run_id>/work-package.zip`
  - `design/*.{md,json,png}`
  - `reviews/*.json`
- Set sharing permissions according to `project_key` policy.
- Update `runs/<orch_run_id>/artifacts/gdrive-links.json`.

**Output / Handoff:**
- `gdrive-links.json` with Drive file IDs and URLs.
- Event: `step.7.trigger`

---

### Step 7 — Task Tracking Update (Asana)
**Input:** `step.7.trigger` + gdrive-links + pr-meta + review report

**Actions:**
- Hermes Agent updates original Asana task (`source_task_id`):
  - Mark task as "In Review" / "Done" based on PR status.
  - Link PR URL(s) in task description and custom fields.
  - Link Google Drive folder.
  - Log milestone comment with timestamps.

**Output / Handoff:**
- Asana task updated.
- `runs/<orch_run_id>/tracking/asana-status.json`

**Event:** `step.8.trigger`

---

### Step 8 — Completion & Notification (Hermes Agent)
**Input:** `step.8.trigger` + all prior artifacts

**Actions:**
- Create a final summary artifact:
  - `runs/<orch_run_id>/summary.md`
- Update Hermes Kanban card:
  - Move to " done " column.
  - Add PR and Drive links as comments.
  - Set completion timestamp.
- Notify user via configured channel (e.g., terminal notification, email stub).
- Run post-flight cleanup (optional archive to tape/object store).

**Output / Handoff:**
- Completed job.
- Human readable notification.

---

## 4. Timeline and State Machine

```
CREATE CARD
   │
   ▼
[0] Bootstrap
   │
   ▼
[1] Requirements (ChatGPT)
   │
   ▼
[2] Architecture (Gemini)
   │
   ▼
[3] Implementation (Claude Code)
   │
   ▼
[4] Review (Codex CLI)
   │   ├─ pass ──▶ [5] PR/CI (Bitbucket + GitHub)
   │   │                │
   │   │                ▼
   │   │         [6] Drive (Upload artifacts)
   │   │                │
   │   │                ▼
   │   │         [7] Asana (Update task)
   │   │                │
   │   │                ▼
   │   │         [8] Complete (Hermes)
   │   │
   │   └─ fail ──▶ [1] Re-evaluate (or escalate)
   │
   ▼
Optional circular loop for fix iterations
```

---

## 5. What Can Be Implemented Today vs Future Work

### 5.1 Implementable Today
- **Hermes Agent as orchestrator:** Use local event queue / filesystem `run-state.json` for coordination.
- **Hermes Kanban integration:** Create a provider plugin for Kanban webhook/polling under `plugins/hermes-kanban`.
- **ChatGPT integration:** Use `hermes-agent` skill wrappers for OpenAI API calls with structured outputs.
- **Google Gemini integration:** Same wrapper API approach.
- **GitHub repo + PR:** Local git operations + GitHub CLI (`gh`) installed.
- **Bitbucket mirror:** Local git + Bitbucket CLI or REST.
- **Google Drive:** Use `gdrive` CLI or Google Drive API with a service account.
- **Asana updates:** Use Asana REST API with fine-grained personal access token.
- **Codex CLI:** Invoke as subprocess in local environment.
- **Claude Code:** Invoke as subprocess scoped to a working tree under `/Users/hys/twohearts/daily-read`.

All of these are achievable with local CLI invocations and API keys stored in Keychain / `~/.config/twohearts/secrets/`.

### 5.2 Requires Future Work
- Fully hosted webhook receiver with OAuth flows for all tools.
- Service mesh / durable queue (e.g., Redis, SQLite WAL, Temporal) for at-least-once delivery.
- Strong typed event schema with protobuf or Zod validators.
- Agent marketplace with pluggable skill definitions.
- Automated provisioning of service accounts and scoped tokens.
- Live dashboard for run visibility.
- Long-running workflow persistence across restarts (Temporal/Cadence).
- Cross-org governance, SAML, and audit trails.

---

## 6. Fallbacks and Human-Approval Gates

### 6.1 Fallbacks
| Failure Mode | Fallback |
|---|---|
| ChatGPT API down | Mark run `degraded` and continue with cached requirements if available. Hermes Agent summarizes Kanban card directly. |
| Gemini API down | Skip architectural artifacts; Hermes Agent emits a placeholder and continues. |
| Claude Code crash | Retry up to 3 times with backoff; if still failing, move card to "Needs Engineer". |
| Codex CLI failure | Treat as `warn` instead of hard block if tests are optional; otherwise escalate. |
| GitHub/Bitbucket unavailable | Queue PR locally; retry exponential backoff; store PR-pending state. |
| Drive unavailable | Save artifacts locally; mark `gdrive_pending=true` to be uploaded by a retry worker. |
| Asana unavailable | Write pending entry to `runs/<run>/tracking/asana-pending.json`; retry later. |
| Orchestrator crash | Run state is persisted to disk; cron poller can resume from last `current_step`. |

### 6.2 Human-Approval Gates (Opt-In Only)
Add `require_human_approval=true` in Kanban `custom_fields` to pause at chosen steps. **Default policy is autonomous execution without human gating.**

| Gate # | Gate Name | Where Configured | Behavior |
|---|---|---|---|
| G1 | Requirements Approve | Kanban custom field | After Step 1, Hermes posts comment "Awaiting approval" and waits for user reaction or manual advance. |
| G2 | Architecture Approve | Kanban custom field | After Step 2, waits for 👍/👎 or explicit "Continue" command. |
| G3 | Code Review Approve | Kanban custom field | After Step 4, if `approved=false` but still within threshold, pause for human sign-off before creating PR. |
| G4 | Merge Approve | Kanban custom field | After CI, require human to approve merge if `priority=high`. |

**Implementation of gates:**
- Polluted or paused runs are tracked in `runs/<run>/run-state.json` with `"paused_at_step": N`.
- Hermes Agent cron checks paused runs every 60 seconds and advances when approval is recorded in Asana or Kanban.

---

## 7. Security & Permissions Model

- **Least privilege:** Each tool uses a dedicated service account / PAT scoped to project `twohearts-project`.
- **Secrets storage:** `/Users/hys/.config/twohearts/secrets/` (NOT in repo).
- **Drive policy:** By default, artifacts are shared only with the project team.
- **Repo policy:** Feature branches are protected; CI must pass before PR auto-merge is attempted (but merge requires human approval by default).
- **Audit:** Every step writes a timestamped log entry to `runs/<run>/audit.log`.

---

## 8. Observability

- **Run ledger:** `runs/<run>/run-state.json`
- **Per-step log:** `runs/<run>/logs/step-01-requirements.log`, etc.
- **Failure metrics:** Count in `runs/<run>/failures.json`.
- **Hermes Kanban trace:** Each comment appended to the card includes `[orch://<run_id>]`.

---

## 9. Example Run Directory Layout

```
/Users/hys/twohearts/daily-read/runs/orch_ab12.../
├─ run-state.json
├─ audit.log
├─ logs/
│  ├─ step-01-requirements.log
│  ├─ step-02-architecture.log
│  ├─ step-03-implementation.log
│  └─ step-04-review.log
├─ requirements.json
├─ design/
│  ├─ architecture.json
│  └─ diagram.md
├─ code/
│  └─ manifest.json
├─ reviews/
│  └─ codex-review.json
├─ pr-meta.json
├─ artifacts/
│  ├─ work-package.zip
│  └─ gdrive-links.json
└─ tracking/
   ├─ asana-status.json
   └─ asana-pending.json
```

---

## 10. Implementation Plan (Ordered)

1. **Phase 1 — Local Orchestrator (Hermes Agent)**
   - Create `plugins/hermes-kanban`.
   - Create `workflows/kanban-trigger` and event handlers.
   - Implement filesystem-backed state machine under `/Users/hys/twohearts/daily-read/runs/`.

2. **Phase 2 — AI Generators**
   - Add `tools/chatgpt.js` and `tools/gemini.js` wrappers.
   - Wire Steps 1 and 2 to structured output contracts.

3. **Phase 3 — Code & Review Loop**
   - Add `tools/claude-code.sh` and `tools/codex.sh` wrappers.
   - Implement retry/fallback logic.
   - Add human-approval gate support (optional).

4. **Phase 4 — Repository & CI**
   - Configure local git + GitHub CLI + Bitbucket CLI.
   - Create PR mirrors and CI handlers.

5. **Phase 5 — Artifacts & Tracking**
   - Add `tools/drive.sh` and `tools/asana.sh`.
   - Validate links and update Kanban.

6. **Phase 6 — Observability & Polish**
   - Run dashboard, audit logs, metrics.
   - Fallback recovery and resumability.

---

## 11. References

- Hermes Agent docs: https://hermes-agent.nousresearch.com/docs
- Hermes Kanban vs Hermes Agent scopes: See plugin authoring guide.
- OpenAI structured outputs: https://platform.openai.com/docs/guides/structured-outputs
- Google Gemini API: https://ai.google.dev/gemini-api/docs
- Anthropic Claude Code SDK: https://docs.anthropic.com/claude/docs/claude-code-sdk
- Codex CLI: https://openai.com/index/introducing-codex/
- Asana API: https://developers.asana.com/docs
- GitHub CLI: https://cli.github.com/manual/
- Bitbucket CLI: https://support.atlassian.com/bitbucket-cloud/docs/using-the-bitbucket-cli/
- Google Drive API: https://developers.google.com/drive/api/guides/about-sdk

---

*End of specification — v0.1*
