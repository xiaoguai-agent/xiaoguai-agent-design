# PRD — Xiaoguai (小怪) Agent Platform

| | |
|---|---|
| Document ID | `PRD-XIAOGUAI-001` |
| Type | PRD (retrofit) |
| Status | `draft` (retrofit baseline, awaiting stakeholder review) |
| Version | v1.4.0-retrofit |
| Author | Zhou Wei (zhouwei008@gmail.com) |
| Reviewers | TBD |
| Created | 2026-05-28 |
| Source documents | `docs/2026-05-21-xiaoguai-agent-design.md` (brainstorm), `HANDOFF-FOR-DESIGN-DOCS.md`, implementation repo `/Users/zw/testany/myskills/xiaoguai/` |

> **Retrofit notice.** The product has been implemented for ~4 months ahead of this PRD. This document captures **what is** at v1.4.0 plus the **near-term roadmap (Tier-1/2/3)**, not a green-field requirement set. Where requirements differ from the 2026-05-21 brainstorm, both states are recorded so design drift is traceable.

---

## 1. Product summary

**Xiaoguai (小怪)** — *Your Little Agent for Big Work / 小怪不小，能办大事.*

A lightweight, MCP-first, **local-LLM-default** AI agent platform optimised for **single-tenant private / air-gapped enterprise deployment**. Native gateways for the three Chinese enterprise IMs (Feishu / DingTalk / WeCom) and four global ones (Slack / Discord / Telegram / Mattermost). Compliance posture rests on the BUSL-1.1 license, an HMAC-chained append-only audit log, and a Human-on-the-Loop (HotL) gate that caps cost and tool blast-radius per tenant.

The thesis is not "be best at any single dimension" but **"be the only product that is lightweight + MCP-first + local-LLM-default + Chinese-IM-native + compliance-aware all at once"**, with private deployment as a first-class delivery mode (not a paid edition).

---

## 2. Goals, non-goals, anti-goals

### 2.1 Goals (v1.4.x current scope)

| ID | Goal | Status |
|---|---|---|
| G1 | Single self-contained binary `xiaoguai` (CLI + server) deployable air-gapped without Valkey/Redis | ✅ shipped (PR #59 + #60) |
| G2 | First-class MCP support (client + supervisor + local sandbox `xiaoguai-mcp-exec`) | ✅ shipped (PR #64) |
| G3 | Local LLM default (Ollama / OpenAI-compat) + 6 cloud providers as opt-in fallbacks | ✅ shipped |
| G4 | Seven IM gateways: Feishu, DingTalk, WeCom, Slack, Discord, Telegram, Mattermost | ✅ shipped |
| G5 | OIDC SSO + Casbin RBAC + Postgres RLS multi-tenancy | ✅ shipped |
| G6 | HMAC-chained append-only audit log with verifiable replay + PII redaction | ✅ shipped (PR #57) |
| G7 | HotL governance: per-tenant spend/count caps with allow-then-escalate semantics (ADR-0015) | ✅ shipped |
| G8 | Pgvector-backed long-term memory with semantic recall | ✅ shipped |
| G9 | Skill-pack marketplace (declarative config, deferred hot-reload, signed packs) | ✅ shipped (ADR-0017) |
| G10 | Kanban-style task board for cross-session async work (`xiaoguai-tasks`) | ✅ shipped |
| G11 | Multi-agent orchestration via `xiaoguai-orchestrator` (was N5 non-goal in brainstorm) | ✅ shipped, design TBD |
| G12 | Persona presets via `xiaoguai-personas` (role templates + persona-scoped memory) | ✅ shipped, design TBD |
| G13 | Outcome telemetry (ROI attribution) — `xiaoguai-outcomes` daily bucketing | ✅ shipped (ADR-0016) |
| G14 | Scheduler with cron / interval / file-watch / webhook / proactive triggers | ✅ shipped |
| G15 | Anomaly detection (Z-score / EWMA) over outcome and usage time-series | ✅ shipped |
| G16 | Multi-arch binaries (linux/amd64, linux/arm64) — 信创 (Xinchuang) ready | ✅ shipped via CI |
| G17 | Eval harness (`xiaoguai-eval`) executing `.eval.yaml` suites for capability + regression coverage | ✅ runtime shipped; suites empty |

### 2.2 Goals (Tier-1/2/3 roadmap)

| Tier | Goal | Status |
|---|---|---|
| Tier-1a | Unified binary (subsume `xiaoguai-core` into `xiaoguai serve`) | ✅ PR #59 |
| Tier-1b | In-process cache fallback (`Cache::connect("")` → DashMap) for air-gap mode | ✅ PR #60 |
| Tier-1c | PII detection + redaction in audit chain and OTel spans | ✅ PR #57 |
| Tier-2a | Programmatic tool calling: `execute_python` MCP server with sandbox | ✅ PR #64 |
| Tier-2b | HotL gate wired into agent ReAct loop's parallel tool-call dispatch | ✅ PR #61 |
| Tier-2c | Agent-authored skills gated by HotL approval | 🔲 not started |
| Tier-2d | Session compaction for long local-model sessions | 🔲 not started |
| Tier-3a | OAuth 2.1 PKCE for authenticated outbound MCP servers | 🔲 not started |
| Tier-3b | Compliance report export (SOC2 / GDPR / HIPAA) generated from the audit chain | 🔲 not started |

### 2.3 Non-goals (v1.4.x)

| ID | Out of scope | Why |
|---|---|---|
| N1 | High availability with auto-failover | Single-instance air-gap is the primary deployment shape; HA reference architecture exists (`docker-compose.ha.yml`, Helm `values-ha.yaml`) but is opt-in. |
| N2 | Multi-region replication | Defer; out of scope for v1.x. |
| N3 | Workflow editor / drag-drop pipeline | Anti-thesis — we are MCP-first, not Dify-clone. |
| N4 | Mobile native clients | Web responsive only; mobile in v2.x at earliest. |
| N5 | Public skill-pack marketplace with anonymous publish | Closed marketplace only; trust model is local-first + signed packs from a curated index. |
| N6 | HIPAA / PCI compliance templates | 等保 2.0 三级 + GDPR + SOC2 only in v1.x. |
| N7 | Auto-scaling executors | Out of scope; tenants size their own pods. |

### 2.4 Anti-goals (never)

| ID | Will not do |
|---|---|
| A1 | Force users onto a hosted SaaS — self-hosting is permanently first-class. |
| A2 | Accept AGPL / SSPL / BSL-incompatible / commercial dependencies in core (license firewall). |
| A3 | Tie core agent functionality to any single LLM provider. |
| A4 | Build a Dify clone — MCP-first and ops-first, not workflow-editor-first. |
| A5 | Embed-then-extract Anthropic / OpenAI API keys server-side; tenants supply their own. |

---

## 3. Target users & deployment model

### 3.1 Primary user profile

| | |
|---|---|
| Organisation size | SMB to mid-market enterprise (50–5 000 employees) |
| Industry tilt | Finance, healthcare, gov / SOE, manufacturing — sectors with internal-data sensitivity |
| IT posture | Internal IT or DevOps team that runs on-prem / private cloud workloads (k8s, systemd, or bare metal) |
| Geography | Greater China primary, EU GDPR jurisdictions secondary |
| Buying motivation | Data must not leave the perimeter; OpenAI/Anthropic APIs blocked or restricted |

### 3.2 Deployment topology — single-tenant air-gap first

**Decision (D-DEPLOY-001):** Default deployment is **single-tenant air-gap**, i.e. one logical tenant per `xiaoguai` install, deployable without outbound internet and without Valkey/Redis. Multi-tenant SaaS shape is supported by the same codebase (RLS-enabled tables, tenant-scoped HotL policies, per-tenant LLM provider rows) but is **opt-in**, not the marketing default.

This decision drives:
- `Cache::connect("")` boots an in-process DashMap (PR #60), so single-tenant deploys don't need a broker.
- `OLLAMA_HOST` defaults to local Ollama; falls back to a deterministic in-process embedder for smoke tests.
- All cloud LLM provider rows are seeded `enabled=false`; only `ollama-local` is enabled at install time (migration 0020).
- The Helm chart's default `values.yaml` does not require S3, Vault, or external secret stores.

### 3.3 Roles

| Role | Capabilities |
|---|---|
| **Operator** | Install, upgrade, configure, monitor, key-rotate, audit-verify. Talks to the CLI + Admin UI. |
| **Tenant admin** | Manage users, MCP servers, skill packs, HotL policies, LLM providers within their tenant. |
| **End user** | Chat with the agent via Chat UI or any registered IM platform; create sessions, fork sessions, view their own memories. |
| **Agent (system)** | Runs the ReAct loop; calls tools through the HotL gate; emits audit + outcome rows. |

---

## 4. Functional requirements

> Each `REQ-*` has acceptance criteria expressed in operator-visible verification commands or UI behaviour. See `docs/test-spec.md` for the test cases that verify each.

### 4.1 Conversation & agent loop

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-CHAT-001 | End user can hold a streaming chat with a single agent over HTTP+SSE | P0 | `POST /v1/sessions/{id}/messages` returns SSE stream with `delta` / `tool_call` / `final` events; `GET /v1/sessions/{id}/messages` lists prior turns. |
| REQ-CHAT-002 | User can cancel an in-flight agent run | P0 | `POST /v1/sessions/{id}/cancel` interrupts the loop at the next tool-call boundary; bounded by slowest in-flight tool. |
| REQ-CHAT-003 | User can fork a session from any prior message | P1 | `POST /v1/sessions/{id}/fork` creates a new session with `parent_session_id` set; replay tested. |
| REQ-CHAT-004 | Agent ReAct loop dispatches parallel tool calls per turn | P0 | LLM-emitted multi-tool-call response → `tokio::join_all` executes them; per-call HotL gate checked before spawn. |
| REQ-CHAT-005 | Max-iteration cap is configurable per tenant | P1 | Default 25; override via tenant config; loop terminates with explicit reason when hit. |

### 4.2 LLM routing

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-LLM-001 | Operator can register multiple LLM providers per tenant | P0 | `POST /v1/admin/llm/providers` accepts `{provider_kind, endpoint, model_aliases, fallback_order, cost_per_1k_input, cost_per_1k_output}`. |
| REQ-LLM-002 | Eight provider kinds supported | P0 | `ollama`, `openai_compat`, `anthropic`, `gemini`, `bedrock`, `azure_openai`, `mistral`, `groq` — `xiaoguai provider register --kind <k>` rejects unknown kinds. |
| REQ-LLM-003 | Provider fallback chain | P0 | If primary provider returns 5xx or times out, router cascades through `fallback_order`; each attempt produces a `token_usage` row. |
| REQ-LLM-004 | Ollama is the air-gap default | P0 | Migration 0020 seeds `ollama-local` enabled; all cloud providers seeded `enabled=false`. |
| REQ-LLM-005 | Token usage and cost are recorded per request | P0 | `token_usage` table populated on every successful stream with `(tenant_id, provider_id, model, input_tokens, output_tokens, cost_usd, user_id, session_id)`. |

### 4.3 MCP — Model Context Protocol

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-MCP-001 | Per-tenant MCP server registry | P0 | `mcp_servers` table holds `(tenant_id, transport, command, args, env, enabled)`; `enabled=false` excludes from agent toolbox. |
| REQ-MCP-002 | Three transports: stdio, http, sse | P0 | `xiaoguai mcp register --transport {stdio,http,sse}` works for all three. |
| REQ-MCP-003 | Marketplace catalog browsable | P1 | `GET /v1/mcp/marketplace` returns catalog; `POST /v1/mcp/marketplace/install` installs a curated server. |
| REQ-MCP-004 | Built-in sandboxed Python executor | P0 | `xiaoguai-mcp-exec` MCP server exposes `execute_python` tool with sandbox limits (timeout, memory, env scrub, stderr PII redaction, 64 KB output cap). |
| REQ-MCP-005 | Xiaoguai can serve as MCP server itself | P2 | `POST /v1/mcp/serve` (when enabled) mounts the Toolbox as an outbound MCP server. |

### 4.4 Memory & RAG

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-MEM-001 | Long-term memory with semantic recall | P0 | `POST /v1/memories` writes; `POST /v1/memories/recall` returns top-k by cosine distance over pgvector HNSW index. |
| REQ-MEM-002 | Backend selectable at boot | P0 | `OLLAMA_HOST` set → `OllamaEmbedder` (384-dim semantic); unset → `InMemoryEmbedder` (deterministic hash, smoke only). |
| REQ-MEM-003 | Memory is tenant-scoped via RLS | P0 | Operator JWT for tenant A cannot read tenant B's rows even with crafted query. |
| REQ-MEM-004 | TTL field on memories | P1 | Memories have optional `expires_at`; rows past TTL are excluded from recall. **Automated purge job: deferred.** |
| REQ-MEM-005 | RAG document ingestion via MCP tools | P1 | `xiaoguai-rag` exposes loaders and chunkers; RAG itself is consumed via MCP tools, not a separate endpoint. |

### 4.5 IM gateways

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-IM-001 | Seven IM platforms supported | P0 | One adapter crate each: feishu, dingtalk, wecom, slack, discord, telegram, mattermost. |
| REQ-IM-002 | Identity mapping table | P0 | `im_identities` maps `(platform, platform_user_id)` to internal `user_id`; bind/unbind via CLI. |
| REQ-IM-003 | Encryption on inbound webhooks | P0 | WeCom XML + AES, Discord Ed25519, Slack HMAC-SHA256, Feishu HMAC verified at gateway. |
| REQ-IM-004 | Trailing message replay | P1 | Up to `XIAOGUAI_IM__MAX_MESSAGES_PER_CONVERSATION` (default 50) prior messages replayed to the agent on each turn. |
| REQ-IM-005 | IM history backend selectable | P2 | Postgres default; in-process opt-in via `XIAOGUAI_IM__USE_IN_PROCESS_HISTORY=true` (dev only). |

### 4.6 Auth, multi-tenancy, RBAC

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-AUTH-001 | OIDC SSO with bearer-JWT API auth | P0 | All `/v1/**` except `GET /healthz` require `Authorization: Bearer <jwt>`; missing/invalid → 401. |
| REQ-AUTH-002 | Casbin RBAC enforcement | P0 | Role/action/resource policies checked per request; admin-only routes verified via integration tests. |
| REQ-AUTH-003 | Postgres RLS double-layer | P0 | App-layer `WHERE tenant_id = ?` + Postgres RLS policy `USING (tenant_id = current_setting(...))`. Defence in depth. |
| REQ-AUTH-004 | Tenant CRUD via admin API | P0 | `GET /v1/admin/tenants` list; further CRUD via CLI. |

### 4.7 Audit & compliance

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-AUD-001 | Append-only audit log with HMAC chain | P0 | Every state-changing call writes one row; `hmac = HMAC(key, prev_hmac ‖ ts ‖ tenant ‖ actor ‖ action ‖ resource ‖ details)`. |
| REQ-AUD-002 | Operator can verify chain integrity | P0 | `GET /v1/admin/audit/verify?tenant_id=...` returns `{valid: bool, head_sequence, broken_at_sequence?}`. |
| REQ-AUD-003 | PII redaction before signing | P0 | When `XIAOGUAI_AUDIT_REDACT_PII=true` (default), emails / IPv4 / `Bearer <token>` / AWS access keys are scrubbed before HMAC is computed. Recursive over JSON values, preserves keys. |
| REQ-AUD-004 | OTel span PII redaction | P0 | `RedactingSpanExporter` decorator scrubs the same patterns from exported spans. |
| REQ-AUD-005 | HMAC key rotation with 30-day verification window | P1 | New key supplied via secret; verifier accepts entries signed by either key during window. |
| REQ-AUD-006 | Compliance report export | P2 (Tier-3) | Generate SOC2 / GDPR / HIPAA-shaped reports from the audit chain. **Not yet implemented.** |

### 4.8 HotL governance

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-HOTL-001 | Per-tenant spend and count budgets | P0 | `hotl_policies` row: `(scope, window_secs, max_count, max_spend_usd, escalate_to)`; checked before tool spawn. |
| REQ-HOTL-002 | Allow-then-escalate semantics (ADR-0015) | P0 | Budget exhaustion does not block; it routes the request to `escalate_to` (operator channel) and records `scheduler.proactive_denied` (or equivalent) in audit. |
| REQ-HOTL-003 | Fail-closed when policy store unreachable | P0 | If `hotl_policies` cannot be read, tool calls are denied with `policy_unavailable` reason (no synthetic allow). |
| REQ-HOTL-004 | HotL CRUD via API | P0 | `GET/POST /v1/hotl/policies`, `DELETE /v1/hotl/policies/{id}`. |

### 4.9 Skill packs

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-SKILL-001 | Tenant can browse curated catalog | P0 | `GET /v1/skills/catalog` returns the list of packs the operator has whitelisted. |
| REQ-SKILL-002 | Tenant can install a pack | P0 | `POST /v1/skills/install` records the install in `skill_packs`; hot-reload deferred to next session boot (ADR-0017). |
| REQ-SKILL-003 | Pack source is local-first and signed | P0 | Packs install from local filesystem or a tenant-private index; pack metadata includes a signature; install rejects unsigned packs unless `--allow-unsigned` (operator opt-in). |
| REQ-SKILL-004 | Knob overrides per install | P1 | `pack_operators` table holds per-install knob overrides without forking the pack. |
| REQ-SKILL-005 | Public marketplace publish flow | OUT OF SCOPE | Anonymous public publish is N5; intentionally not supported in v1.x. |

### 4.10 Scheduler

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-SCHED-001 | Six trigger types | P0 | `cron`, `interval`, `proactive`, `file_watch`, `webhook`, plus reserved `git_push` / `db_poll` (data variants, source TBD). |
| REQ-SCHED-002 | Four push sinks | P0 | `feishu`, `telegram`, `email` (relay webhook), `inbox` (in-memory FIFO). |
| REQ-SCHED-003 | Public webhook auth | P0 | `POST /v1/scheduler/webhooks/{route_id}` requires `X-Xiaoguai-Token` from `scheduler_webhook_tokens`. |
| REQ-SCHED-004 | Proactive trigger budgets | P0 | Daily budget enforced per `(user_id, day_utc)`; exhaustion → audit row `scheduler.proactive_denied`. |
| REQ-SCHED-005 | File-watch debounce | P1 | `FILE_WATCH_DEBOUNCE_MS` (default 250) coalesces filesystem bursts. |

### 4.11 Outcome telemetry (ROI)

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-OUT-001 | Outcome attribution endpoint | P1 | `POST /v1/outcomes` accepts `{revenue_usd, cost_saved_usd, hours_saved, deals_closed, session_id, …}`. |
| REQ-OUT-002 | Daily-bucketed aggregations (ADR-0016) | P1 | `GET /v1/outcomes/summary` and `/timeseries` return tenant-scoped aggregates only. |
| REQ-OUT-003 | Anomaly detection over outcome series | P2 | `xiaoguai anomaly` CLI emits Z-score / EWMA flags; surfaced in Today view. |

### 4.12 Workspaces & tasks

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-WS-001 | Workspaces above sessions | P1 | `GET/POST /v1/workspaces`, `PUT/DELETE /v1/workspaces/{id}` (archive). |
| REQ-TASK-001 | Kanban board with five states | P1 | `boards` + `tasks` (triage→todo→doing→review→done); `task_state_log` is the audit trail. |
| REQ-TASK-002 | Auto-dispatcher | P2 | New tasks in `triage` can be auto-routed to an agent persona. |

### 4.13 Evaluation harness

| ID | Title | Priority | Acceptance criteria |
|---|---|---|---|
| REQ-EVAL-001 | Catalog and run eval suites | P1 | `GET /v1/admin/eval/suites`; `POST /v1/admin/eval/run` executes a suite and records results. |
| REQ-EVAL-002 | Export a session as a regression eval case | P1 | `POST /v1/admin/eval/case-from-session` materialises a `.eval.yaml` case from a real session transcript. |
| REQ-EVAL-003 | Eval philosophy: capability + regression | P2 | Suites split into `capability/` (probe new behaviour, may fail) and `regression/` (must pass 100 %). **Suite content empty as of v1.4.0; runtime ready.** |

---

### 4.14 Web UI surfaces (chat-ui + admin-ui)

The product ships two SPAs under `frontend/` (DEC-025): an end-user **chat-ui** and an operator **admin-ui**. They share `@xiaoguai/shared::XiaoguaiClient` and a Playwright e2e harness.

| ID | Functional requirement | Surface |
|---|---|---|
| REQ-UI-001 | Chat-ui renders the agent loop as a streaming conversation: composer, message bubbles, tool-call chips, inline citations, copy / regenerate / fork on each bubble | `@xiaoguai/chat-ui` |
| REQ-UI-002 | Admin-ui covers every `/v1/admin/*` and operator-only endpoint with a dedicated pane; pane list lives in [`lld-admin-ui.md`](lld/lld-admin-ui.md) §2 | `@xiaoguai/admin-ui` |
| REQ-UI-003 | Audit pane renders the HMAC chain with verified / rotation-amber / broken-red badges per row; supports per-framework compliance export (SOC2 / GDPR / HIPAA) | `@xiaoguai/admin-ui` |
| REQ-UI-004 | Chat-ui surfaces three safety signals as first-class UI: HotL pending verdict (inline panel), watch-loop status (indicator pill), AI-content disclosure (EU AI Act Art. 50(1) banner, non-dismissible by default) | `@xiaoguai/chat-ui` |
| REQ-UI-005 | Both SPAs ship `zh-Hans` + `en` i18n bundles; lint forbids untranslated JSX strings; Axe-core a11y violations fail e2e | both SPAs |
| REQ-UI-006 | Neither SPA hosts a login page; authentication is delegated to the reverse proxy. On 401 → redirect to `VITE_LOGIN_URL` | both SPAs |
| REQ-UI-007 | Admin-ui Personas pane provides CRUD over `/v1/personas`; pane tagged `role/{planner,worker,critic}` chips per DEC-021 (sprint-9 triangle) | `@xiaoguai/admin-ui` (sprint-10b) |
| REQ-UI-008 | Admin-ui Skill Proposals pane provides approve / reject over `/v1/skills/proposals/*` (DEC-014, T3); wrapped in `<RequireScope name="skill.approve">` | `@xiaoguai/admin-ui` (sprint-10b) |

UI non-functional requirements live in §5 below (added: REQ-NFR-011 first-token observable in UI; REQ-NFR-012 Playwright golden paths green on Chrome/Firefox/Safari; REQ-NFR-013 frontend bundle size budget).

## 5. Non-functional requirements

| ID | Class | Statement | Priority | Acceptance criteria |
|---|---|---|---|---|
| REQ-NFR-001 | Performance | Streaming first-token latency P95 ≤ 1.5 s on Ollama local with `qwen2.5-coder:7b` | P1 | Measured by `perf-regression.yml` benchmark fixture |
| REQ-NFR-002 | Performance | Memory recall P95 latency ≤ 300 ms at 100 k rows | P1 | HNSW index build verified; benchmark in `xiaoguai-memory` |
| REQ-NFR-003 | Security | Default-deny network policy at MCP supervisor child processes | P0 | k8s NetworkPolicy template ships denying egress by default |
| REQ-NFR-004 | Security | All inbound IM webhooks verify provider signatures before deserialisation | P0 | Negative test rejects forged Feishu signature with 401 |
| REQ-NFR-005 | Availability | Single-instance restart is safe; in-flight sessions resumable on WebSocket reconnect | P1 | Smoke test interrupts, reconnects, asserts continuation |
| REQ-NFR-006 | Portability | Multi-arch binary build: linux/amd64 + linux/arm64 (信创-ready) | P0 | `release.yml` produces both arch tarballs and Docker tags |
| REQ-NFR-007 | Observability | Prometheus `/metrics` + OTLP span export, both subject to PII redaction | P0 | `RedactingSpanExporter` test rejects unscrubbed exports |
| REQ-NFR-008 | Privacy | Zero telemetry to upstream Anthropic / OpenAI / Xiaoguai authors by default (ADR-0013) | P0 | No outbound calls without explicit operator opt-in |
| REQ-NFR-009 | Footprint | Container image size ≤ 200 MB (distroless runtime) | P1 | CI image-size check |
| REQ-NFR-010 | License compliance | All transitive deps under MIT / Apache-2.0 / BSD-2 / BSD-3 / ISC / Zlib / MPL-2.0 / BSL-1.0 (redis) | P0 | `cargo deny check licenses` zero violations |
| REQ-NFR-011 | UI performance | Chat-ui first-token rendered ≤ 1.5 s P95 against local Ollama (mirrors REQ-NFR-001 at the UI layer — composer submit → first delta visible) | P1 | Playwright perf test on `@xiaoguai/chat-ui` |
| REQ-NFR-012 | UI compatibility | Playwright golden-path suites green on Chromium / Firefox / WebKit (Safari) for both chat-ui and admin-ui | P1 | `pnpm -r test:e2e` in CI |
| REQ-NFR-013 | UI footprint | Each SPA's production build ≤ 1 MB gzipped initial bundle (excluding chunked routes) | P2 | `vite build` size budget assertion in CI |
| REQ-NFR-014 | UI accessibility | Axe-core serious + critical violations = 0 on all golden-path pages | P1 | Axe runs inside Playwright e2e harness |

---

## 6. External behaviours (BEH-*)

These are observable outcomes that the test suite must cover regardless of implementation detail.

| ID | Title | Trigger | Observable outcome |
|---|---|---|---|
| BEH-CHAT-001 | SSE delta stream during agent reasoning | `POST /v1/sessions/{id}/messages` | Client receives a sequence of `event: delta` lines with token deltas, then `event: tool_call`, then `event: final` |
| BEH-AUDIT-001 | Chain detects forged or deleted rows | Operator runs `DELETE FROM audit_log` then `GET /v1/admin/audit/verify` | Endpoint returns `valid=false, broken_at_sequence=<n>` |
| BEH-HOTL-001 | Budget exhaustion produces escalate-not-deny | Tenant exceeds `max_count` in window | Tool call routed to `escalate_to` channel; agent gets synthetic ToolResult; audit row `scheduler.proactive_denied` or `tool.escalated` written |
| BEH-MCPEXEC-001 | Code timeout returns timed_out=true | `execute_python` with infinite loop | Process killed at deadline; result has `timed_out=true`, `exit_code=null` |
| BEH-MCPEXEC-002 | Environment is scrubbed | `execute_python` reads `XIAOGUAI_AUDIT_SIGNING_KEY` | `stdout` shows the env var is unset |
| BEH-IM-001 | Forged Feishu webhook rejected | `POST /v1/im/feishu/event` with mismatched HMAC | 401 before deserialisation |
| BEH-MEM-001 | Cross-tenant recall returns empty | Tenant A queries semantic recall for content stored by tenant B | Empty result, no leaked metadata |

---

## 7. Risks & must-not-regress

### 7.1 Risks (RISK-*)

| ID | Title | Level | Likelihood | Impact | Mitigation |
|---|---|---|---|---|---|
| RISK-SEC-001 | Supply-chain compromise via unsigned skill pack | high | low | high | Signed packs only; operator opt-in `--allow-unsigned`; cargo-deny / cargo-vet for Rust deps |
| RISK-SEC-002 | PII leak via audit OTel spans | critical | medium | high | `RedactingSpanExporter` mandatory; same patterns as audit row redactor |
| RISK-SEC-003 | Sandbox escape from `execute_python` exfiltrating secrets | high | low | high | Env allowlist, fresh tempdir, ulimit, k8s NetworkPolicy egress-deny |
| RISK-SEC-004 | Cross-tenant memory leak via crafted recall query | high | low | high | Postgres RLS + app-layer `WHERE tenant_id = ?` double layer |
| RISK-OPS-001 | Air-gap deploy hits cloud LLM endpoint due to mis-seeded provider row | medium | medium | medium | Migration 0020 seeds `enabled=false` for all cloud providers; smoke test verifies |
| RISK-OPS-002 | HMAC key rotation breaks chain verification | high | medium | high | 30-day dual-key verification window; key rotation runbook |
| RISK-LIC-001 | A transitive dep silently relicenses to AGPL/SSPL | high | low | high | `cargo deny check licenses` in CI; allow-list explicit |
| RISK-LIC-002 | Customer mistakes BUSL self-host for production grant | medium | high | medium | License banner in admin UI, README, `xiaoguai serve` first-run notice |
| RISK-COST-001 | Token-bomb attack (large prompt / long generation) drains tenant budget | high | medium | high | Per-tenant cost quota + token-bomb defense (ADR-0009) |

### 7.2 Must-not-regress

| ID | Title | Priority |
|---|---|---|
| MR-AUTH-001 | All `/v1/**` (except `/healthz`) require bearer JWT | P0 |
| MR-AUTH-002 | Cross-tenant data read returns empty, never leaks IDs | P0 |
| MR-AUDIT-001 | Audit chain remains valid across upgrades (migration must not delete/update rows) | P0 |
| MR-IM-001 | Inbound IM webhook signature verification cannot be disabled | P0 |
| MR-CACHE-001 | `cache.url` empty → in-process backend boots successfully without crashing | P0 |
| MR-MEM-001 | `OLLAMA_HOST` unset → memory store still serves 200 (not 503) | P0 |
| MR-LIC-001 | `cargo deny check licenses` exits 0 on every CI run | P0 |
| MR-PERF-001 | First-token latency does not regress more than 20 % between releases | P1 |

---

## 8. Delta from the 2026-05-21 brainstorm

This table records where v1.4.0 diverges from the original brainstorm so design drift is auditable.

| Area | Brainstorm vision | v1.4.0 reality | Why diverged |
|---|---|---|---|
| Crate count | ~15 crates | 31 crates | Functional expansion: memory, RAG, orchestrator, personas, tasks, scheduler, watch, anomaly, observability split out |
| Binary count | 2 (`xiaoguai-core` + `xiaoguai-im-gateway`) | 3 (`xiaoguai`, `xiaoguai-core` legacy shim, `xiaoguai-mcp-exec`) and unified CLI subsumes IM gateway | Tier-1a unification (PR #59) + Tier-2 added the sandbox bin |
| LLM providers | 3 (Ollama / vLLM / OpenAI-compat) | 8 (added Anthropic, Gemini, Bedrock, Azure-OpenAI, Mistral, Groq) | Demand for cloud providers as opt-in fallback chain |
| IM gateways | 3 (Feishu / DingTalk / WeCom), 钉钉+企微 at v1.1 | 7 (added Slack / Discord / Telegram / Mattermost) | Demand from non-China-only deployments |
| Cross-process transport | gRPC (tonic) between core ↔ im-gateway | In-process (unified binary); gRPC available but unused for IM | Single-binary direction won |
| Cache | Valkey required | Valkey optional; in-process DashMap fallback (PR #60) | Air-gap deployment shape |
| RAG | "v2.0; call MCP RAG tools" | Native crate `xiaoguai-rag` shipped early | Demand sooner than expected |
| Multi-agent | Non-goal N5 | Crate `xiaoguai-orchestrator` exists (G11) | Demand sooner than expected |
| CodeAct execution | Non-goal N6 | `xiaoguai-mcp-exec` ships `execute_python` (G2 + Tier-2a) | Demand sooner than expected; landed safely behind HotL gate |
| Sandbox tech | cgroup v2 + seccomp + userns + netns (Linux-only) | ulimit + tokio timeout + tempdir + env scrub (x-platform) — k8s NetworkPolicy for egress | Operability won over kernel-perfect isolation; wasmtime+pyodide flagged as future upgrade |
| Secret management | age + per-tenant DEK | Deferred; secrets currently from env / file | Not yet implemented |
| Compliance templates | 等保 2.0 三级 self-check | SOC2 + GDPR audit log present; report export deferred (Tier-3b) | Scope shifted; templates still TBD |

---

## 9. Future features (roadmap candidates, not yet committed)

These are listed in the HANDOFF and warrant a separate design conversation. They are **not** in scope for v1.4.x.

| ID | Title | Notes |
|---|---|---|
| FUT-001 | `execute_javascript` MCP server | Deno or QuickJS; separate trust boundary from Python sandbox |
| FUT-002 | wasmtime + pyodide sandbox upgrade | Sibling backend to current ulimit approach |
| FUT-003 | Skill marketplace public publish flow | Currently anti-goal N5; reconsider for v2.x |
| FUT-004 | Multi-pane "Today" view in chat UI | Designer track |
| FUT-005 | Mobile native client | Out of scope until web-responsive saturation reached |
| FUT-006 | HIPAA / PCI templates | After SOC2 / GDPR / 等保 baseline stabilises |
| FUT-007 | Multi-region replication and HA auto-failover | After single-instance baseline matures |

---

## 10. Open questions

Captured for later resolution; PRs should not block on these.

| Q | Owner | Default for v1.4 |
|---|---|---|
| Q1 — `xiaoguai-orchestrator` exact scope (multi-agent dispatch vs workflow engine) | architect | "Multi-agent dispatch + coordination" per HANDOFF decision; detailed contract TBD in LLD |
| Q2 — `xiaoguai-personas` exact scope (presets + role prompts + persona-scoped memory) | product | "Personality presets" per HANDOFF; persona-scoped memory column on `memories` already exists |
| Q3 — Memory TTL purge job | ops | None automated; operator runs `DELETE FROM memories WHERE expires_at < now()` manually; deferred |
| Q4 — Agent-authored skill flow (write-skill tool + HotL approval) | product + security | Not started; design needed before implementation |
| Q5 — Session compaction policy (window vs LLM-summary vs hybrid) | product | Not started; design needed for long local-model sessions |
| Q6 — OAuth 2.1 PKCE outbound MCP scope (internal-only vs marketplace) | security | Not started; Tier-3a |
| Q7 — Eval suite seeding (who writes capability + regression cases) | QA | Not started; suites empty in v1.4.0 |
| Q8 — Telemetry export edge — what is exported, what is never | privacy | ADR-0013 default zero; audit redactor enforces PII guarantee |

---

## 11. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: prd-profile-v1

artifact:
  id: PRD-XIAOGUAI-001
  type: PRD
  title: Xiaoguai Agent Platform — retrofit baseline
  status: draft
  owners:
    - product.xiaoguai
    - engineering.xiaoguai
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents:
    - DESIGN-XIAOGUAI-BRAINSTORM-2026-05-21
    - HANDOFF-XIAOGUAI-DESIGN-DOCS

entities:
  requirements:
    - { id: REQ-CHAT-001, class: functional, title: "Streaming chat with single agent over HTTP+SSE", statement: "End user can hold a streaming chat with a single agent over HTTP+SSE.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["POST /v1/sessions/{id}/messages returns SSE stream with delta/tool_call/final events", "GET /v1/sessions/{id}/messages lists prior turns"] }
    - { id: REQ-CHAT-002, class: functional, title: "Cancel in-flight agent run", statement: "User can cancel an in-flight agent run.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["POST /v1/sessions/{id}/cancel interrupts at next tool-call boundary"] }
    - { id: REQ-CHAT-003, class: functional, title: "Session fork from prior message", statement: "User can fork a session from any prior message.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["POST /v1/sessions/{id}/fork creates new session with parent_session_id set"] }
    - { id: REQ-CHAT-004, class: functional, title: "Parallel tool-call dispatch in ReAct loop", statement: "Agent ReAct loop dispatches parallel tool calls per turn.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Multi-tool-call response executed via tokio::join_all", "Per-call HotL gate checked before spawn"] }
    - { id: REQ-CHAT-005, class: functional, title: "Configurable max iterations", statement: "Max-iteration cap is configurable per tenant.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Default 25", "Override via tenant config", "Termination reason logged"] }
    - { id: REQ-LLM-001, class: functional, title: "Multi-provider per tenant", statement: "Operator can register multiple LLM providers per tenant.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["POST /v1/admin/llm/providers accepts provider config"] }
    - { id: REQ-LLM-002, class: functional, title: "Eight provider kinds", statement: "Provider kinds: ollama, openai_compat, anthropic, gemini, bedrock, azure_openai, mistral, groq.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["xiaoguai provider register --kind <k> rejects unknown kinds"] }
    - { id: REQ-LLM-003, class: functional, title: "Fallback chain", statement: "Provider fallback chain cascades on 5xx or timeout.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Each attempt produces a token_usage row"] }
    - { id: REQ-LLM-004, class: functional, title: "Ollama air-gap default", statement: "Ollama is the air-gap default provider.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Migration 0020 seeds ollama-local enabled", "Cloud providers seeded disabled"] }
    - { id: REQ-LLM-005, class: non_functional, title: "Token usage and cost recorded", statement: "Token usage and cost are recorded per request.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["token_usage table populated on every successful stream"] }
    - { id: REQ-MCP-001, class: functional, title: "Per-tenant MCP registry", statement: "Per-tenant MCP server registry.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["mcp_servers table holds tenant-scoped rows", "enabled=false excludes from agent toolbox"] }
    - { id: REQ-MCP-002, class: functional, title: "Three MCP transports", statement: "stdio, http, sse transports supported.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["xiaoguai mcp register --transport {stdio,http,sse} works for all"] }
    - { id: REQ-MCP-003, class: functional, title: "Marketplace catalog browsable", statement: "Marketplace catalog browsable.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["GET /v1/mcp/marketplace returns catalog", "POST /v1/mcp/marketplace/install installs"] }
    - { id: REQ-MCP-004, class: functional, title: "Sandboxed Python executor", statement: "Built-in sandboxed Python executor via xiaoguai-mcp-exec.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["execute_python tool with timeout/memory/env scrub/64KB cap/stderr PII redaction"] }
    - { id: REQ-MCP-005, class: functional, title: "Outbound MCP serving", statement: "Xiaoguai can serve as MCP server itself.", priority: P2, status: proposed, scope: in, acceptance_criteria: ["POST /v1/mcp/serve mounts Toolbox as outbound MCP"] }
    - { id: REQ-MEM-001, class: functional, title: "Long-term memory with semantic recall", statement: "Long-term memory with semantic recall.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["POST /v1/memories writes", "POST /v1/memories/recall returns top-k by cosine distance over pgvector HNSW"] }
    - { id: REQ-MEM-002, class: functional, title: "Backend selectable at boot", statement: "Memory backend selectable via OLLAMA_HOST.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Set -> OllamaEmbedder", "Unset -> InMemoryEmbedder deterministic hash"] }
    - { id: REQ-MEM-003, class: non_functional, title: "Tenant-scoped via RLS", statement: "Memory is tenant-scoped via RLS.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Cross-tenant query returns empty even with crafted JWT"] }
    - { id: REQ-MEM-004, class: functional, title: "TTL field on memories", statement: "Memories support optional expires_at; auto-purge deferred.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Expired rows excluded from recall"] }
    - { id: REQ-MEM-005, class: functional, title: "RAG via MCP tools", statement: "RAG document ingestion via MCP tools.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["xiaoguai-rag exposes loaders and chunkers"] }
    - { id: REQ-IM-001, class: functional, title: "Seven IM platforms", statement: "Seven IM adapter crates: feishu, dingtalk, wecom, slack, discord, telegram, mattermost.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["One crate per platform"] }
    - { id: REQ-IM-002, class: functional, title: "Identity mapping", statement: "IM identity mapping table.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["im_identities maps (platform, platform_user_id) to user_id"] }
    - { id: REQ-IM-003, class: non_functional, title: "Webhook signature verification", statement: "Inbound webhooks verify signatures before deserialisation.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["WeCom AES, Discord Ed25519, Slack HMAC, Feishu HMAC verified"] }
    - { id: REQ-IM-004, class: functional, title: "Trailing message replay", statement: "Up to MAX_MESSAGES_PER_CONVERSATION prior messages replayed.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Default 50; configurable"] }
    - { id: REQ-IM-005, class: functional, title: "IM history backend selectable", statement: "Postgres default; in-process opt-in for dev.", priority: P2, status: proposed, scope: in, acceptance_criteria: ["XIAOGUAI_IM__USE_IN_PROCESS_HISTORY=true switches backend"] }
    - { id: REQ-AUTH-001, class: non_functional, title: "OIDC SSO with bearer JWT", statement: "All /v1/** except /healthz require bearer JWT.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Missing/invalid token returns 401"] }
    - { id: REQ-AUTH-002, class: non_functional, title: "Casbin RBAC", statement: "Casbin RBAC enforcement per request.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Admin-only routes verified via integration tests"] }
    - { id: REQ-AUTH-003, class: non_functional, title: "RLS double-layer", statement: "App-layer tenant filter + Postgres RLS policy.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Defence-in-depth verified by negative test"] }
    - { id: REQ-AUTH-004, class: functional, title: "Tenant CRUD via admin API", statement: "Tenant list via admin API; further CRUD via CLI.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["GET /v1/admin/tenants list"] }
    - { id: REQ-AUD-001, class: non_functional, title: "Append-only audit log with HMAC chain", statement: "HMAC chain over (prev_hmac, ts, tenant, actor, action, resource, details).", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Every state-changing call writes one row"] }
    - { id: REQ-AUD-002, class: functional, title: "Operator chain verification", statement: "Operator can verify chain integrity via API.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["GET /v1/admin/audit/verify returns valid bool"] }
    - { id: REQ-AUD-003, class: non_functional, title: "PII redaction before signing", statement: "PII scrubbed before HMAC computed.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Emails/IPv4/Bearer/AWS keys scrubbed when XIAOGUAI_AUDIT_REDACT_PII=true"] }
    - { id: REQ-AUD-004, class: non_functional, title: "OTel span PII redaction", statement: "RedactingSpanExporter scrubs exported spans.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Same pattern set as audit"] }
    - { id: REQ-AUD-005, class: non_functional, title: "HMAC key rotation 30-day window", statement: "Verifier accepts entries signed by either key during 30-day window.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Runbook documents rotation; verifier passes dual-key fixture"] }
    - { id: REQ-AUD-006, class: functional, title: "Compliance report export", statement: "Generate SOC2/GDPR/HIPAA reports from audit chain.", priority: P2, status: proposed, scope: in, acceptance_criteria: ["Tier-3b; not yet implemented"] }
    - { id: REQ-HOTL-001, class: functional, title: "Per-tenant budgets", statement: "Per-tenant spend and count budgets.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["hotl_policies row checked before tool spawn"] }
    - { id: REQ-HOTL-002, class: functional, title: "Allow-then-escalate", statement: "Budget exhaustion routes to escalate_to channel, not deny.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Audit row recorded for escalation"] }
    - { id: REQ-HOTL-003, class: non_functional, title: "Fail-closed on policy outage", statement: "If hotl_policies unreadable, tool calls denied.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["No synthetic allow"] }
    - { id: REQ-HOTL-004, class: functional, title: "HotL CRUD via API", statement: "HotL policy CRUD via API.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["GET/POST /v1/hotl/policies, DELETE /v1/hotl/policies/{id}"] }
    - { id: REQ-SKILL-001, class: functional, title: "Browse curated catalog", statement: "Tenant can browse curated skill catalog.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["GET /v1/skills/catalog returns whitelisted packs"] }
    - { id: REQ-SKILL-002, class: functional, title: "Install pack", statement: "Tenant can install a pack.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["POST /v1/skills/install records install", "Hot-reload deferred to next boot"] }
    - { id: REQ-SKILL-003, class: non_functional, title: "Local-first signed packs", statement: "Packs install from local fs or tenant-private index; signature required.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Unsigned packs rejected without --allow-unsigned"] }
    - { id: REQ-SKILL-004, class: functional, title: "Knob overrides per install", statement: "Per-install knob overrides via pack_operators table.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Knob overrides applied without forking pack"] }
    - { id: REQ-SKILL-005, class: functional, title: "Public marketplace publish", statement: "Anonymous public publish flow.", priority: P0, status: proposed, scope: out, acceptance_criteria: ["Intentionally not supported in v1.x"] }
    - { id: REQ-SCHED-001, class: functional, title: "Six trigger types", statement: "cron, interval, proactive, file_watch, webhook plus reserved git_push/db_poll.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["All five primary triggers fire and audit"] }
    - { id: REQ-SCHED-002, class: functional, title: "Four push sinks", statement: "feishu, telegram, email, inbox sinks.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Each sink delivers a fixture message"] }
    - { id: REQ-SCHED-003, class: non_functional, title: "Public webhook auth", statement: "Public scheduler webhooks require X-Xiaoguai-Token.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Token rotation and revocation tested"] }
    - { id: REQ-SCHED-004, class: functional, title: "Proactive budgets", statement: "Daily per-user budget enforced.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Exhaustion writes scheduler.proactive_denied audit"] }
    - { id: REQ-SCHED-005, class: functional, title: "File-watch debounce", statement: "Configurable debounce on filesystem bursts.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["FILE_WATCH_DEBOUNCE_MS default 250"] }
    - { id: REQ-OUT-001, class: functional, title: "Outcome attribution", statement: "Endpoint to record outcome attribution.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["POST /v1/outcomes accepts payload"] }
    - { id: REQ-OUT-002, class: functional, title: "Daily-bucketed aggregations", statement: "Aggregations daily-bucketed and tenant-scoped.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["GET /v1/outcomes/summary, /timeseries"] }
    - { id: REQ-OUT-003, class: functional, title: "Anomaly detection", statement: "Z-score / EWMA flags over outcome series.", priority: P2, status: proposed, scope: in, acceptance_criteria: ["xiaoguai anomaly CLI emits flags"] }
    - { id: REQ-WS-001, class: functional, title: "Workspaces", statement: "Workspaces above sessions.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["CRUD via API"] }
    - { id: REQ-TASK-001, class: functional, title: "Kanban board five states", statement: "Tasks Kanban: triage/todo/doing/review/done.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["task_state_log records transitions"] }
    - { id: REQ-TASK-002, class: functional, title: "Auto-dispatcher", statement: "Auto-route triage tasks to agent persona.", priority: P2, status: proposed, scope: in, acceptance_criteria: ["Configurable per board"] }
    - { id: REQ-EVAL-001, class: functional, title: "Catalog and run eval suites", statement: "Eval catalog + run.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["GET/POST /v1/admin/eval/* working"] }
    - { id: REQ-EVAL-002, class: functional, title: "Session to eval case", statement: "Export session as eval case.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["POST /v1/admin/eval/case-from-session materialises .eval.yaml"] }
    - { id: REQ-EVAL-003, class: functional, title: "Capability vs regression split", statement: "Suites split into capability/ (probe) and regression/ (100%).", priority: P2, status: proposed, scope: in, acceptance_criteria: ["Suite content empty in v1.4.0; runtime ready"] }
    - { id: REQ-NFR-001, class: non_functional, title: "First-token latency P95 <= 1.5s", statement: "Local Ollama qwen2.5-coder:7b.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["perf-regression benchmark"] }
    - { id: REQ-NFR-002, class: non_functional, title: "Memory recall P95 <= 300ms at 100k", statement: "pgvector HNSW.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["xiaoguai-memory benchmark"] }
    - { id: REQ-NFR-003, class: non_functional, title: "Default-deny network at MCP children", statement: "k8s NetworkPolicy template denies egress.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Template shipped in deploy/helm"] }
    - { id: REQ-NFR-004, class: non_functional, title: "IM webhook signature verification", statement: "All inbound IM webhooks signature-verified.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Negative test rejects forged signature"] }
    - { id: REQ-NFR-005, class: non_functional, title: "Single-instance restart safe", statement: "In-flight sessions resumable on WS reconnect.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["Smoke interrupt+reconnect+continue"] }
    - { id: REQ-NFR-006, class: non_functional, title: "Multi-arch binaries", statement: "linux/amd64 + linux/arm64 (信创).", priority: P0, status: proposed, scope: in, acceptance_criteria: ["release.yml produces both"] }
    - { id: REQ-NFR-007, class: non_functional, title: "Prometheus + OTLP with PII redaction", statement: "Both telemetry channels redacted.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["RedactingSpanExporter test"] }
    - { id: REQ-NFR-008, class: non_functional, title: "Zero default telemetry", statement: "No outbound to authors without opt-in.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["Network test confirms"] }
    - { id: REQ-NFR-009, class: non_functional, title: "Container <= 200MB", statement: "Distroless runtime.", priority: P1, status: proposed, scope: in, acceptance_criteria: ["CI image-size check"] }
    - { id: REQ-NFR-010, class: non_functional, title: "License compliance", statement: "All deps permissive + redis BSL-1.0.", priority: P0, status: proposed, scope: in, acceptance_criteria: ["cargo deny check licenses zero violations"] }

  risks:
    - { id: RISK-SEC-001, title: "Unsigned skill pack supply-chain compromise", statement: "An attacker publishes a malicious skill pack that a tenant installs.", status: proposed, scope: in, level: high, likelihood: low, impact: "Tenant data leak or sandbox escape" }
    - { id: RISK-SEC-002, title: "PII leak via OTel spans", statement: "Exported spans carry unredacted PII.", status: proposed, scope: in, level: critical, likelihood: medium, impact: "Compliance violation, customer trust loss" }
    - { id: RISK-SEC-003, title: "Sandbox escape exfiltrates secrets", statement: "execute_python reads env or filesystem outside workdir.", status: proposed, scope: in, level: high, likelihood: low, impact: "Operator key exfiltration" }
    - { id: RISK-SEC-004, title: "Cross-tenant memory leak", statement: "Crafted recall query bypasses RLS.", status: proposed, scope: in, level: high, likelihood: low, impact: "Tenant data leak" }
    - { id: RISK-OPS-001, title: "Air-gap install reaches cloud LLM", statement: "Mis-seeded provider row enables outbound call from air-gap deploy.", status: proposed, scope: in, level: medium, likelihood: medium, impact: "Air-gap posture broken" }
    - { id: RISK-OPS-002, title: "HMAC key rotation breaks chain", statement: "Rotation procedure not followed; verification fails.", status: proposed, scope: in, level: high, likelihood: medium, impact: "Audit non-repudiation broken" }
    - { id: RISK-LIC-001, title: "Transitive dep relicenses to AGPL/SSPL", statement: "An upstream silently switches license.", status: proposed, scope: in, level: high, likelihood: low, impact: "BUSL commercial posture compromised" }
    - { id: RISK-LIC-002, title: "Customer mistakes BUSL self-host for production grant", statement: "Misinterpretation of BUSL.", status: proposed, scope: in, level: medium, likelihood: high, impact: "Revenue leakage" }
    - { id: RISK-COST-001, title: "Token-bomb attack", statement: "Large prompt drains tenant budget.", status: proposed, scope: in, level: high, likelihood: medium, impact: "Cost overrun" }

  must_not_regress:
    - { id: MR-AUTH-001, title: "All /v1/** require bearer JWT", statement: "All /v1/** (except /healthz) require bearer JWT.", status: proposed, scope: in, priority: P0 }
    - { id: MR-AUTH-002, title: "Cross-tenant returns empty", statement: "Cross-tenant data read returns empty, never leaks IDs.", status: proposed, scope: in, priority: P0 }
    - { id: MR-AUDIT-001, title: "Audit chain valid across upgrades", statement: "Migration must not delete or update audit_log rows.", status: proposed, scope: in, priority: P0 }
    - { id: MR-IM-001, title: "IM webhook signature verification on", statement: "Verification cannot be disabled.", status: proposed, scope: in, priority: P0 }
    - { id: MR-CACHE-001, title: "Empty cache.url boots in-process", statement: "cache.url empty boots in-process backend without crash.", status: proposed, scope: in, priority: P0 }
    - { id: MR-MEM-001, title: "Unset OLLAMA_HOST serves 200", statement: "Memory store serves 200 even with OLLAMA_HOST unset.", status: proposed, scope: in, priority: P0 }
    - { id: MR-LIC-001, title: "cargo deny licenses exit 0", statement: "cargo deny check licenses exits 0 on every CI run.", status: proposed, scope: in, priority: P0 }
    - { id: MR-PERF-001, title: "First-token latency <= +20%", statement: "First-token latency does not regress >20% between releases.", status: proposed, scope: in, priority: P1 }

  external_behaviors:
    - { id: BEH-CHAT-001, title: "SSE delta stream during reasoning", statement: "Client receives delta/tool_call/final SSE event sequence.", status: proposed, scope: in, actor: "end_user", trigger: "POST /v1/sessions/{id}/messages", observable_outcome: "Stream of delta then tool_call then final events" }
    - { id: BEH-AUDIT-001, title: "Chain detects tampering", statement: "Deleted or forged row yields valid=false.", status: proposed, scope: in, actor: "operator", trigger: "DELETE on audit_log + verify call", observable_outcome: "valid=false with broken_at_sequence" }
    - { id: BEH-HOTL-001, title: "Budget exhaustion escalates not denies", statement: "Tool call routed to escalate_to; agent receives synthetic ToolResult.", status: proposed, scope: in, actor: "agent", trigger: "tenant exceeds max_count in window", observable_outcome: "Audit row + escalation message" }
    - { id: BEH-MCPEXEC-001, title: "Code timeout returns timed_out=true", statement: "Infinite loop killed at deadline.", status: proposed, scope: in, actor: "agent", trigger: "execute_python with infinite loop", observable_outcome: "timed_out=true, exit_code=null" }
    - { id: BEH-MCPEXEC-002, title: "Env scrubbed", statement: "Secrets not visible to sandbox.", status: proposed, scope: in, actor: "agent", trigger: "execute_python reads env", observable_outcome: "Env var unset in stdout" }
    - { id: BEH-IM-001, title: "Forged IM webhook rejected", statement: "Mismatched signature 401.", status: proposed, scope: in, actor: "external", trigger: "POST with forged signature", observable_outcome: "401 before deserialisation" }
    - { id: BEH-MEM-001, title: "Cross-tenant recall empty", statement: "Tenant A cannot read tenant B's memories.", status: proposed, scope: in, actor: "end_user", trigger: "recall against foreign tenant", observable_outcome: "Empty result, no metadata leak" }

  decisions:
    - { id: DEC-DEPLOY-001, title: "Single-tenant air-gap default deployment", statement: "Default deployment is single-tenant air-gap; multi-tenant SaaS is opt-in.", status: proposed, scope: in, decision: "Single-tenant air-gap is the marketed default; multi-tenant SaaS supported by the same codebase but opt-in.", rationale: "Matches enterprise / private-deploy buyer profile, BUSL commercial posture, and existing in-process cache + Ollama default." }
    - { id: DEC-SKILL-001, title: "Local-first signed skill packs", statement: "Skill marketplace is local-first; packs must be signed; anonymous public publish out of scope.", status: proposed, scope: in, decision: "Local filesystem or tenant-private index; signature required; HotL approval at install time.", rationale: "Trust model for air-gap; aligns with Tier-2c agent-authored skills." }
    - { id: DEC-ORCH-001, title: "Orchestrator is multi-agent dispatch", statement: "xiaoguai-orchestrator scope: multi-agent dispatch and coordination.", status: proposed, scope: in, decision: "Multi-agent dispatch + coordination, not a workflow-engine.", rationale: "Operator demand; aligns with multi-agent-peer architecture doc." }
    - { id: DEC-PERSONA-001, title: "Personas are role presets", statement: "xiaoguai-personas scope: personality / role presets + persona-scoped memory view.", status: proposed, scope: in, decision: "Personality presets and persona-scoped memory.", rationale: "Schema already has personas table; aligns with HANDOFF answer." }

  flows: []
  test_cases: []

relations:
  - { id: REL-001, type: derived_from, from: REQ-CHAT-001, to: DESIGN-XIAOGUAI-BRAINSTORM-2026-05-21, status: active }
  - { id: REL-002, type: derived_from, from: REQ-LLM-002, to: DESIGN-XIAOGUAI-BRAINSTORM-2026-05-21, status: active }
  - { id: REL-003, type: refines, from: DEC-DEPLOY-001, to: REQ-NFR-008, status: active }
  - { id: REL-004, type: refines, from: DEC-SKILL-001, to: REQ-SKILL-003, status: active }
  - { id: REL-005, type: refines, from: DEC-ORCH-001, to: REQ-CHAT-004, status: active }
  - { id: REL-006, type: mitigates, from: REQ-AUD-003, to: RISK-SEC-002, status: active }
  - { id: REL-007, type: mitigates, from: REQ-MCP-004, to: RISK-SEC-003, status: active }
  - { id: REL-008, type: mitigates, from: REQ-AUTH-003, to: RISK-SEC-004, status: active }
  - { id: REL-009, type: mitigates, from: REQ-NFR-010, to: RISK-LIC-001, status: active }
  - { id: REL-010, type: mitigates, from: REQ-HOTL-001, to: RISK-COST-001, status: active }

waivers:
  - { id: WVR-SKILL-001, target_ids: [REQ-SKILL-005], reason: "Public marketplace publish flow is anti-goal N5 for v1.x; revisit for v2.x", status: approved, approved_by: "product.xiaoguai", approved_at: "2026-05-28" }
  - { id: WVR-EVAL-001, target_ids: [REQ-EVAL-003], reason: "Eval runtime is shipped but suite content seeding deferred; not a release blocker for v1.4.0", status: approved, approved_by: "qa.xiaoguai", approved_at: "2026-05-28" }
  - { id: WVR-AUD-001, target_ids: [REQ-AUD-006], reason: "Compliance report export is Tier-3b; the underlying audit chain is sufficient for v1.4.0 attestation", status: approved, approved_by: "compliance.xiaoguai", approved_at: "2026-05-28" }
```
<!-- TRACEABILITY-METADATA:END -->
