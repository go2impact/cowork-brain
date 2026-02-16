# 2. RAG & Knowledge Base

[< Back to index](README.md)

**Their claim:** "Built-in vector database with support for document retrieval and knowledge Q&A"

**Maps to Cowork.ai:** [Context](../../product/product-features.md#6-context) — our ambient capture feeds embeddings; their KB is manual upload. Same vector infrastructure, different input source.

## How It Works

Users manually upload documents (PDF, DOCX, XLSX, images, URLs, plain text). Documents are parsed, chunked, embedded, and stored as vectors in libsql. At query time, cosine similarity search retrieves top-K chunks with optional reranking.

**Key files:**
- `src/main/knowledge-base/index.ts` — KnowledgeBaseManager (531 lines)
- `src/main/mastra/storage/index.ts` — LibSQLVector config
- `src/main/utils/loaders/` — Document parsers (PDF, Word, Excel, PPT, OCR)
- `src/entities/knowledge-base.ts` — TypeORM entities

## Ingestion Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DOCUMENT INGESTION PIPELINE                        │
└──────────────────────────────────────────────────────────────────────────┘

  User uploads file
       │
       ▼
  ┌─────────────────┐
  │  Source Router   │
  │                  │
  │  Text ──────────▶ Direct content
  │  Web URL ───────▶ Fetch + extract
  │  File ──────────▶ Format parser ─┐
  │  Folder ────────▶ (not impl.)    │
  └─────────────────┘                │
                                     ▼
                         ┌───────────────────────┐
                         │    Format Parsers      │
                         │                        │
                         │  .pdf  → pdf-parse     │
                         │          + OCR fallback │
                         │  .docx → mammoth       │
                         │  .doc  → word-extractor│
                         │  .xlsx → xlsx lib      │
                         │  .pptx → officeparser  │
                         │  .png  → @napi-rs/     │
                         │         system-ocr     │
                         │  .jpg  → @napi-rs/     │
                         │         system-ocr     │
                         └───────────┬────────────┘
                                     │ raw text
                                     ▼
                         ┌───────────────────────┐
                         │   Chunking            │
                         │   @mastra/rag v2      │
                         │                        │
                         │   MDocument.fromText() │
                         │   .chunk({             │
                         │     strategy: "markdown"│  ◀── or "recursive"
                         │     maxSize: 512,      │
                         │     overlap: 50,       │
                         │     separators: ["\n"] │
                         │   })                   │
                         └───────────┬────────────┘
                                     │ chunks[]
                                     ▼
                         ┌───────────────────────┐
                         │   Embedding           │
                         │   Vercel AI SDK       │
                         │                        │
                         │   embedMany({          │
                         │     model: provider/id,│  ◀── e.g. ollama/qwen3-embedding
                         │     values: chunks     │
                         │   })                   │
                         └───────────┬────────────┘
                                     │ float32[][]
                                     ▼
                         ┌───────────────────────┐
                         │   Vector Storage      │
                         │   libsql native       │
                         │                        │
                         │   INSERT INTO          │
                         │   [kb_{id}_{dim}]      │
                         │   (id, item_id, chunk, │
                         │    is_enable,           │
                         │    embedding,           │  ◀── vector32(json)
                         │    metadata)            │
                         └───────────────────────┘
```

## Vector Table Schema (per Knowledge Base)

```sql
CREATE TABLE IF NOT EXISTS [kb_{kbId}_{embedding_length}] (
    id            TEXT PRIMARY KEY,
    item_id       TEXT,                           -- FK to knowledge_base_item
    chunk         TEXT,                           -- raw text chunk
    is_enable     BOOLEAN,                        -- soft delete
    embedding     F32_BLOB({embedding_length}),   -- native vector column
    metadata      TEXT DEFAULT '{}'                -- JSON metadata
);
```

## RAG Query Flow

```
User asks question
       │
       ▼
  Embed query via same model
       │
       ▼
  SELECT with cosine similarity:

    WITH vector_scores AS (
      SELECT id, item_id, chunk,
        (1 - vector_distance_cos(embedding, '{query_vec}')) as score,
        metadata
      FROM [kb_{kbId}_{dim}]
      WHERE is_enable = 1
    )
    SELECT * FROM vector_scores
    WHERE score > 0.5           ◀── 50% similarity threshold
    ORDER BY score DESC
    LIMIT 10                    ◀── top-K = 10
       │
       ▼
  Optional: reranker pass
       │
       ▼
  Inject top chunks into agent context
```

## Chunking Parameters

| Source Type | Strategy | Max Size | Overlap | Separators |
|-------------|----------|----------|---------|------------|
| Text | markdown | 512 chars | 50 chars | — |
| Web | recursive | 512 chars | 50 chars | `["\n"]` |
| File | recursive | 512 chars | 50 chars | `["\n"]` |

## What We Should Do

| Action | Detail |
|--------|--------|
| **Copy pattern** | Their libsql vector schema — `F32_BLOB` column type, `vector_distance_cos()`, `vector32()` for insert. This is exactly how we should store activity embeddings. |
| **Copy pattern** | Background task queue for embedding jobs (`TaskQueueManager` with `groupMaxConcurrency: 1` per KB). We need this for activity capture embedding. |
| **Study** | Chunking via `@mastra/rag` v2 — `MDocument.fromText().chunk()`. We should use the same library for chunking activity data. |
| **Study** | Reranker integration — optional second pass with a rerank model. Worth adding for our RAG retrieval. |
| **Skip** | Their manual upload UX — we don't need it. Our context comes from ambient capture, not user uploads. |
| **Note** | Their chunk params (512 chars, 50 overlap) are conservative defaults. We should benchmark different sizes for activity data which has different structure than documents. |
