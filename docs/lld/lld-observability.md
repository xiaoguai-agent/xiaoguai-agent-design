# LLD — `xiaoguai-observability` (SLO module)

| | |
|---|---|
| Document ID | `LLD-OBS-001` |
| Refines | `DEC-HLD-022` (SLO + 4 golden signals), `DEC-HLD-019` (per-tenant JSONB override pattern), `DEC-HLD-009` (observability architecture) |
| Verifies | `REQ-OBS-001` (published SLO per user-facing capability), `REQ-OBS-002` (burn-rate alert → operator page chain), `REQ-OBS-003` (per-tenant SLO override without migration) |

## 1. Module purpose

The SLO module turns the four Google SRE golden signals from *observed metrics* into *contract obligations* on top of the existing `xiaoguai-observability` Prometheus stack. It does not introduce new collection plumbing — the metrics already exist (`prometheus.rs` 15 series, all `xiaoguai_*` prefix) and the alert pipeline already exists (`deploy/helm/xiaoguai-observability/templates/prometheus-rules.yaml`, `alertmanager-config.yaml`). What is new is a *declarative* SLO layer: each user-facing capability has a published latency / traffic / errors / saturation target; a single `xiaoguai_slo_burn_rate{signal,window}` gauge measures distance to breach; two-window burn-rate alerts route to the operator page chain via existing channels. Wave-3 (`wave3-slo-recording-rules`, `wave3-slo-burn-rate-alerts`, `wave3-slo-meta`) is the prototype this LLD generalises to the API tier.

## 2. Public interface

```rust
// crates/xiaoguai-observability/src/slo.rs (NEW — sprint-10 S10-3)

/// One published SLO for one (signal, contract-surface) pair.
pub struct Slo {
    pub signal: Signal,
    pub surface: ContractSurface,
    pub threshold: Threshold,
    pub window: Window,
    pub burn_rate_fast: Duration,   // default 1h per DEC-022
    pub burn_rate_slow: Duration,   // default 6h per DEC-022
}

#[derive(Copy, Clone, Eq, PartialEq, Hash)]
pub enum Signal {
    Latency,
    Traffic,
    Errors,
    Saturation,
}

/// A user-facing capability that has an SLO contract.
pub enum ContractSurface {
    Http { route: &'static str },           // e.g. "/v1/chat/*"
    Queue { name: &'static str },           // background work
    Budget { kind: &'static str },          // e.g. "llm_tokens_vs_tenant_daily"
}

pub enum Threshold {
    LatencyP95Seconds(f64),
    ErrorRatePercent(f64),
    UtilisationRatio(f64),                  // 0.0..=1.0
    RequestsPerSecond { limit_source: &'static str },
}

pub enum Window {
    RollingHours(u32),
    DayBoundary,                             // for "tenant_daily_budget"
}

/// Per-scrape burn-rate computation. Output is published as
/// `xiaoguai_slo_burn_rate{signal,window,tenant?,surface}`.
pub trait BurnRateCalculator: Send + Sync {
    fn observe(&self, slo: &Slo, now: SystemTime) -> Option<f64>;
}

pub enum SloError {
    OverrideParse { tenant: TenantId, key: String, raw: String },
    MetricMissing { metric: &'static str },
    WindowOutOfRange { signal: Signal, requested: Duration },
}
```

Implementation lands in sprint-10 task **S10-3**. This LLD specifies the shape; the impl PR specifies the wiring.

## 3. Module structure

```
crates/xiaoguai-observability/
├── src/
│   ├── lib.rs                      # existing — re-exports init_prometheus, mount_metrics
│   ├── prometheus.rs               # existing — 15 registered metrics; SLO module adds `xiaoguai_slo_burn_rate`
│   ├── otlp.rs                     # existing — OTLP/gRPC trace export
│   ├── instrument.rs               # existing — instrument_* macros
│   └── slo.rs                      # NEW (S10-3) — Slo / Signal / BurnRateCalculator + gauge registration
└── tests/
    └── slo_burn_rate.rs            # NEW (S10-3) — unit + integration cases (§7)
```

Outside the crate:

- `docs/runbooks/slo.md` — the declared SLO table + page chain per signal (NEW, owned by S10-1)
- `deploy/helm/xiaoguai-observability/templates/prometheus-rules.yaml` — extended with `slo-api-tier-recording-rules` and `slo-api-tier-burn-rate-alerts` groups (S10-2; wave-3 groups untouched)
- `observability/grafana/dashboards/slo-overview.json` — 4-signal panels + error-budget tracker (NEW, S10-4)

## 4. Key flows

### 4.1 SLO declaration → runtime contract

```
docs/runbooks/slo.md (YAML table, human-readable)
        │
        ▼  parsed at process start
crates/xiaoguai-observability::slo::load_slos()
        │
        ├─► per-tenant override merge (§4.6)
        │
        ▼
Vec<Slo>  (in-memory, immutable for process lifetime)
        │
        ▼  every scrape interval
BurnRateCalculator::observe(&slo, now)
        │
        ▼
xiaoguai_slo_burn_rate{signal,window,...} gauge updated
        │
        ▼
Prometheus alert rules fire on threshold (§4.5)
        │
        ▼
Alertmanager routes by `severity` label (existing alertmanager-config.yaml)
        │
        ▼
PagerDuty / Slack / email (existing channels — no new plumbing)
```

SLOs are **declarations**, per DEC-022. The runbook file is the source of truth; the Rust struct is the runtime mirror, not the contract.

### 4.2 Four golden signals in xiaoguai context

Each signal maps to one existing metric in `xiaoguai-observability::prometheus`, one DEC-022 default, and one contract surface:

| Signal | Existing metric (real name) | DEC-022 default SLO | Contract surface |
|---|---|---|---|
| **Latency** | `xiaoguai_http_request_duration_seconds`[^1] (histogram, `method`/`path`/`status` labels) — and its first-token-time companion for streaming routes | `/v1/chat/*` p95 < 5 s; `/v1/sessions/*/messages` p95 first-token < 2 s | HTTP routes |
| **Traffic** | derived from `xiaoguai_http_request_duration_seconds_count` (rate) and `xiaoguai_rate_limit_hits_total{decision}` | per-tenant rate limits at `/etc/xiaoguai/config.yaml::rate_limits` (the limits *are* the SLO) | per-tenant request budget |
| **Errors** | `xiaoguai_hotl_usage_total{verdict}` for HotL denials; `xiaoguai_outcomes_recorded_total{kind="failure"}` for tool/agent failures; HTTP 5xx derived from `xiaoguai_http_request_duration_seconds_count{status=~"5.."}` | non-2xx rate < 1 % rolling 1h | HTTP routes + agent loop |
| **Saturation** | `xiaoguai_llm_reasoning_tokens_total` and `xiaoguai_tokens_consumed_total` divided by tenant daily budget (config); `xiaoguai_watch_wakeups_total{outcome}` for watch-loop backpressure | `LLM tokens consumed / tenant_daily_budget < 0.8` | per-tenant cost budget |

[^1]: DEC-022 §statement references `request_duration_seconds`; the actual registered metric (per `crates/xiaoguai-observability/src/prometheus.rs`) is `xiaoguai_http_request_duration_seconds`. This LLD uses the actual name throughout. An HLD touch-up to align DEC-022's prose is queued as a sprint-10 follow-up — out of scope for this LLD because renaming the live metric would invalidate 17 existing alert groups in `prometheus-rules.yaml` and 7 Grafana dashboards under `observability/grafana/dashboards/`.

### 4.3 Default SLO values (verbatim from DEC-022)

| Signal | Default SLO |
|---|---|
| **Latency** | `/v1/chat/*` p95 < 5 s; `/v1/sessions/*/messages` p95 first-token < 2 s |
| **Traffic** | per-tenant rate limits at `/etc/xiaoguai/config.yaml::rate_limits` |
| **Errors** | < 1 % rolling 1h |
| **Saturation** | `LLM tokens consumed / tenant_daily_budget < 0.8` |

These were **approved by the user in the sprint-8 sign-off** (no pre-calibration round on session-7 production data); they ship as the canonical defaults. Operators may override per tenant via §4.6.

### 4.4 `xiaoguai_slo_burn_rate{signal,window}` gauge design

```
Metric type:   Gauge (per-scrape snapshot of current burn ratio)
Series key:    xiaoguai_slo_burn_rate{signal,window,surface,tenant?}
Labels:
  signal       latency | traffic | errors | saturation
  window       fast | slow                              (per DEC-022 — see §4.5 for richer pattern)
  surface      route or queue or budget identifier
  tenant       OPTIONAL — present only when override active (avoid global cardinality explosion)
Cardinality:   4 signals × 2 windows × ~10 surfaces ≈ 80 base series + N per overriding-tenant
Scrape:        15 s default (matches existing Prometheus scrape config)
Computation:   error_rate_observed / error_budget_per_window   (>1.0 means burning faster than budget)
```

The gauge lives in the new `slo.rs`. Registration follows the existing `register_*_with_registry!` pattern in `prometheus.rs` (lines 132 onward).

### 4.5 Burn-rate alert pattern (fast 1h + slow 6h)

DEC-022 specifies the SRE workbook two-window pattern: a fast (1 h) and a slow (6 h) window per signal. The alert rule pair is:

```promql
# fast burn — high severity, high precision
alert: ApiLatencySLOBurnRateFast
expr: xiaoguai_slo_burn_rate{signal="latency",window="fast",surface="/v1/chat/*"} > 14.4
for: 2m
labels:  { severity: critical, team: platform, slo: api-latency-chat-p95-5s }

# slow burn — medium severity, lower precision but catches sustained leak
alert: ApiLatencySLOBurnRateSlow
expr: xiaoguai_slo_burn_rate{signal="latency",window="slow",surface="/v1/chat/*"} > 6
for: 15m
labels:  { severity: warning,  team: platform, slo: api-latency-chat-p95-5s }
```

Burn-rate thresholds 14.4× and 6× follow Google SRE workbook chapter 5 (budget burnt in ~2 days and ~5 days respectively at a 30-day budget). The shape mirrors the existing `wave3-slo-burn-rate-alerts` group at `deploy/helm/xiaoguai-observability/templates/prometheus-rules.yaml:406–467`, which ships a richer 4-pair scheme (14.4× / 6× / 3× / 1× across {1 h+5 m, 6 h+30 m, 1 d+2 h, 3 d+6 h}). Operators who want the 4-pair scheme can extend any signal — DEC-022's 2-window pattern is the floor, not the ceiling.

### 4.6 Wave-3 precursor relationship

Wave-3 already ships SLO infrastructure for the HotL latency contract. The three groups in `prometheus-rules.yaml` are the **prototype** for what DEC-022 generalises:

| Wave-3 group (existing) | Lines | Role | Sprint-10 generalisation |
|---|---|---|---|
| `wave3-slo-recording-rules` | 363–404 | Computes `xiaoguai:hotl_latency_sli:ratio_rate{5m,1h,6h,1d,3d}` recordings | New group `slo-api-tier-recording-rules` adds analogous recordings for `/v1/chat/*` latency, errors, saturation |
| `wave3-slo-burn-rate-alerts` | 406–467 | 4-pair burn-rate alerts on HotL latency SLO | New group `slo-api-tier-burn-rate-alerts` adds 2-window pair (DEC-022 default) per API-tier signal |
| `wave3-slo-meta` | 472+ | Error budget remaining over 28-day window | New group `slo-api-tier-meta` mirrors for API tier |

Wave-3 groups are **not migrated** — they keep their `wave3-*` names and their richer 4-pair alert pattern. Sprint-10 adds parallel `slo-api-tier-*` groups. Naming is a soft convention; both schemes coexist under the same PrometheusRule CR (per DEC-022 "no new alerting plumbing").

### 4.7 Per-tenant SLO override via `tenant_settings.settings` JSONB

Mirrors the DEC-019 `sandbox_tier` pattern (`crates/xiaoguai-tasks/src/skill_author.rs:183–206`). The `tenant_settings` table from migration `0021_skill_proposals.sql:19–23` already exists; SLO overrides add new top-level keys to the same JSONB column — **no new migration**.

Override keys (all optional, lenient parse, fall back to DEC-022 defaults):

| JSONB key | Type | Default | Example |
|---|---|---|---|
| `settings->>'slo_latency_p95_ms'` | integer (ms) | 5000 for `/v1/chat/*` | `2500` (stricter for premium tier) |
| `settings->>'slo_error_budget_pct'` | float (0..1) | 0.01 | `0.005` |
| `settings->>'slo_saturation_ratio'` | float (0..1) | 0.8 | `0.95` (allow burn closer to cap) |
| `settings->>'slo_alert_severity'` | string | `critical` | `warning` (downgrade page chain) |

```rust
// Pattern lifted from skill_author.rs::SandboxTier::from_str_lenient
impl SloLatencyOverride {
    pub fn from_jsonb(raw: Option<&str>) -> Self {
        match raw.and_then(|s| s.trim().parse::<u32>().ok()) {
            Some(ms) if (100..=600_000).contains(&ms) => Self::Override(ms),
            _ => Self::DefaultFromDec022,
        }
    }
}
```

Read path: existing `tenant_settings` cache (no new lock). Override is read at SLO-evaluation time, not cached separately from the wider tenant_settings row.

### 4.8 Integration with `xiaoguai-watch` and alert routing

Per DEC-022: "Alert routing via the existing `xiaoguai-watch` infrastructure — no new alerting plumbing." Mapping:

- **Prometheus alert fires** → Alertmanager (existing pipeline).
- **Alertmanager routes by `severity` label** to the existing receivers in `alertmanager-config.yaml`: `severity=critical` → PagerDuty + Slack; `severity=warning` → Slack + email.
- **For SLO breaches that need agent-loop response** (e.g. saturation burn → trigger compaction), the alert is mirrored into a `WatchSpec` consumed by `crates/xiaoguai-watch::WatchRunner` (`src/runner.rs:39`). The `on_match.action` resolves to an existing executor (e.g. `FeishuPushSink` for IM notification, or an internal compaction trigger) — sprint-10 ships no new executor.

Explicit non-goals:

- No new alertmanager receiver.
- No new IM gateway adapter.
- No new Prometheus exporter.
- No SLO-specific channel — pages go through the same `team=platform` / `team=tenant-ops` routing as wave-3 alerts.

### 4.9 Failure modes

| Failure | Detection | Recovery |
|---|---|---|
| Metric scrape gap (Prometheus down, target unreachable) | `xiaoguai_slo_burn_rate` becomes stale (`absent_over_time`) | Existing `up == 0` alert in `prometheus-rules.yaml` covers; SLO alerts auto-recover when scrape resumes |
| `Slo::from_yaml` parse failure at startup | Process exits with non-zero code + log line citing the offending key | Fail fast — declarative SLOs must parse; misconfigured deploys must not boot. |
| Tenant override malformed (`slo_latency_p95_ms = "fast"`) | `SloError::OverrideParse` logged with `tenant` + raw value | Lenient parse falls back to DEC-022 default; alert `xiaoguai_slo_override_parse_failed_total{tenant}` increments (NEW counter, S10-3) |
| Burn-rate gauge stuck at last value when underlying metric stops | `BurnRateCalculator::observe` returns `None`; gauge unset (Prometheus drops the series) | Two-window pattern: slow-burn alert still fires if breach persists across the slow window; fast-burn alert auto-clears on metric absence (intentional — see SRE workbook on incident-suppression during outage). |
| Alertmanager backlog (existing risk) | Wave-3 already has `wave3.anomaly.storm` group covering alert-storm; SLO alerts inherit | No new mitigation; sprint-10 documents the dependency. |
| SLO threshold itself wrong (e.g. p95 < 5 s impossible to meet) | Slow-burn pages within hours; runbook entry (S10-6) tells operator how to file an SLO-revision PR | Process: revise `docs/runbooks/slo.md`, re-PR, re-deploy. SLO is a declaration; revisions go through code review. |

## 5. Error handling

```rust
pub enum SloError {
    /// Tenant override JSONB value failed to parse to expected type/range.
    OverrideParse { tenant: TenantId, key: String, raw: String },

    /// SLO references a metric not registered in this build (e.g. typo in runbook).
    MetricMissing { metric: &'static str },

    /// Window duration outside the supported range (1 min … 30 d).
    WindowOutOfRange { signal: Signal, requested: Duration },
}
```

All variants are recoverable except `MetricMissing` at startup (fail fast). `OverrideParse` is logged at `WARN` and counted; the tenant gets the DEC-022 default for that key, not a hard failure.

## 6. Concurrency / transactions

- Gauge updates use `prometheus::Gauge::set`, which is internally a `RwLock<f64>` — lock-free for the read path that Prometheus uses; brief write lock on update. No new synchronisation primitives.
- The tenant-override read path piggybacks on the existing `tenant_settings` cache (Wave-3 introduced the cache; not duplicated here).
- `BurnRateCalculator::observe` is called from the scrape thread (one call per Slo per scrape); no concurrent invocation per Slo.
- SLO declarations are loaded once at process start and never mutated — `Arc<[Slo]>` shared read-only across threads.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | `Slo::from_yaml_str` round-trip; `Threshold` parse (latency_p95_ms, error_rate, utilisation, rps); `Window` parse + range validation; `SloLatencyOverride::from_jsonb` lenient-parse (valid, out-of-range, garbage, null) — mirrors `SandboxTier::from_str_lenient` test pattern; `BurnRateCalculator::observe` math (numerator/denominator/edge ratios). |
| Integration | `slo_gauge_registers_and_publishes` — boot process, scrape `/metrics`, assert `xiaoguai_slo_burn_rate` series present with expected labels. `slo_alert_rules_validate` — run `promtool test rules` against generated `slo-api-tier-burn-rate-alerts` group with synthetic input series; assert fast-burn fires at burn=15, slow-burn fires at burn=7. `slo_tenant_override_applied` — set `tenant_settings.settings->>'slo_latency_p95_ms'='2500'` for tenant T; scrape; assert the corresponding `xiaoguai_slo_burn_rate{tenant="T",surface="/v1/chat/*"}` reflects the 2.5 s threshold. |
| System | One end-to-end synthetic-breach case (sprint-10 S10-5): generate sustained 5xx on `/v1/chat/*` for 30 min; assert `ApiErrorsSLOBurnRateFast` alert fires; assert Alertmanager dispatches to the configured Slack channel; assert `xiaoguai-watch` consumes the mirrored watch event and records a `WatchEvent` row. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-OBS-001
  type: LLD
  title: xiaoguai-observability SLO module LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-29
  updated_at: 2026-05-29
  source_documents: [HLD-XIAOGUAI-001, PRD-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-OBS-001, title: "Two-window burn-rate default", statement: "Default burn-rate pattern is the DEC-022 two-window (fast 1h + slow 6h) scheme; the wave-3 four-pair scheme is available to operators who opt in.", status: approved, scope: in, decision: "Two-window default.", rationale: "Floor coverage with minimal config; richer scheme is a deliberate opt-in." }
    - { id: DEC-LLD-OBS-002, title: "Per-tenant override via JSONB, no migration", statement: "SLO overrides land as new top-level keys in tenant_settings.settings JSONB. No schema change.", status: approved, scope: in, decision: "JSONB extension.", rationale: "Mirrors sandbox_tier (DEC-019); operators can override per tenant without re-deploy of storage layer." }
    - { id: DEC-LLD-OBS-003, title: "Actual metric name used in LLD", statement: "LLD uses xiaoguai_http_request_duration_seconds (real metric) rather than DEC-022's xiaoguai_request_duration_seconds. HLD prose touch-up is queued as follow-up; rename of the runtime metric is out of scope.", status: approved, scope: in, decision: "Actual name + HLD touch-up.", rationale: "Renaming invalidates 17 alert groups + 7 dashboards." }
    - { id: DEC-LLD-OBS-004, title: "Wave-3 SLO groups stay; sprint-10 adds parallel groups", statement: "wave3-slo-* groups in prometheus-rules.yaml are not migrated. Sprint-10 introduces slo-api-tier-* groups alongside.", status: approved, scope: in, decision: "Parallel groups, no migration.", rationale: "DEC-022 'no new alerting plumbing' — reuse the same PrometheusRule CR; do not destabilise an in-production SLO." }
  flows:
    - { id: FLOW-LLD-OBS-001, title: "Declaration to gauge", statement: "YAML SLO declaration → Slo struct → BurnRateCalculator → xiaoguai_slo_burn_rate gauge.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-OBS-002, title: "Per-tenant override read", statement: "tenant_settings.settings JSONB → SloLatencyOverride lenient parse → effective threshold at SLO-eval time.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-OBS-003, title: "Burn-rate alert routing", statement: "Prometheus alert → Alertmanager → severity-routed receiver (existing alertmanager-config.yaml) → PagerDuty/Slack/email.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-OBS-001, type: refines, from: DEC-LLD-OBS-001, to: DEC-HLD-022, status: active }
  - { id: REL-LLD-OBS-002, type: refines, from: DEC-LLD-OBS-002, to: DEC-HLD-019, status: active }
  - { id: REL-LLD-OBS-003, type: refines, from: FLOW-LLD-OBS-001, to: DEC-HLD-022, status: active }
  - { id: REL-LLD-OBS-004, type: refines, from: FLOW-LLD-OBS-003, to: DEC-HLD-009, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
