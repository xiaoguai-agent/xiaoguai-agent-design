# LLD Index — Xiaoguai v1.4

This directory contains the low-level design for each starred crate in [`hld.md`](../hld.md) §2.

| Crate | LLD file | Profile selection |
|---|---|---|
| `xiaoguai-storage` | [`lld-storage.md`](lld-storage.md) | Core + Storage + Observability |
| `xiaoguai-audit` | [`lld-audit.md`](lld-audit.md) | Core + Storage + Security + Observability |
| `xiaoguai-llm` | [`lld-llm.md`](lld-llm.md) | Core + Async + Observability |
| `xiaoguai-mcp` | [`lld-mcp.md`](lld-mcp.md) | Core + Async + Infra |
| `xiaoguai-mcp-exec` | [`lld-mcp-exec.md`](lld-mcp-exec.md) | Core + Infra + Security |
| `xiaoguai-agent` | [`lld-agent.md`](lld-agent.md) | Core + Async + Observability |
| `xiaoguai-runtime` | [`lld-runtime.md`](lld-runtime.md) | Core + Async |
| `xiaoguai-orchestrator` | [`lld-orchestrator.md`](lld-orchestrator.md) | Core + Async + Event |
| `xiaoguai-memory` | [`lld-memory.md`](lld-memory.md) | Core + Storage + API + Observability |
| `xiaoguai-rag` | [`lld-rag.md`](lld-rag.md) | Core + Storage |
| `xiaoguai-im-gateway` | [`lld-im-gateway.md`](lld-im-gateway.md) | Core + API + Event |
| `xiaoguai-personas` | [`lld-personas.md`](lld-personas.md) | Core + Storage |
| `xiaoguai-observability` (SLO module) | [`lld-observability.md`](lld-observability.md) | Core + Async + Observability |
| `@xiaoguai/admin-ui` | [`lld-admin-ui.md`](lld-admin-ui.md) | Frontend (React + Vite + TS) |
| `@xiaoguai/chat-ui` | [`lld-chat-ui.md`](lld-chat-ui.md) | Frontend (React + Vite + TS) |

All LLDs share the upstream baseline:

- PRD: [`PRD-XIAOGUAI-001`](../prd-xiaoguai.md)
- API contract: [`API-XIAOGUAI-001`](../api-contract.md)
- HLD: [`HLD-XIAOGUAI-001`](../hld.md)
- Guardrails: [`GUARDRAILS-XIAOGUAI-001`](../guardrails.md)

## Manifest summary

Every LLD module section follows this structure:

1. **Module purpose** — one paragraph
2. **Public interface** — exposed types and functions
3. **Module structure** — file layout
4. **Key flows** — happy path + exception branches
5. **Error handling** — typed error enum
6. **Concurrency / transactions** — locks, async boundaries, idempotency
7. **Test design** — unit (in scope for crate), system integration (delegated to test-strategy)
8. **Traceability metadata**

Each LLD's `decisions[]` refer to a single `DEC-` in HLD §3 (or further refine one); LLD `flows[]` use kind `module_interaction` to refine an HLD `system_flow`.
