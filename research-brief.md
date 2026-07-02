# Research Brief: Autonomous Multi-Tool AI Orchestration & Workflow Automation

**Date:** 2026-07-02  
**Focus:** Eliminating manual handoff between Claude Code, Codex CLI, Hermes Agent, ChatGPT, Google Gemini, Google Drive, Asana, Bitbucket, and GitHub using Hermes Kanban as a unified trigger.

---

## 1. Key Findings

### Control-Plane Pattern is the Dominant Architecture
The most production-proven pattern for autonomous multi-tool orchestration is **"issue-tracker-as-control-plane"** — using a task board (Linear, GitHub Issues, Asana, or Hermes Kanban) to drive agent execution, similar to how humans organize engineering work. OpenAI's internal **Symphony** (open-sourced April 2026) demonstrated this by turning Linear tickets into persistent agent workspaces, achieving a **500% increase in landed PRs** in 3 weeks by eliminating human session micromanagement. Hermes Kanban can adopt the same pattern: every active card becomes a guaranteed running agent.

### Coding Agents Are Commoditized; Coordination Is the New Bottleneck
Tools like Claude Code, Codex CLI, OpenHands, and Devin are mature enough that the limiting factor is no longer agent capability but **orchestration overhead**. Addy Osmani's March 2026 CodCon talk identified three solo-agent ceilings (context overload, no specialization, no coordination) that multi-agent patterns solve. OpenHands (44K+ GitHub stars) provides a composable SDK with MCP support and agent delegation primitives. The `awesome-agent-orchestrators` list documents 40+ tool-specific orchestrators.

### Standardization Is Emerging via MCP
The **Model Context Protocol (MCP)** has become the de-facto middleware standard for integrating AI agents with external tools. With 97M+ monthly downloads in 2026, MCP provides a universal SDK layer for connecting Claude, Codex, and custom agents to file systems, APIs, and databases. Hermes Agent already surfaces MCP integrations. This is the closest existing mechanism to a neutral bus spanning the user's tool stack.

### Middleware Exists but Is Fragmented
Zapier and Make natively connect ChatGPT, Gemini, and Google Drive, but they **do not orchestrate local coding agents** (Claude Code / Codex CLI). n8n, LangGraph, and CrewAI fill coding-agent orchestration but lack native enterprise PM integrations (Asana/Bitbucket/GitHub upstream triggers). AWS's new **CLI Agent Orchestrator** (open-source) specifically targets Claude Code → Codex CLI → Gemini CLI parallelism. The gap is true cross-domain orchestration spanning **project management + local coding CLIs + cloud AI APIs**.

---

## 2. Existing Tools / Patterns

### A. Kanban-Driven Autonomous Execution
| Tool/Pattern | Value |
|---|---|
| **OpenAI Symphony** | Linear ticket → isolated workspace → coding agent. Subtask DAGs, auto-CI, spec-driven. Ref impl in Elixir/TS/Python/Go/Rust/Java. |
| **Hermes Multi-Profile Kanban** | Built-in durable task board across profiles. Supports cron, webhooks, multi-agent handoffs. Native task pick-up/update lifecycle. |
| **Notion + AI** | Triggers via databases/cron, but limited native coding-agent runtime. |

### B. Multi-Agent Coding Orchestration
| Tool | Value |
|---|---|
| **Bernstein** | Deterministic orchestrator spawning Claude Code, Codex CLI, Gemini CLI in parallel with git worktrees. Zero LLM tokens on coordination. |
| **Claude Command Center (CCC)** | Local dashboard for parallel Claude Code / Codex / Cursor sessions. Spawn, monitor, resume. |
| **OpenHands Agent SDK** | Cloud coding agent platform. REST API for multi-agent, event-sourced state, MCP tool support. |
| **Agent Tier / AgentBox** | Kubernetes / Docker sandboxed parallel agents with webhook invoke APIs. |
| **AGX / Amux / CMux** | Terminal UIs and open-source platforms for running parallel coding agents. |

### C. AI Workflow Middleware
| Tool | Value |
|---|---|
| **n8n / LangGraph / CrewAI** | Workflow API or state graph execution. Good for API-heavy flows but weaker in local CLI process spawning. |
| **Zapier / Make** | 9,000+ app integrations. Native Asana, GitHub, Google Drive, ChatGPT, Gemini. **Critical gap:** no local coding-agent runtime. |
| **MCP** | Cross-platform tool-binding standard. Can normalize Asana/GitHub/GDrive into MCP resources, then route to coding agents. |

### D. Handoff & State Patterns
| Pattern | Source | Value |
|---|---|---|
| Orchestrator / Conductor | Addy Osmani, Azure | Central planner assigns to specialist agents. Handles dependency DAGs. |
| Peer Handoff | Augmentcode, Rasa | Agent delegates entire task+context to another agent (useful for model-switching). |
| Agent-as-Tool | Medium / Yu | Wraps another agent inside an MCP tool endpoint. |
| Subagent Layers | Addy Osmani | Parent → feature leads → specialists. Mimics org charts. |

---

## 3. Gaps vs. The User's Desired Outcome

### Desired Architecture
> Hermes Kanban card → triggers → Claude Code, Codex CLI, Hermes Agent, ChatGPT, Google Gemini, Google Drive, Asana, Bitbucket, GitHub → autonomous execution → result sync back to card → no human in the middle.

### Critical Gaps in Current Landscape
1. **No unified Kanban-to-CLI bus for multi-vendor local agents**  
   Hermes Kanban can trigger webhooks/cron, but there is no off-the-shelf runtime that spawns Claude Code + Codex CLI side-by-side, scopes them into git worktrees, feeds them a spec, and reconciles state back into Asana/Bitbucket/GitHub automatically.

2. **Cross-stakeholder tool separation**  
   Asana, Bitbucket, and GitHub have their own issue trackers and automation layers (native webhooks / GitHub Actions), but they do **not** natively call local CLIs or Hermes Agent. Reversing this (local agent → push state back upstream) requires custom glue.

3. **Google Drive + Gemini + ChatGPT routing**  
   These are cloud APIs with auth, rate limits, and file contexts. Middleware like Make/Zapier can chain them, but integrating their outputs into coding-agent workspaces requires custom MCP adapters or a local proxy. No one orchestrator connects Drive/Gemini/ChatGPT to Claude Code/Codex in a single workflow today.

4. **Hermes as central orchestrator vs. Hermes as agent**  
   Hermes Agent has Kanban and multi-profile support, but invoking it as the **central conductor** that dispatches subprocesses for Claude Code and Codex CLI requires building a thin local orchestrator service around its API or CLI.

5. **State recovery and determinism**  
   If an agent crashes mid-task, no existing turnkey product safely rolls back Bitbucket branches, reschedules Asana cards, and re-spawns with checkpoint context without bespoke code.

---

## 4. Actionable Recommendations

### Immediate (Week 1-2)
1. **Adopt the "Kanban as Control Plane" pattern**  
   Designate Hermes Kanban cards as the single source of truth. Enforce that every piece of execution state (spawned agents, git branches, PRs, AI-generated docs) references the card ID. This aligns with Symphony's proven model.

2. **Build a local orchestrator service under `/Users/hys/twohearts/daily-read`**  
   Create a lightweight daemon/script running on a devbox 24/7 that:
   - Polls Hermes Kanban (or accepts webhooks)
   - Spawns Claude Code / Codex CLI into isolated git worktrees per card
   - Uses Bernstein-style deterministic coordination to avoid orchestration LLM costs
   - Writes structured output back to the card comment feed

3. **Normalize tool access via MCP**  
   Deploy MCP servers for Asana, GitHub, Bitbucket, and Google Drive. This lets Hermes Agent (or the local orchestrator) call them uniformly without writing separate API clients for each.

### Near-Term (Month 1-2)
4. **Model-aware handoff routing**  
   Route tasks by capability:
   - Hard coding tasks → Claude Code / Codex CLI (scoped worktrees)
   - Research, writing, document generation → ChatGPT / Gemini
   - Drive storage / asset management → Google Drive MCP
   Use the "Agent-as-Tool" handoff pattern: wrap each external agent as an MCP endpoint the orchestrator can call.

5. **Git worktree isolation per card**  
   Avoid merge conflicts and enable parallel execution by creating a unique worktree per active Kanban card. Commit/push boundaries are explicit and revertible.

6. **Cron + Webhook hybrid trigger**  
   Hermes Kanban already supports cron; complement with native Asana webhooks and GitHub webhook receivers that post back into the local orchestrator. This covers both push and pull models.

### Medium-Term (Month 2-4)
7. **Evaluating ACM (Agent Conductor Middleware)**  
   Consider whether the existing tooling satisfies the "single bus" requirement. Options:
   - **n8n + local agent runner node**: Fast to prototype, but weak on git worktree lifecycle.
   - **LangGraph state machine**: Best for complex DAGs, but steep learning curve and less native integration with Hermes Kanban.
   - **Custom Rust/Elixir orchestrator (Symphony approach)**: Most control, highest long-term ROI. Symphony's SPEC.md is a ready-made blueprint; replace Linear with Hermes Kanban + Asana APIs.

8. **Observability & Guardrails**  
   Add:
   - Confidence scores on agent completions (Augmentcode handoff patterns)
   - Time-to-complete SLAs per card type
   - Budget caps per agent (Hermes mission-control-style kill switches)
   - Flaky-test retry and CI reconciliation (Symphony pattern for monorepos)

9. **Eliminate duplicate state stores**  
   Currently, context lives in Hermes Kanban cards, Asana tasks, GitHub PRs, and agent stdout. Standardize on the Hermes card as the event log, with external systems treated as side-effects, not sources of truth.

---

## 5. Top Relevant Sources
- OpenAI Symphony spec: https://openai.com/index/open-source-codex-orchestration-symphony/
- Addy Osmani - Code Agent Orchestra: https://addyosmani.com/blog/code-agent-orchestra/
- AWS CLI Agent Orchestrator: https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/
- awesome-agent-orchestrators: https://github.com/andyrewlee/awesome-agent-orchestrators
- OpenHands Agent SDK: https://arxiv.org/html/2511.03690v1
- Hermes Kanban docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
- Azure AI Agent Orchestration Patterns: https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
- MCP: https://modelcontextprotocol.io/docs/getting-started/intro
- Agent Handoff Patterns: https://www.augmentcode.com/guides/agent-handoff-patterns-human-agent-interface
