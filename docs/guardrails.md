# Guardrails — Xiaoguai

| | |
|---|---|
| Document ID | `GUARDRAILS-XIAOGUAI-001` |
| Type | Project guardrails |
| Status | `draft` |
| Source | `PRD-XIAOGUAI-001`, `HLD-XIAOGUAI-001`, implementation repo CI configs and `deny.toml` |
| Updated | 2026-05-31 |
| Review cadence | Quarterly, plus event-driven on the triggers in §10 |

Guardrails are **project-level engineering rules** that downstream documents (LLD, runbook, code) MUST follow. They are not per-feature requirements; they are invariants enforced by CI or by operational policy.

---

## 1. License & supply-chain

| ID | Rule | Enforcement |
|---|---|---|
| GR-LIC-01 | License of the project is **BUSL-1.1**, Licensor = Zhou Wei (`zhouwei008@gmail.com`), Change Date = release-date + 4 years, Change License = Apache 2.0. | `LICENSE` file checked in CI |
| GR-LIC-02 | All transitive dependencies of `xiaoguai-*` crates MUST be under one of: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, Zlib, MPL-2.0, BSL-1.0 (redis only). | `deny.toml` allow list + `cargo deny check licenses` in `deny.yml` workflow |
| GR-LIC-03 | AGPL, SSPL, BSL (other than redis BSL-1.0), Commons Clause, and "source-available" are blocked without exception. | Same |
| GR-LIC-04 | Frontend `package.json` dependencies are subject to the same allow list; AG Grid Enterprise / MUI X Pro / Tailwind UI Paid / Material Icons are explicitly disallowed. | Manual review in `frontend.yml` |
| GR-LIC-05 | Redis ≥ 7.4 SSPL is replaced by Valkey (BSD-3). MongoDB is not a dependency. Elasticsearch ≥ 8 is replaced by OpenSearch if needed. MinIO (AGPL 2025+) is replaced by SeaweedFS or cloud S3. | Per release inventory check |
| GR-LIC-06 | First-party Cargo packages publish to `crates.io` only with the BUSL header in every `lib.rs`/`main.rs`; verified in pre-release. | release script |

---

## 2. Toolchain & MSRV

| ID | Rule | Enforcement |
|---|---|---|
| GR-TC-01 | Rust toolchain is **pinned at 1.88** via `rust-toolchain.toml`; bumps require a coordinated PR. | `rust.yml` pinned action |
| GR-TC-02 | MSRV (`rust-version` in `Cargo.toml`) MUST match the pinned toolchain. | CI grep check |
| GR-TC-03 | `cargo fmt --check` and `cargo clippy --all-targets --all-features -- -D warnings` MUST be clean on every PR. | `rust.yml` |
| GR-TC-04 | `cargo build --all-targets` MUST succeed on Linux. macOS-only build failures are acceptable for `#[cfg(target_os = "linux")]` code (sd-notify etc.). | `rust.yml` matrix |
| GR-TC-05 | `cargo test --workspace` MUST pass; DB-dependent tests are marked `#[ignore]` and executed in `migrations-smoke` job via `--include-ignored`. | `rust.yml` + `e2e.yml` |
| GR-TC-06 | Front-end pnpm workspace pins Node ≥ 20 LTS; pnpm version pinned in `package.json` packageManager field. | `frontend.yml` |
| GR-TC-07 | Multi-arch image build (linux/amd64, linux/arm64). | `release.yml` |

---

## 3. Security baseline

| ID | Rule | Enforcement |
|---|---|---|
| GR-SEC-01 | No secrets in code, history, or fixtures. `gitleaks` runs on every PR with full-history scan. | `.gitleaks.toml` allow-list only for two test fixtures (wecom sample key, jwt test key) |
| GR-SEC-02 | All `/v1/**` (except `/healthz`) require bearer-JWT auth; this is a must-not-regress (MR-AUTH-001). | Integration test asserts 401 on missing token |
| GR-SEC-03 | Postgres RLS double-layer: app-layer `WHERE tenant_id = $1` AND Postgres RLS `USING (tenant_id = current_setting(...))` on every tenant-scoped table. | Code review + RLS audit script |
| GR-SEC-04 | `XIAOGUAI_AUDIT_REDACT_PII=true` is the default. Disabling requires explicit operator action and is logged. | Boot log assertion in smoke |
| GR-SEC-05 | OTel span export passes through `RedactingSpanExporter`. Direct `OtlpExporter::build()` calls are forbidden in non-test code. | clippy lint + code review |
| GR-SEC-06 | Inbound IM webhook signature verification is mandatory and cannot be disabled at runtime. (MR-IM-001) | Negative test in `xiaoguai-im-*` crates |
| GR-SEC-07 | `xiaoguai-mcp-exec` MUST scrub environment variables to a fixed allow-list (`PATH`, `LANG`, `LC_*`) before spawning Python. | Unit test in `xiaoguai-mcp-exec` |
| GR-SEC-08 | Skill packs MUST be signature-verified at install. `--allow-unsigned` is a per-install operator opt-in and is audited. | `xiaoguai-skills` integration test |
| GR-SEC-09 | HotL policy store outage MUST fail closed. No synthetic allow. (REQ-HOTL-003) | Unit test in `xiaoguai-runtime` |
| GR-SEC-10 | `cargo audit` (advisory DB) MUST be clean on every PR; ignored advisories are documented in `deny.toml` with an expiry. | `audit.yml` workflow |
| GR-SEC-11 | `cargo vet` trust list reviewed quarterly; new crate dependencies require a vet entry. | `cargo-vet.yml` workflow |
| GR-SEC-12 | OWASP A01–A10 self-check applied to every new HTTP route at code-review time. | `security-review` skill or PR template checkbox |
| GR-SEC-13 | HotL `args_redacted` MUST be computed by `xiaoguai-auth::redaction::RedactionRules::apply` before any SSE emission or audit row write. Direct copies of raw tool-call args into `AgentEvent::HotlPending` are forbidden. (DEC-HLD-014) | PR #148 + test in `crates/xiaoguai-core/tests/hotl_args_redaction.rs` |
| GR-SEC-14 | `POST /v1/hotl/decisions` MUST be guarded by the Casbin scope `hotl:decide`. Path-based fallback rules for this route are forbidden. (DEC-HLD-016) | PR #143 + test in `crates/xiaoguai-api/tests/hotl_decide_scope.rs` |
| GR-SEC-15 | Every scope-gated HTTP route MUST express its scope requirement via the `xiaoguai-api::middleware::require_scope::RequireScope<&'static str>` axum extractor. Inline `if !claims.scopes.contains(...)` checks in route handlers are forbidden — the gate must surface in the handler signature so OpenAPI generation and `cargo doc` produce an authoritative scope-route map. (DEC-HLD-018) | Sprint-14 PR (tracked) + lint in `tools/lint-scope-gates.sh` grepping for the forbidden pattern + test `require_scope_extractor.rs` |
| GR-SEC-16 | `hotl_redaction_policies` mutations are insert-only with an `active` flag. UPDATE statements on the table are forbidden in production code paths; only the migration toolchain (`xiaoguai-storage::migrations`) may run an UPDATE, and only to set `active=false` from a transactional code path that simultaneously inserts the superseding revision. (DEC-HLD-017) | Sprint-14 PR (tracked) + repo-grep CI gate forbidding `UPDATE hotl_redaction_policies` outside migrations + test in `crates/xiaoguai-storage/tests/hotl_redaction_revisions.rs` |
| GR-SEC-17 | OIDC issuers fronting `xiaoguai-api` MUST emit a `scopes` claim containing at minimum `hotl:decide` for operator roles and `hotl:policy:{read,write}` for tenant-admin roles. Absence is observable via the `xiaoguai_oidc_scopes_claim_present{issuer}` gauge; alert fires when `avg_over_time(...[10m]) < 0.95`. (DEC-HLD-019) | Runbook §9.8 (issuer-specific recipes) + boot-log diagnostic for missing claim + Grafana alert in `observability/grafana/dashboards/xiaoguai-tenant.json` |

### 3.1 Redaction policy (HotL operator-visibility surface)

The HotL operator-visibility surface — the chat-ui banner that displays a pending tool call to the human reviewer — is downstream of every other redaction backstop (audit chain redaction per DEC-HLD-004, OTel span redaction per GR-SEC-05). Without a dedicated rule layer there, a raw `password` or `account_number` argument shown in a banner becomes the leak point.

DEC-HLD-014 introduces per-tenant redaction policies in table `hotl_redaction_policies`:

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | uuid | RLS-scoped per GR-SEC-03 |
| `scope` | text | Tool-call scope this rule fires for (e.g. `tool_call.execute_python`); `*` matches all |
| `jsonpath` | text | JSONPath-style selector against the tool-call argument JSON; matched nodes are replaced with `"***"` |
| `applies_to` | text[] | `["sse"]`, `["audit"]`, or both. `["sse"]` is the minimum-allowed configuration |

Policy evaluation lives in `xiaoguai-auth::redaction` alongside Casbin (see DEC-HLD-014 rationale for crate choice). The rule layer is mandatory on the emission path: `SuspendingHotlGate` cannot bypass `RedactionRules::apply` — there is no operator opt-out. An empty rule set degrades to "args verbatim + boot warning" so tenants are not silently exposed.

Operators MUST configure at least one `applies_to=["sse"]` policy per tenant before flipping `agent.hotl.suspend_on_escalate = true`. The boot-time warning is GR-SEC-13's failure surface; the runbook references this paragraph.

> **Shipped:** xiaoguai PR #144 (`xiaoguai-auth::redaction::RedactionRules` + `HotlRedactionRepo` wiring), PR #148 (`SuspendingHotlGate` applies `RedactionRules` before SSE/audit emission), PR #138 (migration 0027 introduces `redaction_policies` table).

### 3.2 Policy authoring surface (sprint-14 — DEC-HLD-017)

Sprint-13 shipped the runtime path; sprint-14 adds the tenant-admin authoring surface. The CRUD contract (api-contract §2.13) is **insert-only with an `active` flag**:

- `POST` inserts a new row.
- `PUT` inserts a **new** row with `supersedes_policy_id = <prior>` and atomically flips the prior to `active=false` in the same transaction. The new row receives a fresh `policy_id`.
- `DELETE` only sets `active=false` (the row stays for audit lineage).

Audit rows in `hotl_pending` carry the exact `redaction_policy_id` that was applied at decision time (DEC-HLD-014); insert-only revisions preserve that pointer's resolvability for the life of the audit chain. GR-SEC-16 enforces the contract at the CI layer — any `UPDATE hotl_redaction_policies` outside the migration toolchain is a lint failure.

> **Planned:** sprint-14 PR series — admin-ui pane (LLD-ADMIN-UI §4.11), backend repo + routes (api-contract §2.13), audit hooks for the mutations (`hotl_redaction_policy.{create,update,delete}` audit kinds).

### 3.3 Scope-gate extractor and OIDC contract (sprint-14 — DEC-HLD-018, DEC-HLD-019)

Sprint-13's S13-10 added a `hotl:decide` scope check inline in `routes/hotl_decisions.rs`. Sprint-14 extracts that check into a reusable axum extractor (`xiaoguai-api::middleware::require_scope::RequireScope<&'static str>`); GR-SEC-15 makes the extractor mandatory for every scope-gated route. The extractor reads from JWT claims, not Casbin per-request, accepting the hot-reload trade-off explicitly (DEC-HLD-018 "Hot-reload" subsection).

The production deployment contract is GR-SEC-17: OIDC issuers must emit a `scopes` claim. Until they do, operators get `403 scope_required` on `POST /v1/hotl/decisions` (and any other scope-gated route). Runbook §9.8 documents per-issuer recipes (Keycloak / Auth0 / Okta / Azure AD / Authelia); the boot-log diagnostic surfaces the absence within 10 s of the first verified token.

---

## 4. Performance baseline

| ID | Rule | Enforcement |
|---|---|---|
| GR-PERF-01 | First-token latency on Ollama qwen2.5-coder:7b local: P95 ≤ 1.5 s. (REQ-NFR-001) | `perf-regression.yml` (currently gated behind paid runner; see GR-OPS-04) |
| GR-PERF-02 | Memory recall P95 ≤ 300 ms at 100 k rows. (REQ-NFR-002) | Benchmark in `xiaoguai-memory` |
| GR-PERF-03 | Container image ≤ 200 MB. (REQ-NFR-009) | CI image-size check |
| GR-PERF-04 | First-token latency must not regress by more than 20 % between releases. (MR-PERF-001) | Comparative benchmark in release runbook |

---

## 5. Coding standards

| ID | Rule | Enforcement |
|---|---|---|
| GR-CODE-01 | File size: 200–400 lines typical, 800 max. Larger files trigger refactor review. | Reviewer judgement |
| GR-CODE-02 | Functions ≤ 50 lines body. | Same |
| GR-CODE-03 | Prefer immutable data (`#[derive(Clone)]` + builder, not `&mut`). Mutation only at module boundaries (registries, repositories). | Clippy + reviewer |
| GR-CODE-04 | Errors are typed (`thiserror` enums per crate); never `Box<dyn Error>` in public APIs except at the very top. | Code review |
| GR-CODE-05 | Comments only for non-obvious *why*. No `// removed:`, no echoing the code. | Code review |
| GR-CODE-06 | All `unsafe` blocks require a `// SAFETY:` comment explaining the invariant. | clippy `clippy::undocumented_unsafe_blocks` |
| GR-CODE-07 | Public types are documented (`///`) at least at the type level. `cargo doc --no-deps` runs in CI but is non-gating. | CI advisory |
| GR-CODE-08 | No `unwrap()` / `expect()` outside tests, examples, or boot setup. Use typed errors. | clippy + code review |

---

## 6. Database & migrations

| ID | Rule | Enforcement |
|---|---|---|
| GR-DB-01 | New tenant-scoped table MUST add RLS policy in the same migration. | Migration template + reviewer |
| GR-DB-02 | Migrations are forward-only; no destructive operations on `audit_log`. | Migration lint |
| GR-DB-03 | New columns to `audit_log` MUST be backwards-compatible (NULLable) so the HMAC computation order does not change. | Reviewer |
| GR-DB-04 | Use of pgvector requires extension check at boot; the smoke test verifies `CREATE EXTENSION IF NOT EXISTS vector`. | Boot smoke |
| GR-DB-05 | Schema changes affecting embedding dimensions are explicit migrations (see PRD §4.4 REQ-MEM-002 dimension table). | Reviewer |

---

## 7. CI gates

The following gates MUST be green for a PR to merge to `main`:

| Workflow | Gate |
|---|---|
| `rust.yml` | fmt, clippy `-D warnings`, build all-targets, test workspace |
| `deny.yml` | `cargo deny check licenses advisories sources` |
| `audit.yml` | `cargo audit` clean |
| `cargo-vet.yml` | vet entries up to date |
| `migrations-smoke` | DB migrations apply cleanly; ignored tests pass |
| `helm-unittest.yml` | Helm chart tests pass (when `deploy/helm/` touched) |
| `manifest-validators.yml` | YAML schema validation (when `deploy/` touched) |

Non-gating advisory workflows (zombies acceptable in v1.4 but tracked for cleanup):

| Workflow | Status |
|---|---|
| `e2e.yml` | Pact / Playwright consumers, gated by repo label |
| `loadtest-smoke.yml` | k6 nightly only |
| `perf-regression.yml` | Needs `ubuntu-latest-4-cores` (paid); queued unless gated |
| `docs.yml` | mdBook publish to GitHub Pages; advisory |
| `release.yml` / `release-packages.yml` / `release-tarball.yml` / `pip-wheel.yml` | Only fires on tags |

---

## 8. Operational guardrails

| ID | Rule | Enforcement |
|---|---|---|
| GR-OPS-01 | Air-gap default deployment values (`values.yaml`) do NOT require Valkey, S3, or external IdP at install. Operator must opt in to these. | Helm template review |
| GR-OPS-02 | Migration 0020 seeds `ollama-local` enabled and all cloud providers disabled. Any future provider seed migration must continue this default. | Migration reviewer |
| GR-OPS-03 | HMAC key rotation MUST preserve a 30-day dual-key verification window. Operator runbook documents the procedure (see [`runbook.md`](runbook.md) §6). | Runbook reviewer |
| GR-OPS-04 | Long-running CI workflows that require paid runners (`perf-regression.yml`) MUST be gated by a label rather than running on every PR. | Workflow file review |
| GR-OPS-05 | Disaster recovery procedures for Postgres, Valkey, and audit chain MUST be in the operator runbook and reviewed at every release. | Release checklist |
| GR-OPS-06 | All operator runbooks include at least one verification command per step. | Runbook reviewer |

---

## 9. China / EU regulatory hooks (placeholder)

These items are not enforced by CI today but are tracked as obligations for the v1.5+ compliance track. They are placeholders to be filled when compliance work begins.

| ID | Obligation | Status |
|---|---|---|
| GR-REG-CN-01 | 等保 2.0 三级 self-check template (audit-trail integrity, identity, access logging) | Audit chain exists; report template TBD |
| GR-REG-CN-02 | 密评 (cryptographic compliance) for HMAC algorithm choice and key length (current: HMAC-SHA256, 256-bit) | Conforms; documentation TBD |
| GR-REG-CN-03 | 信创 hardware compatibility (linux/arm64) | Multi-arch build green |
| GR-REG-EU-01 | GDPR data-subject-export / data-subject-erasure flows | Manual today; automated job TBD |
| GR-REG-EU-02 | OAuth client registration is logged with `data_processing_purpose` | Manual today |
| GR-REG-US-01 | SOC2 control mapping (export from audit chain) | Tier-3b roadmap |
| GR-REG-US-02 | HIPAA template | Out of v1.x scope (PRD N6) |

---

## 10. Triggers requiring guardrail re-review

This document is reviewed quarterly. It is also reviewed immediately when any of the following occur:

| Trigger | Sections to re-review |
|---|---|
| New LLM provider added | §1 (license), §3 (security baseline), §4 (perf budget) |
| New IM platform added | §3 GR-SEC-06, §7 CI gates |
| New top-level binary or crate | §1 license, §2 toolchain |
| Audit chain schema change | §6 GR-DB-03, §8 GR-OPS-03 |
| Cache backend added / removed | §5 GR-OPS-01, runbook |
| Sandbox technology change (e.g., wasmtime+pyodide) | §3 GR-SEC-07, threat model in HLD |
| Compliance regime onboarded (HIPAA, FedRAMP) | §9 placeholders → real rules |
| Incident retro identifies repeat root cause | Add a new GR-* line plus a regression test |

---

## 11. Downstream impact

When a guardrail changes, the following downstream artefacts MUST be reviewed:

| Change | Downstream artefacts to re-review |
|---|---|
| §1 license rules | `Cargo.toml` metadata, `deny.toml`, frontend `package.json`, release notes |
| §2 toolchain | `rust-toolchain.toml`, every `rust-version`, CI |
| §3 security | HLD §6, every LLD, runbook |
| §4 perf | API contract §8 SLOs, perf-regression workflow |
| §5 coding standards | CONTRIBUTING.md |
| §6 DB | Migration template, every migration PR |
| §7 CI | Branch protection rules |
| §8 ops | Runbook §6 (key rotation), §10 (disaster recovery) |
| §9 compliance | Compliance docs, audit export jobs |

---

## 12. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: prd-profile-v1
artifact:
  id: GUARDRAILS-XIAOGUAI-001
  type: PRD
  title: Xiaoguai project guardrails
  status: draft
  owners: [engineering.xiaoguai, security.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-31
  source_documents:
    - PRD-XIAOGUAI-001
    - HLD-XIAOGUAI-001
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions: []
  flows: []
  test_cases: []
relations:
  - { id: REL-GR-001, type: refines, from: GUARDRAILS-XIAOGUAI-001, to: PRD-XIAOGUAI-001, status: active }
  - { id: REL-GR-002, type: refines, from: GUARDRAILS-XIAOGUAI-001, to: HLD-XIAOGUAI-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
