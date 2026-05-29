# Handoff: Retroactive Design Documentation for Xiaoguai

> **For the next Claude Code session that runs `/testany-eng:guide`.** This file
> is the briefing — read it first, then read [`README.md`](README.md) and the
> existing brainstorm at [`docs/2026-05-21-xiaoguai-agent-design.md`](docs/2026-05-21-xiaoguai-agent-design.md),
> then start the testany-eng flow.

---

## What the user wants

The user (Zhou Wei) wants the **xiaoguai** project's design captured retroactively
in this directory, using the `testany-eng` skill family (PRD writer, HLD writer,
LLD writer, etc.). The product has been implemented heavily over the past
~4 months and the original 2026-05-21 brainstorm in `docs/` is now outdated.

Goal: a comprehensive set of design docs covering
1. Current architecture and components
2. What's been built
3. What's in flight
4. What's planned but not started
5. Possible future features worth considering

The user said: **interact / ask questions where unclear**.

---

## Project at a glance

- **Name**: Xiaoguai (小怪) — "Your Little Agent for Big Work / 小怪不小，能办大事"
- **Repo**: https://github.com/xiaoguai-agent/xiaoguai (PUBLIC, latest tag v1.4.0)
- **Implementation directory**: `/Users/zw/testany/myskills/xiaoguai/` (the real codebase)
- **Design directory**: `/Users/zw/testany/myskills/xiaoguai-agent-design/` (THIS dir; where docs go)
- **License**: BUSL-1.1 (Licensor: Zhou Wei, zhouwei008@gmail.com), Change Date = release + 4 years → Apache 2.0
- **Stack**: Rust (server) + React+TS (frontends) + Postgres + pgvector + optional Valkey/Redis + Ollama (default local LLM)
- **Differentiation**: lightweight, MCP-first, **local-LLM default**, three Chinese IM gateways native (Feishu/DingTalk/WeCom), compliance posture (BUSL + audit chain + HotL governance)
- **Owner intent**: A good **local agent for enterprise private deployment**; commercial licensing for production use

---

## Codebase shape (current snapshot)

- **31 Rust crates** under `crates/`, **~93,000 lines of Rust**
- Top-level binaries:
  - `xiaoguai` (unified CLI + server, `xiaoguai serve` replaces legacy `xiaoguai-core serve`)
  - `xiaoguai-core` (kept as a thin shim for backward-compat with existing systemd units / .deb packaging)
  - `xiaoguai-mcp-exec` (sandboxed Python code-execution MCP server, Tier-2)
- React+TS frontends: chat-ui, admin-ui (Gemini-style redesign 2026-05-27, PR #38)
- CI: GitHub Actions, all real gates green (build-and-test, migrations-smoke, cargo-deny, cargo vet); k6 / mdbook / perf-regression workflows are non-gating zombies

### Crate inventory (group by domain)

| Domain | Crates |
|---|---|
| Foundations | `xiaoguai-types`, `xiaoguai-config`, `xiaoguai-storage`, `xiaoguai-auth`, `xiaoguai-audit` |
| LLM + MCP | `xiaoguai-llm`, `xiaoguai-mcp`, `xiaoguai-mcp-exec` |
| Agent runtime | `xiaoguai-agent`, `xiaoguai-runtime`, `xiaoguai-orchestrator` |
| Memory + RAG | `xiaoguai-memory`, `xiaoguai-rag` |
| Scheduling | `xiaoguai-scheduler`, `xiaoguai-watch`, `xiaoguai-anomaly` |
| Eval + Eval | `xiaoguai-eval` |
| API + CLI | `xiaoguai-api`, `xiaoguai-cli`, `xiaoguai-core` |
| IM gateways | `xiaoguai-im-gateway`, `xiaoguai-im-feishu`, `xiaoguai-im-dingtalk`, `xiaoguai-im-wecom`, `xiaoguai-im-discord`, `xiaoguai-im-slack`, `xiaoguai-im-telegram`, `xiaoguai-im-mattermost` |
| Other | `xiaoguai-personas`, `xiaoguai-tasks`, `xiaoguai-observability` |

---

## What's shipped (recent)

### v1.4.0 baseline (2026-05-26 release)
- Wave-3 dep cleanup, CI core green for the first time
- Frontend Gemini-style redesign
- Ollama is the local-first default LLM backend
- 30 deferred merges landed (memory/workspace/tasks/dispatcher/cli-tasks)

### Session-4 work (2026-05-28, PR #57, merged)
- `PgMemoryStore` wired into `xiaoguai-core` (was 503; now `/v1/memories` live)
- Audit chain PII redaction (`xiaoguai-audit::redact`: email, IPv4, Bearer tokens, AWS keys)
- OTel trace export redaction via `RedactingSpanExporter` decorator
- Live local E2E proven (PG17 + pgvector + Ollama all-minilm + cosine recall ranks correctly)

### Session-5 work (2026-05-28, this just-finished session — 4 PRs)

| PR | Branch | Status | Summary |
|---|---|---|---|
| #59 | `feat/tier1-unified-binary` | **MERGED** (`bb30e38`) | Tier-1a: `xiaoguai-core/main.rs` → lib.rs; unified `xiaoguai serve` subcommand; legacy bin kept as shim |
| #60 | `feat/tier1b-inprocess-cache` | **MERGED** (`ee8be10`) | Tier-1b: `Cache::connect("")` boots in-process DashMap (no Valkey needed for air-gapped single-tenant) |
| #61 | `feat/tier2-prereq-hotl-gate` | **MERGED** (`959de4c`) | Tier-2 prereq: `HotlGate` plumbed into agent ReAct loop's parallel tool-call dispatch |
| #64 | `feat/tier2-mcp-exec` | **MERGED** (`81cd853`) | Tier-2: new `xiaoguai-mcp-exec` crate — sandboxed Python via ulimit + tokio timeout + tempdir + env scrub + 64KB output cap + stderr PII redaction |

**Live-validated end-to-end** during session-5:
- `xiaoguai serve` in air-gap mode (no Valkey) → memory recall works
- `xiaoguai-mcp-exec` via real MCP stdio handshake: env isolation verified (parent `XIAOGUAI_AUDIT_SIGNING_KEY` did NOT leak), 30s sleep killed at 1s deadline, PII redaction works on stderr

---

## Roadmap (pi/Hermes-informed)

| Tier | Item | Status |
|---|---|---|
| **Tier-1** | Ollama/llama.cpp first-class default | ✅ Done |
| **Tier-1** | Single self-contained binary | ✅ Done (PR #59 + #60) |
| **Tier-1** | PII detection/redaction in audit + traces | ✅ Done (PR #57) |
| **Tier-2** | Programmatic tool calling (`execute_code`) | ✅ Done (PR #64) |
| **Tier-2** | Agent-authored skills gated by HotL | 🔲 Not started — HotL gate is in (PR #61), skill-authoring path is open |
| **Tier-2** | Session compaction for long local-model sessions | 🔲 Not started |
| **Tier-3** | OAuth 2.1 PKCE for authed MCP servers | 🔲 Not started |
| **Tier-3** | Compliance report export (SOC2/GDPR/HIPAA) from audit chain | 🔲 Not started |

### Possible future features worth a design conversation

- `execute_javascript` MCP server (deno or QuickJS, separate trust boundary)
- wasmtime + pyodide upgrade for the sandbox (most-isolated future path; the
  current crate's file layout supports adding a sibling `wasmtime_backend.rs`)
- Multi-agent dispatch (already a crate `xiaoguai-orchestrator`, scope undocumented)
- `xiaoguai-personas` (already a crate, undocumented)
- `xiaoguai-tasks` (Kanban-style, v1.4-ready, REST routes exist)
- `xiaoguai-anomaly` (Z-score/EWMA monitors, CLI wired)
- `xiaoguai-watch` (declarative active-wakeup watchers via SQL/HTTP)

---

## Documents and runbooks that already exist in the implementation repo

The next session should grep these — they're the authoritative source of truth
for what each component does.

```
/Users/zw/testany/myskills/xiaoguai/docs/
├── designs/
│   └── tier2-mcp-exec.md            ← full design rationale for the sandbox
├── runbooks/
│   ├── local-memory-and-redaction.md ← Tier-1b/PII operator guide
│   ├── cache-fallback.md             ← when to pick in-process vs Redis
│   ├── mcp-exec-sandbox.md           ← sandbox operator + threat model
│   └── operator.md                   ← general operator guide
├── HANDOFF-2026-05-26-v1.4-public.md  ← v1.4 baseline handoff
├── HANDOFF-2026-05-27-session2.md     ← session-2 wrap
└── ... (more handoffs, see git log)
```

Plus `/Users/zw/testany/myskills/xiaoguai/CLAUDE.md` is the project's primary
"long-form context" doc but it's mostly about VMware skills release management
— don't drown in it.

---

## Open design questions worth raising with the user

The previous session bounced off these without consensus — they're good
interview material:

1. **Single-tenant vs multi-tenant default posture**: today the code is
   multi-tenant-aware (RBAC + per-tenant memory/policy), but the air-gap mode
   is single-tenant in practice. What's the deployment story for SMB vs
   enterprise? Should there be a "personal" install mode?

2. **Skill marketplace governance**: skill-packs exist (`xiaoguai-skills` table,
   PR for v1.3 marketplace). Who can publish, who can install, what's the
   trust model? Important for the **agent-authored skills** Tier-2 item.

3. **Memory ownership and retention**: `memories` table has TTL but no
   automated purge job. GDPR right-to-be-forgotten flow?

4. **Multi-LLM routing strategy**: 11 providers wired into LlmRouter
   (`xiaoguai provider register`), Ollama is default. What's the operator
   policy for fallback order, cost vs latency vs privacy tradeoffs?

5. **`xiaoguai-orchestrator` scope**: undocumented crate. Multi-agent? Workflow
   engine? Both? Owner needs to articulate.

6. **`xiaoguai-personas` scope**: undocumented crate. Personality presets?
   Persona-scoped memory? Role-based system prompts?

7. **OAuth 2.1 PKCE for outbound MCP** (Tier-3): which MCP servers need
   authenticated access? Internal-corporate or public-marketplace?

8. **Eval harness scope**: `EvalService` is wired but `eval-suites/` empty.
   What's the eval philosophy — capability tests, regression tests, both?

9. **Production telemetry posture**: Prometheus + OTLP wired, redaction in
   place. What's exported, what's NOT? Operator opt-in or out?

10. **Frontend roadmap**: chat-ui Gemini-style redesign shipped, admin-ui
    exists. Mobile app? Multi-pane "today" view scope?

---

## Recommended new-session workflow

```
1. New Claude Code session opens here:
   cd /Users/zw/testany/myskills/xiaoguai-agent-design

2. Run /testany-eng:guide
   → it scans this dir + the implementation repo's docs/
   → tells you which phase to enter

3. Because xiaoguai is "implementation ahead of design":
   → SKIP /brd-interviewer and /uc-interviewer (those are for ideation)
   → START with /testany-eng:prd-writer in retrofit mode (capture WHAT IS)
   → use the question list above to drive interactive clarification

4. Then sequence:
   /testany-eng:prd-writer        → docs/prd-xiaoguai.md
   /testany-eng:api-writer        → docs/api-contract.md  (HTTP /v1/* + MCP tools)
   /testany-eng:hld-writer        → docs/hld.md
   /testany-eng:lld-writer        → docs/lld-{crate}.md   (one per major crate)
   /testany-eng:test-strategy-writer → docs/test-strategy.md
   /testany-eng:test-spec-writer  → docs/test-spec.md
   /testany-eng:runbook-writer    → docs/runbook.md
   /testany-eng:guardrails-writer → docs/guardrails.md    (Rust 1.88 MSRV, BUSL, etc.)

5. Reviewer skills should be paired after each writer:
   /testany-eng:prd-reviewer, /testany-eng:hld-reviewer, etc.
```

---

## What's already in this directory (don't blow it away)

```
/Users/zw/testany/myskills/xiaoguai-agent-design/
├── README.md                                    ← intro, preserve
├── HANDOFF-FOR-DESIGN-DOCS.md                   ← THIS file
└── docs/
    └── 2026-05-21-xiaoguai-agent-design.md      ← brainstorm phase, OUTDATED but useful as reference
```

The 2026-05-21 brainstorm doc reflects the **original vision** before
implementation. The retrofit PRD should note where reality diverged from that
vision.

---

## TL;DR for the next session

- Project is **xiaoguai**, a Rust agent platform, v1.4.0 shipped, 31 crates, 93k LOC.
- 4 PRs landed today (Tier-1a, Tier-1b, Tier-2 prereq, Tier-2 mcp-exec).
- Your job: read the existing code + runbooks, use testany-eng skills to capture
  the design retroactively, interview the user on the 10 open questions above.
- Output goes into `docs/` here, NOT into the implementation repo.
- Start with `/testany-eng:guide`, then `/testany-eng:prd-writer` (retrofit mode).
- Language: default 中文（user is 周伟, native zh, working zh+en mixed; respect
  the `output_language` knob in each skill).
