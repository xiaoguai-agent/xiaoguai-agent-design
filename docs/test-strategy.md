# Test Strategy — Xiaoguai v1.4

| | |
|---|---|
| Document ID | `TSTRAT-XIAOGUAI-001` |
| Type | Test strategy |
| Status | `draft` |
| Source | [`PRD-XIAOGUAI-001`](prd-xiaoguai.md), [`API-XIAOGUAI-001`](api-contract.md), [`HLD-XIAOGUAI-001`](hld.md), [`GUARDRAILS-XIAOGUAI-001`](guardrails.md) |
| Updated | 2026-05-28 |

> This document defines the **independent test scope** — what gets tested outside the developer inner-loop (`cargo test`). The dev inner-loop covers unit + code-level integration + provider-side contracts; this strategy is responsible for system integration, e2e, regression, compatibility, non-functional, security, and eval.

---

## 1. Quality goals

| ID | Goal |
|---|---|
| QG-01 | No `/v1/**` route ships without an authn/authz negative test. |
| QG-02 | Multi-tenant isolation (RLS) is verified by adversarial cross-tenant queries on every release. |
| QG-03 | Audit chain integrity (forge/delete detection) is verified on every release. |
| QG-04 | HotL allow-then-escalate path is verified end-to-end with a real Postgres + cache. |
| QG-05 | `xiaoguai-mcp-exec` sandbox isolation properties (timeout, memory cap, env scrub, PII redaction) are verified with real processes, not mocks. |
| QG-06 | Air-gap deployment shape (no Valkey, no cloud LLM enabled) boots and serves `/v1/memories` returning 200. |
| QG-07 | First-token latency does not regress > 20 % between releases. |
| QG-08 | Coverage of P0 requirements is 100 % at acceptance-criterion level; P1 ≥ 80 %. |

---

## 2. In scope

- System integration tests across multiple crates with real Postgres + cache backend.
- E2E tests against a booted `xiaoguai serve` over HTTP/SSE.
- Regression tests derived from real session transcripts (REQ-EVAL-002).
- Compatibility tests across the cache backend matrix (`""` in-process vs `redis://`).
- Non-functional: first-token latency, recall latency, container image size.
- Security: authn/authz, RLS adversarial, sandbox escape probes, signature forgery, license scan.
- Eval suites: capability (probe new behaviour) and regression (must pass 100 %).

## 3. Out of scope

- Unit tests of individual functions (developer inner-loop).
- Code-level integration tests within a single crate (developer inner-loop).
- Provider-side contracts of upstream services (Ollama API, OpenAI Chat Completions schema) — assumed stable by SDK; deviations caught at runtime.
- Performance characterisation of upstream LLMs (we measure our own latency, not theirs).
- Manual exploratory testing.

## 4. Test layers and ownership

| Layer | Owner | Tooling | Trigger |
|---|---|---|---|
| L0 — Unit | Developer | `cargo test --lib` | Inner loop |
| L0 — Code-level integration | Developer | `cargo test` per crate | Inner loop |
| L1 — System integration | This strategy | `cargo test --workspace --include-ignored` against real PG + cache | CI `migrations-smoke` + nightly |
| L2 — E2E HTTP/SSE | This strategy | `tests/e2e/` test harness using `reqwest` + `eventsource-client` against `xiaoguai serve` | Nightly + pre-release |
| L3 — Regression eval | This strategy | `xiaoguai eval run --suite regression/` | Nightly + pre-release |
| L4 — Capability eval | This strategy | `xiaoguai eval run --suite capability/` | Pre-release advisory |
| L5 — Performance regression | This strategy | `cargo bench` + `hyperfine` + flamegraph | Pre-release |
| L6 — Security | Security + this strategy | Authn/authz negatives, RLS adversarial, signature forgery, `cargo audit`, `cargo deny` | Every PR + pre-release |
| L7 — Manual UAT | Product + first customer | Helm install in customer staging | Pre-release |

---

## 5. Environment matrix

| Layer | Postgres | Cache | LLM | OS / arch |
|---|---|---|---|---|
| L1 system integration | PG 17 + pgvector container | in-process AND `redis://` (matrix) | InMemoryEmbedder (deterministic) | linux/amd64 |
| L2 E2E | PG 17 + pgvector | in-process | Ollama all-minilm + qwen2.5-coder:0.5b (smallest viable) | linux/amd64 |
| L3 regression | Same as L2 | Same | Same models | linux/amd64 |
| L4 capability | Same as L2 | Same | Same models + opt-in cloud provider (Anthropic) | linux/amd64 |
| L5 performance | PG 17 prewarmed | in-process | Ollama qwen2.5-coder:7b | linux/amd64 with 16 GB RAM minimum |
| L6 security | PG 17 | in-process | none required | linux/amd64 |

Pre-release adds an arm64 smoke pass for L1+L2 to verify multi-arch.

---

## 6. Test data strategy

| Data class | Source | Privacy | Refresh |
|---|---|---|---|
| Synthetic users / tenants | Fixtures in `tests/fixtures/` | Synthetic, no PII | Per test run |
| Synthetic IM payloads | Per-platform fixtures; include the two allowlisted test keys from `.gitleaks.toml` | Synthetic | Static |
| Synthetic LLM responses | Recorded once via `xiaoguai eval case-from-session`; stored as `.eval.yaml` | Synthetic; no real customer data | On capability-suite update |
| Real customer transcripts | NOT used unless redacted via `xiaoguai-audit::redact` first; customer opt-in required | Redacted | Manual |
| PII probe corpus | Synthetic emails, IPv4, Bearer tokens, AWS keys; used to verify redactor | Synthetic | Static |

PII probe corpus is a must-not-regress fixture: every redactor change MUST keep this corpus producing zero leaks.

---

## 7. Coverage targets by requirement class

| Requirement class | L1 | L2 | L3 | L5 | L6 |
|---|---|---|---|---|---|
| P0 functional | 100 % | 100 % | n/a | n/a | 100 % (authn/authz) |
| P0 non-functional | n/a | 80 % | n/a | 100 % | 100 % (license) |
| P0 must-not-regress | 100 % | 100 % | 100 % | n/a | 100 % |
| P1 functional | 100 % | 80 % | n/a | n/a | n/a |
| P2 functional | 80 % | optional | n/a | n/a | n/a |

Coverage is measured at **acceptance-criterion granularity**, not line coverage. Each AC in PRD §4 must map to at least one test case in `docs/test-spec.md`.

---

## 8. Risk-driven test design

Each PRD-listed risk gets at least one verifying test case:

| Risk | Verification |
|---|---|
| RISK-SEC-001 unsigned skill pack | L6 security: install unsigned pack without `--allow-unsigned` → expect rejection; install with `--allow-unsigned` → expect audit row |
| RISK-SEC-002 PII leak via traces | L1: assert exported spans contain no email / IPv4 / Bearer / AWS-key patterns |
| RISK-SEC-003 sandbox escape | L1: `execute_python` reading `XIAOGUAI_AUDIT_SIGNING_KEY` returns unset; writing outside workdir fails |
| RISK-SEC-004 cross-tenant leak | L1: tenant A JWT querying tenant B memories returns empty; verify via Postgres logs that no rows were read |
| RISK-OPS-001 air-gap reaches cloud | L2: boot air-gap profile, assert outbound DNS to `api.openai.com` is zero |
| RISK-OPS-002 HMAC rotation breaks chain | L1: dual-key fixture — sign N entries with key A, rotate to key B, verifier accepts both within window, rejects after window expiry |
| RISK-LIC-001 transitive AGPL | L6: `cargo deny check licenses` blocks merge |
| RISK-LIC-002 BUSL misinterpretation | manual; license banner test asserts presence at boot and in admin UI |
| RISK-COST-001 token-bomb | L1: send 100 KB prompt → cost quota gate triggers per ADR-0009 |
| RISK-OPS-003 long-session context overflow (v0.5.4.1) | L1+L2: synthetic 100-turn agent run with compaction enabled — assert `xiaoguai_compaction_triggered_total` increments, post-compaction token estimate ≤ 50 % of `max_context_tokens`, agent can answer a follow-up that depends on a fact stated 80 turns earlier (preserved via summary). Healthy gate: `compaction_fallback_total / triggered_total < 5 %`. |
| RISK-SEC-005 agent-authored skill manifest abuse (v0.5.4.2, T3) | L1: validator rejects manifests that (a) reference unknown tools, (b) include `propose_skill` itself, (c) declare any MCP server / native code / filesystem path. L2: `xiaoguai-api/tests/skill_proposals.rs` `tenant_settings.allow_skill_authoring=false` → 503 with no audit emission. L2: HotL bucket `skill_author` enforces 5/tenant/day cap before any proposal reaches `Pending` state. **Required eval case**: `skill-author-allowlist-bypass` (capability suite) — adversarial agent attempts to inject `tool_allowlist=["execute_python", "rm_rf"]`; expect rejection with `InvalidManifest("rm_rf not in known tools")`. |
| RISK-SEC-006 outbound MCP OAuth refresh-token leakage (v0.5.4.2, T4) | L1: refresh_token is stored in `mcp_oauth_tokens` with RLS policy `tenant_id = current_setting('app.current_tenant_id')`. L1: TLS verify-off mode requires *both* the env var `XIAOGUAI_MCP_OAUTH_INSECURE=1` AND the warn log emission — the absence of either reverts to verify-on. L6 (compliance): hardening item documented — refresh_token at rest is NOT encrypted in v0.5.4.2; that's a follow-up PR. Operator runbook covers DB filesystem encryption as the interim mitigation. |
| RISK-COMP-001 audit-chain export over tampered window (v0.5.4.2, T5) | L1: `crates/xiaoguai-audit/tests/export_integration.rs` deletes one audit_log row inside the export window → `export_bundle()` returns `ChainBroken{first_broken_id, ts}`, HTTP 409, CLI exit non-zero. **There is no `--skip-verify` flag** — by design. L6: confirms the compliance promise that "export reflects an intact chain" is structurally enforced, not just documented. |
| RISK-SEC-007 polyglot sandbox cross-runtime escape (v0.5.4.2, T6, DEC-017) | L1: `xiaoguai-mcp-exec` and `xiaoguai-mcp-exec-js` are **separate binaries** loaded via separate `xiaoguai mcp register` rows. A CVE in V8 cannot reach the Python interpreter process and vice versa. L2: HotL buckets `tool_call.execute_python` and `tool_call.execute_javascript` are independent — operators can grant agents one without the other. Gated test on JS path validates env scrub of `XIAOGUAI_AUDIT_SIGNING_KEY` (mirrors CASE-MCPEXEC-003 for Python). |

---

## 9. Authn / authz / RLS adversarial harness

A single test fixture `tests/adversarial/` runs the following against every `/v1/**` route:

1. **No token** → 401.
2. **Expired token** → 401.
3. **Token for tenant A** querying tenant B → 200 with empty body (no leak) **or** 403 (whichever the API contract specifies) — never 200 with data.
4. **Token with role `chat`** calling `/v1/admin/*` → 403.
5. **Token replay** (same token after revocation) → 401 within 1 minute.

This harness MUST be extended whenever a new route is added to API contract §2.

---

## 10. Eval suite philosophy

### 10.1 Two suites

| Suite | Purpose | Acceptance |
|---|---|---|
| `eval-suites/regression/` | Lock in behaviour we have shipped; every case has a known-good outcome | 100 % pass rate; PR blocking |
| `eval-suites/capability/` | Probe behaviour we are adding or improving; cases may fail | Advisory; trend tracked |

### 10.2 Provenance

Regression cases derive from **real session transcripts** via `POST /v1/admin/eval/case-from-session` (REQ-EVAL-002). Capability cases are authored by hand from PRD requirements. Each case stores:

- `prompt`: user input
- `tools_allowed`: list
- `expected_artefact`: deterministic content (e.g., a function definition) OR a checker prompt (LLM-judge)
- `max_iterations`
- `seed` (when applicable)

### 10.3 Grading

Two graders are supported (see `xiaoguai-eval`):

- **Code grader**: deterministic check on the final assistant message (regex, JSON-path, equality).
- **Transcript grader**: checks the sequence of tool calls (e.g., "must have called `read_file` before `edit_file`").

LLM-judge grading is supported as a tertiary signal but never as the only signal for regression suites.

### 10.4 Seeding plan

`xiaoguai-eval` runtime is shipped but suite content is empty as of v1.4.0 (PRD waiver WVR-EVAL-001). The seeding plan:

| Phase | What lands | Owner |
|---|---|---|
| 1 | 20 regression cases from production session transcripts (real customer, redacted) | QA + product |
| 2 | 10 capability cases per major PRD feature (chat / memory / hotl / sandbox / IM / **compaction**) | QA |
| 3 | LLM-judge fixture cases for subjective tasks | QA + product |

**Compaction-specific eval cases** (Phase 2 sub-track, v0.5.4.1+):

- `compaction-fact-recall`: 80-turn synthetic session establishes a fact at turn 5 (e.g., "the deploy target is cluster `prod-east-7`"); at turn 50 trigger compaction; at turn 60 ask a question whose correct answer requires the turn-5 fact. Pass = the fact survives the summary.
- `compaction-tool-pair-integrity`: a conversation with assistant `tool_calls` / `Role::Tool` reply pairs spread across the would-be split boundary. Post-compaction, every retained `Role::Tool` must have its issuing assistant message also retained. Code-grader.
- `compaction-fallback-on-flake`: summariser backend returns 500 once, then 200; pass = `CompactionOutcome::FellBack` recorded, next iteration retries summarisation cleanly, no loss of recent context.

---

## 11. Performance regression strategy

| Benchmark | Tool | Source of baseline |
|---|---|---|
| First-token latency, Ollama 7B local | `cargo bench --bench first_token` | Locked at v1.4.0 release; future runs compared with ± 20 % threshold |
| Memory recall @100k | `cargo bench --bench mem_recall` | Same |
| Audit chain verify, 1M rows | `hyperfine` | Same |
| Image size | `docker image ls` post-build | Locked at v1.4.0 |

Performance regression workflow currently requires `ubuntu-latest-4-cores` (paid runner) — guardrail GR-OPS-04 requires gating behind a label.

---

## 12. Smoke and regression packages

| Package | Scope | Run before |
|---|---|---|
| `smoke` | `/healthz`, `xiaoguai serve` boots in air-gap, migrations apply, `/v1/memories` returns 200 | Every deploy |
| `regression-min` | Adversarial harness §9, audit chain forge detection, HotL allow-then-escalate, sandbox isolation, license scan | Every PR |
| `regression-full` | regression-min + L3 eval suite + L5 perf regression + arm64 smoke | Every release tag |

---

## 13. Entry / exit criteria

### 13.1 Entry to release

- All P0 + P1 ACs traced to a test case in `docs/test-spec.md`.
- `regression-full` package green on the release candidate commit.
- Adversarial harness §9 green.
- License scan green.
- No open P0/P1 bug in the bug tracker.
- Runbook updated to cover any new operational surface.

### 13.2 Exit from release (post-deploy)

- Audit-chain verification green on the deployed instance (`GET /v1/admin/audit/verify`).
- `/healthz` and `xiaoguai-core smoke` succeed.
- First-token latency P95 within 20 % of baseline (random sample from production).
- No new error class trending in Prometheus alerts within the first 24 h.

---

## 14. Test environment hygiene

| Rule | Statement |
|---|---|
| Isolation | Each L1/L2 test run uses a fresh Postgres DB (`CREATE DATABASE xiaoguai_test_<run_id>`); dropped at teardown. |
| Fixtures | Loaded from `tests/fixtures/`; never edited by tests at runtime. |
| Network | Tests assume no internet; tests that need Ollama get a local container. Cloud LLM tests skip when API key env is unset. |
| Logs | All test stdout/stderr captured to `target/test-logs/<suite>/<case>/`; uploaded as CI artefacts. |
| Flakes | Three consecutive flakes → quarantine label; reviewed weekly; cause fixed or test removed. |

---

## 15. Independent verification responsibilities

This strategy explicitly does NOT own:

- Unit tests for individual functions (developer inner-loop responsibility).
- Code-level integration tests within a single crate (developer responsibility).
- Provider-side contract testing for upstream LLM APIs (upstream responsibility; we maintain SDKs with version pins and rely on `xiaoguai-llm` integration smoke).

This strategy DOES own:

- System integration tests across crates with real dependencies.
- End-to-end HTTP/SSE behaviour.
- Eval harness operation and suite curation.
- Performance regression measurement.
- Security baseline regression (RLS, sandbox, signature verification, license scan).

---

## 16. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: test-strategy-profile-v1
artifact:
  id: TSTRAT-XIAOGUAI-001
  type: TEST_STRATEGY
  title: Xiaoguai independent test strategy
  status: draft
  owners: [qa.xiaoguai, engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents:
    - PRD-XIAOGUAI-001
    - API-XIAOGUAI-001
    - HLD-XIAOGUAI-001
    - GUARDRAILS-XIAOGUAI-001
entities:
  requirements: []
  risks:
    - { id: RISK-TS-001, title: "Eval suite empty at release", statement: "Regression eval has no cases; behavioural drift will not be caught.", status: proposed, scope: in, level: high }
    - { id: RISK-TS-002, title: "Perf regression workflow unrunnable", statement: "perf-regression.yml needs paid runner; sits queued.", status: proposed, scope: in, level: medium }
    - { id: RISK-TS-003, title: "Adversarial harness drift", statement: "New /v1/** routes added without harness update.", status: proposed, scope: in, level: high }
  must_not_regress:
    - { id: MR-TS-001, title: "Adversarial harness runs on every PR", statement: "Authn/authz/RLS adversarial suite runs on every PR.", status: proposed, scope: in, priority: P0 }
    - { id: MR-TS-002, title: "License scan blocks merge", statement: "cargo deny licenses non-zero blocks merge.", status: proposed, scope: in, priority: P0 }
    - { id: MR-TS-003, title: "Regression eval green at release", statement: "Regression eval suite 100% before tag.", status: proposed, scope: in, priority: P0 }
  external_behaviors:
    - { id: BEH-TS-001, title: "Adversarial harness covers all routes", statement: "Every /v1/** route has a negative authn/authz test.", status: proposed, scope: in, actor: "ci", trigger: "PR open", observable_outcome: "Harness exits 0" }
    - { id: BEH-TS-002, title: "Air-gap smoke green", statement: "Air-gap boot + /v1/memories smoke green.", status: proposed, scope: in, actor: "ci", trigger: "deploy", observable_outcome: "200 on /v1/memories" }
  decisions: []
  flows: []
  test_cases: []
relations:
  - { id: REL-TS-001, type: refines, from: TSTRAT-XIAOGUAI-001, to: PRD-XIAOGUAI-001, status: active }
  - { id: REL-TS-002, type: mitigates, from: MR-TS-001, to: RISK-SEC-004, status: active }
  - { id: REL-TS-003, type: mitigates, from: MR-TS-002, to: RISK-LIC-001, status: active }
  - { id: REL-TS-004, type: mitigates, from: BEH-TS-002, to: RISK-OPS-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
