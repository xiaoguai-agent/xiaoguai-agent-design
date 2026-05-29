# LLD — `xiaoguai-im-gateway`

| | |
|---|---|
| Document ID | `LLD-IMGW-001` |
| Verifies | `REQ-IM-001`…`REQ-IM-005`, `REQ-NFR-004`, `BEH-IM-001`, `MR-IM-001` |

## 1. Module purpose

Common abstraction over seven per-platform IM adapters. Provides:

- The `ImProvider` trait every platform implements.
- A `Dispatcher` that receives a normalised `ImEvent` and routes to the agent.
- An `IdentityResolver` that maps platform-side user ids to internal `UserId` via the `im_identities` table.
- Conversation history backend selection (Postgres default; in-process for dev).

## 2. Public interface

```rust
#[async_trait]
pub trait ImProvider: Send + Sync {
    fn platform(&self) -> &'static str;
    async fn parse_event(&self, raw: HttpRequest) -> Result<ImEvent, ImError>;
    async fn reply(&self, conv: ConversationRef, message: ImMessage) -> Result<(), ImError>;
}

pub struct ImEvent {
    pub platform: &'static str,
    pub conversation: ConversationRef,
    pub from: PlatformUserId,
    pub kind: ImEventKind,           // text / file / button / command
    pub raw_signature_verified: bool,
}

pub struct Dispatcher { /* binds providers to /v1/im/<platform>/event routes */ }
```

## 3. Module structure

```
crates/xiaoguai-im-gateway/
├── src/
│   ├── lib.rs               # ImProvider trait, Dispatcher
│   ├── identity.rs          # IdentityResolver
│   ├── history/
│   │   ├── mod.rs
│   │   ├── postgres.rs
│   │   └── in_process.rs    # dev-only opt-in
│   ├── error.rs
│   └── routes.rs            # axum sub-router mounted under /v1/im
└── tests/
    └── signature_negative.rs
```

Platform-specific crates (`xiaoguai-im-feishu`, …) depend on this crate.

## 4. Key flows

### 4.1 Inbound event

```
POST /v1/im/<platform>/event
   ├─ dispatcher.handle(platform, raw_request)
   ├─ provider.parse_event(raw_request)
   │     ├─ verify signature (HMAC / Ed25519 / AES)
   │     └─ if fail: return Err(SignatureMismatch) -> 401
   ├─ event.from -> IdentityResolver -> internal UserId
   │     if unknown: enqueue "bind your account" challenge; return 200
   ├─ load trailing N messages from history (N=cfg.max_messages_per_conversation)
   ├─ enqueue agent run on worker pool
   └─ ack platform within ≤ 1.5s
```

### 4.2 Outbound reply

```
agent emits Final -> provider.reply(conv, message)
   ├─ marshal to platform format
   └─ POST / WS / API call
```

### 4.3 History backend

```
cfg.use_in_process_history == false (default) -> PostgresHistory (im_messages table)
cfg.use_in_process_history == true            -> InProcessHistory (dev only)
```

## 5. Error handling

```rust
pub enum ImError {
    SignatureMismatch,
    UnknownIdentity,
    Transport(String),
    Parse(String),
    HistoryUnavailable,
}
```

`SignatureMismatch` maps to 401 BEFORE deserialisation (REQ-NFR-004).

## 6. Concurrency / transactions

- Inbound webhook handlers ack the platform synchronously; the agent run is enqueued.
- Reply is best-effort; failure increments a retry counter and audits `im.reply_failed`.
- Identity binding is a one-shot transaction over `im_identities`.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (per platform) | Signature verification negative; payload parsing positive + edge (button, file) |
| Integration | `signature_negative.rs` — forged signature returns 401 before deserialisation; PostgresHistory writes survive restart |
| System | Test strategy L2: round-trip an IM message through Feishu webhook → agent → reply |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-IMGW-001
  type: LLD
  title: xiaoguai-im-gateway LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, API-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-IMGW-001, title: "Signature-verify before deserialise", statement: "Inbound webhook signature verification happens before any deserialisation of the body.", status: approved, scope: in, decision: "Verify-then-parse.", rationale: "Prevents JSON parser fuzz from unauthenticated traffic." }
    - { id: DEC-LLD-IMGW-002, title: "Identity bind is one-shot", statement: "Unknown platform user receives a bind challenge; no auto-create.", status: approved, scope: in, decision: "Explicit bind.", rationale: "Prevents tenant cross-pollution from misrouted webhooks." }
  flows:
    - { id: FLOW-LLD-IMGW-001, title: "Inbound webhook", statement: "Verify signature -> resolve identity -> load history -> enqueue agent -> ack.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-IMGW-001, type: mitigates, from: DEC-LLD-IMGW-001, to: BEH-IM-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
