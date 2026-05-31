# HLD — Xiaoguai v1.4 (retrofit)

| | |
|---|---|
| Document ID | `HLD-XIAOGUAI-001` |
| Type | High-Level Design |
| Status | `draft` |
| Author | Zhou Wei |
| Source | [`PRD-XIAOGUAI-001`](prd-xiaoguai.md), [`API-XIAOGUAI-001`](api-contract.md), implementation repo |
| Updated | 2026-05-31 |

> **Retrofit notice.** This HLD describes the architecture as it exists at v1.4.0. ADR identifiers (`ADR-0001`…`ADR-0018`) refer to records in the implementation repo `docs/architecture/adr/`.

> **Philosophy.** The design rationale behind everything below — the R.E.S.T product goals (Reliability, Efficiency, Security, Traceability), the harness-as-REPL-container abstraction, the control-plane/data-plane split, and the six design principles we negotiate trade-offs against — lives in [`harness-engineering.md`](harness-engineering.md). Read that doc first if you are new to the project; it gives you the vocabulary to read this one.

---

## 1. System context

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              External actors                              │
│   end users   tenant admins   operators   IM platforms   curated catalog  │
└──────┬──────────┬─────────────┬──────────────┬───────────────┬───────────┘
       │ chat-ui  │ admin-ui    │ CLI / API    │ webhook+WS    │ skill pack
       ▼          ▼             ▼              ▼               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                            xiaoguai (binary)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Edge:  HTTP/SSE (axum, /v1/**)  ·  CLI (clap)  ·  IM gateways      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Agent runtime:  ReAct loop → HotL gate → parallel tool dispatch    │  │
│  │  Orchestrator (multi-agent)  ·  Personas (role presets)              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────┬──────────────┬───────────────┬──────────────────────┐  │
│  │ LLM router   │ MCP client + │ Memory + RAG  │ Scheduler / Watch /  │  │
│  │ (8 providers)│ supervisor   │ (pgvector)    │ Anomaly              │  │
│  └──────────────┴──────────────┴───────────────┴──────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Cross-cutting:  Auth (OIDC + Casbin + RLS)  ·  Audit (HMAC chain)   │  │
│  │                  Observability (OTel + Prometheus, both PII-redacted)│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────┬────────────────────────────┬────────────────────┬──────────────────┘
      │                            │                    │
      ▼                            ▼                    ▼
┌───────────────┐         ┌────────────────┐    ┌──────────────────────┐
│ PostgreSQL    │         │ Cache backend  │    │ External services    │
│  + pgvector   │         │ (Valkey or     │    │ (Ollama, OpenAI,     │
│  (RLS, HNSW)  │         │  in-process)   │    │  IM APIs, MCP srv)   │
└───────────────┘         └────────────────┘    └──────────────────────┘
```

**Deployment posture (DEC-DEPLOY-001):** the marketed default is **single-tenant air-gap** — one xiaoguai binary plus one Postgres, no cache broker, no outbound internet. Multi-tenant SaaS uses the same binary with cache promoted to Valkey/Redis and OIDC pointed at the tenant IdP.

---

## 2. Module map (32 crates + 4 frontend packages)

Grouped by domain. Crate names are the source of truth; one LLD per starred (★) crate ships alongside this HLD. Frontend packages live in `frontend/` (pnpm monorepo) and have their own LLDs — see [`lld/lld-admin-ui.md`](lld/lld-admin-ui.md), [`lld/lld-chat-ui.md`](lld/lld-chat-ui.md).

| Domain | Crate | Role |
|---|---|---|
| Foundations | `xiaoguai-types` | Shared DTOs, ID newtypes, error enum |
|  | `xiaoguai-config` | YAML + env loader, hot-reload boundary |
|  | `xiaoguai-storage` ★ | sqlx repositories, RLS plumbing, migrations; in-process DashMap cache fallback when `cache.url == ""` (Tier-1b, PR #60) |
|  | `xiaoguai-auth` | OIDC verify, Casbin enforcer |
|  | `xiaoguai-audit` ★ | HMAC chain, PII redactor, OTel decorator; **per-framework compliance export module (SOC2 CC7.2 / GDPR Art. 30 / HIPAA §164.312, PR #74, T5)** |
| LLM + MCP | `xiaoguai-llm` ★ | LlmRouter, **9 provider backends** (ollama, openai_compat, anthropic, gemini, bedrock, azure_openai, mistral, groq, **minimax** — DEC-024), UsageSink, conservative `token_count` estimator (PR #66) |
|  | `xiaoguai-mcp` ★ | MCP client, supervisor, transports; **outbound OAuth 2.1 PKCE auth module + `TokenStore` (PR #73, T4)** |
|  | `xiaoguai-mcp-exec` ★ | Sandboxed Python `execute_python` server |
|  | `xiaoguai-mcp-exec-js` ★ | **Sandboxed JavaScript `execute_javascript` server, Deno default (PR #75, T6); separate trust boundary** |
| Agent runtime | `xiaoguai-agent` ★ | ReAct loop, parallel tool dispatch, HotL gate (Tier-2 prereq, PR #61), LLM-summarisation compaction (Tier-2d, PR #66, v0.5.4.1), **`propose_skill` synthetic MCP tool routing to skill-author flow (PR #72, T3)** |
|  | `xiaoguai-runtime` ★ | Worker pool, queue, cancellation registry |
|  | `xiaoguai-orchestrator` ★ | Multi-agent dispatch + coordination |
| Memory + RAG | `xiaoguai-memory` ★ | pgvector-backed long-term memory |
|  | `xiaoguai-rag` ★ | Document loaders, chunkers, retrievers exposed as MCP tools |
| Scheduling | `xiaoguai-scheduler` | Six triggers, four sinks, audit-on-fire |
|  | `xiaoguai-watch` | Active-wakeup watchers (SQL / HTTP) |
|  | `xiaoguai-anomaly` | Z-score / EWMA monitors |
| Eval | `xiaoguai-eval` | `.eval.yaml` runner, capability + regression split |
| API + CLI | `xiaoguai-api` | Axum router, SSE, Toolbox state; **`/v1/skills/proposals/*` (T3), `/v1/audit/exports` (T5) routes** |
|  | `xiaoguai-cli` | clap-based unified CLI; **`xiaoguai skills proposals {list,approve,reject}` (T3), `xiaoguai audit export --framework …` (T5), `xiaoguai mcp register --auth oauth2-pkce …` (T4)** |
|  | `xiaoguai-core` | (legacy shim; thin re-export for back-compat with systemd units) |
| IM | `xiaoguai-im-gateway` ★ | `ImProvider` trait, dispatcher |
|  | `xiaoguai-im-feishu`, `-dingtalk`, `-wecom`, `-slack`, `-discord`, `-telegram`, `-mattermost` | Per-platform adapters |
| Other | `xiaoguai-personas` ★ | Role presets, persona-scoped memory views |
|  | `xiaoguai-tasks` ★ | Kanban board, auto-dispatcher; **`skill_author` module (manifest validator + `propose/approve/reject` lifecycle, PR #72, T3)** |
|  | `xiaoguai-observability` | OTel + Prometheus exporters |
| Frontend (`frontend/` pnpm monorepo) | `@xiaoguai/admin-ui` ★ | Operator-facing SPA — 16 panes (Today / Audit / Scheduler / Eval / Usage / Outcomes / Anomaly / Kanban / Marketplace / Skills / McpServers / Tenants / Providers / HotlPolicies / Memory + sprint-10b Personas + SkillProposals); React 18 + Vite + TS + recharts + i18next |
|  | `@xiaoguai/chat-ui` ★ | End-user SPA — chat surface with SSE streaming, HotL pending inline panel, watch indicator, EU AI Act disclosure banner; React 18 + Vite + TS |
|  | `@xiaoguai/shared` | Type-safe `XiaoguaiClient` (mirrors all `xiaoguai-types` wire types); ~1600 LOC client + SSE consumer; one source of truth for both SPAs |
|  | `@xiaoguai/e2e` | Playwright multi-browser matrix (Chrome / Firefox / Safari) covering chat-ui + admin-ui + scheduler golden paths |

`xiaoguai-core` exists as a legacy entry point for systemd / .deb compatibility (PR #59 unified binary). New deployments use `xiaoguai serve`.

---

## 3. Architectural decisions

### DEC-001 — Single self-contained binary (Tier-1a)

**Statement:** `xiaoguai-core/main.rs` was reduced to a thin shim; the real entry point is `xiaoguai serve` in the unified CLI. The IM gateway, agent loop, MCP supervisor, and API all link into one binary.

**Rationale:** Operations told us deploys with separate `xiaoguai-core` + `xiaoguai-im-gateway` doubled the failure surface (mTLS provisioning, gRPC reconnection logic) for marginal benefit. A single binary is air-gap-friendly and reduces what an operator has to monitor.

**Trade-off:** loses the ability to scale IM gateway independently; mitigated by horizontal pod scaling and the in-process channel design.

**Refines:** REQ-NFR-006, MR-CACHE-001.

### DEC-002 — Cache backend selectable at boot (Tier-1b)

**Statement:** `Cache::connect("")` boots an in-process DashMap. Any `redis://` / `rediss://` URL switches to a Valkey/Redis client. No live failover.

**Rationale:** Air-gap deployments cannot afford a Valkey container. In-process backend honours sub-second TTLs (Redis clamps at 1 s); state is intentionally ephemeral (rate-limit windows, IM session locks, smoke heartbeats).

**Trade-off:** in-process state does not survive restart and cannot be shared across replicas — explicitly opt-out of HA.

**Refines:** DEC-DEPLOY-001 (from PRD), MR-CACHE-001.

### DEC-003 — Memory embedder selectable at boot

**Statement:** Boot-time `OLLAMA_HOST` envar selects `OllamaEmbedder` (semantic) or `InMemoryEmbedder` (deterministic hash). No runtime fail-over between them.

**Rationale:** Decouples integration testing (no Ollama) from production (semantic recall). Single knob avoids configuration drift.

**Trade-off:** dimension is hard-wired to `all-minilm`'s 384 unless the schema is altered.

**Refines:** REQ-MEM-002, MR-MEM-001.

### DEC-004 — Audit chain redacts PII before signing (Tier-1c)

**Statement:** `xiaoguai-audit::redact` strips emails, IPv4, `Bearer <token>` and AWS access keys from `actor`, `resource`, and `details` before HMAC. Signature is computed over the redacted form, so the chain remains verifiable.

**Rationale:** PII may leak through tool results into the audit log; if redaction happened *after* signing the chain becomes a leak vector.

**Trade-off:** if redaction is later turned off (or pattern set is changed) historical rows are still signed under their original setting. Verifier accepts because the rows match their own redacted form.

**Mitigates:** RISK-SEC-002. **Refines:** REQ-AUD-003, REQ-NFR-007.

### DEC-005 — Sandbox via ulimit + tokio timeout (Tier-2a)

**Statement:** `xiaoguai-mcp-exec` runs Python snippets in a fresh tempdir, with `ulimit -v` (address-space), tokio `timeout` (wall-clock), env scrub, 64 KiB output cap, and stderr PII redaction. Network egress is not blocked at this layer — Kubernetes NetworkPolicy or `--network none` is the defense.

**Rationale:** wasmtime+pyodide is the more isolated future option but heavy to ship today; landlock+seccomp is Linux-only and breaks macOS development. ulimit+timeout is x-platform, ships now, and combines with NetworkPolicy at the deployment layer.

**Trade-off:** macOS `ulimit -v` is advisory; production must run Linux containers. No JS runtime yet.

**Mitigates:** RISK-SEC-003. **Refines:** REQ-MCP-004.

### DEC-006 — HotL gate enforces allow-then-escalate

**Statement:** Per-tenant `hotl_policies` row defines `(scope, window_secs, max_count, max_spend_usd, escalate_to)`. Agent ReAct loop dispatches every tool call through `HotlGate.check(scope, amount)`. Budget exhaustion does not deny; it routes the request to the escalate channel and feeds a synthetic ToolResult back to the agent. Policy-store outage fails closed (denies tool calls).

**Rationale:** Hard-deny breaks long-running agent runs unexpectedly; allow-then-escalate gives operators time to react. Fail-closed on store outage stops runaway behaviour.

**Mitigates:** RISK-COST-001. **Refines:** REQ-HOTL-001, REQ-HOTL-002, REQ-HOTL-003.

### DEC-007 — Skill packs are local-first and signed

**Statement:** Packs install from local filesystem or a tenant-private index. Each pack metadata includes a signature; install rejects unsigned packs unless the operator explicitly opts in. Hot-reload is deferred to next session boot (ADR-0017).

**Rationale:** Air-gap deployments cannot reach an internet marketplace. Signature requirement contains supply-chain risk in environments where the operator does not pre-vet every pack.

**Mitigates:** RISK-SEC-001. **Refines:** REQ-SKILL-003.

### DEC-008 — Postgres RLS double-layer multi-tenancy

**Statement:** Every tenant-scoped table carries an RLS policy `USING (tenant_id = current_setting('app.current_tenant_id'))`. The application also injects `WHERE tenant_id = $1` in every query.

**Rationale:** Defence in depth. App-layer prevents accidental SQL bugs; RLS prevents disaster if app-layer is bypassed.

**Mitigates:** RISK-SEC-004. **Refines:** REQ-AUTH-003.

### DEC-009 — Zero default outbound telemetry (ADR-0013)

**Statement:** No telemetry leaves the process without operator opt-in. Prometheus `/metrics` is local-pull. OTLP export is configured per-deploy. PII redaction is mandatory on both channels.

**Mitigates:** RISK-SEC-002 partially. **Refines:** REQ-NFR-007, REQ-NFR-008.

### DEC-010 — Outcomes daily-bucketed and tenant-scoped reads (ADR-0016)

**Statement:** Outcome rows are written raw, but reads via `/v1/outcomes/{summary,timeseries}` are aggregated by day and scoped to the calling tenant. No cross-tenant rollup endpoint.

**Refines:** REQ-OUT-002.

### DEC-011 — Orchestrator is multi-agent dispatch, not a workflow engine

**Statement:** `xiaoguai-orchestrator` schedules agent peers (each a `xiaoguai-agent` instance with its own persona) and coordinates intermediate results. It is not a BPMN-style stateful workflow engine.

**Rationale:** Demand observed during v1.4 was for agents-cooperate-on-one-task, not for designer-defined branching pipelines. The orchestrator depends on `xiaoguai-personas` and inherits per-agent HotL budgets.

**Refines:** REQ-CHAT-004 (parallel dispatch generalises to peer agents), DEC-PERSONA-001.

### DEC-012 — Personas are role presets + persona-scoped memory views

**Statement:** `xiaoguai-personas` stores `(persona_id, system_prompt, default_tools, default_skills)`. Memory rows carry an optional `persona_id`; recall queries can be persona-filtered.

**Refines:** REQ-CHAT-001 (entry point passes a persona_id), REQ-MEM-001.

### DEC-013 — History compaction is LLM-summarisation with slide fallback (Tier-2d, v0.5.4.1)

**Statement:** When the conservative token-count estimate of the in-flight conversation crosses `compaction.trigger_at_pct` (default 75 %) of `compaction.max_context_tokens` (default 30 000), the agent loop replaces the older head of the message list with a single synthetic `Role::System` message carrying an LLM-generated summary, preserving the original `Role::System` prompt and the last `keep_recent` (default 6) turns verbatim. On summariser failure (network, empty output, backend error), the path falls back to `history::slide(keep_recent + 1)` so the loop never blocks on a flaky summary model. Compaction is **opt-in** per tenant via `agent.compaction.enabled` in `local.yaml` — legacy callers (`AgentConfig.compaction = None`) keep the v0.5.4 slide-only behaviour.

**Rationale:** Long agent sessions on local 32k-context Ollama models used to fail with provider 400s once history grew past the window. Pure slide dropped facts the agent later needed. LLM-summarisation preserves concrete facts (names, IDs, paths) while bounding tokens; falling back to slide keeps reliability when the summariser is unavailable. Plan D.2 of session-6 (PR #66) ships it. Provider-native compaction (Anthropic `cache_control`, OpenAI conversation IDs) is preferred when available and the operator can keep this feature off on those backends.

**Refines:** REQ-CHAT-007 (long-session reliability), DEC-001 (single binary still ships compaction inside `xiaoguai-agent`), R.E.S.T Efficiency (token budget) + Reliability (fallback path).

**Metrics:** `xiaoguai_compaction_triggered_total{reason}`, `xiaoguai_compaction_fallback_total{reason}`, `xiaoguai_compaction_token_savings` histogram. Healthy ratio `fallback / triggered < 5 %`; above that, swap `summary_model`.

### DEC-014 — Agent-authored skills are whitelist-only, HotL-gated, admin-approved (PR #72, T3)

**Statement:** A new synthetic MCP tool `propose_skill` lets the agent author skill-pack manifests at runtime. Each proposal must pass three gates: (a) **validator** in `xiaoguai-tasks::skill_author` enforces that the manifest references existing tool names only — it CANNOT declare new MCP servers, load native code, or include `propose_skill` itself (no recursion); (b) **HotL gate** under bucket `skill_author` with a default budget of 5 proposals per tenant per day; (c) **admin approval** via `POST /v1/skills/proposals/:id/approve` (Casbin action `skill.approve`). Approved manifests materialise to `~/.xiaoguai/skills/<name>-<version>.yaml`. The feature is **off by default**; per-tenant config `tenant_settings.allow_skill_authoring=true` is required.

**Rationale:** Hermes Agent parity — agent self-extension is a core capability. But unrestricted skill authoring is a sandbox-escape vector. The three-gate model (validator + HotL + admin) is the harness equivalent of "data-driven evolution with human guard rails" from PHILO §5 dimension 育马 (Cultivating). Disabled tenants silently drop proposals with no audit emission, so the surface is invisible until explicitly enabled.

**Refines:** REQ-SKILL-001 (skill marketplace), PHILO §5 (data-driven evolution + 育马), PHILO §11 (anti-pattern "letting the LLM author its own escapes" — the gates make this safe). R.E.S.T Security dominant.

**Audit:** `skill.propose`, `skill.hotl_gate`, `skill.approve` / `skill.reject` rows all enter the HMAC chain. Three-row sequence per lifecycle ensures full traceability.

### DEC-015 — Outbound MCP OAuth 2.1 PKCE is hand-rolled, tokens per-tenant in RLS table (PR #73, T4)

**Statement:** Authed remote MCP servers (Linear, Notion, GitHub MCP, Anthropic-hosted connectors) require OAuth 2.1 with PKCE (RFC 7636). Implementation is **hand-rolled** in `xiaoguai-mcp::auth::oauth2_pkce` using existing workspace deps (`rand 0.10`, `sha2 0.11`, `base64 0.22`) — no new `oauth2` crate dependency. Tokens stored in new `mcp_oauth_tokens` table with RLS isolating `(server_id, tenant_id)` rows. `HttpMcpClient::connect_with_store` auto-refreshes when `expires_at < now + 60s` and atomically persists rotated refresh tokens per RFC 6749 §6. TLS verification is **on by default**; `XIAOGUAI_MCP_OAUTH_INSECURE=1` is the only escape hatch, logged at `warn`.

**Rationale:** Hand-rolled PKCE is ~120 LOC vs adding `oauth2 = "5"` which pulls a new `getrandom 0.3` chain and alternative async-http abstraction. Avoids transitive-dep churn. RLS on tokens means the api/cli crates never need to filter by tenant — Postgres enforces it. Refresh-token encryption-at-rest is **deferred to a separate hardening PR** (HMAC signing key ≠ symmetric encryption key; reusing audit infrastructure would conflate concerns).

**Refines:** REQ-MCP-AUTH-001, PHILO §15 (policy gateway — OAuth tokens are *outbound* authn, not *inbound* policy). R.E.S.T Security dominant.

**Audit:** OAuth flow emits `mcp.oauth.consent` and `mcp.oauth.refresh` rows. Refused TLS / invalid state errors emit `mcp.oauth.deny` with reason.

### DEC-016 — Compliance export from audit chain is chain-verify-non-bypassable (PR #74, T5)

**Statement:** A new `xiaoguai audit export --framework {soc2|gdpr|hipaa} …` CLI subcommand and matching `POST /v1/audit/exports` HTTP endpoint produce per-framework audit-report bundles (SOC2 CC7.2, GDPR Art. 30, HIPAA §164.312). Each bundle's header carries a `ChainProof { first_id, last_id, count, start_prev_hmac_hex, end_hmac_hex }`. **Chain verification runs before every export** — there is no `--skip-verify` flag, no override env var, no admin bypass. If `verify_chain` finds a break, the export returns `ChainBroken { first_broken_id, first_broken_ts }` and exits non-zero. JSON is the canonical format; CSV is a deterministic projection with identical row counts and semantics. PDF is stubbed (`PdfUnimplemented` → HTTP 501) — deferred to a separate follow-up PR.

**Rationale:** The audit chain's whole point is tamper-evidence. An export path that lets you skip verification would make the chain compliance-useless. The trade-off (no way to extract data when chain is broken) is intentional — the answer to a broken chain is to investigate, not to ship a tainted report. Auditors get the `ChainProof` header as evidence the export reflects an intact chain at the verified time window.

**Refines:** REQ-AUDIT-EXPORT-001, ADR-0009 (HMAC chain integrity), PHILO §17 (anti-pattern "trusting tool output as user input" — we don't trust a broken chain to be reported as fine). R.E.S.T Traceability dominant.

**Architecture:** New trait `AuditChainExporter` in `xiaoguai-api`; impl `PgAuditAdapter` in `xiaoguai-core`. Mirrors the existing `AuditReader` / `AuditVerifier` bridge pattern so the api crate never sees the signing key.

### DEC-017 — Polyglot sandbox tier is separate crates per language, shared L1 contract (PR #75, T6)

**Statement:** `execute_javascript` ships as a **new sibling crate** `xiaoguai-mcp-exec-js`, not as a `--runtime js` flag on `xiaoguai-mcp-exec`. Both crates implement the same L1 contract per PHILO §14: `ulimit -v` memory cap + tokio deadline timeout + env-scrubbed subprocess (`PATH`, `LANG`, `LC_ALL`, `LC_CTYPE` only) + fresh tempdir CWD + 64 KB output cap + stderr PII redaction via `xiaoguai-types::redact_str`. Default JS runtime is **Deno** (with `--allow-none` for runtime-enforced sandboxing); `--runtime node` is supported as an operator opt-in. Default memory raised to 1024 MB (V8 needs more headroom than CPython's default 256 MB).

**Rationale:** **Trust boundary physical separation.** A Python sandbox escape and a JavaScript sandbox escape have different attack surfaces (V8 JIT vs CPython interpreter vulnerability classes). Sharing the binary means a single CVE in either path compromises both. Separate crates → separate binaries → separate process spaces → composable failures stay isolated. The HotL bucket is also separate (`tool_call.execute_javascript`) so operators decide per-language whether to grant agents access.

Why **Deno over Node**: `--allow-none` makes network/filesystem capability denial enforced by the runtime itself, not by our wrapper. Operator-auditable (`--allow-net` etc. are explicit grants). Rejected `boa_engine` (embedded JS engine) as a third option because it's not feature-complete enough for real agent workloads. Documented in `docs/designs/tier2-mcp-exec-js.md`.

**Refines:** REQ-TOOL-EXEC-002 (Hermes parity), PHILO §14 (sandbox tiering), PHILO §17 (anti-pattern "one giant Agent that does everything" — the same logic applies to sandbox binaries). R.E.S.T Efficiency (small snippet sizes for JSON/DOM/regex) + Security (trust isolation).

**Future-compatible:** ADR-0020 (L3 sandbox feasibility) decided wasmtime+pyodide as the L3 path. The L1 crates' module structure (`exec.rs` / `tools.rs` / `server.rs`) is preserved so they can grow a `runtime: Option<Box<dyn ExecBackend>>` field where L1 stays the default and L3 (`xiaoguai-mcp-exec-wasm`) becomes an opt-in via config.

### DEC-018 — L3 sandbox path is wasmtime+pyodide, deferred 2–3 sprints (ADR-0020)

**Statement:** When the L3 sandbox tier from PHILO §14 lands, it will be implemented via **wasmtime + pyodide** (Python) and **wasmtime + QuickJS-WASM** (JavaScript), not Firecracker micro-VMs and not gVisor. Implementation is **deferred** to a follow-up sprint (~3 weeks calendar across 2–3 sprints per ADR-0020 §"Implementation phasing"). This decision locks in the design commitment so the L1 crates (`xiaoguai-mcp-exec`, `xiaoguai-mcp-exec-js`) can extract an `ExecBackend` trait in their next refactor without ambiguity about the future L3 shape.

**Rationale (from ADR-0020):** wasmtime+pyodide wins on (a) cross-platform dev parity — works on macOS, which Firecracker doesn't and gVisor doesn't; (b) Rust-native — already in our build chain; (c) capability model fits PHILO §15 policy gateway exactly (no default Linux ABI to over-grant); (d) cold-start cache via `Engine::precompile_module` brings per-call latency to ~10 ms; (e) polyglot story is trivial (each language is a WASM module).

**Refines:** PHILO §14 (sandbox tiering), DEC-017 (polyglot sandbox tier). R.E.S.T Security dominant.

**Trade accepted:** pyodide CPython is 2–4× slower than native on CPU-heavy work; pyodide stdlib gaps (e.g., `subprocess`, `socket`) require explicit error messages when forbidden imports are used. Both documented in the future LLD when the implementation lands.

### DEC-019 — `ExecBackend` trait extraction enables L1↔L3 swap per tenant (sprint-8 prereq)

**Statement:** Both L1 crates (`xiaoguai-mcp-exec`, `xiaoguai-mcp-exec-js`) extract their subprocess wrapper into a shared `ExecBackend` trait. The trait surface is:

```rust
#[async_trait]
pub trait ExecBackend: Send + Sync {
    fn name(&self) -> &'static str;        // "process-l1" | "wasmtime-l3"
    async fn run(&self, snippet: &str, cfg: &ExecConfig) -> ExecResult;
    fn capability_summary(&self) -> CapabilitySummary;  // for runbook + audit
}
```

L1 implementations (`ProcessL1Python`, `ProcessL1JavaScript`) become the default. Per-tenant config `agent.sandbox_tier = "L1" | "L3"` (defaults to `L1`) selects the backend at MCP-tool dispatch time. The selector lives in `xiaoguai-mcp-exec::server::ExecServer::new` — config drives which backend impl is constructed; agent code never sees the tier.

**Rationale:** Without trait extraction, an L3 implementation forks the binary at compile time, which makes it impossible for a single deployment to serve some tenants L1 and others L3. The trait is the architectural commitment that **tier selection is a runtime per-tenant decision, not a build-time switch**. ADR-0020 §"Implementation phasing" §1 commits to this extraction as Phase 1.

**Refines:** DEC-017 (polyglot sandbox tier — same logic applies to tier switching), DEC-018 (L3 path).

**Migration safety:** L1 stays the default. Operators opt into L3 per tenant via `tenant_settings.sandbox_tier = 'L3'`. No DB migration needed beyond a new column on `tenant_settings`.

### DEC-020 — L3 sandbox implementation (`xiaoguai-mcp-exec-wasm`) is one crate hosting two language modules

**Statement:** L3 ships as **a single new crate** `xiaoguai-mcp-exec-wasm` exposing:
- `WasmtimePythonBackend` — wraps `wasmtime::Engine` + cached pyodide module
- `WasmtimeJavaScriptBackend` — wraps `wasmtime::Engine` + cached QuickJS-WASM module

Both implement `ExecBackend` (from DEC-019). They share a single `wasmtime::Engine` instance per process (the engine is heavy; instances are cheap). Cold-start cache uses `Engine::precompile_module` at process start; per-call instantiation is ~10 ms.

The crate ships as two separate binaries (`xiaoguai-mcp-exec-wasm-py`, `xiaoguai-mcp-exec-wasm-js`) so operators can register them independently in the MCP supervisor — even though they share the engine, the operational surface stays two-MCP-servers.

**Rationale:** Sharing the wasmtime engine cuts memory by ~50 % vs two separate processes (each would carry its own engine). The dual-binary surface preserves the per-language HotL bucket model from DEC-017. **Trust boundary is now at the WASM module boundary**, not the process boundary, but that's strictly stronger than L1's process boundary (WASM is capability-confined per module).

**Refines:** DEC-018 (L3 path), DEC-019 (trait extraction), PHILO §14 (sandbox tiering).

**Metrics:** `xiaoguai_wasm_cold_start_seconds`, `xiaoguai_wasm_engine_warm_cache_hits_total`, `xiaoguai_wasm_module_load_failed_total`.

### DEC-021 — Multi-agent topology is planner-worker-critic triangle with shared memory view

**Statement:** The orchestrator's current "spawn N agent peers, aggregate results" model evolves into an explicit **three-role triangle**:

| Role | Responsibility | LLM call shape |
|---|---|---|
| **Planner** | Decompose user goal into a JSON plan of sub-tasks; assign each to one Worker | One LLM call per re-plan; small context (goal + tool list + persona) |
| **Worker** | Execute one sub-task as a full ReAct loop; report `WorkerResult { artefact, citations, confidence, cost }` | N LLM calls per sub-task (the ReAct loop) |
| **Critic** | Review every Worker result; either approve, request revision, or reject with reason | One LLM call per Worker result; small context (result + acceptance criteria) |

**Shared memory view**: all three roles read the same per-session memory snapshot via a new `MemoryView` trait. Workers write to a private scratchpad that the Critic can read; only Critic-approved artefacts get promoted to the session-level memory. This prevents one Worker from contaminating another Worker's context.

**Cost budget**: parent budget splits 50 / 40 / 10 (Worker / Planner / Critic) by default; operators tune per-persona. Critic budget being smallest is by design — it makes accept/reject decisions, not artefacts.

**Rationale:** The current orchestrator is "M sub-agents do the same kind of work in parallel". Real multi-agent workflows (per Hermes, per the OpenAI Practices for Governing Agentic Systems paper) are heterogeneous roles with **separation of concerns matching the §5 decision-execution principle**. The planner-worker-critic triangle is the smallest non-trivial topology that captures this; larger topologies (planner-orchestrator-N-workers-critic-evaluator) are a follow-up.

**Refines:** DEC-011 (orchestrator is multi-agent dispatch — refined into role-specialised dispatch), PHILO §5 (Decision ↔ Execution separation), PHILO §13 (Plan-and-Execute mode), R.E.S.T Reliability (Critic catches Worker errors before they propagate).

**Metrics:** `xiaoguai_orchestrator_role_calls_total{role}`, `xiaoguai_orchestrator_critic_rejection_rate`, `xiaoguai_orchestrator_replan_count_total`.

### DEC-022 — SLO model and 4 golden signals are first-class contract obligations

**Statement:** Each user-facing capability has a published **Service Level Objective** with the four Google SRE golden signals:

| Signal | Definition | Default SLO |
|---|---|---|
| **Latency** | p95 of `request_duration_seconds` | `/v1/chat/*` p95 < 5 s; `/v1/sessions/*/messages` p95 first-token < 2 s |
| **Traffic** | requests/sec | per-tenant rate limits at `/etc/xiaoguai/config.yaml::rate_limits` |
| **Errors** | non-2xx rate | < 1 % rolling 1h |
| **Saturation** | resource consumption vs cap | `LLM tokens consumed / tenant_daily_budget < 0.8` |

SLOs are **declarations**, not configurations — they live in `docs/runbooks/slo.md` and are referenced by alert rules. When an SLO is breached, the operator's runbook entry has the specific page chain.

Burn-rate alerts (per SRE workbook): fast-burn (1-hour window) and slow-burn (6-hour window) for each signal. Alert routing via the existing `xiaoguai-watch` infrastructure — no new alerting plumbing.

**Rationale:** R.E.S.T §2 names Traceability + Reliability + Efficiency + Security as the four goals; the four golden signals are the **operationalisation** of the first three. Without published SLOs, "is this deployment healthy?" is a judgement call rather than a measurable state. This DEC makes the answer mechanical: meet the SLO or page.

**Refines:** PHILO §16 (metrics taxonomy — adds the SLO/burn-rate framing on top), R.E.S.T §2.

**Adds:** `docs/runbooks/slo.md` (this DEC's deliverable), Grafana dashboard `slo-overview.json`, Alertmanager rules for fast-burn + slow-burn.

### DEC-023 — Tier-2/3 follow-up hardening: refresh-token AES-GCM, PDF backend, agent-loop wiring (sprint-8 ⇄ 9 catch-up)

**Statement:** Four debt items from sprint-7 ship in a single "follow-up hardening" track:

1. **T4 refresh-token encryption at rest** — AES-GCM with a 32-byte key handle stored in the operator's env (`XIAOGUAI_MCP_OAUTH_TOKEN_KEY`). New `crates/xiaoguai-mcp/src/auth/at_rest.rs` module. Key-rotation via dual-key window (mirrors the existing audit-key rotation pattern from DEC-001's `XIAOGUAI_AUDIT_SIGNING_KEY`). RLS already handles tenant isolation; encryption-at-rest handles disk-snapshot threats.
2. **T5 PDF rendering backend** — replaces the `PdfUnimplemented` stub with `typst`-based rendering (Rust-native, no JVM). Template files in `crates/xiaoguai-audit/templates/{soc2,gdpr,hipaa}.typ`. PDF output is deterministic for a given input bundle (auditors can verify byte-for-byte reproducibility).
3. **T3 production wiring** — implements `PgSkillProposalRepository`, `PgTenantSettings`, and bridges in `xiaoguai-core::skill_author_bridge` so `propose_skill` works against a real Postgres without test stubs.
4. **T6 agent-loop integration test** — mirrors PR #66 (compaction): an end-to-end test that boots `xiaoguai serve`, registers `xiaoguai-mcp-exec-js`, drives a chat that invokes `execute_javascript`, asserts the HotL counter bump and the audit row. Cements the operator workflow.

**Rationale:** Each item is small individually but together they finish the v1.5.x line. Splitting across PRs avoids one mega-PR while keeping the scope coherent under one sprint heading.

**Refines:** DEC-014 (T3 production wiring), DEC-015 (T4 refresh-token), DEC-016 (T5 PDF), DEC-017 (T6 wiring).

### DEC-024 — MiniMax LLM provider as dedicated backend, NOT via OpenAiCompatBackend

**Statement:** Add `MinimaxBackend` as a new sibling of `GroqBackend` / `MistralBackend` in `crates/xiaoguai-llm/`, **not** via the existing `OpenAiCompatBackend` configured with a custom `base_url`. The backend targets the OpenAI-compatible endpoint `https://api.minimax.io/v1` (which MiniMax stably maintains), supports the `M1` / `M2.x` / `abab` model families, and **passes through `reasoning_content`** from M1-and-up thinking-mode responses without flattening it into `content`.

```rust
pub struct MinimaxBackend {
    api_key: String,
    base_url: String,           // default "https://api.minimax.io/v1"
    http: reqwest::Client,
}

impl MinimaxBackend {
    pub fn new(api_key: impl Into<String>) -> Self;
    pub fn with_base_url(self, url: impl Into<String>) -> Self;
}

impl LlmBackend for MinimaxBackend {
    async fn chat_stream(&self, req: ChatRequest) -> Result<ChatStream, LlmError>;
    fn name(&self) -> &'static str { "minimax" }
}
```

Seed migration `0023_minimax_provider_seed.sql` adds one row to `llm_providers` with `kind='minimax'`, `enabled=false` (operator opt-in), `models=['MiniMax-M1', 'MiniMax-M2', 'MiniMax-M2.5', 'MiniMax-M2.7', 'abab6.5-chat']`.

**Rationale (3 reasons it's NOT a thin OpenAiCompat wrapper):**

1. **`reasoning_content` field** on the response delta — present on M1/M2/M2.5/M2.7 streaming chunks for thinking-mode runs. `OpenAiCompatBackend` ignores this field. We want to expose it to downstream consumers (the agent loop, OTel spans, audit). A new field on `ChatChunk { reasoning_delta: Option<String> }` lands in `xiaoguai-types`, and `MinimaxBackend` is the first writer.
2. **Model alias mapping is provider-specific.** MiniMax names models with a `-highspeed` suffix variant (`MiniMax-M2.7-highspeed` vs `MiniMax-M2.7`). The user-facing alias resolution belongs in the provider, not in `OpenAiCompatBackend`'s generic path.
3. **MiniMax publishes an Anthropic-compatible endpoint** as a recommended alternative. Today we use OpenAI-compatible (more stable wire format) but the dedicated crate gives us the option to migrate without disturbing existing OpenAI-compat users.

**Rejected alternative — `OpenAiCompatBackend` with `base_url=https://api.minimax.io/v1`:** zero new code, but drops `reasoning_content` and forces operators to repeat model lists in config. Acceptable as a fallback if `MinimaxBackend` is unavailable, documented in the runbook.

**Refines:** the existing 8-backend `LlmRouter` family (now 9: ollama, openai_compat, anthropic, gemini, bedrock, azure_openai, mistral, groq, **minimax**); R.E.S.T Efficiency (per-tenant provider routing); §16 metrics gets `xiaoguai_llm_reasoning_tokens_total{provider,model}` for cost attribution of thinking-mode traffic.

**Sprint placement:** lands in sprint-8 hardening track as task S8-10 (alongside DEC-023 follow-ups). Not blocking on the L3 pipeline.

### DEC-025 — Frontend is a pnpm monorepo of two independent SPAs sharing one type-safe client

**Statement:** `frontend/` is a **pnpm workspace** containing four packages:

| Package | Role | Tech |
|---|---|---|
| `@xiaoguai/admin-ui` | Operator SPA (audit / scheduler / eval / HotL policies / kanban / personas / skill-proposals / …) | React 18 + Vite + TS + recharts + i18next |
| `@xiaoguai/chat-ui` | End-user SPA (chat surface, SSE streaming, HotL pending, watch indicator, AI disclosure) | React 18 + Vite + TS + react-markdown + i18next |
| `@xiaoguai/shared` | Type-safe `XiaoguaiClient` mirroring `xiaoguai-types` wire DTOs; SSE consumer; ApiError discriminated union | TS only, no UI framework |
| `@xiaoguai/e2e` | Playwright multi-browser test harness | Playwright 1.48 |

Five sub-decisions that follow from the dual-SPA split:

1. **Two SPAs, not one shell.** Admin-ui and chat-ui are independent build artefacts on separate dev ports (`:5174` admin, `:5173` chat). Rationale: trust boundary — end users (chat-ui) and operators (admin-ui) never share a frontend security context; an XSS in one cannot pivot into the other.
2. **One client, one type system.** `@xiaoguai/shared::XiaoguaiClient` is the **only** HTTP layer; both SPAs import it. No second SDK, no per-pane fetch. The wire types are TypeScript mirrors of `xiaoguai-types` Rust DTOs — a contract change in Rust forces a TypeScript change.
3. **No login page in either SPA.** Authentication is delegated to the surrounding reverse proxy / OIDC (Kong, nginx with `oauth2-proxy`, corporate SSO). The SPA receives a bearer token via injected `Authorization` header and reads it from `VITE_API_URL` env config. On 401 → redirect to `VITE_LOGIN_URL`. Rationale: air-gapped enterprise deployments already standardise on SSO at the proxy; one OIDC integration in the proxy is safer than two OIDC integrations in the SPAs.
4. **State is pane-local + URL.** No Redux/Zustand/global store. Each pane uses `useState`/`useReducer`; cross-pane state (tenant context) is the URL. Rationale: 16-pane SPA; global store invites cross-feature coupling and complicates the trust boundary between admin and chat (a global store would tempt sharing).
5. **RBAC scope hints are UX-only.** `<RequireScope name="…">` hides actions when the operator lacks the scope, but fails open if `/v1/admin/me/scopes` is unavailable (older backend). Enforcement remains in backend Casbin (PHILO §17 anti-pattern: never trust a frontend gate for security).

**Rationale:** The frontend has been growing organically for months under feature PRs (chat-ui Gemini-style welcome, admin-ui Kanban, anomaly dashboard, HotL editor) without an architectural anchor. DEC-025 retro-actively encodes the choices that have been working, and declares the trust boundary between the two SPAs explicit. The split-vs-merge question is decided in favour of split because the threat model — operator vs end user vs malicious tool output — is fundamentally different in each surface.

**Refines:** R.E.S.T Security (operator/end-user surface separation), R.E.S.T Reliability (composer-doesn't-block, partial-bubble-preservation in chat-ui), DEC-006 (HotL gate has a first-class UI surface, not an admin afterthought), DEC-014 (skill proposals get an approval pane).

**Adds:** `lld/lld-admin-ui.md`, `lld/lld-chat-ui.md`, sprint-10b task line in roadmap §4, PRD §4.14 UI requirements, test-spec §3.14 UI golden-path cases.

### DEC-026 — DecisionRegistry is PG-backed; in-process waiters replay on boot (sprint-13)

**Statement:** The `DecisionRegistry` introduced in sprint-12 (DEC-006 follow-up, see [`lld-agent.md`](lld/lld-agent.md) §4.5) graduates from an in-memory `DashMap<Uuid, oneshot::Sender<_>>` to a durable layer backed by the `hotl_pending` table (extended in DEC-029). Verdicts continue to flow through the in-process oneshot when a waiter exists; the new property is that **pending escalations survive `xiaoguai-api` restarts**. On boot, `xiaoguai-core::run_serve` scans `hotl_pending WHERE status='pending' AND expires_at > now()`, re-mints in-process `HotlSuspensionTicket` senders for each row, and re-arms the per-row `tokio::time::sleep_until(expires_at)` timeout future. The in-process registry stays the source of truth for **live oneshot delivery**; Postgres is the source of truth for **which escalations exist** and their lifecycle state. The chat session itself does not need to be resumed — sessions are stateless across requests; on the next `POST /v1/sessions/{id}/messages` the loop reads back its history (which already includes the prior `hotl_pending` event) and the operator decision arrives via the replayed waiter exactly as if the API had never restarted.

**Rationale:** The sprint-12 MVP loses every live waiter on `xiaoguai-api` restart, which forces the next scheduler tick to synthesise `verdict=timeout` even when the operator approved seconds earlier. For production tenants that already trust HotL for cost-bounded autonomy, that failure mode is unacceptable — restarts (rolling deploys, k8s pod recycling, OS patching) happen routinely. Choosing **PG row + in-process waiter** over **Redis pub/sub + ephemeral waiter** keeps the single-binary air-gap deployment (DEC-002) intact and reuses RLS for tenant isolation; the Valkey/Redis backend stays opt-in for cache, not required for HotL. Sub-second oneshot delivery preserves operator UX; the durable layer is only consulted on boot and at decision-write time.

**Refines:** DEC-006 (HotL allow-then-escalate — adds restart durability), DEC-008 (RLS double-layer — extends to escalations), R.E.S.T Reliability dominant + Traceability (durable lifecycle audit trail).

**Metrics:** `xiaoguai_hotl_registry_replayed_total{outcome}` (on boot: how many pending rows reattached, expired-on-replay, or were marked dropped), `xiaoguai_hotl_registry_size_gauge` (live waiter count), `xiaoguai_hotl_decision_replay_latency_seconds` histogram.

### DEC-027 — HotL args redaction is policy-driven, applied before SSE emission (sprint-13)

**Statement:** `HotlPending.args_redacted` stops being a pass-through of the raw tool-call arguments. A new per-tenant **redaction policy** declares JSONPath-style field selectors whose values are masked to the fixed sentinel `"***"` before the gate emits `AgentEvent::HotlPending`. Policy lives in a new table `hotl_redaction_policies (tenant_id, scope, jsonpath, applies_to)` with RLS, and is loaded into `xiaoguai-auth::redaction::RedactionRules` at request time. The policy DSL ships in **`xiaoguai-auth`** (not a new crate) because the same crate already owns Casbin policy parsing — colocating the two policy languages keeps the operator surface coherent and avoids a third crate dependency edge for `xiaoguai-agent` and `xiaoguai-api` (both already depend on `xiaoguai-auth`). Selectors are matched against the tool-call argument JSON tree; matches are replaced in-place. Redaction is mandatory on the emission path — there is no opt-out flag — and an empty policy degrades to "emit args verbatim" with a boot warning so tenants notice they have no rules configured.

**Rationale:** A passwords-in-clear example tool call shown to a tenant operator in the chat-ui banner is an exfiltration vector that defeats DEC-004's PII redaction (audit) and DEC-009's telemetry redaction (logs) — both backstops sit downstream of the operator's eyes. Driving redaction from policy rather than hard-coded patterns lets each tenant declare what counts as sensitive in their domain (a fintech tenant masks `account_number`; a healthcare tenant masks `mrn`). Reusing `xiaoguai-auth` over creating a new `xiaoguai-redaction` crate avoids cross-crate plumbing for a feature that is fundamentally policy evaluation — the same logic that gates `who can decide` should gate `what they see when they decide`.

**Deferred decision:** the alternative "spawn `xiaoguai-redaction` as a sibling crate of `xiaoguai-auth`" was considered. Rejected for sprint-13; reconsider only if the policy language grows past Casbin's expressive frame (e.g., regex selectors plus tree rewrites). Documented here so the next reviewer does not relitigate.

**Refines:** DEC-006 (HotL — completes the operator-visibility surface), DEC-004 (audit redaction — same redact-before-emit philosophy), R.E.S.T Security dominant.

**Audit:** Every `HotlPending` audit row carries a `redaction_policy_id` foreign key so post-hoc compliance can prove which policy was applied at decision time, even if the policy is later updated.

### DEC-028 — HotL timeouts are per-scope; global `default_expiry` is the fallback (sprint-13)

**Statement:** The single `agent.hotl.default_expiry` Duration is split into a per-scope namespace under `agent.hotl.expiry.<scope_class>` in `config.yaml` (and the corresponding `XIAOGUAI_AGENT__HOTL__EXPIRY__<SCOPE>` env vars):

```yaml
agent:
  hotl:
    default_expiry: 24h        # fallback when no per-scope key matches
    expiry:
      tool: 24h                # tool_call.* scopes
      mcp: 4h                  # mcp.* scopes (network egress; tighter)
      skill: 72h               # skill_author scope (admin approval; longer)
```

Lookup order at `HotlEnforcer::escalate(scope, ...)`: split `scope` on `.`, take the first component, look up `expiry.<first_component>`; on miss, fall back to `default_expiry`. The fallback path is the only backward-compat surface — tenants who have not set per-scope keys keep the v1.9.x behaviour byte-identically.

**Rationale:** Different scopes have fundamentally different operator response cadences. A `tool_call.execute_python` escalation is acute (operator wants to decide within minutes); a `skill_author.propose_skill` (DEC-014) is asynchronous (operator may batch-approve overnight); an `mcp.oauth.consent` (DEC-015) is even slower (waiting on a human to click consent in another tab). Forcing all three onto the same 24h window either denies-by-timeout on the slow ones or makes the fast ones loiter in the queue. Per-scope timeouts are also a precondition for tightening the `mcp.*` window to hours, which limits the blast radius of a stuck network-egress escalation.

**Refines:** DEC-006 (HotL — operationalises the timeout dimension), DEC-026 (registry persistence — sleep_until is now per-row not per-class), R.E.S.T Reliability.

**Metrics:** `xiaoguai_hotl_timeout_total{scope_class}` (broken out so saturation in one class does not mask in another), `xiaoguai_hotl_pending_age_seconds{scope_class}` histogram.

### DEC-029 — `escalation_id` is the canonical identifier; `hotl_escalations` parent table splits the schema (sprint-13)

**Statement:** Three coupled cleanups land as one decision because they share the same migration:

1. **Naming.** The SSE contract, decision endpoint, and persistence layer all use **`escalation_id`** as the canonical identifier. The sprint-12 `#[serde(alias = "escalation_id")]` shim on `request_id` is removed. Frontend (`chat-ui` HotlBanner), backend (`AgentEvent::HotlPending`, `DecisionRegistry`, `POST /v1/hotl/decisions`), audit rows, and metrics labels all rename in lockstep. Old clients sending `request_id` get `400 Bad Request` with a discriminator-error pointing at the field name — there is no transparent compatibility path.

2. **Casbin scope.** `POST /v1/hotl/decisions` requires the Casbin **scope `hotl:decide`** (not a path-based fallback rule). The scope is added to `tenant_admin` and `operator` roles in migration 0027; ordinary chat users receive `403 Forbidden`. The sprint-11 path-based rule (`p, *, /v1/hotl/decisions, POST`) is removed in the same migration.

3. **Schema split.** Migration **0027** introduces `hotl_escalations` as the parent table — one row per top-level invocation (a single agent loop iteration that triggered HotL) — and refactors the sprint-11 `hotl_pending` into the child table with a `escalation_id` foreign key. A multi-gate iteration (one tool call gated under `tool_call.execute_python` and again under a nested `mcp.oauth.consent` from inside that call) now produces one `hotl_escalations` row plus N `hotl_pending` children, instead of N orphaned `hotl_pending` rows that all share a session id and have no traceable parent. The migration backfills existing `hotl_pending` rows into the new parent table 1-to-1 (each existing row gets its own parent), preserving lifecycle history without forensic loss.

**Rationale:** The alias shim was a sprint-12 expediency; it leaves a two-name surface for an identifier that the contract calls one thing in prose. A clean rename is a small, contained breaking change at a moment where the SSE contract still has very few external integrators (chat-ui is the only consumer in tree). Path-based Casbin rules are pre-RBAC-redesign sediment — every other write-endpoint already uses scope rules, and leaving `POST /v1/hotl/decisions` on the path-based path is exactly the kind of inconsistency that produces the wrong-default-policy bug later. The single-table schema modelled one row per `(tenant, scope)` decision; the production cases for nested gating (orchestrator + per-peer HotL, DEC-021 triangle + Worker tool calls) require a parent dimension that the schema does not currently express. Adding it now is cheaper than retrofitting later through every aggregation query.

**Refines:** DEC-006 (HotL — finishes the contract cleanups outstanding from sprint-12), DEC-008 (RLS double-layer — extends to `hotl_escalations`), DEC-021 (triangle topology — nested gating now has a parent record), R.E.S.T Security (scope rule) + Traceability (parent table) + Reliability (single-name surface eliminates one class of bug).

**Migration risk:** Migration 0027 is the largest of the four sprint-13 items (parent-table backfill plus column rename across two tables). It is forward-only per GR-DB-02; pre-flight runs in `migrations-smoke` against a snapshot containing live `hotl_pending` rows. Rollback is via a forward-only `0028_revert` if the snapshot test fails — not by reversing 0027.

---

## 4. Runtime model

### 4.1 Request lifecycle (single-agent chat)

```
[chat-ui] --POST /v1/sessions/{id}/messages--> [xiaoguai-api]
                                                  │
                                                  ├─ Auth: verify JWT → tenant_id, user_id, roles
                                                  ├─ Casbin: check (role, action, resource)
                                                  ├─ Set local SQL: app.current_tenant_id
                                                  ├─ Persist user message row (messages)
                                                  ├─ Audit: session.message.create
                                                  │
                                                  ▼
                                            [xiaoguai-agent: ReAct loop]
                                                  │
                                                  ├─ PreProcessor:
                                                  │     • token estimate (xiaoguai-llm::token_count)
                                                  │     • if est > trigger_threshold → history::compact
                                                  │            (LLM summary; slide on fallback)
                                                  │     • else                       → history::slide
                                                  │     • sys prompt inject
                                                  ├─ LLM call (stream) via xiaoguai-llm router
                                                  ├─ if tool_calls:
                                                  │     for each call:
                                                  │       HotlGate.check(scope, amount)
                                                  │       if allowed: dispatch to MCP client (tokio::join_all)
                                                  │       else if escalated: synth ToolResult; route to escalate_to
                                                  │       else if policy_unavailable: synth ToolResult; deny
                                                  ├─ Audit each tool call
                                                  ├─ Loop until max_iterations or no tool_calls
                                                  │
                                                  ▼
                                            [PostProcessor]
                                                  │
                                                  ├─ token_usage row (per LLM stream segment)
                                                  ├─ persist assistant message
                                                  ├─ audit: session.message.complete
                                                  │
                                                  ▼
                                            SSE: final event → client
```

### 4.2 Multi-agent flow (orchestrator)

```
                  ┌─[Persona A: planner]──┐
[orchestrator] ─→ ├─[Persona B: researcher]├─→ aggregate → final
                  └─[Persona C: critic]────┘
```

Each peer runs an independent ReAct loop with its own HotL budget scope. The orchestrator passes intermediate results between peers and stops when consensus / max-rounds is reached.

### 4.3 Cancellation registry

`xiaoguai-runtime` exposes a `CancellationRegistry` keyed by `session_id`. `POST /v1/sessions/{id}/cancel` writes a cancel token; the ReAct loop polls the token between iterations and at tool-call boundaries. Bounded latency is the slowest in-flight tool call.

---

## 5. Data model

20 sqlx migrations under `crates/xiaoguai-storage/migrations/`. Highlights:

| Migration | Key tables | Indexes / RLS |
|---|---|---|
| 0001 | `tenants`, `users`, `user_roles`, `sessions`, `messages` | RLS on all five |
| 0002 | `audit_log` | append-only; `hmac` not null; partial index on `(tenant_id, sequence)` |
| 0003 | `llm_providers` | RLS; index on `(tenant_id, kind)` |
| 0004 | `token_usage` | RLS; partition-by-month candidate |
| 0005 | `mcp_servers` | RLS |
| 0006 | `im_identities` | RLS; unique `(platform, platform_user_id)` |
| 0007 | `scheduled_jobs` | RLS |
| 0008 | `sessions.parent_session_id` | self-FK |
| 0009 | `scheduler_webhook_tokens` | RLS; hashed token |
| 0010 | `llm_providers.cost_per_1k_*` | (alter) |
| 0011 | `hotl_policies` | RLS |
| 0012 | `outcomes` | RLS; daily bucket index `(tenant_id, recorded_at::date)` |
| 0013 | `audit_export_state` | export checkpoint |
| 0014 | `tenant_rate_limit_config` | RLS |
| 0015 | `skill_packs`, `pack_operators` | RLS; signature column |
| 0016 | `personas` | RLS |
| 0017 | `workspaces` | RLS |
| 0018 | `boards`, `tasks`, `task_state_log` | RLS |
| 0019 | `memories`, `recall_traces` | RLS; pgvector HNSW on `content_embedding vector(384)` cosine |
| 0020 | seed `ollama-local` provider | data-only |

RLS plumbing is centralised in `xiaoguai-storage`. Repositories must call `Storage::with_tenant(tenant_id)` to set the `SET LOCAL` for the transaction.

---

## 6. Cross-cutting concerns

### 6.1 Auth

- OIDC token verification: `xiaoguai-auth::OidcVerifier` caches JWKs with `Cache-Control` honoured.
- IM-based OAuth (Feishu / DingTalk / WeCom) is wrapped as an OIDC-adapter trait so the rest of the system treats them identically.
- Casbin policy reload is on-demand (`POST /v1/admin/casbin/reload`) and on boot.

### 6.2 Audit

- Append-only, never UPDATE / DELETE.
- HMAC key in Kubernetes Secret `xiaoguai-audit`; rotation window 30 days; verifier accepts dual keys during rotation.
- `XIAOGUAI_AUDIT_REDACT_PII=true` by default.

### 6.3 Observability

- Prometheus `/metrics` (local-pull). OTLP traces and metrics exported only when `XIAOGUAI_OTEL__EXPORT_URL` is configured.
- `RedactingSpanExporter` wraps every exporter to strip PII before egress.
- Structured JSON logs via `tracing-subscriber`.

### 6.4 Cache

- DashMap backend by default. `XIAOGUAI_CACHE__URL=redis://…` opts into Valkey/Redis.
- Switching modes requires restart.

### 6.5 Secrets

- Kubernetes Secrets, env vars, or local file. `age`-encrypted-per-tenant secret store is a v2 candidate.
- No secrets are propagated to MCP child processes.

---

## 7. Deployment topologies

### 7.1 Single-tenant air-gap (DEFAULT)

```
[Node]
 ├─ xiaoguai serve         (binary, single process)
 ├─ Postgres 17 + pgvector (container or systemd)
 └─ Ollama                  (container or systemd; serves models locally)

Cache: in-process DashMap (no Valkey)
LLM:   ollama-local enabled; cloud providers seeded disabled
Audit: HMAC key in local Secret/file
```

Helm chart `deploy/helm/xiaoguai` ships a `values.yaml` profile for this shape: no Valkey, no S3, single replica.

### 7.2 Multi-tenant SaaS (opt-in)

```
[Ingress]
 └─ xiaoguai serve × N replicas
       │
       ├─ Postgres primary + read replicas
       ├─ Valkey cluster (cache backend)
       ├─ Ollama / vLLM cluster (local models) or
       │   tenant-supplied cloud provider keys
       └─ Prometheus + OTel collector
```

`values-ha.yaml` shows the multi-replica template. Each tenant retains its own RLS scope, HotL policies, MCP server registry, and provider rows.

### 7.3 Deployment artefacts inventory

| Artefact | Purpose |
|---|---|
| `deploy/Dockerfile` | Multi-stage Rust build → distroless runtime (~ 180 MB) |
| `deploy/docker-compose.yml` | Local dev compose |
| `deploy/docker-compose.ha.yml` | HA reference (3× xiaoguai, 3× PG, pgBouncer, HAProxy) |
| `deploy/helm/xiaoguai/` | Helm chart (12 template files, 9 test files) |
| `deploy/kustomize/` | Base + staging + production overlays |
| `deploy/istio/` | VirtualService / DestinationRule samples |
| `deploy/systemd/xiaoguai-core.service` | systemd unit for bare-metal |
| `deploy/terraform/` | AWS / GCP IaC |
| `deploy/otel-collector-*.yaml` | OTel collector configs (basic + advanced) |
| `deploy/prometheus.yml` | Scrape config |

---

## 8. Failure modes & resilience

| Failure | System response |
|---|---|
| Postgres unreachable | All `/v1/**` return `503 dependency_unavailable`; agent loop fails the in-flight tool call; HotL fail-closed. |
| Cache backend lost (Valkey mode) | Rate-limit windows reset; sessions continue (cache is ephemeral); cluster operator can fail back to in-process by restart. |
| Ollama unreachable | LLM router cascades to next provider in `fallback_order`; if Ollama is the only enabled provider, the request returns `503 dependency_unavailable` with the underlying message. |
| MCP child process hangs | Per-call `timeout_secs` deadline kills it; HotL increment still applied. |
| Audit write fails | Chain protected (no partial write); request returns `500 internal_error`; operator alerted via PagerDuty (Prometheus rule). |
| Bad clock skew | JWT verification fails (`invalid_token`); operator must NTP-sync. |

---

## 9. Risk mapping

| HLD decision | Mitigates | Refines |
|---|---|---|
| DEC-001 unified binary | OPS sprawl | REQ-NFR-006 |
| DEC-002 in-process cache | Operational complexity in air-gap | DEC-DEPLOY-001 |
| DEC-003 boot-time embedder | Configuration drift | REQ-MEM-002 |
| DEC-004 redact-before-sign | RISK-SEC-002 | REQ-AUD-003 |
| DEC-005 sandbox ulimit + tempdir | RISK-SEC-003 | REQ-MCP-004 |
| DEC-006 HotL allow-then-escalate | RISK-COST-001 | REQ-HOTL-002 |
| DEC-007 signed skill packs | RISK-SEC-001 | REQ-SKILL-003 |
| DEC-008 RLS double-layer | RISK-SEC-004 | REQ-AUTH-003 |
| DEC-009 zero default telemetry | RISK-SEC-002 (partial) | REQ-NFR-008 |
| DEC-010 daily-bucketed outcomes | Cross-tenant inference | REQ-OUT-002 |
| DEC-011 orchestrator scope | Q1 ambiguity | REQ-CHAT-004 |
| DEC-012 persona scope | Q2 ambiguity | REQ-CHAT-001 |

---

## 10. PRD coverage summary

| PRD section | Covered by |
|---|---|
| Conversation & agent loop | §4.1 runtime model, DEC-011 |
| LLM routing | LLD `xiaoguai-llm`, fallback chain in §6.4 |
| MCP | LLD `xiaoguai-mcp`, DEC-005 sandbox |
| Memory & RAG | DEC-003, §5 migration 0019 |
| IM gateways | LLD `xiaoguai-im-gateway`, §6.1 OIDC wrappers |
| Auth & multi-tenancy | DEC-008 |
| Audit | DEC-004, §6.2 |
| HotL | DEC-006 |
| Skill packs | DEC-007 |
| Scheduler | §6.3 observability, LLD `xiaoguai-scheduler` (not in this batch) |
| Outcomes | DEC-010 |
| Orchestrator | DEC-011 |
| Personas | DEC-012 |
| NFR | §6, §7, §8 |

---

## 11. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: hld-profile-v1
artifact:
  id: HLD-XIAOGUAI-001
  type: HLD
  title: Xiaoguai high-level design — v1.4 retrofit
  status: draft
  owners: [engineering.xiaoguai, architecture.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-31
  source_documents:
    - PRD-XIAOGUAI-001
    - API-XIAOGUAI-001
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-HLD-001, title: "Single self-contained binary", statement: "xiaoguai serve subsumes legacy xiaoguai-core; IM gateways link into the same binary.", status: approved, scope: in, decision: "One binary; legacy xiaoguai-core kept as shim.", rationale: "Air-gap simplicity; halves failure surface." }
    - { id: DEC-HLD-002, title: "Cache backend selectable at boot", statement: "Cache::connect('') boots in-process DashMap; redis:// switches to Valkey/Redis.", status: approved, scope: in, decision: "Boot-time selection, no live failover.", rationale: "Air-gap mode needs no broker; sub-second TTL precision." }
    - { id: DEC-HLD-003, title: "Memory embedder selectable at boot", statement: "OLLAMA_HOST envar selects OllamaEmbedder or InMemoryEmbedder.", status: approved, scope: in, decision: "Boot-time selection.", rationale: "Decouples integration from production." }
    - { id: DEC-HLD-004, title: "Audit chain redacts PII before signing", statement: "Redaction patterns applied before HMAC.", status: approved, scope: in, decision: "Redact-then-sign.", rationale: "Sign-then-redact would invalidate verification; redact-before-sign keeps chain verifiable." }
    - { id: DEC-HLD-005, title: "Sandbox via ulimit + tokio timeout", statement: "Python sandbox uses ulimit -v, tokio timeout, env scrub, tempdir.", status: approved, scope: in, decision: "ulimit + tokio timeout + tempdir; network egress at deployment layer.", rationale: "x-platform vs Linux-only landlock/seccomp; wasmtime+pyodide deferred." }
    - { id: DEC-HLD-006, title: "HotL allow-then-escalate", statement: "Budget exhaustion escalates; policy outage fails closed.", status: approved, scope: in, decision: "Allow-then-escalate, fail-closed on store outage.", rationale: "Avoids surprise denials; protects against runaway behaviour when policy store is unreachable." }
    - { id: DEC-HLD-007, title: "Local-first signed skill packs", statement: "Packs install from local fs or tenant index; signature required.", status: approved, scope: in, decision: "Signed local packs only; --allow-unsigned operator opt-in.", rationale: "Air-gap-compatible; bounded supply-chain risk." }
    - { id: DEC-HLD-008, title: "RLS double-layer multi-tenancy", statement: "App-layer WHERE + Postgres RLS policy.", status: approved, scope: in, decision: "Both layers mandatory.", rationale: "Defence in depth." }
    - { id: DEC-HLD-009, title: "Zero default outbound telemetry", statement: "No telemetry leaves without operator opt-in.", status: approved, scope: in, decision: "Opt-in OTel + Prometheus pull only.", rationale: "Enterprise privacy posture." }
    - { id: DEC-HLD-010, title: "Outcomes daily-bucketed reads", statement: "Outcome reads aggregated by day, tenant-scoped.", status: approved, scope: in, decision: "Daily bucket + tenant-scoped.", rationale: "Prevent cross-tenant inference." }
    - { id: DEC-HLD-011, title: "Orchestrator is multi-agent dispatch", statement: "xiaoguai-orchestrator coordinates peer agents.", status: approved, scope: in, decision: "Multi-agent dispatch + coordination.", rationale: "Demand observed; not a workflow engine." }
    - { id: DEC-HLD-012, title: "Personas are role presets", statement: "xiaoguai-personas stores presets and persona-scoped memory view.", status: approved, scope: in, decision: "Presets + memory.persona_id column.", rationale: "Matches existing schema and HANDOFF answer." }
    - { id: DEC-HLD-013, title: "DecisionRegistry is PG-backed with boot replay", statement: "hotl_pending rows back the in-memory waiter map; on xiaoguai-api boot, pending+unexpired rows reattach in-process oneshot senders and re-arm per-row sleep_until timeouts.", status: approved, scope: in, decision: "PG row durability + in-process oneshot delivery; no Redis pub/sub.", rationale: "Restarts must not synthesise verdict=timeout when the operator already approved; keeps single-binary air-gap deployment intact." }
    - { id: DEC-HLD-014, title: "HotL args redaction is policy-driven before SSE emission", statement: "Per-tenant hotl_redaction_policies declares JSONPath selectors masked to ***; redaction is mandatory and lives in xiaoguai-auth alongside Casbin.", status: approved, scope: in, decision: "Policy-driven redaction in xiaoguai-auth; no opt-out flag.", rationale: "Operator-visible banners are downstream of audit and telemetry redaction backstops; per-tenant rules let domains declare what is sensitive without hard-coded patterns." }
    - { id: DEC-HLD-015, title: "HotL timeouts are per-scope class with default fallback", statement: "agent.hotl.expiry.<class> map (tool/mcp/skill) overrides agent.hotl.default_expiry; HotlEnforcer.escalate looks up by first scope component.", status: approved, scope: in, decision: "Per-scope class config, fallback to global default.", rationale: "Tool-call escalations are acute (minutes); skill_author is async (overnight); mcp.oauth is even slower. Single window denies-by-timeout the slow ones or loiters the fast ones." }
    - { id: DEC-HLD-016, title: "escalation_id rename + hotl:decide scope + hotl_escalations parent table", statement: "Migration 0027 renames request_id→escalation_id end-to-end, adds Casbin hotl:decide scope rule, and splits hotl_pending into hotl_escalations (parent, one per top-level invocation) + hotl_pending (child).", status: approved, scope: in, decision: "Clean breaking rename + parent/child schema + scope rule.", rationale: "Alias shim leaves a two-name surface; path-based Casbin rules are pre-RBAC sediment; nested gating from DEC-021 needs a parent dimension the single-table schema does not express." }
  flows:
    - { id: FLOW-CHAT-001, title: "Single-agent chat lifecycle", statement: "User message flows through Auth → Casbin → ReAct loop → HotL gate → tool dispatch → SSE final.", status: approved, scope: in, kind: system_flow }
    - { id: FLOW-MULTI-001, title: "Multi-agent orchestration", statement: "Orchestrator dispatches to planner/researcher/critic peers, aggregates results.", status: approved, scope: in, kind: system_flow }
    - { id: FLOW-CANCEL-001, title: "Session cancellation propagation", statement: "Cancel token polled at iteration boundary; bounded by slowest in-flight tool.", status: approved, scope: in, kind: state_transition }
  test_cases: []
relations:
  - { id: REL-HLD-001, type: refines, from: DEC-HLD-001, to: REQ-NFR-006, status: active }
  - { id: REL-HLD-002, type: refines, from: DEC-HLD-002, to: MR-CACHE-001, status: active }
  - { id: REL-HLD-003, type: refines, from: DEC-HLD-003, to: REQ-MEM-002, status: active }
  - { id: REL-HLD-004, type: mitigates, from: DEC-HLD-004, to: RISK-SEC-002, status: active }
  - { id: REL-HLD-005, type: mitigates, from: DEC-HLD-005, to: RISK-SEC-003, status: active }
  - { id: REL-HLD-006, type: mitigates, from: DEC-HLD-006, to: RISK-COST-001, status: active }
  - { id: REL-HLD-007, type: mitigates, from: DEC-HLD-007, to: RISK-SEC-001, status: active }
  - { id: REL-HLD-008, type: mitigates, from: DEC-HLD-008, to: RISK-SEC-004, status: active }
  - { id: REL-HLD-009, type: refines, from: DEC-HLD-011, to: REQ-CHAT-004, status: active }
  - { id: REL-HLD-010, type: refines, from: DEC-HLD-012, to: REQ-MEM-001, status: active }
  - { id: REL-HLD-011, type: refines, from: FLOW-CHAT-001, to: REQ-CHAT-001, status: active }
  - { id: REL-HLD-012, type: refines, from: FLOW-MULTI-001, to: REQ-CHAT-004, status: active }
  - { id: REL-HLD-013, type: refines, from: FLOW-CANCEL-001, to: REQ-CHAT-002, status: active }
  - { id: REL-HLD-014, type: refines, from: DEC-HLD-013, to: DEC-HLD-006, status: active }
  - { id: REL-HLD-015, type: refines, from: DEC-HLD-014, to: DEC-HLD-006, status: active }
  - { id: REL-HLD-016, type: refines, from: DEC-HLD-015, to: DEC-HLD-006, status: active }
  - { id: REL-HLD-017, type: refines, from: DEC-HLD-016, to: DEC-HLD-006, status: active }
  - { id: REL-HLD-018, type: refines, from: DEC-HLD-014, to: DEC-HLD-004, status: active }
  - { id: REL-HLD-019, type: refines, from: DEC-HLD-013, to: DEC-HLD-008, status: active }
  - { id: REL-HLD-020, type: refines, from: DEC-HLD-016, to: DEC-HLD-008, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
