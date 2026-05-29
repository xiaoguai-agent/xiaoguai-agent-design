# LLD — `xiaoguai-chat-ui`

| | |
|---|---|
| Document ID | `LLD-CHAT-UI-001` |
| Refines | `DEC-HLD-025` (frontend monorepo, dual-SPA split), `BEH-CHAT-001` (SSE delta stream) |
| Verifies | `REQ-UI-001` (chat SPA covers chat surface), `REQ-UI-004` (HotL pending + Watch indicator + AI disclosure visible), `REQ-NFR-001` (first-token P95 ≤ 1.5 s observable in UI) |

## 1. Module purpose

`@xiaoguai/chat-ui` is the end-user-facing SPA — the conversational surface. React 18 + Vite + TypeScript single-page app deployed to `:5173` in dev. Its job is **one thing**: render a chat session as a streaming conversation between the user and an agent, with the harness's safety signals (HotL pending verdict, watch-loop status, AI-content disclosure per EU AI Act Art. 50(1)) made first-class UI elements.

This is the SPA that end users in the air-gapped enterprise deployment will actually open. The SPA does not host admin/audit/scheduler — those are in [`lld-admin-ui.md`](lld-admin-ui.md); the split is a deliberate trust/blast-radius boundary (DEC-025): end users never see operator-only data.

## 2. Public interface

```typescript
// frontend/chat-ui/src/App.tsx — top-level router
<BrowserRouter>
  <Routes>
    <Route path="/"                element={<ChatPage />} />     // default — new session
    <Route path="/sessions/:id"    element={<ChatPage />} />     // resume existing
    <Route path="/skills"          element={<SkillsPage />} />   // skill-pack browse
  </Routes>
</BrowserRouter>
```

Three routes only. Session list is rendered in the sidebar inside `ChatPage`; `:id` resumes the session by loading its messages from `GET /v1/sessions/{id}/messages`. The Skills page lists installed skill packs (`GET /v1/skills/installed`) — read-only consumer view (install/uninstall is admin-only).

## 3. Module structure

```
frontend/chat-ui/
├── package.json                          # @xiaoguai/chat-ui v0.2.0; React 18 + Vite + i18next + react-markdown
├── vite.config.ts                        # dev :5173, proxy /v1 → :8080
└── src/
    ├── main.tsx                          # bootstrap + i18n + client
    ├── App.tsx                           # router + sidebar SessionList
    ├── pages/
    │   ├── ChatPage.tsx                  # ~15 kB; the entire conversation surface
    │   ├── ChatPage.test.tsx
    │   └── Skills.tsx
    ├── components/
    │   ├── SessionList.tsx               # left sidebar — new / select / fork
    │   ├── Composer.tsx                  # message input + send + cancel + attach
    │   ├── MessageBubble.tsx             # markdown render + code block + citation
    │   ├── WatchIndicator.tsx + test     # running / paused / error pill (top-right of conversation)
    │   ├── AiDisclosureBanner.tsx + test # EU AI Act Art. 50(1) disclosure (configurable)
    │   ├── HotlPendingPanel.tsx          # when HotL gate suspends a tool call
    │   ├── RecentOutcomesPanel.tsx       # tail of outcome stream for the session
    │   └── ThemeToggle.tsx               # dark default (DEC-025 §3)
    ├── stream/
    │   └── agentEventStream.ts           # SSE consumer for AgentEvent (text_delta, tool_call_*, hotl_pending, outcome_recorded, final)
    ├── hooks/
    │   ├── useSession.ts                 # session lifecycle
    │   └── useStreamingMessage.ts        # accumulates text_delta into the current bubble
    └── i18n/                             # zh-Hans + en
```

The conversation engine is **one page (`ChatPage.tsx`)** plus a small set of components. State lives in `useState` / `useRef`; no global store (same rationale as admin-ui — DEC-LLD-ADMIN-UI-001 applies here too).

## 4. Key flows

### 4.1 Send a message → first token (BEH-CHAT-001, REQ-NFR-001)

```
user types → Composer.onSubmit
        │
        ▼
client.sendMessage(sessionId, text)   ──► POST /v1/sessions/{id}/messages
        │                                  (SSE response, AbortController in place)
        │
        ▼
agentEventStream.ts iterates events:
        │
        ├─ text_delta       → useStreamingMessage appends to current bubble; streaming=true
        │                     first text_delta arrival = first-token-time (perf P95 target)
        ├─ tool_call_started → bubble shows "calling <tool>…" inline tool chip
        ├─ tool_call_finished → tool chip resolves with result preview
        ├─ hotl_pending     → suspend stream; render HotlPendingPanel (§4.3)
        ├─ outcome_recorded → push to RecentOutcomesPanel
        └─ final            → streaming=false; persist message to session view
```

The Composer is **never blocked** on the stream — the user can type the next message while the previous one streams. On a second submit while streaming, the previous AbortController fires → `POST /v1/sessions/{id}/cancel` → server emits `final{stop_reason: Cancelled}` → bubble marks "cancelled" → new message starts.

### 4.2 Resume a session (`/sessions/:id`)

```
ChatPage mount with :id
        │
        ▼
client.listMessages(id)  → render historical bubbles
client.getSession(id)    → header shows persona, model, watcher status
client.listSessionWatchers(id) [404-safe] → WatchIndicator state
        │
        ▼
SSE not opened until the user submits a new message — historical replay is a pure read.
```

If `listMessages` returns 404, the route renders "session not found; start a new one" CTA. No silent redirect.

### 4.3 HotL pending — the suspended-tool-call UX (§DEC-006, REQ-UI-004)

When the agent decides to call a tool that falls into a `policy.default_verdict=escalate` scope, the backend emits `hotl_pending{ tool, args_redacted, scope, request_id }` and suspends the agent loop. The chat-ui:

1. Pauses the streaming bubble at the boundary; renders a `<HotlPendingPanel>` inline (not a modal — operators may scroll up and look at the conversation context).
2. The panel shows the tool name, redacted args, the policy scope, the elapsed timeout, and three buttons: **Approve** / **Deny** / **Approve & remember (raise policy)**.
3. Approve → `POST /v1/hotl/decisions` (request_id, verdict=allow); backend resumes the loop; SSE continues.
4. Deny → same endpoint with verdict=deny; backend rejects the tool call; agent receives the rejection in the loop and decides what to do next.
5. "Approve & remember" opens a small form to create a `HotlPolicy` raising the verdict to `allow` for this `{scope, tool}` pair so the same call does not page again. This is a chat-ui convenience — the underlying API is the admin one (`POST /v1/hotl/policies`).

If the user closes the tab during a pending verdict, the request stays open until the policy default timeout (backend default: 24h). On reconnect, the panel re-renders from the SSE replay.

#### 4.3.1 In-conversation Approve / Deny — sprint-11 status

> **Sprint-11 status (2026-05-30) — drift to close.** As of v1.8.0 `frontend/chat-ui/src/HotlBanner.tsx` renders only an informational banner with a "Review in approval queue →" link out to admin-ui — it does **not** yet expose the three inline action buttons (**Approve** / **Deny** / **Approve & remember**) specified above. The chat-ui SSE-handling code already pauses the streaming bubble on `hotl_pending`, so the gap is purely the inline affordances + `POST /v1/hotl/decisions` wiring. The `frontend/e2e/tests/chat-ui/chat-hotl-suspend-resume.spec.ts` Playwright spec carries a `test.fixme()` placeholder for the inline-decision case. Gap-close checklist owned by sprint-11 task **S11-3**:
>
> 1. Promote `HotlBanner` to a `HotlPendingPanel` (or extend it) that renders the three buttons in the order Approve → Deny → Approve & remember; keep the existing "Review in approval queue →" link as a fall-through for operators who prefer the admin queue.
> 2. Add `XiaoguaiClient.submitHotlDecision({request_id, verdict, raise_policy?})` in `frontend/shared/src/index.ts` posting to `/v1/hotl/decisions`.
> 3. "Approve & remember" opens a small form to capture `{scope, tool}` and chains a `POST /v1/hotl/policies` call (admin endpoint, see §4.3 step 5); surface auth errors as a non-fatal toast — the inline panel must keep working when raising a policy fails.
> 4. Optimistic UX: on Approve/Deny click, the panel switches to a "submitting…" state; the SSE replay's eventual `final` or `hotl_resolved` event clears the panel. Failure → revert to interactive + show error inside the panel.
> 5. Flip the `test.fixme()` marker off in `chat-hotl-suspend-resume.spec.ts` (or split into a sibling `chat-hotl-inline-decision.spec.ts` as the e2e author noted) — it becomes the acceptance test for S11-3.
>
> The link-out behaviour stays as a tested fallback for operators on browsers where the SSE stream is unreliable; it is **not** removed. **(Carry: refines DEC-LLD-CHAT-UI-002 — the inline panel is still inline; sprint-11 fleshes out the affordances DEC-LLD-CHAT-UI-002 already specifies.)**

### 4.4 Watch indicator (BEH-CHAT-001 sibling)

The `<WatchIndicator>` pill shows the running state of any `xiaoguai-watch` watcher attached to the session (`running` / `paused` / `error`). It is read from `listSessionWatchers(id)` on mount and updated by the SSE `watch_event` (sprint-10b backend task: emit this event over the existing message SSE stream). Click → opens a dropdown with pause / resume buttons → `POST /v1/sessions/{id}/watchers/{watcher_id}/{pause|resume}` (sprint-10b backend task; currently 404).

### 4.5 AI disclosure banner (EU AI Act Art. 50(1))

Top of `ChatPage`: `<AiDisclosureBanner>` renders a one-line disclosure stating the conversational partner is an AI system. Configurable via `getAiDisclosureConfig()` in `@xiaoguai/shared` — operators can:

- Disable entirely (non-EU deployments).
- Customise the wording (translations, brand voice).
- Set `severity=mandatory` to make the banner non-dismissible (default).

The default text is the EU Commission's plain-language template; jurisdictions with stricter requirements override via config.

### 4.6 Skill packs read-only view

`/skills` lists installed skill packs from `GET /v1/skills/installed`. Each card: name, version, signature status (`signed/unsigned/revoked`), tools provided, install date. No install/uninstall here — admins use admin-ui. Click → drawer with the full manifest (read-only `<SkillManifestPreview>` — same component as admin-ui's SkillProposals pane uses, lives in `@xiaoguai/shared` per DRY).

### 4.7 Failure modes

| Failure | Detection | Recovery |
|---|---|---|
| SSE drop mid-stream | reader error inside `XiaoguaiClient.sendMessage` (current impl uses `fetch` + `getReader()`, not `EventSource`) | Banner "connection lost; reconnecting…" (`data-testid="sse-reconnect-banner"`); backoff exponential (1 s → 2 s → 4 s → 8 s → 16 s, cap 30 s); on stream resume clear banner; if no reconnect in 30 s → toast + manual retry button. **The partial bubble is preserved**, never deleted. See §4.7.1 for the sprint-11 status. |
| HotL pending — user closes tab | None client-side | On reload, `listMessages` replay includes the pending event; `<HotlPendingPanel>` re-renders. Server-side timeout (24h default) drives eventual deny. |
| Composer submit while streaming | Local state guard | Confirm dialog "cancel current response?"; cancel route as §4.1 |
| Backend older than SPA (e.g. no `watch_event` in SSE) | Unknown event kind in `agentEventStream.ts` | Ignored silently; WatchIndicator stays in initial state |
| `sendMessage` 429 (rate limited) | XiaoguaiClient typed error | Bubble marked "rate limited; retry in N s"; backoff per `Retry-After` |
| Network offline | `navigator.onLine === false` | Composer disabled; banner "offline"; on `online` event re-enabled |

#### 4.7.1 SSE reconnect banner — sprint-11 status

> **Sprint-11 status (2026-05-30) — drift to close.** As of v1.8.0 `XiaoguaiClient.sendMessage()` in `frontend/shared/src/index.ts` calls `onError?(err)` on a stream/reader failure and stops; there is **no** automatic retry loop and **no** reconnect banner component. The partial-bubble-preservation property (DEC-LLD-CHAT-UI-003) already holds because the reader appends as it reads and never tears down the bubble on error; that case is covered by a passing e2e test. The `frontend/e2e/tests/chat-ui/chat-sse-reconnect.spec.ts` Playwright spec carries a `test.fixme()` placeholder for the reconnect-banner case. Gap-close checklist owned by sprint-11 task **S11-2**:
>
> 1. `ChatPage` (or a small surrounding `<SseLifecycle>` component) exposes a banner UI hook rendered when the SSE stream is in a `reconnecting` state, with `data-testid="sse-reconnect-banner"` so the e2e assertion is stable.
> 2. `XiaoguaiClient.sendMessage()` gains an internal retry loop: on a non-`AbortError` reader/network failure during streaming, sleep with exponential backoff (1 s, 2 s, 4 s, 8 s, 16 s; cap 30 s) and re-issue the POST with a `resume_from_sequence` parameter (or, if the backend cannot resume, restart the in-progress message — server-side dedup makes this safe). After 30 s with no success → emit `onError` with a `kind: "reconnect_failed"` `StreamError` and surface a manual retry button.
> 3. The retry callback is observable: `sendMessage` accepts an optional `onReconnect?(attempt: number)` so `ChatPage` can flip the banner UI on/off.
> 4. The reconnect path **must not** delete the partial bubble (DEC-LLD-CHAT-UI-003) — the reader keeps the same `useStreamingMessage` buffer; only `streaming=true` stays asserted across the gap.
> 5. Flip the `test.fixme()` marker off in `chat-sse-reconnect.spec.ts` — it becomes the acceptance test for S11-2.
>
> The corresponding row in the §4.7 failure-mode table above has been updated to point here and to clarify that the impl uses `fetch` + reader, not `EventSource`. **(Carry: refines DEC-LLD-CHAT-UI-003 — preserved-partial guarantee survives the reconnect.)**

## 5. Error handling

Same `ApiError` discriminated union as admin-ui (defined in `@xiaoguai/shared`). Chat-ui never silently swallows; every error becomes a visible bubble annotation or a banner. The SSE stream additionally produces a `StreamError` union:

```typescript
type StreamError =
  | { kind: "Dropped",  partialMessage: Message }
  | { kind: "ParseFailed", offendingLine: string }
  | { kind: "AgentLoopError", message: string };   // from final{stop_reason: Error}
```

The `partialMessage` field is what lets us preserve user work on a dropped SSE.

## 6. Concurrency / state

- One SSE per active stream; closed via AbortController on user-initiated cancel.
- Composer and the current streaming bubble are decoupled — composer remains responsive.
- Session list in sidebar is shared via React Context (chat-ui only; not a global store — context scope is the route tree).
- No service worker; reconnect logic is in-page (DEC-025 §3 — service workers add a cache-invalidation surface that air-gapped deployments do not need).

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (Vitest) | `agentEventStream` parse fixtures for every `AgentEvent` kind including the sprint-10b `watch_event`; `useStreamingMessage` accumulation correctness; `getAiDisclosureConfig` precedence (env → config file → default); `<AiDisclosureBanner>` mandatory-mode non-dismissible. |
| Component (Vitest + RTL) | Existing: `ChatPage.test.tsx`, `AiDisclosureBanner.test.tsx`, `WatchIndicator.test.tsx`. Sprint-10b adds: `HotlPendingPanel.test.tsx` (approve/deny/remember branches), `Composer.test.tsx` (cancel-during-stream interaction). |
| E2E (Playwright `@xiaoguai/e2e`) | Existing: `e2e/tests/chat-ui/golden-path.spec.ts` (168 LOC; new session → send → first delta → final). Sprint-10b adds: `chat-hotl-suspend-resume.spec.ts` (full HotL pending flow against test fixture), `chat-sse-reconnect.spec.ts` (kill the SSE; assert partial preserved + reconnect), `chat-ai-disclosure-mandatory.spec.ts` (banner not dismissible in mandatory mode). |
| Performance | `chat-ui` Playwright project measures first-token latency against a stubbed Ollama; assert P95 ≤ 1.5 s (REQ-NFR-001). |
| Accessibility | Axe-core on golden path — chat input is keyboard-only navigable; bubbles announce to screen readers when streaming completes (`aria-live="polite"`). |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-CHAT-UI-001
  type: LLD
  title: xiaoguai-chat-ui LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-29
  updated_at: 2026-05-30
  source_documents: [HLD-XIAOGUAI-001, PRD-XIAOGUAI-001, API-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-CHAT-UI-001, title: "Composer never blocks on SSE", statement: "User can type the next message while a previous response streams; concurrent submit triggers cancel + new stream.", status: approved, scope: in, decision: "Decoupled composer.", rationale: "Long agent responses must not paralyse the UI; cancel route already exists at /v1/sessions/{id}/cancel." }
    - { id: DEC-LLD-CHAT-UI-002, title: "HotL pending is inline panel, not modal", statement: "Suspended tool-call decisions render inline in the conversation; users may scroll up for context.", status: approved, scope: in, decision: "Inline panel.", rationale: "Operator decisions need full conversation context; modal hides the surrounding bubbles." }
    - { id: DEC-LLD-CHAT-UI-003, title: "Partial bubble preserved on SSE drop", statement: "On EventSource error, the partial streamed text is retained as 'partial' annotation; never deleted on reconnect.", status: approved, scope: in, decision: "Preserve partial.", rationale: "Local-first ethos (R.E.S.T Reliability); losing user-visible work on a network blip is unacceptable." }
    - { id: DEC-LLD-CHAT-UI-004, title: "AI disclosure mandatory by default", statement: "AiDisclosureBanner renders mandatory (non-dismissible) by default to satisfy EU AI Act Art. 50(1); operators may override per deployment.", status: approved, scope: in, decision: "Mandatory default.", rationale: "Compliance-by-default; opt-out is an operator decision, not an end-user decision." }
  flows:
    - { id: FLOW-LLD-CHAT-UI-001, title: "Send → first token", statement: "Composer → POST /v1/sessions/{id}/messages → SSE → text_delta → bubble", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-CHAT-UI-002, title: "HotL suspend → resume", statement: "hotl_pending event → HotlPendingPanel → POST /v1/hotl/decisions → SSE resumes", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-CHAT-UI-001, type: refines, from: FLOW-LLD-CHAT-UI-001, to: BEH-CHAT-001, status: active }
  - { id: REL-LLD-CHAT-UI-002, type: refines, from: FLOW-LLD-CHAT-UI-002, to: DEC-HLD-006, status: active }
  - { id: REL-LLD-CHAT-UI-003, type: refines, from: DEC-LLD-CHAT-UI-004, to: DEC-HLD-025, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
