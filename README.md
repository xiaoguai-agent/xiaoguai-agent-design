# Xiaoguai (小怪) — Design Workspace

> **Retrofit design workspace.** Xiaoguai is implemented at `/Users/zw/testany/myskills/xiaoguai/` (public repo `github.com/xiaoguai-agent/xiaoguai`, v1.4.0 released 2026-05-26). This directory holds the **design documents written after the fact** so the system has a normalised, reviewable design baseline.

## Project at a glance

| | |
|---|---|
| Name | Xiaoguai (小怪) — *Your Little Agent for Big Work / 小怪不小，能办大事* |
| Implementation repo | `/Users/zw/testany/myskills/xiaoguai/` (public, BUSL-1.1) |
| License | BUSL-1.1, Licensor = Zhou Wei, Change Date = release + 4 years → Apache 2.0 |
| Stack | Rust (server) + React+TS (frontend) + Postgres + pgvector + optional Valkey + Ollama |
| Differentiation | lightweight · MCP-first · local-LLM default · 7 IM gateways native · BUSL + audit chain + HotL governance |
| Default deployment | **single-tenant air-gap**, multi-tenant SaaS opt-in |

## Document set (English, 2026-05-28)

| Document | Purpose |
|---|---|
| [`docs/prd-xiaoguai.md`](docs/prd-xiaoguai.md) | PRD (retrofit baseline) — scope, requirements, risks, must-not-regress, delta from 2026-05-21 brainstorm |
| [`docs/api-contract.md`](docs/api-contract.md) | HTTP REST + SSE + MCP tool surface, error envelope, SLOs |
| [`docs/hld.md`](docs/hld.md) | High-level design — module map, decisions (DEC-HLD-001…012), runtime model, data model, topologies |
| [`docs/guardrails.md`](docs/guardrails.md) | Project-level engineering rules (license, MSRV, security, perf, CI gates, regulatory hooks) |
| [`docs/test-strategy.md`](docs/test-strategy.md) | Independent test scope — layers, environment matrix, coverage targets, eval philosophy |
| [`docs/test-spec.md`](docs/test-spec.md) | Concrete test case package — 75 cases across 13 domains, RTM, Testany handoff |
| [`docs/runbook.md`](docs/runbook.md) | Operator runbook — deploy, upgrade, rollback, key rotation, DR, troubleshooting, on-call |
| [`docs/lld/`](docs/lld/) | Low-level design — one LLD per starred core crate (12 crates) |

## LLD inventory

See [`docs/lld/index.md`](docs/lld/index.md). Twelve crates:

- `xiaoguai-storage` — RLS, migrations, pgvector
- `xiaoguai-audit` — HMAC chain, PII redactor, OTel decorator
- `xiaoguai-llm` — router + 8 provider backends
- `xiaoguai-mcp` — client + supervisor + transports
- `xiaoguai-mcp-exec` — sandboxed Python `execute_python`
- `xiaoguai-agent` — ReAct loop with HotL gate
- `xiaoguai-runtime` — worker pool, cancel registry, cache
- `xiaoguai-orchestrator` — multi-agent dispatch (plan-then-execute, research+critic, round-robin)
- `xiaoguai-memory` — pgvector recall, embedder selection
- `xiaoguai-rag` — loaders, chunkers, retriever exposed via MCP
- `xiaoguai-im-gateway` — common IM abstraction + 7 platform adapters
- `xiaoguai-personas` — role presets, persona-scoped memory views

## Reference inputs

- [`HANDOFF-FOR-DESIGN-DOCS.md`](HANDOFF-FOR-DESIGN-DOCS.md) — the briefing for this writing session.
- [`docs/2026-05-21-xiaoguai-agent-design.md`](docs/2026-05-21-xiaoguai-agent-design.md) — the brainstorm that preceded implementation. See PRD §8 for the delta between vision and v1.4.0 reality.
- Implementation runbooks live in the code repo at `/Users/zw/testany/myskills/xiaoguai/docs/runbooks/` and were consolidated into the runbook above.

## Traceability

Every document carries a `<!-- TRACEABILITY-METADATA:BEGIN -->` block conforming to `testany-traceability v1.0.0`. The relationship graph:

```
PRD-XIAOGUAI-001
  ├── API-XIAOGUAI-001 (derived_from)
  ├── HLD-XIAOGUAI-001 (derived_from)  ─── refines ──► PRD-* requirements
  │     └── LLD-* (refines DEC-HLD-*)
  ├── GUARDRAILS-XIAOGUAI-001 (refines PRD + HLD)
  ├── TSTRAT-XIAOGUAI-001 (refines PRD + API + HLD + GUARDRAILS)
  │     └── TSPEC-XIAOGUAI-001 (verifies REQ-* / BEH-* / MR-* / RISK-*)
  └── RUNBOOK-XIAOGUAI-001 (refines HLD + GUARDRAILS)
```

Open design questions (see PRD §10) that were defaulted in this writing pass:

- **Q3** memory TTL purge job — manual today.
- **Q4** agent-authored skill flow — not started.
- **Q5** session compaction policy — Tier-2d, not started.
- **Q6** OAuth 2.1 PKCE outbound MCP scope — Tier-3a.
- **Q7** eval suite seeding — runtime ready, suites empty.

## Next steps

1. Stakeholder review of PRD and HLD; ratify or amend `DEC-HLD-*` and `DEC-DEPLOY-001` (single-tenant air-gap default).
2. Open the design questions above; seed `eval-suites/regression/` and `eval-suites/capability/`.
3. Lift these docs into the implementation repo under `docs/design/` once approved; remove this workspace.
