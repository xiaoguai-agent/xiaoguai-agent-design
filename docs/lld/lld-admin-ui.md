# LLD — `xiaoguai-admin-ui`

| | |
|---|---|
| Document ID | `LLD-ADMIN-UI-001` |
| Refines | `DEC-HLD-025` (frontend monorepo, dual-SPA split), `BEH-AUDIT-001`, `BEH-CHAT-001` |
| Verifies | `REQ-UI-002` (admin SPA covers all admin/* endpoints), `REQ-UI-003` (HMAC chain visualisation), `REQ-UI-005` (i18n + a11y baseline) |

## 1. Module purpose

`@xiaoguai/admin-ui` is the operator-facing SPA. It is a React 18 + Vite + TypeScript single-page app deployed to `:5174` in dev and to a static-asset bucket (or nginx sidecar) in prod. Its job is to surface every Casbin-gated admin capability the backend exposes — audit, scheduler, eval, HotL policies, tenants, MCP marketplace, skills, outcomes, anomaly, kanban, memory — through pane-based navigation. It does **not** host the conversational chat surface (that is `@xiaoguai/chat-ui`, [`lld-chat-ui.md`](lld-chat-ui.md)); the split is a deliberate trust/blast-radius boundary (DEC-025).

The SPA consumes one type-safe client (`@xiaoguai/shared::XiaoguaiClient`) — there is no second HTTP layer. Authentication is delegated entirely to the surrounding reverse proxy / OIDC; the SPA receives the bearer token from `VITE_API_URL` env + `Authorization` header injection at the proxy. No login page lives in this SPA (DEC-025 §3).

## 2. Public interface

The "interface" of a frontend module is its **route table + the API surface it consumes**. The Rust types in `xiaoguai-types` are the wire contract; admin-ui mirrors them via `@xiaoguai/shared`.

```typescript
// frontend/admin-ui/src/App.tsx — top-level router
<BrowserRouter>
  <Routes>
    <Route path="/today"          element={<TodayPane />} />        // default landing
    <Route path="/scheduler"      element={<SchedulerPane />} />
    <Route path="/eval"           element={<EvalPane />} />
    <Route path="/usage"          element={<UsagePane />} />
    <Route path="/outcomes"       element={<OutcomesPane />} />
    <Route path="/anomaly"        element={<AnomalyPane />} />
    <Route path="/kanban"         element={<KanbanPane />} />
    <Route path="/marketplace"    element={<MarketplacePane />} />
    <Route path="/skills"         element={<SkillPacksPane />} />
    <Route path="/mcp-servers"    element={<McpServersPane />} />
    <Route path="/tenants"        element={<TenantsPane />} />
    <Route path="/providers"      element={<ProvidersPane />} />
    <Route path="/hotl-policies"  element={<HotlPoliciesPane />} />
    <Route path="/audit"          element={<AuditPane />} />
    <Route path="/memory"         element={<MemoryPane />} />
    {/* sprint-10b additions (DEC-025): */}
    <Route path="/personas"          element={<PersonasPane />} />
    <Route path="/skill-proposals"   element={<SkillProposalsPane />} />
  </Routes>
</BrowserRouter>
```

Single-pane budget: each pane must (1) render an empty/loading/error state, (2) call only the `XiaoguaiClient` methods it needs (no direct `fetch`), (3) degrade gracefully when an endpoint returns 404 (the backend may be older than the SPA), (4) emit no PII to console.

## 3. Module structure

```
frontend/admin-ui/
├── package.json                          # @xiaoguai/admin-ui v0.2.0; React 18 + Vite + recharts + i18next
├── vite.config.ts                        # dev :5174, proxy /v1 → :8080
├── public/                               # static assets, i18n bundles
└── src/
    ├── main.tsx                          # bootstrap + i18n init + client construction
    ├── App.tsx                           # router + sidebar nav (14 panes → 16 in sprint-10b)
    ├── layout/
    │   ├── Sidebar.tsx                   # nav + tenant switcher
    │   └── TopBar.tsx                    # tenant / scope / theme toggle
    ├── panes/                            # one file per route — pane is a self-contained feature unit
    │   ├── Today.tsx          + Today.test.ts
    │   ├── Audit.tsx                     # HMAC chain visualisation
    │   ├── Scheduler.tsx                 # job editor + fire-now
    │   ├── Eval.tsx                      # suite list + run
    │   ├── Usage.tsx                     # token timeseries (recharts)
    │   ├── Outcomes.tsx                  # raw + summary + timeseries
    │   ├── Anomaly.tsx        + Anomaly.test.ts        # z-score / EWMA dashboard
    │   ├── Kanban.tsx         + Kanban.test.ts         # 6-column board (DEC-025 §4 ADR-0019)
    │   ├── Marketplace.tsx               # MCP one-click install
    │   ├── SkillPacks.tsx     + SkillPacks.test.ts
    │   ├── McpServers.tsx
    │   ├── Tenants.tsx
    │   ├── Providers.tsx                 # placeholder pane (LLM router config — backend stub)
    │   ├── HotlPolicies.tsx   + HotlPolicies.test.ts   # budget rules CRUD
    │   ├── Memory.tsx         + Memory.test.ts         # 404-safe (ADR-0019, task #155 in flight)
    │   ├── Personas.tsx                  # NEW sprint-10b
    │   └── SkillProposals.tsx            # NEW sprint-10b — approve/reject T3 proposals
    ├── components/                       # shared widgets (Table, Card, JsonViewer, ChainBadge)
    ├── hooks/                            # useClient, useTenant, useFeature (RBAC scope gate)
    └── i18n/                             # zh-Hans + en bundles (a11y labels live here)
```

Sprint-10b adds two panes (Personas, SkillProposals) and one widget category (RBAC scope guard `<RequireScope name="audit.export">`). No restructuring of the existing 14 panes.

## 4. Key flows

### 4.1 Pane lifecycle (canonical)

Every pane follows the same three-state contract — encoded in a shared `<PaneShell>` wrapper (sprint-10b cleanup task):

```
Mount → useEffect(() => client.<read>())
        │
        ├─► Loading  (skeleton rows / spinner)
        │
        ├─► Error    (typed by XiaoguaiClient — 401/403/404/5xx render distinct copy)
        │
        └─► Ready    (data table / chart / form)
                │
                └─► Mutation (POST/PATCH/DELETE) → optimistic update + revert-on-error
```

No global store. Pane-local `useState` is the default; cross-pane state is the URL (tenant switcher writes `?tenant=X`).

### 4.2 Audit pane — HMAC chain visualisation (BEH-AUDIT-001)

```
GET /v1/admin/audit?limit=100&tenant=<id>
        │
        ▼
Render rows with ChainBadge column:
        ├─ green  → row's hmac_prev matches predecessor (verified)
        ├─ amber  → row signed under rotation-window key (still valid, see DEC-LLD-AUD-002)
        └─ red    → broken chain — show `broken_at_sequence` from verify endpoint

On row click → JsonViewer modal: entry body (post-redaction), hmac_prev, hmac_self, signing_key_id.
On "Export…" → POST /v1/audit/exports {framework: soc2|gdpr|hipaa}; SSE progress; download ChainProof bundle.
```

The pane shows the redaction state explicitly — an `<AbsenceMarker fields={['email','phone']} />` widget renders the placeholder positions so operators understand the chain signs the post-redaction payload (DEC-004, REQ-NFR-007).

### 4.3 Kanban pane — auto-dispatch + block (ADR-0019)

Six fixed columns: `triage → todo → ready → running → blocked → done`. Drag handle calls `PATCH /v1/tasks/{id}/column`. Block button opens reason modal → `POST /v1/tasks/{id}/block`. Dispatch button on the board header → `POST /v1/tasks/dispatch?board=X` (server selects next eligible task per persona-budget rules).

Failure modes: backend 404 on board → render "no board configured; create one" CTA (degrade, do not crash). Concurrent move conflict (409) → snap card back + toast "moved by another operator; refresh".

### 4.4 HotL Policies pane — budget rules CRUD

Form fields mirror `xiaoguai-tasks::HotlPolicy`: scope (tool name / persona id / tenant), budget (`{calls_per_hour, calls_per_day}`), default verdict (`allow|deny|escalate`). Submission is `POST` or `PUT /v1/hotl/policies`. The pane refuses to submit a policy whose `default_verdict=deny` without a confirm modal (operator-experience guard, not a security guard — the backend enforces).

### 4.5 Scheduler pane — job editor + fire-now (REQ-NFR-005)

Editor renders the six trigger types (cron / interval / once / webhook / on-event / on-startup) as discriminated-union forms. Save → `POST /v1/admin/scheduler/jobs`. Webhook tokens managed via `/v1/admin/scheduler/tokens`. Fire-now button calls `POST /v1/admin/scheduler/jobs/{id}/fire-now` with audit-log link returned in response → toast "fired; audit row #N" with click-through.

### 4.6 Personas pane (sprint-10b NEW)

Lists personas from `GET /v1/personas` (route exists at `xiaoguai-personas/src/routes.rs:52-64` but is not yet mounted on the main router — sprint-10b backend task). CRUD form mirrors `Persona { id, name, system_prompt, model_preference, memory_view_scope, tags? }`. Tag chips show `role/planner|worker|critic` per DEC-021 (sprint-9). Memory-view scope dropdown surfaces the `persona_scoped_view_id` from `xiaoguai-memory`. On submit → `POST /v1/personas` or `PUT /v1/personas/{id}`.

### 4.7 Skill Proposals pane (sprint-10b NEW)

Lists agent-authored skill proposals from `GET /v1/skills/proposals` (DEC-014, sprint-7 T3). Each row: proposer agent, persona, requested tool composition (read-only `<SkillManifestPreview>`), submission timestamp, HotL bucket usage. Approve button → `POST /v1/skills/proposals/{id}/approve` (Casbin `skill.approve` scope required — pane wraps in `<RequireScope name="skill.approve">`). Reject button opens reason modal → `POST /v1/skills/proposals/{id}/reject`. Audit row link in response toast.

### 4.8 RBAC scope gate (sprint-10b NEW)

Casbin scopes the backend enforces (e.g. `audit.export`, `skill.approve`, `tenants.write`) need a frontend hint so operators see "Forbidden" upfront instead of clicking and getting a 403. The mechanism is one component:

```tsx
<RequireScope name="audit.export">
  <ExportButton />
</RequireScope>
```

The scope list is fetched once per session from a new `GET /v1/admin/me/scopes` endpoint (sprint-10b backend task). On absence (older backend), the gate fails open — render the child and let the backend enforce. This is **UX**, not security; the backend is the enforcement point (PHILO §17 "Bypassable compliance gates" anti-pattern).

### 4.9 i18n + a11y

`react-i18next` with `zh-Hans` + `en` bundles. All visible strings go through `t()`; aria labels and screen-reader-only headings are in the same bundles. The lint rule (`@xiaoguai/admin-ui` ESLint config) forbids string literals in JSX children outside `t()` (sprint-10b will introduce). Color tokens use `prefers-color-scheme`; dark mode is the default (matches operator workflow).

### 4.10 Failure modes

| Failure | Detection | Recovery |
|---|---|---|
| Backend endpoint 404 (older deploy than SPA) | XiaoguaiClient typed error | Render "feature not available in this deployment"; pane stays functional for available data |
| 401 (token expired) | `XiaoguaiClient` 401 error | Redirect to surrounding-app login (via `window.location` to `VITE_LOGIN_URL`); SPA never tries to refresh |
| 403 (scope missing) | XiaoguaiClient 403 error | `<RequireScope>` rendered "Forbidden" copy; action disabled, no retry |
| 5xx | Toast + retry button | No silent retry; operator decides |
| SSE drop on Today timeline | `EventSource.onerror` | Auto-reconnect with exponential backoff; banner "reconnecting…"; clear on `onopen` |
| Drag-and-drop conflict on Kanban (409) | XiaoguaiClient 409 | Card snaps back; toast "concurrent edit; refresh" |
| i18n key missing | `t()` fallback to key name | Renders the key as visible string; CI lint catches missing keys before merge |

## 5. Error handling

Frontend errors are values, not exceptions, modulo React boundaries. `XiaoguaiClient` returns `Result<T, ApiError>` (TypeScript discriminated union):

```typescript
type ApiError =
  | { kind: "Unauthorized" }                    // 401
  | { kind: "Forbidden", scope?: string }       // 403
  | { kind: "NotFound", path: string }          // 404
  | { kind: "Conflict", detail?: string }       // 409
  | { kind: "Server", status: number, message?: string }  // 5xx
  | { kind: "Network", cause: string };         // fetch threw
```

Top-level `<ErrorBoundary>` in `App.tsx` catches render-time exceptions (component bugs); it does **not** catch API errors — those are handled per-pane.

## 6. Concurrency / state

- No global store. Pane-local `useState`/`useReducer` only.
- The `XiaoguaiClient` is one instance per app, constructed in `main.tsx`. It is stateless beyond the bearer token; safe to share.
- SSE streams (`Today` timeline live updates; future watch-event feed) use `EventSource`; one per pane; closed on unmount.
- URL is the source of truth for tenant context (`?tenant=X`); a tenant switch is a route navigation, not a state mutation.
- Mutations are optimistic + revert-on-error; for chained mutations (e.g. Kanban move + block) the pane serialises (no concurrent in-flight per row).

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (Vitest) | Pane render contracts: Loading / Error / Ready / Empty for each of the 16 panes (existing 14 + Personas + SkillProposals); `XiaoguaiClient` error union narrowing; `<RequireScope>` open-fail behaviour; i18n key coverage (all visible strings have zh-Hans + en); `<ChainBadge>` colour decision matrix on Audit pane. |
| Component (Vitest + React Testing Library) | `Anomaly.test.ts`, `Kanban.test.ts`, `SkillPacks.test.ts`, `HotlPolicies.test.ts`, `Memory.test.ts` — exist. Sprint-10b adds `Personas.test.ts`, `SkillProposals.test.ts`, `Audit.test.ts` (chain badge + export modal). |
| E2E (Playwright `@xiaoguai/e2e`) | Existing: `e2e/tests/admin-ui/golden-path.spec.ts` (173 LOC; covers Today → Audit → Scheduler smoke). Sprint-10b adds: `admin-personas.spec.ts`, `admin-skill-proposals.spec.ts`, `admin-audit-export.spec.ts` (HMAC chain → export → ChainProof download). Multi-browser matrix (Chrome / Firefox / Safari) per existing `playwright.config.ts:19-108`. |
| Accessibility | Axe-core integrated in Playwright on the same golden paths (sprint-10b adds); fail on serious/critical violations. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-ADMIN-UI-001
  type: LLD
  title: xiaoguai-admin-ui LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-29
  updated_at: 2026-05-29
  source_documents: [HLD-XIAOGUAI-001, PRD-XIAOGUAI-001, API-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-ADMIN-UI-001, title: "Pane-local state, no global store", statement: "Each pane owns its state via useState/useReducer. No Redux/Zustand. Cross-pane state lives in the URL.", status: approved, scope: in, decision: "Local state.", rationale: "14+ pane SPA; global store invites cross-feature coupling. URL as state is browser-native." }
    - { id: DEC-LLD-ADMIN-UI-002, title: "RBAC scope gate is UX, backend enforces", statement: "<RequireScope> hides actions when scope is absent; fails open when /me/scopes is unavailable. Backend Casbin is the enforcement point.", status: approved, scope: in, decision: "UX hint only.", rationale: "Frontend gates are bypassable; only backend gates are safe (PHILO §17)." }
    - { id: DEC-LLD-ADMIN-UI-003, title: "No login page in this SPA", statement: "Authentication is delegated to the surrounding reverse proxy / OIDC; SPA reads bearer from injected header. On 401 → redirect to VITE_LOGIN_URL.", status: approved, scope: in, decision: "Delegated auth.", rationale: "Air-gapped deployments use Kong / nginx / corporate SSO; one OIDC integration in the proxy, none in the SPA (DEC-025)." }
  flows:
    - { id: FLOW-LLD-ADMIN-UI-001, title: "Pane lifecycle", statement: "Mount → loading → error|ready → mutation (optimistic + revert).", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-ADMIN-UI-002, title: "Audit chain visualisation", statement: "GET /v1/admin/audit → ChainBadge colouring → optional /v1/audit/exports flow.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-ADMIN-UI-001, type: refines, from: DEC-LLD-ADMIN-UI-002, to: DEC-HLD-006, status: active }
  - { id: REL-LLD-ADMIN-UI-002, type: refines, from: FLOW-LLD-ADMIN-UI-002, to: DEC-HLD-004, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
