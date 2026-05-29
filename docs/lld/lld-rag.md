# LLD — `xiaoguai-rag`

| | |
|---|---|
| Document ID | `LLD-RAG-001` |
| Verifies | `REQ-MEM-005` |

## 1. Module purpose

Retrieval-augmented generation primitives consumed via MCP tools. Provides:

- Document loaders (markdown, txt, pdf, docx, html).
- Chunkers (by tokens, by structure).
- Retrievers backed by `xiaoguai-storage`'s pgvector recall.
- An in-tree MCP server `rag_lookup` that exposes `rag.search(query, namespace, k)`.

It does NOT expose its own `/v1/rag/**` HTTP surface — RAG is intentionally consumed through the MCP tool surface so tenants can swap providers (e.g., external Qdrant) without API churn.

## 2. Public interface

```rust
pub trait Loader: Send + Sync {
    fn load(&self, path: &Path) -> Result<Vec<Document>, RagError>;
}

pub trait Chunker: Send + Sync {
    fn chunk(&self, doc: &Document) -> Vec<Chunk>;
}

pub struct Retriever { /* pgvector-backed */ }
impl Retriever {
    pub async fn search(&self, namespace: &str, query: &str, k: usize)
        -> Result<Vec<Hit>, RagError>;
}

pub struct RagMcpServer { /* in-tree MCP server */ }
```

## 3. Module structure

```
crates/xiaoguai-rag/
├── src/
│   ├── lib.rs
│   ├── loaders/{markdown.rs, pdf.rs, html.rs, txt.rs}
│   ├── chunkers/{by_token.rs, by_structure.rs}
│   ├── retriever.rs
│   └── mcp_server.rs       # rag_lookup tool
└── tests/
    └── chunker_token.rs
```

## 4. Key flows

### 4.1 Ingest

```
loader.load(path) -> Vec<Document>
   for doc in docs:
     chunks = chunker.chunk(doc)
     for chunk in chunks:
       embedding = embedder.embed(chunk.text)
       memories.put(tenant, NewMemory{ content: chunk.text, tags: ["rag", namespace], ... })
```

### 4.2 Search

```
rag_lookup(query, namespace, k) -> MemoryService::recall(tenant, RecallRequest{
   query, limit: k, tags_filter: Some(vec![namespace])
})
```

## 5. Error handling

```rust
pub enum RagError {
    LoaderFailed(String),
    ChunkerEmpty,
    Memory(MemoryError),
}
```

## 6. Concurrency / transactions

- Loader + chunker run on the worker pool; ingest is parallel per document.
- Search delegates to `MemoryService::recall` and inherits its concurrency model.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | Markdown loader extracts headings; token chunker respects max_tokens; structure chunker keeps headings with body |
| Integration | RAG ingest then `rag_lookup` returns chunks with `namespace` filter |
| System | Test strategy L2: agent calls `rag_lookup`, then uses results in subsequent reasoning |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-RAG-001
  type: LLD
  title: xiaoguai-rag LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-RAG-001, title: "RAG consumed via MCP tool, not REST", statement: "No /v1/rag/* surface; rag_lookup is an MCP tool.", status: approved, scope: in, decision: "MCP-only.", rationale: "Allows tenants to swap providers without API churn." }
  flows:
    - { id: FLOW-LLD-RAG-001, title: "Ingest + search", statement: "Load -> chunk -> embed -> store; embed query -> recall.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-RAG-001, type: refines, from: DEC-LLD-RAG-001, to: REQ-MCP-005, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
