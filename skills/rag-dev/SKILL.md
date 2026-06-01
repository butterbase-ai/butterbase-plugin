---
name: rag-dev
description: Use when building knowledge bases, ingesting documents, running semantic search, or adding LLM-synthesized Q&A over private content with Butterbase RAG
---

# Butterbase RAG (Retrieval-Augmented Generation)

Two tools cover the entire RAG surface:

- **`manage_rag_content`** — collections, document ingestion, status polling, deletion
- **`rag_query`** — semantic search, optional LLM synthesis

Documents are ingested asynchronously: text or files become embeddings stored in pgvector, and queries do a similarity search at runtime.

---

## 1. The mental model

```
Collection                        Documents                     Chunks
──────────                       ──────────                     ──────
"product-faq" ──────────────►   doc_1 (PDF) ───────────►       chunk 1, 2, 3...
                                doc_2 (text) ──────────►       chunk 4, 5...
                                doc_3 (markdown) ──────►       chunk 6...
```

A **collection** holds documents; a **document** is split into **chunks** and embedded; **`rag_query`** searches by cosine similarity across chunks within a collection.

`chunk_size` and `chunk_overlap` are set **once at collection creation** and immutable — to change them, delete and recreate the collection.

---

## 2. End-to-end workflow

```
┌────────────────────────────────────────────┐
│ 1. create_collection (once per knowledge)  │
├────────────────────────────────────────────┤
│ 2. ingest_document (text OR storage_object)│
├────────────────────────────────────────────┤
│ 3. poll get_document_status until "ready"  │
├────────────────────────────────────────────┤
│ 4. rag_query (with or without synthesis)   │
└────────────────────────────────────────────┘
```

### Step 1 — create the collection

```js
manage_rag_content({
  app_id: "app_abc123",
  action: "create_collection",
  name: "product-faq",
  description: "Customer-facing product knowledge",
  chunk_size: 512,         // optional, default 512 tokens
  chunk_overlap: 50,       // optional, default 50 tokens
  access_mode: "shared"    // optional: "private" | "shared" | "custom"
})
```

| `access_mode` | Who can query |
|----------------|---------------|
| `private` (default) | Only the app owner / service key |
| `shared` | Any authenticated end-user with a valid JWT |
| `custom` | Respects RLS policies — for fine-grained control |

### Step 2a — ingest raw text

```js
manage_rag_content({
  app_id: "app_abc123",
  action: "ingest_document",
  collection: "product-faq",
  text: "Our return policy is 30 days from purchase...",
  filename: "return-policy.txt",          // optional, for display
  metadata: { category: "returns", tier: "all" }   // filter later in rag_query
})
// → { document_id: "doc_xyz", status: "pending" }
```

### Step 2b — ingest an uploaded file

Files come from `manage_storage` first. Two-step:

```js
// 1. Upload the file via the storage skill — get an object_id
const { object_id } = await uploadPdfViaStorage(...);

// 2. Hand that object_id to RAG ingestion
manage_rag_content({
  app_id: "app_abc123",
  action: "ingest_document",
  collection: "product-faq",
  storage_object_id: object_id,
  filename: "manual.pdf",
  metadata: { product: "v3" }
})
```

Supported file types: **PDF, TXT, Markdown, CSV, HTML, DOCX, XLSX, PPTX**.

### Step 3 — poll until ready

Ingestion is fire-and-forget. The document moves through `pending → processing → ready` (or `failed`). Poll:

```js
manage_rag_content({
  app_id: "app_abc123",
  action: "get_document_status",
  collection: "product-faq",
  document_id: "doc_xyz"
})
// → { id, filename, status: "processing", processedAt, errorMessage? }
```

Recommended cadence: poll every 2–5 seconds for the first minute, back off after that. Bigger files (large PDFs, XLSX) take longer.

### Step 4 — query

Two modes: raw retrieval (just chunks back) or synthesized (LLM answer + sources).

#### Raw retrieval

```js
rag_query({
  app_id: "app_abc123",
  collection: "product-faq",
  query: "How long do I have to return an item?",
  top_k: 5,                  // default 5, max 20
  threshold: 0.7,            // optional similarity floor (0..1)
  filter: { category: "returns" }    // optional metadata filter
})
// → { chunks: [{ text, score, document_id, metadata }, ...] }
```

#### Synthesized answer

```js
rag_query({
  app_id: "app_abc123",
  collection: "product-faq",
  query: "How long do I have to return an item?",
  synthesize: true,
  model: "anthropic/claude-haiku-4.5"   // default
})
// → { answer, chunks, model }
```

`synthesize: true` runs the retrieved chunks through an LLM and returns a grounded answer. `chunks` is still included so you can show citations.

---

## 3. Listing and cleanup

```js
manage_rag_content({ app_id, action: "list_collections" })
manage_rag_content({ app_id, action: "get_collection", name: "product-faq" })
manage_rag_content({ app_id, action: "list_documents", collection: "product-faq" })
manage_rag_content({ app_id, action: "delete_document", collection: "product-faq", document_id: "doc_xyz" })
manage_rag_content({ app_id, action: "delete_collection", name: "product-faq" })
```

`get_collection` returns `{ name, description, accessMode, chunkSize, chunkOverlap, createdAt, documentCount: { pending, processing, ready, failed } }` — handy for a dashboard view.

> **Both `delete_document` and `delete_collection` are irreversible** and remove embeddings. To replace a document, delete then re-ingest.

---

## 4. Choosing chunk size and overlap

| Use case | Suggested `chunk_size` | `chunk_overlap` |
|----------|------------------------|------------------|
| Q&A over short FAQs / docs | 256–512 | 50 |
| Long-form documentation, manuals | 512–1024 | 100 |
| Code or structured content | 1024–2048 | 0–50 |
| Conversational logs / transcripts | 256 | 50 |

Larger chunks preserve more context but reduce retrieval granularity (you may pull in irrelevant nearby content). Overlap prevents semantic splits at boundaries from losing meaning. **You can't change these without recreating the collection** — pick them deliberately the first time.

---

## 5. Metadata-driven filtering

Anything you pass in `metadata` at ingest time is available as a filter at query time. Use it to scope queries:

```js
// at ingest:
metadata: { product: "v3", region: "EU", language: "en" }

// at query:
filter: { product: "v3", language: "en" }
```

Filters are exact-match key/value. There's no full-text search beyond chunk content; design your metadata schema to match how you'll segment queries.

---

## 6. Common patterns

### Customer-support bot

1. Create `support-kb` (access_mode: `shared`).
2. Ingest your help-center articles (markdown) and product PDFs.
3. From a serverless function: `rag_query` with `synthesize: true`, return the answer + top 3 chunks as citations.

### Per-tenant knowledge base

1. Create one collection per tenant (`access_mode: "custom"`).
2. Tag every document with `metadata: { tenant_id }`.
3. Query with `filter: { tenant_id: ctx.user.tenant_id }` from a function.
4. Use RLS on the wrapping table to gate which tenant a user can query.

### Versioned docs

Tag with `metadata: { version: "v3" }`. Query with `filter: { version: "v3" }`. To deprecate `v2`, delete just those documents — no need to rebuild the collection.

---

## 7. Errors and pitfalls

| Error | Cause |
|-------|-------|
| `RESOURCE_NOT_FOUND` | App / collection / document doesn't exist |
| `VALIDATION_DUPLICATE_NAME` | Collection name already taken |
| `VALIDATION_ERROR` | `ingest_document` with neither `text` nor `storage_object_id` |
| `COLLECTION_EMPTY` | `rag_query` against a collection with no `ready` docs |

**Pitfalls:**

- Polling too aggressively wastes quota; backoff after the first minute.
- `chunk_size` / `chunk_overlap` are immutable — get them right up front.
- `synthesize: true` adds LLM latency + cost. For low-latency UX, do raw retrieval and synthesize on the frontend asynchronously.
- Metadata is exact-match only — no LIKE, no ranges. Pre-bucket continuous values (e.g. `tier: "free" | "pro"`) before ingesting.
- File ingestion **requires** prior upload to `manage_storage`; you cannot stream raw bytes into `ingest_document`.
- A failed document stays in the collection with `status: "failed"` and an `errorMessage`. Delete and re-ingest to retry.

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-rag` so the journey orchestrator stays in sync.
