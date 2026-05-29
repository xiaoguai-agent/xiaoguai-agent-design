# LLD — `xiaoguai-audit`

| | |
|---|---|
| Document ID | `LLD-AUDIT-001` |
| Refines | `DEC-HLD-004` (redact-before-sign), `DEC-HLD-009` (zero default telemetry) |
| Verifies | `REQ-AUD-001` … `REQ-AUD-005`, `REQ-NFR-007`, `MR-AUDIT-001`, `BEH-AUDIT-001` |

## 1. Module purpose

Owns the append-only audit chain, the PII redactor, and the OTel span decorator. Provides:

- `append(entry)` — write a new row inside the caller's transaction; computes the HMAC over the redacted form.
- `verify(tenant_id, from?, to?)` — recompute the chain and compare against on-disk HMACs.
- `redact(value)` — public utility (also used by `xiaoguai-mcp-exec` stderr scrubber and the OTel exporter).
- `RedactingSpanExporter<T>` — decorator over any OTel SpanExporter.

## 2. Public interface

```rust
pub struct AuditWriter<'a> { /* &mut Tx<'a> */ }

impl<'a> AuditWriter<'a> {
    pub async fn append(&mut self, entry: AuditEntry) -> Result<AuditRow, AuditError>;
}

pub struct Verifier { hmac_keys: Vec<HmacKey> }

impl Verifier {
    pub async fn verify(
        &self,
        tx: &mut Tx<'_>,
        tenant_id: TenantId,
        from: Option<i64>,
        to: Option<i64>,
    ) -> Result<VerifyReport, AuditError>;
}

pub struct VerifyReport {
    pub valid: bool,
    pub head_sequence: i64,
    pub head_hmac_b64: String,
    pub broken_at_sequence: Option<i64>,
}

pub fn redact(value: &mut serde_json::Value);

pub struct RedactingSpanExporter<T: SpanExporter> { inner: T }
```

## 3. Module structure

```
crates/xiaoguai-audit/
├── src/
│   ├── lib.rs
│   ├── entry.rs           # AuditEntry, AuditRow
│   ├── chain.rs           # HMAC computation, prev_hmac chaining
│   ├── verifier.rs        # Verifier struct + dual-key support
│   ├── redact.rs          # patterns: email, IPv4, Bearer, AWS-key
│   ├── otel.rs            # RedactingSpanExporter<T>
│   └── error.rs
└── tests/
    ├── chain_forge.rs     # delete row, verifier returns broken_at
    ├── redact_corpus.rs   # PII probe corpus, zero leaks
    └── otel_redact.rs     # exported spans contain no patterns
```

## 4. Key flows

### 4.1 Append

```
caller (e.g., agent loop) ──► AuditWriter::append(entry)
                                  │
                                  ├─ redact(&mut entry.actor)
                                  ├─ redact(&mut entry.resource)
                                  ├─ redact_recursive(&mut entry.details)   # values only, keys preserved
                                  ├─ load prev_hmac from current tenant chain head
                                  ├─ hmac = HMAC_SHA256(key, prev_hmac ‖ ts ‖ tenant ‖ actor ‖ action ‖ resource ‖ details_json)
                                  ├─ INSERT INTO audit_log(... , prev_hmac, hmac)
                                  └─ return AuditRow
```

### 4.2 Verify

```
Verifier::verify(tx, tenant_id, from, to)
   ├─ stream rows ordered by sequence
   ├─ for each row:
   │      expected = HMAC_SHA256(key, prev_hmac ‖ ts ‖ ... ‖ details)
   │      if expected != row.hmac:
   │          try each key in self.hmac_keys (rotation window)
   │          if none match: broken_at = row.sequence; return
   └─ return valid=true, head_sequence, head_hmac
```

### 4.3 Span export

```
RedactingSpanExporter::export(batch)
   ├─ for each span:
   │      for each attribute value:
   │          if matches pattern: replace with [redacted-*]
   └─ inner.export(redacted_batch)
```

### 4.4 Compliance export (v0.5.4.2, T5, PR #74)

`crates/xiaoguai-audit/src/export.rs` ships the framework-specific export layer. Per DEC-016, chain verification is the **non-bypassable** gate before any export bytes are produced.

```rust
pub enum Framework { Soc2, Gdpr, Hipaa }
pub enum OutputFormat { Json, Csv, Pdf }

pub struct ChainProof {
    pub first_id: i64,
    pub last_id: i64,
    pub count: u64,
    pub start_prev_hmac_hex: String,
    pub end_hmac_hex: String,
}

pub struct ExportBundle<'a> {
    pub framework: Framework,
    pub from: DateTime<Utc>,
    pub to: DateTime<Utc>,
    pub proof: ChainProof,           // embedded in every output
    pub rows: Vec<ExportRow<'a>>,    // framework-projection of audit_log
}

pub enum ExportError {
    ChainBroken { first_broken_id: i64, first_broken_ts: DateTime<Utc> },
    EmptyWindow,
    StorageError(String),
    PdfUnimplemented,                // 501 — deferred PDF backend
}

/// Build the bundle for `framework` over `[from, to]`. Re-runs verify_chain
/// over the window and refuses if a break is detected — no override.
pub async fn export_bundle<R: AuditChainExporter>(
    reader: &R,
    framework: Framework,
    tenant_id: &str,
    from: DateTime<Utc>,
    to: DateTime<Utc>,
) -> Result<ExportBundle<'_>, ExportError>;

/// Render bundle to JSON (canonical) or CSV (deterministic projection).
pub fn render(bundle: &ExportBundle, format: OutputFormat) -> Result<Vec<u8>, ExportError>;
```

**Framework projections** are hardcoded in the source (no runtime template DSL per DEC-016):

- `Framework::Soc2` → SOC2 CC7.2 (system-event-monitoring evidence). Filters to `audit.action IN {tool.execute, llm.usage, hotl.deny, skill.approve, session.start, session.end}`.
- `Framework::Gdpr` → GDPR Art. 30 (records of processing activity). Filters to rows with `actor` not null and `resource` referencing user-data tables. Projects `tenant_id`, `actor`, `purpose` (derived from `action`), `data_categories`.
- `Framework::Hipaa` → HIPAA §164.312 (audit controls). Filters to rows touching tables flagged as containing PHI. Projects `who`, `what`, `when`, `where`, `why` (derived).

**Architecture (cross-crate):** the new `AuditChainExporter` trait lives in `xiaoguai-api` (not in `xiaoguai-audit`) so the api crate can be constructed without the HMAC signing key. `xiaoguai-core::PgAuditAdapter` implements the trait by combining `PgAuditSink` (knows the key) + chain verification. Same bridge pattern as the existing `AuditReader` / `AuditVerifier`.

**Output formats**: JSON is canonical. CSV is a deterministic projection with identical row counts and column semantics (just flat). PDF was originally stubbed (`PdfUnimplemented` → HTTP 501) — see §4.5 below for the sprint-8 implementation.

### 4.5 PDF rendering via `pdf-writer` (v0.5.4.3, DEC-023.2, PR #79)

`crates/xiaoguai-audit/src/pdf.rs` replaces the `PdfUnimplemented` stub with a `pdf-writer`-based renderer.

**Plan adjustment recorded in PR #79's S8-6 commit**: the original DEC-023.2 statement specified `typst` (full typesetting engine with `.typ` templates). Sub-agent pivoted to `pdf-writer` (low-level PDF byte-emit library) on inspection of the actual requirements:

- Templates are 4-5 fixed tables per framework + a `ChainProof` header. Not the use case a typesetting engine is designed for.
- Typst defaults to embedding `datetime.now()` and a random `/ID` in the trailer; suppressing both requires hacks. `pdf-writer` is deterministic by construction.
- DEC-016's commitment is "auditors verify byte-for-byte reproducibility". `pdf-writer` makes that a structural property, not a configuration discipline.

The DEC-023.2 promise (auditor reproducibility) is preserved; only the implementation library changed.

```rust
pub fn render_pdf(bundle: &ExportBundle) -> Result<Vec<u8>, ExportError>;
```

Output structure: PDF 1.7 with deterministic `/CreationDate`, no random `/ID`, no embedded fonts (Helvetica via PDF standard 14). Per framework:

- SOC2 CC7.2: tables for system-event-monitoring evidence + chain-proof footer.
- GDPR Art. 30: records of processing activity with per-row `actor` / `purpose` / `data_categories`.
- HIPAA §164.312: audit-controls table with five-W projection.

Tests: 6 tests covering `%PDF-` header sniff, byte-determinism per framework (regenerate twice → equal), empty-bundle path, framework-label-in-footer.

## 5. Error handling

```rust
pub enum AuditError {
    Storage(StorageError),
    HmacKeyMissing,
    SerializationFailure(serde_json::Error),
    ChainBroken { at_sequence: i64 },
    ExportError(export::ExportError),    // v0.5.4.2
}
```

`HmacKeyMissing` is fatal at startup; the smoke command fails before serving.

## 6. Concurrency / transactions

- `AuditWriter` takes `&mut Tx<'_>` — same transaction as the change being audited.
- `prev_hmac` load uses `SELECT … FOR UPDATE` on the tenant's chain tail to serialise appends within a transaction.
- `Verifier::verify` uses a read-only transaction and streams rows; no locks.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | redact patterns: emails, IPv4, Bearer, AWS keys, IPv6 (NOT redacted); HMAC computation matches RFC 2104; chain helper composes correctly |
| Integration | delete a row, verify returns `valid=false, broken_at_sequence=<n>`; dual-key fixture for rotation |
| OTel | `RedactingSpanExporter` decorator zero-leak fixture across the PII probe corpus |
| System | Test strategy: chain integrity across deploys; redaction toggle observed in boot logs |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-AUDIT-001
  type: LLD
  title: xiaoguai-audit LLD
  status: draft
  owners: [engineering.xiaoguai, security.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, GUARDRAILS-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-AUD-001, title: "HMAC-SHA256 256-bit key", statement: "Audit HMAC uses SHA-256 with 256-bit key.", status: approved, scope: in, decision: "HMAC-SHA256.", rationale: "密评 compatible; widely audited." }
    - { id: DEC-LLD-AUD-002, title: "Dual-key verifier", statement: "Verifier accepts entries signed by any key in its list (rotation window).", status: approved, scope: in, decision: "Dual-key.", rationale: "30-day rotation per GR-OPS-03." }
    - { id: DEC-LLD-AUD-003, title: "RedactingSpanExporter decorator", statement: "All OTel exporters wrapped by RedactingSpanExporter.", status: approved, scope: in, decision: "Decorator pattern.", rationale: "Single redaction code path; clippy lint forbids raw exporter use." }
  flows:
    - { id: FLOW-LLD-AUD-001, title: "Append (redact-then-sign)", statement: "Redact entry; load prev_hmac FOR UPDATE; sign; insert.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-AUD-002, title: "Verify dual-key", statement: "Stream rows, try each key; first mismatch breaks chain.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-AUD-001, type: refines, from: DEC-LLD-AUD-001, to: DEC-HLD-004, status: active }
  - { id: REL-LLD-AUD-002, type: refines, from: DEC-LLD-AUD-002, to: REQ-AUD-005, status: active }
  - { id: REL-LLD-AUD-003, type: refines, from: DEC-LLD-AUD-003, to: REQ-NFR-007, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
