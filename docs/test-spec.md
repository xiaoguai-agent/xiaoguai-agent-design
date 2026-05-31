# Test Spec — Xiaoguai v1.4

| | |
|---|---|
| Document ID | `TSPEC-XIAOGUAI-001` |
| Type | Test specification + case package |
| Status | `draft` |
| Source | [`PRD-XIAOGUAI-001`](prd-xiaoguai.md), [`API-XIAOGUAI-001`](api-contract.md), [`HLD-XIAOGUAI-001`](hld.md), [`TSTRAT-XIAOGUAI-001`](test-strategy.md), LLD set in `lld/` |
| Updated | 2026-05-31 |

> This document is the concrete test-case package for the **independent test scope** defined in the Test Strategy. Cases here are layer L1+ (system integration and above). Unit / code-level integration cases live in each crate's `tests/` directory and are not enumerated here.

---

## 1. Conventions

- IDs follow `CASE-<domain>-NNN`.
- `layer` values: `system_integration`, `e2e`, `regression`, `compatibility`, `non_functional`.
- `priority` values: `P0`/`P1`/`P2`/`P3`.
- `automation` values: `must`, `should`, `manual_ok`.
- All cases declare a `verifies` relation to a PRD `REQ-*`, `BEH-*`, `MR-*`, or `RISK-*`.

---

## 2. Coverage summary

| Class | Total in PRD | Cases authored | Coverage |
|---|---:|---:|---|
| P0 requirements | 33 | 33 | 100 % |
| P1 requirements | 18 | 15 | 83 % |
| P2 requirements | 7 | 3 | 43 % |
| External behaviours | 7 | 7 | 100 % |
| Must-not-regress | 8 | 8 | 100 % |
| Risks | 9 | 9 | 100 % |
| Total | 82 | 75 | 91 % |

Uncovered are P2 cases waived in `TSTRAT-XIAOGUAI-001` §7 (capability suite seeding).

---

## 3. Test matrix (cases by domain)

### 3.1 Conversation & agent loop

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-CHAT-001 | Streaming chat returns delta/tool_call/final sequence | e2e | P0 | REQ-CHAT-001, BEH-CHAT-001 |
| CASE-CHAT-002 | Mid-flight cancel returns Final(cancelled) within slowest-tool deadline | e2e | P0 | REQ-CHAT-002 |
| CASE-CHAT-003 | Fork creates new session with parent_session_id | system_integration | P1 | REQ-CHAT-003 |
| CASE-CHAT-004 | Parallel tool calls dispatch via tokio::join_all and return interleaved | system_integration | P0 | REQ-CHAT-004 |
| CASE-CHAT-005 | Max-iteration cap honoured per tenant override | system_integration | P1 | REQ-CHAT-005 |
| CASE-CHAT-006 | Token-count estimator rounds up + handles CJK + per-message overhead | unit | P0 | DEC-013 |
| CASE-CHAT-007 | `compact()` no-op when tail ≤ keep_recent | unit | P0 | DEC-013 |
| CASE-CHAT-008 | `compact()` replaces head with synthetic summary and preserves recent tail | unit | P0 | DEC-013 |
| CASE-CHAT-009 | `compact()` falls back to `slide` on summariser backend error | unit | P0 | DEC-013 (Reliability) |
| CASE-CHAT-010 | `compact()` preserves assistant-`tool_calls` ↔ `Role::Tool` pairing across the split | unit | P0 | DEC-013, FC lifecycle step 4 |
| CASE-CHAT-011 | 100-turn synthetic conversation with 30 tool-call cycles compacts to ≤ 50 % of `max_context_tokens` | system_integration | P0 | DEC-013, REQ-CHAT-007 |
| CASE-CHAT-012 | Compacted history preserves last 6 messages bit-identically | system_integration | P0 | DEC-013 |
| CASE-CHAT-013 | `xiaoguai_compaction_triggered_total` increments per compaction; `fallback / triggered < 5 %` in healthy runs | system_integration | P1 | DEC-013 (Traceability) |

#### CASE-CHAT-001 — Streaming SSE shape

- **Preconditions:** `xiaoguai serve` booted with Ollama qwen2.5-coder:0.5b; tenant `t-test` with operator JWT.
- **Steps:**
  1. `POST /v1/sessions` → session_id.
  2. `POST /v1/sessions/{id}/messages` with body `{"role":"user","content":"What is 2+2? Use the calculator tool if available."}` accepting `text/event-stream`.
  3. Collect events until `event: final`.
- **Expected evidence:**
  - At least one `event: delta` received before any tool_call.
  - If tool_call present, a matching tool_result with same `id` follows.
  - `event: final` carries `stop_reason in {stop, max_iterations}` and a `message_id`.
- **Automation:** `must`. Stored under `tests/e2e/chat_sse.rs`.

#### CASE-CHAT-002 — Cancel during tool call

- **Preconditions:** Tool fixture `slow_sleep` registered with 10 s sleep; HotL allows it.
- **Steps:**
  1. Start streamed request that the LLM responds to with `slow_sleep(5)`.
  2. While tool_call event is in flight, `POST /v1/sessions/{id}/cancel`.
  3. Observe final event.
- **Expected evidence:** Final event arrives ≤ 6 s after cancel (slowest-tool bounded); `stop_reason=cancelled`; audit row `session.cancel`.
- **Automation:** `must`.

#### CASE-CHAT-004 — Parallel dispatch

- **Preconditions:** Two tool fixtures `delay_a(1s)` and `delay_b(1s)`.
- **Steps:** Force the LLM to emit both calls in one turn (prompt instructs it; fall back to a fixture LLM if non-deterministic).
- **Expected evidence:** Wall-clock between tool_call events and tool_result events ≤ 1.3 s (parallel) rather than ≥ 2 s (serial).
- **Automation:** `must`.

#### CASE-CHAT-011 — 100-turn synthetic conversation compaction (Tier-2d)

- **Preconditions:** `CompactionConfig { max_context_tokens: 8_000, trigger_at_pct: 75, keep_recent: 6, summary_model: "fixed-summary" }`. Mock backend `FixedSummaryBackend` returns a deterministic ~30-word summary on every `chat_stream` call.
- **Steps:**
  1. Build synthetic conversation: 1 system + 70 user/assistant pairs + 30 tool-call cycles (each = user + assistant_tool_calls + Role::Tool + assistant). Total ≈ 261 messages (141 + 120).
  2. Assert `estimate_message_tokens` is > 5 000 before compaction.
  3. Call `compact(messages, &backend, cfg).await`.
  4. Assert `outcome == CompactionOutcome::Compacted`.
  5. Assert `estimate_message_tokens(out) ≤ max_context_tokens / 2`.
  6. Assert the synthetic summary text appears in `out[1].content`.
- **Expected evidence:** All asserts pass. The synthetic system summary preserves concrete facts from the canned summary string (`cluster-id=prod-east-7`, `audit signing key rotation`).
- **Automation:** `must`. Stored under `crates/xiaoguai-agent/tests/compaction_integration.rs`.

#### CASE-CHAT-012 — Recent-turn verbatim preservation

- **Preconditions:** Same as CASE-CHAT-011.
- **Steps:** Call `compact()`; inspect the last message of the result.
- **Expected evidence:** `out.last().content == "Pod pod-29 is healthy."` (bit-identical to the last message of the input). Demonstrates `keep_recent` semantics.
- **Automation:** `must`. Stored in the same test file as CASE-CHAT-011.

### 3.2 LLM routing

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-LLM-001 | Register provider via API | system_integration | P0 | REQ-LLM-001 |
| CASE-LLM-002 | Reject unknown provider kind | system_integration | P0 | REQ-LLM-002 |
| CASE-LLM-003 | Cascade through fallback_order on 5xx | system_integration | P0 | REQ-LLM-003 |
| CASE-LLM-004 | Migration 0020 seeds ollama-local enabled, cloud providers disabled | system_integration | P0 | REQ-LLM-004, RISK-OPS-001 |
| CASE-LLM-005 | token_usage row written per stream | system_integration | P0 | REQ-LLM-005 |

#### CASE-LLM-003 — Fallback chain

- **Preconditions:** Primary provider mocked with `wiremock` returning 503; secondary provider returning a real Ollama-like stream.
- **Steps:** Issue chat → expect success.
- **Expected evidence:** `token_usage` row exists for the secondary provider only; primary attempt logged with `Provider(... , "503 ...")`.
- **Automation:** `must`.

#### CASE-LLM-004 — Air-gap seed verification

- **Steps:**
  1. Fresh DB; apply migrations through 0020.
  2. `SELECT kind, enabled FROM llm_providers ORDER BY kind`.
- **Expected evidence:** `ollama` row `enabled=true`; all other 7 kinds `enabled=false`.
- **Automation:** `must`. Maps directly to `RISK-OPS-001` mitigation.

### 3.3 MCP & sandbox

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-MCP-001 | stdio transport roundtrip | system_integration | P0 | REQ-MCP-002 |
| CASE-MCP-002 | http and sse transports roundtrip | system_integration | P0 | REQ-MCP-002 |
| CASE-MCP-003 | Marketplace install records signed pack | system_integration | P1 | REQ-MCP-003, REQ-SKILL-003 |
| CASE-MCPEXEC-001 | execute_python returns expected output | system_integration | P0 | REQ-MCP-004 |
| CASE-MCPEXEC-002 | Timeout returns timed_out=true, exit_code=null | system_integration | P0 | BEH-MCPEXEC-001 |
| CASE-MCPEXEC-003 | Env scrub hides XIAOGUAI_AUDIT_SIGNING_KEY | system_integration | P0 | BEH-MCPEXEC-002, RISK-SEC-003 |
| CASE-MCPEXEC-004 | Stderr PII redaction matches audit corpus | system_integration | P0 | REQ-AUD-003 (extended) |
| CASE-MCPEXEC-005 | Output cap 64 KiB with truncated=true | system_integration | P0 | REQ-MCP-004 |
| CASE-MCPEXEC-JS-001 | execute_javascript happy path on Deno | system_integration | P0 | DEC-017, REQ-TOOL-EXEC-002 |
| CASE-MCPEXEC-JS-002 | execute_javascript timeout → kill_on_drop SIGKILL | system_integration | P0 | DEC-017 |
| CASE-MCPEXEC-JS-003 | execute_javascript env scrub hides XIAOGUAI_AUDIT_SIGNING_KEY | system_integration | P0 | DEC-017, RISK-SEC-003 |
| CASE-MCPEXEC-JS-004 | execute_javascript runtime not on PATH → SKIPPED (gated test) | unit | P1 | DEC-017 (graceful degrade) |
| CASE-MCPAUTH-001 | OAuth PKCE happy path: verifier matches SHA256(challenge) | system_integration | P0 | DEC-015 |
| CASE-MCPAUTH-002 | Expired access_token triggers refresh before next MCP call | system_integration | P0 | DEC-015 |
| CASE-MCPAUTH-003 | Refresh-token rotation: new refresh_token in response → stored bundle atomic update | system_integration | P0 | DEC-015 |
| CASE-MCPAUTH-004 | refresh_token absent in response → old preserved (RFC 6749 §6) | unit | P0 | DEC-015 |
| CASE-MCPAUTH-005 | TLS verify-off requires XIAOGUAI_MCP_OAUTH_INSECURE=1 env + warn log | system_integration | P1 | DEC-015 |
| CASE-MCPAUTH-006 | CLI redirect listener validates state, rejects mismatch | unit | P0 | DEC-015 |

#### CASE-MCPEXEC-002 — Timeout semantics

- **Preconditions:** `xiaoguai-mcp-exec` server started with `--timeout-secs 2`.
- **Code:** `while True: pass`.
- **Expected:** result JSON has `timed_out=true`, `exit_code=null`, `duration_ms ≥ 1900 and ≤ 3000`.
- **Automation:** `must`.

#### CASE-MCPEXEC-003 — Env scrub

- **Preconditions:** Spawn xiaoguai-mcp-exec with `XIAOGUAI_AUDIT_SIGNING_KEY=secret-key-do-not-leak` in parent env.
- **Code:** `import os; print(os.environ.get('XIAOGUAI_AUDIT_SIGNING_KEY','MISSING'))`.
- **Expected:** `stdout` contains `MISSING`, never `secret-key-do-not-leak`.
- **Automation:** `must`.

### 3.4 Memory

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-MEM-001 | Put then recall returns the row with score | system_integration | P0 | REQ-MEM-001 |
| CASE-MEM-002 | OLLAMA_HOST unset boots InMemoryEmbedder, /v1/memories returns 200 | e2e | P0 | REQ-MEM-002, MR-MEM-001 |
| CASE-MEM-003 | OLLAMA_HOST set boots OllamaEmbedder, recall quality > 0.5 cosine for synonym query | system_integration | P0 | REQ-MEM-002 |
| CASE-MEM-004 | Cross-tenant recall returns empty | system_integration | P0 | REQ-AUTH-003, BEH-MEM-001, RISK-SEC-004 |
| CASE-MEM-005 | Expired rows excluded from recall | system_integration | P1 | REQ-MEM-004 |
| CASE-MEM-006 | Persona-scoped recall returns persona + global, never others | system_integration | P1 | DEC-PERSONA-001 |

#### CASE-MEM-004 — Cross-tenant isolation

- **Preconditions:** Two tenants `t-A`, `t-B`; each puts 10 memories with distinct content.
- **Steps:**
  1. With `t-A` JWT, `POST /v1/memories/recall` query that lexically matches `t-B`'s content verbatim.
  2. With Postgres logs at `log_min_duration_statement=0`, capture the executed SQL.
- **Expected:** Response items empty; SQL contains `WHERE tenant_id = $1` with `t-A`'s id; no `t-B` row was read.
- **Automation:** `must`.

### 3.5 IM gateways

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-IM-001 | Feishu forged signature returns 401 before parse | system_integration | P0 | REQ-NFR-004, BEH-IM-001, MR-IM-001 |
| CASE-IM-002 | WeCom AES decryption roundtrip | system_integration | P0 | REQ-IM-003 |
| CASE-IM-003 | Discord Ed25519 verification | system_integration | P0 | REQ-IM-003 |
| CASE-IM-004 | Slack Socket Mode reconnect | compatibility | P1 | REQ-IM-001 |
| CASE-IM-005 | Identity bind challenge sent for unknown user | system_integration | P1 | REQ-IM-002 |
| CASE-IM-006 | Trailing message replay caps at MAX_MESSAGES_PER_CONVERSATION | system_integration | P1 | REQ-IM-004 |

### 3.6 Auth, tenancy, RBAC

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-AUTH-001 | Missing token returns 401 on every /v1/** route | regression | P0 | REQ-AUTH-001, MR-AUTH-001 |
| CASE-AUTH-002 | Expired token returns 401 | regression | P0 | REQ-AUTH-001 |
| CASE-AUTH-003 | Tenant A token rejected on tenant B resource | regression | P0 | REQ-AUTH-003, MR-AUTH-002 |
| CASE-AUTH-004 | Role `chat` rejected on /v1/admin/* | regression | P0 | REQ-AUTH-002 |
| CASE-AUTH-005 | RLS double-layer: app filter removed by mock, RLS still blocks | system_integration | P0 | DEC-HLD-008, RISK-SEC-004 |

CASE-AUTH-001..004 are generated by the adversarial harness §9 in the Test Strategy and run on every PR.

### 3.7 Audit

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-AUD-001 | Chain valid for a fresh tenant | system_integration | P0 | REQ-AUD-001 |
| CASE-AUD-002 | Deleting a row makes verifier return broken_at_sequence | system_integration | P0 | REQ-AUD-002, BEH-AUDIT-001, MR-AUDIT-001 |
| CASE-AUD-003 | PII probe corpus is fully scrubbed before signing | system_integration | P0 | REQ-AUD-003, RISK-SEC-002 |
| CASE-AUD-004 | OTel exported spans contain no PII patterns | system_integration | P0 | REQ-AUD-004, REQ-NFR-007 |
| CASE-AUD-005 | Dual-key verifier accepts entries from either key in window | system_integration | P1 | REQ-AUD-005, RISK-OPS-002 |
| CASE-AUD-006 | XIAOGUAI_AUDIT_REDACT_PII=false bypasses redactor (operator opt-out) | system_integration | P1 | REQ-AUD-003 |
| CASE-AUD-EXP-001 | SOC2 export over intact chain returns bundle + ChainProof | system_integration | P0 | DEC-016 |
| CASE-AUD-EXP-002 | GDPR Art. 30 projection filters to actor-non-null + PHI tables | unit | P0 | DEC-016 |
| CASE-AUD-EXP-003 | HIPAA §164.312 projection emits who/what/when/where/why | unit | P0 | DEC-016 |
| CASE-AUD-EXP-004 | Broken chain in window → ChainBroken{first_broken_id, ts} + HTTP 409 | system_integration | P0 | DEC-016 |
| CASE-AUD-EXP-005 | JSON and CSV output have identical row counts and column semantics | unit | P0 | DEC-016 |
| CASE-AUD-EXP-006 | PDF format returns PdfUnimplemented → HTTP 501 | system_integration | P1 | DEC-016 (deferred PDF) |
| CASE-AUD-EXP-007 | Empty window returns EmptyWindow (HTTP 400, not 200 with empty body) | unit | P1 | DEC-016 |
| CASE-AUD-EXP-008 | `xiaoguai audit export` CLI exits non-zero on ChainBroken | system_integration | P0 | DEC-016 |

#### CASE-AUD-002 — Forged chain detection

- **Preconditions:** 50 audit rows for tenant `t-X` produced by E2E flows.
- **Steps:**
  1. `DELETE FROM audit_log WHERE tenant_id = 't-X' AND sequence = 23`.
  2. `GET /v1/admin/audit/verify?tenant_id=t-X` with operator JWT.
- **Expected:** Response `valid=false, broken_at_sequence in {23, 24}` (depending on whether the verifier surfaces gap-detection at the deleted row or the next).
- **Automation:** `must`.

### 3.8 HotL

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-HOTL-001 | Allowed when under budget | system_integration | P0 | REQ-HOTL-001 |
| CASE-HOTL-002 | Escalate when over budget; synthetic ToolResult returned | system_integration | P0 | REQ-HOTL-002, BEH-HOTL-001 |
| CASE-HOTL-003 | Policy store unreachable → fail closed | system_integration | P0 | REQ-HOTL-003 |
| CASE-HOTL-004 | Policy CRUD via API | system_integration | P0 | REQ-HOTL-004 |
| CASE-HOTL-005 | API restart with 5 pending escalations: all 5 oneshot waiters replay on boot; recorded_at + expires_at preserve bit-identically; a subsequent `POST /v1/hotl/decisions` for each resolves the live waiter (`resumed=true`). *Shipped: PR #145 + #149 → `crates/xiaoguai-api/tests/hotl_persistence_replay.rs`, `crates/xiaoguai-api/tests/hotl_cross_feature_e2e.rs`* | system_integration | P0 | DEC-HLD-013 |
| CASE-HOTL-006 | API restart with 1 pending escalation that expired during downtime: replay path synthesises `verdict=timeout` exactly once; no late `decisions` POST flips it. *Shipped: PR #145 → `crates/xiaoguai-api/tests/hotl_persistence_replay.rs::replay_drops_expired`* | system_integration | P0 | DEC-HLD-013 |
| CASE-HOTL-007 | Tool-call args `{password: "secret", note: "ok"}` under tenant policy `{jsonpath: "$.password", applies_to: ["sse"]}`: `HotlPending.args_redacted == {password: "***", note: "ok"}`; audit row carries non-null `redaction_policy_id` matching policy version. *Shipped: PR #148 + #149 → `crates/xiaoguai-core/tests/hotl_args_redaction.rs`, `crates/xiaoguai-core/tests/hotl_cross_feature_e2e.rs`* | system_integration | P0 | DEC-HLD-014 |
| CASE-HOTL-008 | Empty redaction policy set: emits boot warning once per tenant; `HotlPending.args_redacted` equals raw args verbatim (degraded path, behaviour documented). *Shipped: PR #148 → `crates/xiaoguai-core/tests/hotl_args_redaction.rs::{gate_emits_verbatim_args_when_no_rule,gate_fail_closed_when_required_and_no_rule}`* | unit | P0 | DEC-HLD-014 |
| CASE-HOTL-009 | Config sets `expiry.mcp = 4h`, `expiry.tool = 24h`, `default_expiry = 24h`: `mcp.oauth.consent` escalation's `expires_at` is `now+4h`; `tool_call.execute_python` is `now+24h`; an unknown-class scope falls back to `now+24h`. *Shipped: PR #142 + #149 → `crates/xiaoguai-core/tests/hotl_per_scope_expiry.rs`, `crates/xiaoguai-core/tests/hotl_cross_feature_e2e.rs`* | unit | P0 | DEC-HLD-015 |
| CASE-HOTL-010 | `POST /v1/hotl/decisions` body using legacy `request_id` field returns `400 Bad Request` with `field: escalation_id` discriminator; body using `escalation_id` succeeds. *Shipped: PR #146 → `crates/xiaoguai-api/tests/hotl_escalation_id_rename.rs`* | system_integration | P0 | DEC-HLD-016 |
| CASE-HOTL-011 | Operator JWT lacking `hotl:decide` Casbin scope receives `403 Forbidden` on `POST /v1/hotl/decisions`; same JWT with `hotl:decide` succeeds; sprint-11 path-based fallback rule is absent from the loaded policy. *Shipped: PR #143 + #149 → `crates/xiaoguai-api/tests/hotl_decide_scope.rs`, `crates/xiaoguai-api/tests/hotl_cross_feature_e2e.rs`* | system_integration | P0 | DEC-HLD-016 |
| CASE-HOTL-012 | Migration 0027 backfill: tenant fixture with 7 pre-sprint-13 `hotl_pending` rows produces 7 new `hotl_escalations` parent rows + 7 `hotl_pending` child rows with matching `escalation_id` FKs; row counts identical before/after; `pg_dump` diff scoped to the two tables. *Shipped: PR #138 → `crates/xiaoguai-storage/tests/migrations_hotl_escalations.rs` (`#[ignore]`, requires Docker)* | system_integration | P0 | DEC-HLD-016 |
| CASE-HOTL-013 | Nested-gating audit: an orchestrator triangle Worker iteration that escalates two gates produces exactly one `hotl_escalations` parent row plus two `hotl_pending` children sharing that parent id. *Shipped: PR #141 → `crates/xiaoguai-storage/tests/hotl_escalations_repo.rs` (`#[ignore]`, requires Docker)* | system_integration | P1 | DEC-HLD-016, DEC-021 |

#### CASE-HOTL-002 — Allow-then-escalate

- **Preconditions:** Policy `scope=tool_call.execute_python, max_count=2, window=60s, escalate_to=inbox`.
- **Steps:** Issue 3 chats that each invoke `execute_python`.
- **Expected:** Third call escalates: agent observes synthetic ToolResult with reason `budget_exceeded`; one `tool.escalated` audit row; inbox receives one escalation message.
- **Automation:** `must`.

### 3.8a Agent-authored skills (T3)

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-SKILLAUTH-001 | propose_skill rejected on disabled tenant — no audit emission | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-002 | Invalid manifest (bad semver) rejected before HotL gate; no audit | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-003 | tool_allowlist references unknown tool → InvalidManifest | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-004 | tool_allowlist contains propose_skill → InvalidManifest (no recursion) | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-005 | tool_allowlist declares an MCP server → InvalidManifest | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-006 | HotL deny propagates reason back to agent | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-007 | Daily budget (5) exhausted → BudgetExceeded | system_integration | P0 | DEC-014 |
| CASE-SKILLAUTH-008 | Approve writes YAML to skills_dir then flips Approved → Installed | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-009 | Audit chain exact sequence: skill.propose → skill.hotl_gate → skill.approve | unit | P0 | DEC-014 |
| CASE-SKILLAUTH-010 | E2E revise-after-deny: propose denied → revise → allow → approve | system_integration | P0 | DEC-014 |

### 3.9 Skill packs

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-SKILL-001 | Install signed pack succeeds | system_integration | P0 | REQ-SKILL-002 |
| CASE-SKILL-002 | Install unsigned pack rejected without --allow-unsigned | system_integration | P0 | REQ-SKILL-003, RISK-SEC-001 |
| CASE-SKILL-003 | Install unsigned pack with --allow-unsigned writes audit | system_integration | P0 | REQ-SKILL-003 |
| CASE-SKILL-004 | Knob overrides applied without forking pack | system_integration | P1 | REQ-SKILL-004 |

### 3.10 Scheduler

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-SCHED-001 | Cron fires within tick + 1 s of schedule | system_integration | P0 | REQ-SCHED-001 |
| CASE-SCHED-002 | Public webhook with valid token fires; invalid token 401 | system_integration | P0 | REQ-SCHED-003 |
| CASE-SCHED-003 | Each of four sinks delivers a fixture message | system_integration | P0 | REQ-SCHED-002 |
| CASE-SCHED-004 | Proactive budget exhaustion writes scheduler.proactive_denied | system_integration | P0 | REQ-SCHED-004 |
| CASE-SCHED-005 | File-watch debounces 10 events into 1 fire | system_integration | P1 | REQ-SCHED-005 |

### 3.11 Outcomes

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-OUT-001 | POST /v1/outcomes persists row | system_integration | P1 | REQ-OUT-001 |
| CASE-OUT-002 | Summary endpoint scopes to tenant only | system_integration | P1 | REQ-OUT-002, DEC-HLD-010 |
| CASE-OUT-003 | Anomaly Z-score flags injected spike | system_integration | P2 | REQ-OUT-003 |

### 3.12 Eval

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-EVAL-001 | List, run, and gather results for a smoke suite | system_integration | P1 | REQ-EVAL-001 |
| CASE-EVAL-002 | Materialise eval case from session | system_integration | P1 | REQ-EVAL-002 |

### 3.14 Web UI (chat-ui + admin-ui golden paths)

UI cases are owned by the Playwright harness in `frontend/e2e/`. Each row maps to a `.spec.ts` file; multi-browser matrix (Chromium / Firefox / WebKit) is the run dimension.

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-UI-CHAT-001 | New session: composer submit → SSE delta → final | e2e (Playwright) | P0 | REQ-UI-001, BEH-CHAT-001 |
| CASE-UI-CHAT-002 | Resume `/sessions/:id` renders historical messages without opening SSE | e2e | P1 | REQ-UI-001 |
| CASE-UI-CHAT-003 | HotL pending event suspends stream; approve resumes; deny ends with rejection bubble | e2e | P0 | REQ-UI-004, DEC-006 |
| CASE-UI-CHAT-004 | SSE dropped mid-stream: partial bubble preserved + auto-reconnect banner | e2e | P1 | DEC-LLD-CHAT-UI-003 |
| CASE-UI-CHAT-005 | AI disclosure banner non-dismissible in mandatory mode (EU AI Act Art. 50(1)) | e2e | P1 | REQ-UI-004 |
| CASE-UI-CHAT-006 | Composer accepts next-message submit while previous stream in flight → cancel-then-new | e2e | P1 | DEC-LLD-CHAT-UI-001 |
| CASE-UI-CHAT-007 | First-token P95 ≤ 1.5 s on local Ollama fixture | e2e (perf) | P1 | REQ-NFR-011 |
| CASE-UI-ADMIN-001 | Today pane renders unified timeline; clicking row → audit link | e2e | P0 | REQ-UI-002 |
| CASE-UI-ADMIN-002 | Audit pane HMAC chain badges colour-correctly on intact / rotated / broken fixtures | e2e | P0 | REQ-UI-003, BEH-AUDIT-001 |
| CASE-UI-ADMIN-003 | Audit export (SOC2) → ChainProof bundle download | e2e | P1 | REQ-UI-003 |
| CASE-UI-ADMIN-004 | Scheduler pane: create cron job → fire-now → audit row link | e2e | P0 | REQ-UI-002 |
| CASE-UI-ADMIN-005 | HotL Policies pane: create deny-default policy requires confirm modal | e2e | P1 | REQ-UI-002 |
| CASE-UI-ADMIN-006 | Kanban: drag triage → ready; concurrent-edit 409 snaps back | e2e | P1 | REQ-UI-002 |
| CASE-UI-ADMIN-007 | Personas pane CRUD against `/v1/personas`; role tag chip render | e2e | P1 | REQ-UI-007 (sprint-10b) |
| CASE-UI-ADMIN-008 | Skill Proposals pane: approve flow against `/v1/skills/proposals/{id}/approve`; RBAC scope absent → action hidden | e2e | P1 | REQ-UI-008 (sprint-10b) |
| CASE-UI-A11Y-001 | Axe-core: 0 serious/critical violations across chat-ui + admin-ui golden paths | e2e (a11y) | P1 | REQ-NFR-014 |
| CASE-UI-I18N-001 | All visible JSX strings translated in `zh-Hans` + `en`; missing-key lint fail | unit (Vitest) | P1 | REQ-UI-005 |
| CASE-UI-AUTH-001 | 401 from API → redirect to `VITE_LOGIN_URL`; no in-SPA login form | e2e | P0 | REQ-UI-006, DEC-025 §3 |

### 3.13 Non-functional

| ID | Title | Layer | Priority | Verifies |
|---|---|---|---|---|
| CASE-NFR-001 | First-token P95 ≤ 1.5 s with Ollama 7B local | non_functional | P1 | REQ-NFR-001, MR-PERF-001 |
| CASE-NFR-002 | Memory recall P95 ≤ 300 ms @ 100k rows | non_functional | P1 | REQ-NFR-002 |
| CASE-NFR-003 | Container image ≤ 200 MB | non_functional | P1 | REQ-NFR-009 |
| CASE-NFR-004 | Multi-arch image manifest contains linux/amd64 and linux/arm64 | non_functional | P0 | REQ-NFR-006 |
| CASE-NFR-005 | Zero outbound network calls when no providers enabled but ollama | non_functional | P0 | REQ-NFR-008, RISK-OPS-001 |
| CASE-NFR-006 | cargo deny check licenses exits 0 | non_functional | P0 | REQ-NFR-010, MR-LIC-001, RISK-LIC-001 |

#### CASE-NFR-005 — Air-gap network probe

- **Preconditions:** Fresh deploy via `values.yaml` (no Valkey, no cloud providers); install fake DNS sinkhole capturing all egress.
- **Steps:** Run smoke suite (`xiaoguai-core smoke`).
- **Expected:** Sinkhole observes zero queries to `api.openai.com`, `api.anthropic.com`, `generativelanguage.googleapis.com`, etc.
- **Automation:** `must`.

---

## 4. Regression package

The following subset runs on every PR (Test Strategy §12 `regression-min`):

`CASE-AUTH-001..004, CASE-AUD-002, CASE-AUD-003, CASE-MEM-002, CASE-MEM-004, CASE-MCPEXEC-002, CASE-MCPEXEC-003, CASE-HOTL-002, CASE-HOTL-003, CASE-NFR-006`.

The release-tag package (`regression-full`) runs every case above plus CASE-NFR-001/002/003/005.

---

## 5. Smoke package (deploy verification)

Runs after every deploy:

- `GET /healthz` returns 200.
- `xiaoguai-core smoke` exits 0.
- `POST /v1/memories` with seed payload returns 200.
- `GET /v1/admin/audit/verify?tenant_id=<bootstrap>` returns `valid=true`.

---

## 6. Testany Automation Handoff

The block below is intended for `/case-writing` (testany-bot) to consume; it lists the regression cases that should be wired into the pipeline.

```yaml
testany_automation_handoff:
  workspace: xiaoguai-private
  pipeline: regression-min-on-pr
  trigger: github-webhook
  cases:
    - id: CASE-AUTH-001
      label: harness:adversarial
      gate: must-pass
    - id: CASE-AUTH-002
      label: harness:adversarial
      gate: must-pass
    - id: CASE-AUTH-003
      label: harness:adversarial
      gate: must-pass
    - id: CASE-AUTH-004
      label: harness:adversarial
      gate: must-pass
    - id: CASE-AUD-002
      label: integrity
      gate: must-pass
    - id: CASE-AUD-003
      label: privacy
      gate: must-pass
    - id: CASE-MEM-002
      label: air-gap
      gate: must-pass
    - id: CASE-MEM-004
      label: rls
      gate: must-pass
    - id: CASE-MCPEXEC-002
      label: sandbox
      gate: must-pass
    - id: CASE-MCPEXEC-003
      label: sandbox
      gate: must-pass
    - id: CASE-HOTL-002
      label: hotl
      gate: must-pass
    - id: CASE-HOTL-003
      label: hotl
      gate: must-pass
    - id: CASE-NFR-006
      label: licensing
      gate: must-pass
```

---

## 7. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: test-spec-profile-v1
artifact:
  id: TSPEC-XIAOGUAI-001
  type: TEST_SPEC
  title: Xiaoguai test case package v1.4
  status: draft
  owners: [qa.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-31
  source_documents:
    - PRD-XIAOGUAI-001
    - API-XIAOGUAI-001
    - HLD-XIAOGUAI-001
    - TSTRAT-XIAOGUAI-001
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions: []
  flows: []
  test_cases:
    - { id: CASE-CHAT-001, title: "Streaming chat SSE shape", statement: "POST /v1/sessions/{id}/messages returns delta/tool_call/final SSE sequence.", status: proposed, scope: in, layer: e2e, priority: P0, automation: must }
    - { id: CASE-CHAT-002, title: "Cancel during tool call", statement: "Cancel returns Final(cancelled) bounded by slowest tool.", status: proposed, scope: in, layer: e2e, priority: P0, automation: must }
    - { id: CASE-CHAT-003, title: "Session fork", statement: "Fork creates session with parent_session_id.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-CHAT-004, title: "Parallel tool dispatch", statement: "Wall-clock confirms join_all parallelism.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-CHAT-005, title: "Max-iteration cap", statement: "Tenant override honoured.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-LLM-001, title: "Register provider via API", statement: "POST creates row.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-LLM-002, title: "Reject unknown kind", statement: "400 error code.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-LLM-003, title: "Fallback cascade", statement: "Secondary succeeds after primary 503.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-LLM-004, title: "Air-gap seed", statement: "Migration 0020 leaves only ollama enabled.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-LLM-005, title: "token_usage per stream", statement: "Row written.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCP-001, title: "stdio roundtrip", statement: "Tool call returns expected result.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCP-002, title: "http+sse roundtrip", statement: "All three transports verified.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCP-003, title: "Marketplace install signed", statement: "Signature validated; install row recorded.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-MCPEXEC-001, title: "execute_python happy path", statement: "Returns expected exit_code 0 + stdout.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCPEXEC-002, title: "execute_python timeout", statement: "timed_out=true, exit_code=null.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCPEXEC-003, title: "Env scrub", statement: "Secret env var invisible.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCPEXEC-004, title: "Stderr PII redacted", statement: "Probe corpus passes.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MCPEXEC-005, title: "Output cap 64KiB", statement: "truncated=true.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MEM-001, title: "Put then recall", statement: "Returns the row with score.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MEM-002, title: "Air-gap memory smoke", statement: "OLLAMA_HOST unset still serves 200.", status: proposed, scope: in, layer: e2e, priority: P0, automation: must }
    - { id: CASE-MEM-003, title: "Ollama recall quality", statement: "Synonym query > 0.5 cosine.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: should }
    - { id: CASE-MEM-004, title: "Cross-tenant recall empty", statement: "RLS double-layer verified.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-MEM-005, title: "Expired rows excluded", statement: "TTL respected.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-MEM-006, title: "Persona-scoped recall", statement: "Persona-filtered query includes global rows.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-IM-001, title: "Feishu forged signature 401", statement: "Rejected before deserialise.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-IM-002, title: "WeCom AES roundtrip", statement: "Decrypts ok.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-IM-003, title: "Discord Ed25519", statement: "Verifies real signature.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-IM-004, title: "Slack Socket reconnect", statement: "Auto reconnect after drop.", status: proposed, scope: in, layer: compatibility, priority: P1, automation: should }
    - { id: CASE-IM-005, title: "Identity bind challenge", statement: "Unknown user gets challenge.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-IM-006, title: "Trailing replay capped", statement: "MAX_MESSAGES honoured.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-AUTH-001, title: "Missing token 401", statement: "Every /v1/** returns 401.", status: proposed, scope: in, layer: regression, priority: P0, automation: must }
    - { id: CASE-AUTH-002, title: "Expired token 401", statement: "JWT exp respected.", status: proposed, scope: in, layer: regression, priority: P0, automation: must }
    - { id: CASE-AUTH-003, title: "Tenant A on tenant B", statement: "Cross-tenant rejected.", status: proposed, scope: in, layer: regression, priority: P0, automation: must }
    - { id: CASE-AUTH-004, title: "Role chat on admin route", statement: "403 forbidden.", status: proposed, scope: in, layer: regression, priority: P0, automation: must }
    - { id: CASE-AUTH-005, title: "RLS without app filter", statement: "RLS still blocks if app filter removed.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-AUD-001, title: "Chain valid fresh tenant", statement: "verify returns valid=true.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-AUD-002, title: "Chain forge detected", statement: "Delete row → broken_at_sequence.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-AUD-003, title: "PII probe scrubbed", statement: "Corpus zero leaks.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-AUD-004, title: "OTel spans redacted", statement: "Exported spans pass corpus.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-AUD-005, title: "Dual-key rotation", statement: "Both keys accepted in window.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-AUD-006, title: "Redaction off bypass", statement: "Operator opt-out works.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-HOTL-001, title: "Allowed under budget", statement: "Tool call dispatched.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-002, title: "Escalate over budget", statement: "Synthetic ToolResult + audit row.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-003, title: "Policy store outage fail closed", statement: "Tool calls denied.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-004, title: "Policy CRUD via API", statement: "Round trip works.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-005, title: "Registry boot-replay happy path", statement: "5 pending rows survive API restart; all waiters replay; decisions resolve resumed=true.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-006, title: "Registry boot-replay expired-during-downtime", statement: "Pending row expires during restart; replay synthesises verdict=timeout once.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-007, title: "Args redaction applied before SSE emit", statement: "JSONPath selector $.password replaced with ***; audit row carries redaction_policy_id.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-008, title: "Empty redaction policy degraded path", statement: "Boot warning emitted once; args emitted verbatim.", status: proposed, scope: in, layer: unit, priority: P0, automation: must }
    - { id: CASE-HOTL-009, title: "Per-scope expiry lookup", statement: "expiry.mcp=4h beats default 24h for mcp.* scopes; unknown class falls back.", status: proposed, scope: in, layer: unit, priority: P0, automation: must }
    - { id: CASE-HOTL-010, title: "Legacy request_id field rejected", statement: "POST /v1/hotl/decisions with request_id returns 400 with field=escalation_id discriminator.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-011, title: "hotl:decide scope enforced", statement: "Operator without hotl:decide gets 403; with scope succeeds; path-based fallback removed.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-012, title: "Migration 0027 backfill correctness", statement: "Pre-sprint-13 hotl_pending rows backfill 1-to-1 into hotl_escalations; row counts preserved.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-HOTL-013, title: "Nested gating writes one parent + N children", statement: "Triangle Worker iteration with 2 gates produces 1 hotl_escalations row + 2 hotl_pending children sharing FK.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-SKILL-001, title: "Install signed pack", statement: "Signature validated.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SKILL-002, title: "Reject unsigned default", statement: "Install fails without --allow-unsigned.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SKILL-003, title: "Allow unsigned audited", statement: "Audit row records operator opt-in.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SKILL-004, title: "Knob overrides", statement: "Override applied without pack edit.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-SCHED-001, title: "Cron fires", statement: "Within tick + 1s.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SCHED-002, title: "Webhook token gate", statement: "Valid 200; invalid 401.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SCHED-003, title: "Four sinks deliver", statement: "feishu/telegram/email/inbox each.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SCHED-004, title: "Proactive deny audit", statement: "scheduler.proactive_denied row.", status: proposed, scope: in, layer: system_integration, priority: P0, automation: must }
    - { id: CASE-SCHED-005, title: "File-watch debounce", statement: "10 events -> 1 fire.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-OUT-001, title: "Outcome persists", statement: "Row written.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-OUT-002, title: "Summary tenant-scoped", statement: "No cross-tenant leak.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-OUT-003, title: "Anomaly Z-score", statement: "Spike detected.", status: proposed, scope: in, layer: system_integration, priority: P2, automation: should }
    - { id: CASE-EVAL-001, title: "Eval suite run", statement: "Results gathered.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-EVAL-002, title: "Eval case from session", statement: "Materialised .eval.yaml.", status: proposed, scope: in, layer: system_integration, priority: P1, automation: must }
    - { id: CASE-NFR-001, title: "First-token P95", statement: "≤ 1.5s qwen2.5-coder:7b.", status: proposed, scope: in, layer: non_functional, priority: P1, automation: should }
    - { id: CASE-NFR-002, title: "Recall P95", statement: "≤ 300ms @ 100k.", status: proposed, scope: in, layer: non_functional, priority: P1, automation: should }
    - { id: CASE-NFR-003, title: "Image size", statement: "≤ 200 MB.", status: proposed, scope: in, layer: non_functional, priority: P1, automation: must }
    - { id: CASE-NFR-004, title: "Multi-arch manifest", statement: "amd64 + arm64.", status: proposed, scope: in, layer: non_functional, priority: P0, automation: must }
    - { id: CASE-NFR-005, title: "Air-gap zero egress", statement: "No outbound calls.", status: proposed, scope: in, layer: non_functional, priority: P0, automation: must }
    - { id: CASE-NFR-006, title: "License scan clean", statement: "cargo deny licenses exit 0.", status: proposed, scope: in, layer: non_functional, priority: P0, automation: must }
relations:
  - { id: REL-TSPEC-001, type: verifies, from: CASE-CHAT-001, to: REQ-CHAT-001, status: active }
  - { id: REL-TSPEC-002, type: verifies, from: CASE-CHAT-001, to: BEH-CHAT-001, status: active }
  - { id: REL-TSPEC-003, type: verifies, from: CASE-CHAT-002, to: REQ-CHAT-002, status: active }
  - { id: REL-TSPEC-004, type: verifies, from: CASE-CHAT-004, to: REQ-CHAT-004, status: active }
  - { id: REL-TSPEC-005, type: verifies, from: CASE-MCPEXEC-002, to: BEH-MCPEXEC-001, status: active }
  - { id: REL-TSPEC-006, type: verifies, from: CASE-MCPEXEC-003, to: BEH-MCPEXEC-002, status: active }
  - { id: REL-TSPEC-007, type: mitigates, from: CASE-MCPEXEC-003, to: RISK-SEC-003, status: active }
  - { id: REL-TSPEC-008, type: verifies, from: CASE-MEM-002, to: MR-MEM-001, status: active }
  - { id: REL-TSPEC-009, type: verifies, from: CASE-MEM-004, to: BEH-MEM-001, status: active }
  - { id: REL-TSPEC-010, type: mitigates, from: CASE-MEM-004, to: RISK-SEC-004, status: active }
  - { id: REL-TSPEC-011, type: verifies, from: CASE-AUD-002, to: BEH-AUDIT-001, status: active }
  - { id: REL-TSPEC-012, type: verifies, from: CASE-AUD-002, to: MR-AUDIT-001, status: active }
  - { id: REL-TSPEC-013, type: mitigates, from: CASE-AUD-003, to: RISK-SEC-002, status: active }
  - { id: REL-TSPEC-014, type: verifies, from: CASE-HOTL-002, to: BEH-HOTL-001, status: active }
  - { id: REL-TSPEC-015, type: mitigates, from: CASE-HOTL-002, to: RISK-COST-001, status: active }
  - { id: REL-TSPEC-016, type: verifies, from: CASE-HOTL-003, to: REQ-HOTL-003, status: active }
  - { id: REL-TSPEC-017, type: mitigates, from: CASE-SKILL-002, to: RISK-SEC-001, status: active }
  - { id: REL-TSPEC-018, type: verifies, from: CASE-LLM-004, to: REQ-LLM-004, status: active }
  - { id: REL-TSPEC-019, type: mitigates, from: CASE-LLM-004, to: RISK-OPS-001, status: active }
  - { id: REL-TSPEC-020, type: verifies, from: CASE-NFR-006, to: MR-LIC-001, status: active }
  - { id: REL-TSPEC-021, type: mitigates, from: CASE-NFR-006, to: RISK-LIC-001, status: active }
  - { id: REL-TSPEC-022, type: verifies, from: CASE-IM-001, to: BEH-IM-001, status: active }
  - { id: REL-TSPEC-023, type: verifies, from: CASE-AUTH-001, to: MR-AUTH-001, status: active }
  - { id: REL-TSPEC-024, type: verifies, from: CASE-AUTH-003, to: MR-AUTH-002, status: active }
  - { id: REL-TSPEC-025, type: verifies, from: CASE-AUD-005, to: REQ-AUD-005, status: active }
  - { id: REL-TSPEC-026, type: mitigates, from: CASE-AUD-005, to: RISK-OPS-002, status: active }
  - { id: REL-TSPEC-027, type: verifies, from: CASE-HOTL-005, to: DEC-HLD-013, status: active }
  - { id: REL-TSPEC-028, type: verifies, from: CASE-HOTL-006, to: DEC-HLD-013, status: active }
  - { id: REL-TSPEC-029, type: verifies, from: CASE-HOTL-007, to: DEC-HLD-014, status: active }
  - { id: REL-TSPEC-030, type: verifies, from: CASE-HOTL-008, to: DEC-HLD-014, status: active }
  - { id: REL-TSPEC-031, type: verifies, from: CASE-HOTL-009, to: DEC-HLD-015, status: active }
  - { id: REL-TSPEC-032, type: verifies, from: CASE-HOTL-010, to: DEC-HLD-016, status: active }
  - { id: REL-TSPEC-033, type: verifies, from: CASE-HOTL-011, to: DEC-HLD-016, status: active }
  - { id: REL-TSPEC-034, type: verifies, from: CASE-HOTL-012, to: DEC-HLD-016, status: active }
  - { id: REL-TSPEC-035, type: verifies, from: CASE-HOTL-013, to: DEC-HLD-016, status: active }
waivers:
  - { id: WVR-TSPEC-001, target_ids: [CASE-EVAL-001, CASE-EVAL-002], reason: "Capability suite content seeding is deferred per PRD WVR-EVAL-001; runtime smoke is in scope but content depth is advisory until suites are seeded.", status: approved, approved_by: qa.xiaoguai, approved_at: 2026-05-28 }
```
<!-- TRACEABILITY-METADATA:END -->
