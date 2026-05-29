# Design-doc update pass — input briefing

> Read by the next `cd /Users/zw/testany/myskills/xiaoguai-agent-design && claude` session.
> Phase-1 prep has already been done from the *implementation* repo
> session on 2026-05-28; this file is the bridge.

## Read order

1. **This file** (you are here).
2. [`docs/harness-engineering.md`](docs/harness-engineering.md) — newly
   added philosophy doc; vocabulary the rest of the work will use.
3. [`docs/RELEASE-LOG.md`](docs/RELEASE-LOG.md) — the running ledger of
   updates to this design repo.
4. [`/Users/zw/testany/myskills/xiaoguai/docs/HANDOFF-2026-05-28-session5.md`](../xiaoguai/docs/HANDOFF-2026-05-28-session5.md)
   — what was delivered in session-5 of the implementation repo.
5. [`/Users/zw/testany/myskills/xiaoguai/docs/plans/2026-05-28-retro-design-docs.md`](../xiaoguai/docs/plans/2026-05-28-retro-design-docs.md)
   — **Plan C** (this work). The §2 success criteria are the
   definition-of-done for the interactive Phase 2.
6. Existing design docs under `docs/` (hld.md, prd-xiaoguai.md, etc.).

## What's been done already (Phase 1 prep, 2026-05-28)

- [x] `docs/harness-engineering.md` — new philosophy doc covering R.E.S.T,
      REPL container, control/data plane split, sandbox tiering, policy
      gateway, six design principles, with each concept mapped to a
      Xiaoguai module.
- [x] `docs/hld.md` — top-of-doc "Philosophy" pointer added.
- [x] `docs/RELEASE-LOG.md` — created (this update is the first row).
- [x] Implementation repo cross-link committed at
      `xiaoguai/docs/architecture/design-link.md`.
- [x] Implementation-repo PRs covering session-5 deltas:
      - PR #66 — session compaction (Plan D.2 code)
      - PR #67 — systemd ExecStart + README install matrix (Plan B safe)
      - PR #68 — mcp-exec E2E demo script + runbook (Plan A docs)
      - PR #65 — the four task plans themselves

## What this session should do (Phase 2)

Walk plan C §4.5–4.9. The 10 success criteria from plan C §2 map to:

| # | Criterion | Likely edit target |
|---|---|---|
| 1 | `adr/` directory in design repo mirroring implementation ADRs | New `docs/adr/` with one short stub per implementation ADR (~18 stubs) |
| 2 | HLD §2 module map includes in-process cache fallback (#60) and clarifies `xiaoguai-core` legacy shim | `docs/hld.md` §2 |
| 3 | HLD has a "HotL gating and the policy gateway" section pointing at philosophy §9 | `docs/hld.md` §6.5 |
| 4 | PRD has a sandboxed-code-execution sub-requirement (#64) | `docs/prd-xiaoguai.md` |
| 5 | Guardrails: redact-before-HMAC ordering + mcp-exec env allowlist | `docs/guardrails.md` §3 |
| 6 | LLD agent has HotL gate subsection (#61) | `docs/lld/lld-agent.md` |
| 7 | Every LLD has a header link to philosophy + HLD | `docs/lld/*.md` (12 files) |
| 8 | testany-eng reviewer skills report no CRITICAL/HIGH | Reviewer skills, run after each edit |
| 9 | RELEASE-LOG.md is current | `docs/RELEASE-LOG.md` |
| 10 | docs/README.md links top-level docs + philosophy | `docs/README.md` (create if absent) |

## How to drive

Open this dir in a fresh Claude Code session so the testany-eng plugin
loads. Then, in order:

```
/testany-eng:guide
```

(Confirm the flow shape — the prd-writer/hld-writer/lld-writer skills
are for new work; we use the *reviewer* skills here.)

For each criterion above, edit the file *manually* (do not regenerate
from scratch — the v1.4 retrofit is good). After each file, run the
matching reviewer:

```
/testany-eng:hld-reviewer
/testany-eng:prd-reviewer
/testany-eng:lld-reviewer
/testany-eng:guardrails-reviewer
```

CRITICAL/HIGH findings must be addressed before committing.
MEDIUM findings can be deferred via TODO comments.

## Conventions (matching the v1.4 retrofit)

- Do **not** change `Document ID:` fields — they're stable identifiers.
- Do **not** rewrite sections from scratch; surgical additions only.
- Each new section ends with a "Last updated: YYYY-MM-DD" line so the
  next pass can diff.
- Cross-link with relative paths (`[`harness-engineering.md`](harness-engineering.md)`),
  not absolute.
- The design repo is **English-only** by convention; do not translate.

## When you're done

1. `git init && git remote add origin <url>` if this repo isn't versioned yet
2. Commit: `docs: post-session-5 update pass (Harness Engineering + Tier-1/2 delta)`
3. Push.
4. Open a tracking issue in the implementation repo referencing the
   commit so future contributors can find the design doc deltas.

## What's out of scope (per plan C §7)

- Writing a fresh PRD from scratch.
- Adding ADRs for things that *should* have an ADR but don't (track
  separately).
- Chinese translation.
- Mermaid / PlantUML diagrams.
- Cost / pricing sections.

## Related files

- This file: `HANDOFF-DESIGN-UPDATE.md`
- Plan: `/Users/zw/testany/myskills/xiaoguai/docs/plans/2026-05-28-retro-design-docs.md`
- Philosophy doc just added: `docs/harness-engineering.md`
- Release log: `docs/RELEASE-LOG.md`
- Cross-link from implementation repo: `/Users/zw/testany/myskills/xiaoguai/docs/architecture/design-link.md`
