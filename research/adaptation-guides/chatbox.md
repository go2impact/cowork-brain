# Chatbox → Cowork.ai Adaptation Guide

**Purpose:** Maps the key patterns from Chatbox to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our multi-process architecture.

**Audience:** Engineering (Rustan + team)
**Target architecture:** [`architecture/system-architecture.md`](../../architecture/system-architecture.md)

**Why Chatbox?** Chatbox is the only repo in our competitive set that uses **Mastra LibSQLVector** (via `@mastra/libsql`) — the exact same vector storage stack we're building on. It also has production patterns for libsql in Electron (packaging, migration, corruption avoidance) and an MCP stdio-over-IPC bridge that solves a problem we'll face. These patterns don't exist in aime-chat or Cherry Studio.

---

## Sections

| # | Section | Status |
|---|---|---|
| 0 | [Mastra LibSQLVector in Electron](#0-mastra-libsqlvector-in-electron) | Done |
| 1 | [Resumable Ingestion Pipeline](#1-resumable-ingestion-pipeline) | Done |
| 2 | [libsql Schema Migrations](#2-libsql-schema-migrations) | Done |
| 3 | [MCP stdio-over-IPC Bridge](#3-mcp-stdio-over-ipc-bridge) | Done |
| 4 | [libsql Packaging in Electron](#4-libsql-packaging-in-electron) | Done |
| 5 | [Summary](#5-summary) | Done |

---

## 0. Mastra LibSQLVector in Electron

**Source:** Chatbox's `src/main/knowledge-base/db.ts`

This is the most directly relevant pattern in our entire competitive set. Chatbox uses `@mastra/libsql`'s `LibSQLVector` with a local `file:` URL — the same setup we plan for `cowork.db`. Their hard-won lessons about single-client access and corruption avoidance are directly applicable.

### Copy

#### Single-client access pattern

Chatbox's most critical libsql pattern. Instead of creating a separate `libsql` `Client` instance for SQL queries, they extract the internal `turso` client from `LibSQLVector` and reuse it for all database operations.

**Chatbox's approach (`src/main/knowledge-base/db.ts:116-119`):**

```
const vectorStore = new LibSQLVector({ connectionUrl: `file:${dbPath}` })
const db = (vectorStore as any).turso  // Extract internal client
```

**Why this matters:** libsql in WAL mode allows one writer. If two `Client` instances both try to write (one for vector operations, one for schema queries), you get `SQLITE_BUSY` errors or corruption. By reusing the same client, all writes serialize through one connection.

**Our equivalent:**

```
Agents & RAG utility process:
  LibSQLVector({ connectionUrl: 'file:cowork.db' })
      ↓
  Extract turso client → use for:
    - LibSQLVector operations (embeddings, similarity search)
    - LibSQLStore operations (agent memory, threads)
    - App table queries (capture data reads, context queries)
```

What changes:
- **Scope is broader.** Chatbox has one `chatbox_kb.db` for knowledge base only. We have one `cowork.db` for everything — capture data, agent memory, embeddings. The single-client principle is even more critical because more code paths write.
- **Two processes write.** Chatbox has one writer (main process). We have two: the Capture Utility process (sync writes via `libsql`) and the Agents & RAG utility process (async writes via `@mastra/libsql`). WAL mode supports this — one writer at a time, readers don't block. But we must ensure each process has exactly one client. The capture process creates its own `libsql` Client (sync API for high-frequency writes). The agents process extracts from `LibSQLVector` (Chatbox's pattern).
- **No `(vectorStore as any).turso` hack.** Chatbox reaches into a private field. We should check if Mastra exposes the client via a public API (e.g., `vectorStore.getClient()`). If not, we file an issue or wrap the access in a utility that's clearly marked as depending on Mastra internals.

**Reference:** [system-architecture.md — Database Access Pattern](../../architecture/system-architecture.md#database-access-pattern)

#### Vector index per namespace

Chatbox creates vector indexes named `kb_{kbId}` — one per knowledge base. This isolates different document collections.

We adopt the same pattern with different namespaces:

| Chatbox index | Our index | What it stores |
|---|---|---|
| `kb_{kbId}` (per knowledge base) | `embed_activity` | Embedded window titles, URLs, app sessions |
| — | `embed_conversation` | Embedded past conversations |
| — | `embed_observation` | Embedded observational memory summaries |

Separate indexes let us query specific data types independently or merge results across types with different relevance weights. `LibSQLVector.createIndex({ indexName, dimension })` handles creation — same API call as Chatbox.

### Adapt

#### Search flow — our queries span multiple indexes

Chatbox queries a single vector index per knowledge base:

```
query → embed → vectorStore.query({ indexName: 'kb_1', queryVector, topK: 20 })
```

Our RAG retrieval queries multiple indexes and merges results:

```
query → embed via Qwen3-0.6B →
  parallel:
    vectorStore.query({ indexName: 'embed_activity', topK: 10 })
    vectorStore.query({ indexName: 'embed_conversation', topK: 10 })
    vectorStore.query({ indexName: 'embed_observation', topK: 5 })
  → merge by relevance score
  → priority: conversation > activity > observation
  → assemble context window
```

This is architecturally different from Chatbox's single-index search. We need a merge strategy (simple score-based ranking for v0.1, potentially learned weights later).

### Study

#### Direct SQL for complex vector queries

Chatbox's `readChunks()` function bypasses `LibSQLVector`'s query API and directly queries the underlying SQLite via `(vectorStore as any).turso.execute()`. It uses `json_extract()` on metadata columns and composite WHERE clauses to avoid SQLite's 999-parameter limit.

**When we might need this:** If `LibSQLVector.query()` doesn't support filtering by metadata fields (e.g., "find activity embeddings from the last 24 hours"), we'll need direct SQL queries against the vector table. Study Chatbox's pattern for metadata extraction and parameterization.

**Prefer:** Check if `LibSQLVector.query()` supports a `filter` parameter first. Direct SQL is a last resort — it couples us to the internal table schema.

### Skip

#### Separate KB database file

Chatbox creates `chatbox_kb.db` as a separate database file from its main storage. We don't need this — `cowork.db` holds everything (see [system-architecture.md — Database Layer](../../architecture/system-architecture.md#database-layer)). One file simplifies backup, migration, and atomic operations across tables.

---

## 1. Resumable Ingestion Pipeline

**Source:** Chatbox's `src/main/knowledge-base/file-loaders.ts`

Chatbox's ingestion pipeline processes documents into vector embeddings with full resumability — if the app crashes mid-ingestion, it picks up where it left off. This is critical for us because our capture process generates a continuous stream of embeddable data that can be interrupted at any time.

### Copy

#### Status state machine for ingestion jobs

Chatbox tracks each file's ingestion state with explicit transitions:

```
pending → processing → done
              ↓
           failed
              ↓
           paused (crash recovery or user pause)
              ↓
           pending (user resume)
```

**Schema fields (`src/main/knowledge-base/db.ts:39-50`):**

| Column | Type | Purpose |
|---|---|---|
| `status` | TEXT | State machine: `pending`, `processing`, `done`, `failed`, `paused` |
| `chunk_count` | INTEGER | Chunks processed so far (progress checkpoint) |
| `total_chunks` | INTEGER | Total chunks in document (set after chunking) |
| `processing_started_at` | DATETIME | When processing began (for timeout detection) |
| `error` | TEXT | Human-readable error message on failure |

**We adopt this for our embedding queue.** Our input is different (capture data instead of uploaded files), but the state machine is identical:

| Chatbox field | Our equivalent | Source |
|---|---|---|
| `status` | `embed_status` | Capture flush → `pending`. Embedding picks up → `processing`. Done → `done`. |
| `chunk_count` | `chunks_embedded` | How many chunks from this data batch are embedded |
| `total_chunks` | `total_chunks` | Total after chunking |
| `processing_started_at` | `embed_started_at` | Timeout detection (same 5-minute threshold) |
| `error` | `embed_error` | Error message for debugging |

#### Batch processing with progress checkpoints

Chatbox embeds in batches of 50 chunks with a progress update after each batch:

```
remaining = allChunks.slice(currentChunkCount)  // Resume from checkpoint

for batch in remaining (BATCH_SIZE=50):
    1. Check if paused → stop gracefully
    2. embedMany({ model, values: batchTexts })
    3. vectorStore.upsert({ indexName, vectors, metadata })
    4. UPDATE kb_file SET chunk_count = newCount  // Checkpoint
    5. sleep(100ms)  // Rate limit
```

**Critical detail:** Progress is persisted to the database after every batch, not just at completion. If the app crashes after batch 3 of 10, it resumes from chunk 150 (not chunk 0).

We adopt this pattern for our embedding pipeline in the Agents & RAG utility process. The batch size may differ (our chunks are shorter — window titles and URLs — so larger batches may be efficient), but the checkpoint-after-each-batch principle is essential.

#### Startup crash recovery

Chatbox recovers from crash states at startup:

1. **`cleanupProcessingFiles()` (`db.ts:233`):** Moves any files stuck in `processing` state to `paused`. This handles the case where the app crashed during ingestion.
2. **`checkProcessingTimeouts()` (`db.ts:255`):** Marks files as `failed` if `processing_started_at` is older than 5 minutes. This handles hangs.

```
App startup →
  cleanupProcessingFiles():
    UPDATE kb_file SET status='paused' WHERE status='processing'
  checkProcessingTimeouts():
    UPDATE kb_file SET status='failed' WHERE status='processing'
      AND processing_started_at < (now - 5min)
```

We need the same for our embedding queue. When the Agents & RAG utility process restarts (auto-restart via Electron), it must recover any in-flight embedding jobs.

### Adapt

#### Input source — continuous capture vs. file upload

Chatbox's pipeline starts when a user uploads a file. Our pipeline starts when the capture process flushes data to `cowork.db`. The ingestion trigger is different, but the processing pipeline is the same.

| Stage | Chatbox | Cowork.ai |
|---|---|---|
| **Trigger** | User uploads file → `pending` row in `kb_file` | Capture process flushes → `pending` row in embedding queue |
| **Polling** | Worker loop polls every 3s for `status='pending'` | Same polling pattern in Agents & RAG utility process |
| **Parsing** | Format-specific parsers (PDF, DOCX, etc.) | No parsing needed — capture data is already text |
| **Chunking** | `MDocument.fromText().chunk({ strategy: 'recursive', size: 512, overlap: 50 })` | Same library, potentially different sizes (activity data is shorter) |
| **Embedding** | `embedMany()` via configurable AI SDK provider | `embedMany()` via Qwen3-Embedding-0.6B (Ollama, local) |
| **Storage** | `vectorStore.upsert()` via LibSQLVector | Same |
| **Process** | Main process | Agents & RAG utility process |

#### Chunking parameters — tune for activity data

Chatbox uses `size: 512, overlap: 50` for all document types. Our activity data has different characteristics:

- **Window titles:** 10-80 chars. No chunking needed — embed as-is.
- **URLs:** Short. No chunking needed.
- **Keystroke chunks:** Up to 1000 chars per flush. May benefit from smaller chunks (256) for more precise retrieval.
- **Conversation messages:** Variable. AI SDK message boundaries are natural chunk points.

We start with Chatbox's defaults and benchmark retrieval quality with our data types. The `@mastra/rag` v2 chunking library supports multiple strategies — `recursive` with Chatbox's parameters is the baseline.

### Skip

#### File upload UI and management

Chatbox has IPC handlers for upload, retry, pause, resume, and delete operations. We don't need these — our ingestion is automatic from the capture pipeline, with no user-facing file management.

#### User-configurable parser selection

Chatbox lets users choose between local parsing and remote "chatbox-ai" parsing for documents. We don't parse documents in v0.1 — our input is already text.

---

## 2. libsql Schema Migrations

**Source:** Chatbox's `src/main/knowledge-base/db.ts:56-98`

Chatbox uses a simple but effective migration strategy: additive ALTER TABLE statements that silently ignore "duplicate column" errors. This lets the app add new columns without tracking migration versions.

### Copy

#### Additive migration with duplicate-column tolerance

Chatbox's migration approach:

```typescript
async function initDB(db: Client) {
  // Create tables if not exist
  await db.execute(`CREATE TABLE IF NOT EXISTS knowledge_base (...)`)
  await db.execute(`CREATE TABLE IF NOT EXISTS kb_file (...)`)

  // Additive migrations — each silently ignores "duplicate column"
  try {
    await db.execute('ALTER TABLE kb_file ADD COLUMN document_parser TEXT')
  } catch (e) {
    // Column already exists — ignore
  }
  try {
    await db.execute('ALTER TABLE kb_file ADD COLUMN provider_mode TEXT')
  } catch (e) {
    // Column already exists — ignore
  }
  // ... more additive columns
}
```

**Why this works for Electron:** Desktop apps don't have a server-side migration runner. The app starts, runs `initDB()`, and all schema changes are applied idempotently. No migration version table, no rollback — just forward progress.

We adopt this for `cowork.db`. Our schema is not finalized (see [system-architecture.md — Database Layer](../../architecture/system-architecture.md#database-layer)), and the additive pattern lets us evolve it across app updates without a formal migration framework.

**What we add:**
- **Column type validation.** Chatbox only checks for duplicate column names. We should also validate that existing columns have the expected type (e.g., if we change a column from TEXT to INTEGER in a future release, the additive pattern won't catch it).
- **Logging.** Log which migrations ran vs. which were already applied. Useful for debugging schema-related issues on user machines.

#### Transaction wrapper for multi-step operations

Chatbox wraps multi-step operations in manual transactions:

```typescript
async function withTransaction(db: Client, fn: (tx: Transaction) => Promise<void>) {
  const txId = crypto.randomUUID()
  await db.execute('BEGIN')
  try {
    await fn({ execute: (sql, args) => db.execute({ sql, args }) })
    await db.execute('COMMIT')
  } catch (e) {
    try {
      await db.execute('ROLLBACK')
    } catch (rollbackError) {
      // Distinguish rollback failure from original error
      console.error(`Transaction ${txId} rollback failed`, rollbackError)
    }
    throw e
  }
}
```

We need this for operations that touch multiple tables atomically — e.g., deleting a user's capture data (clear activity rows + clear embedding rows + clear vector index entries). Without a transaction, a crash mid-deletion leaves inconsistent state.

### Adapt

#### Migration scope — more tables, more processes

Chatbox migrates 2 tables in one process. We migrate multiple table groups across two writer processes:

| Table group | Process that writes | Migration location |
|---|---|---|
| App tables (activities, input events, context streams) | Capture Utility | Capture process startup |
| LibSQLStore tables (agent memory, threads, traces) | Agents & RAG Utility | Mastra handles internally |
| LibSQLVector tables (embeddings) | Agents & RAG Utility | Mastra handles internally |
| Embedding queue table | Agents & RAG Utility | Agents process startup |

**Key constraint:** Both processes must agree on the schema. The capture process creates app tables; the agents process creates its own tables. Neither should ALTER tables owned by the other process. Schema ownership is per-process, not global.

### Skip

#### Version-tracked migrations

Chatbox doesn't track migration versions — each `ALTER TABLE ADD COLUMN` is idempotent. For v0.1, this is sufficient. If we later need destructive migrations (dropping columns, changing types), we'll add a `schema_version` table. But not now — premature migration infrastructure adds complexity for a schema that isn't finalized.

---

## 3. MCP stdio-over-IPC Bridge

**Source:** Chatbox's `src/renderer/packages/mcp/ipc-stdio-transport.ts` (renderer) + `src/main/mcp/ipc-stdio-transport.ts` (main)

MCP servers that use stdio transport (running as child processes) can't be spawned from the renderer — it's sandboxed. Chatbox solves this with a clean proxy pattern: the renderer creates a proxy transport that forwards all MCP protocol messages to the main process via IPC, where the real `StdioClientTransport` spawns and manages the child process.

### Adapt

#### Bridge location — main process vs. utility process

Chatbox bridges stdio transports through the main process because that's where their MCP logic lives. Our MCP connections live in the Agents & RAG utility process.

**Chatbox's flow:**

```
Renderer → IPC → Main process → StdioClientTransport → MCP server child process
```

**Our flow:**

```
Agents & RAG Utility → direct → StdioClientTransport → MCP server child process
```

**Key difference:** We don't need the renderer-to-main IPC bridge for stdio transports because our MCP connections aren't in the renderer. The Agents & RAG utility process has full Node.js access and can spawn child processes directly.

**Where the bridge pattern IS useful for us:** If we ever need the renderer to communicate with MCP servers directly (e.g., for the Apps feature where a third-party app calls MCP tools), we'd need a similar bridge. But for v0.1, all MCP traffic flows through the agents process.

#### What we take from the pattern

Even though we don't need the renderer→main bridge, the implementation patterns are valuable:

1. **UUID-per-transport instance.** Chatbox assigns a unique ID to each stdio transport and uses it for event routing (`mcp:stdio-transport:${transportId}:onmessage`). This pattern isolates multiple concurrent MCP server connections.

2. **Stderr capture with encoding detection.** Chatbox captures stderr from the child process, detects encoding via `chardet`, and decodes with `iconv`. MCP servers can emit debug output or errors on stderr — capturing it is essential for connection health reporting.

3. **Lifecycle management.** `transportMap` in main tracks all active transports. `closeAllTransports()` at app shutdown ensures clean process cleanup. We need the same — dangling child processes after app exit is a common Electron bug.

4. **Event posting with try-catch guards.** Chatbox wraps `event.sender.send()` in try-catch because the renderer may have navigated away or closed. We need the same for any IPC message from the utility process to main — the main window may have been destroyed.

### Copy

#### Tool namespacing: `mcp__{server}__{tool}`

Chatbox namespaces MCP tools as `mcp__{serverName}__{toolName}`:

```
"Brave Search" server + "search" tool → mcp__brave_search__search
```

With regex safety: server names are lowercased and spaces replaced with underscores. Names that don't match `/^[A-Za-z0-9_-]+$/` fall back to just `mcp__{toolName}`.

Cherry Studio uses `serverId__toolName` (single underscore separator). We adopt Chatbox's double-underscore convention because it's unambiguous — tool names themselves may contain single underscores.

#### Error-as-return for tool execution

Chatbox wraps each MCP tool's `execute()` in a try-catch that returns the error instead of throwing:

```typescript
execute: async (args, options) => {
  try {
    return await tool.execute?.(args, options)
  } catch (err) {
    return err  // Return error, don't throw
  }
}
```

This prevents one failed tool call from aborting the entire agent stream. The LLM sees the error in the tool result and can decide what to do (retry, use a different tool, tell the user).

We adopt this with one addition: our safety rails count consecutive tool errors. If a tool fails 3 times with the same input, we pause and surface the error to the user (see [system-architecture.md — Safety Rails](../../architecture/system-architecture.md#safety-rails)).

### Study

#### MCP controller status subscriptions

Chatbox's `MCPServer` class extends `Emittery` for reactive status updates. UI components subscribe to status changes and update connection health indicators in real-time:

```
MCPServer emits status → { state: 'running' }
  → subscribed React component updates → green dot
MCPServer emits status → { state: 'idle', error: 'auth expired' }
  → subscribed React component updates → red dot + error message
```

**Evaluate for:** Our MCP Integrations feature needs the same reactive health indicators. Study whether `Emittery` or a simpler EventEmitter-based pattern fits our architecture (the status events need to cross the IPC boundary from the agents process to the renderer via main).

### Skip

#### Renderer-side MCP orchestration

Chatbox runs its MCP controller and tool merging in the renderer process. HTTP transports connect directly from the renderer; stdio transports proxy through main.

We skip this entirely. All MCP connections live in the Agents & RAG utility process. The renderer never touches MCP — it sees tool results via the chat stream (agent chunks forwarded through IPC).

---

## 4. libsql Packaging in Electron

**Source:** Chatbox's `electron-builder.yml` + `.erb/scripts/patch-libsql.cjs` + `patches/libsql@0.5.22.patch`

libsql has a native Node.js binding (`@libsql/client`) that requires special handling in Electron's ASAR-packed builds. Chatbox has production-tested patterns for packaging, patching, and graceful degradation.

### Copy

#### ASAR unpack for native modules

Chatbox's `electron-builder.yml` explicitly unpacks libsql from the ASAR archive:

```yaml
asar: true
asarUnpack:
  - "**\\*.{node,dll}"           # All native binaries
  - "**/node_modules/libsql/**"  # Entire libsql package
```

Native `.node` files can't be loaded from inside an ASAR archive (they need a real filesystem path for `dlopen()`). Unpacking libsql ensures the native binding is accessible at runtime.

We need the same for `cowork.db`'s libsql access AND for our native capture addons (`coworkai-keystroke-capture`, `coworkai-activity-capture`). Our `asarUnpack` list:

```yaml
asarUnpack:
  - "**\\*.{node,dll}"
  - "**/node_modules/libsql/**"
  - "**/node_modules/coworkai-keystroke-capture/**"
  - "**/node_modules/coworkai-activity-capture/**"
```

#### Post-pack patching for cross-platform compatibility

Chatbox's `.erb/scripts/patch-libsql.cjs` runs after `electron-builder` packages the app. It patches libsql's JavaScript loader to:

1. **Fix error message detection:** Handles both `@libsql/target` and `@libsql\target` (backslash-escaped) in error messages.
2. **Add platform guards:** Returns empty object on unsupported platforms (Windows ARM64) instead of crashing.
3. **Wrap require() in try-catch:** Graceful degradation if native module isn't found — falls back to pure JS.

We adopt this pattern. Our target is Mac-only for v0.1, but the try-catch wrapper is still valuable — if the native libsql binary doesn't load for any reason (corrupt download, permission issue), the app should report the error clearly instead of crashing on startup.

#### Dual patching strategy

Chatbox patches libsql at two points:

1. **Install-time:** `pnpm` applies `patches/libsql@0.5.22.patch` during `npm install`.
2. **Post-pack:** `.erb/scripts/patch-libsql.cjs` re-applies patches after electron-builder creates the distributable.

This belt-and-suspenders approach ensures patches survive regardless of how the build pipeline transforms `node_modules`.

### Adapt

#### Version-specific patches

Chatbox's patches are for `libsql@0.5.22`. Our libsql version may differ (we use whatever `@mastra/libsql` depends on). We need to:

1. Check what libsql version Mastra pulls in.
2. Test if the same issues exist in that version.
3. Create version-specific patches if needed.

The patching infrastructure (patch file + post-pack script) transfers regardless of version — only the patch content changes.

### Skip

#### Windows ARM64 guard

Chatbox adds a guard for Windows ARM64 because there's no native libsql binary for that platform. We're Mac-only for v0.1. If we add Windows support later, we'll revisit platform-specific guards.

---

## 5. Summary

### Copy These Patterns

| Pattern | Chatbox source | Cowork.ai target | Why |
|---|---|---|---|
| Single-client libsql access | `db.ts:116-119` (extract `turso` from LibSQLVector) | `src/core/store/db.ts` | Prevent SQLITE_BUSY / corruption with shared cowork.db |
| Vector index per namespace | `kb_{kbId}` naming convention | `embed_activity`, `embed_conversation`, `embed_observation` | Separate data types for independent or merged queries |
| Resumable ingestion state machine | `kb_file.status` + `chunk_count` + `total_chunks` | Embedding queue table in cowork.db | Capture data can be interrupted by crashes or thermal shutdown |
| Batch embedding with checkpoints | `BATCH_SIZE=50`, update chunk_count per batch | Same pattern in Agents & RAG utility | Progress survives crashes |
| Startup crash recovery | `cleanupProcessingFiles()` + `checkProcessingTimeouts()` | Agents process startup routine | Recover from utility process crashes (auto-restart) |
| Additive schema migrations | `ALTER TABLE ADD COLUMN` with duplicate-column tolerance | `initDB()` per process | Schema evolution without migration framework |
| Transaction wrapper | `withTransaction(db, fn)` with rollback error handling | Same utility in `src/core/store/` | Atomic multi-table operations |
| Tool namespacing (`mcp__server__tool`) | `controller.ts:222` | `src/core/mcp/tool-registry.ts` | Disambiguate tools from different MCP servers |
| Error-as-return for tool execution | Tool execute() wrapped in try-catch → return error | Same pattern in our tool executor | Prevent one failed tool from aborting agent stream |
| ASAR unpack for native modules | `electron-builder.yml:4-6` | Same in our electron-builder config | libsql + capture addons need real filesystem paths |
| Post-pack libsql patching | `.erb/scripts/patch-libsql.cjs` | Same pattern in our build pipeline | Graceful degradation if native module fails to load |

### Study These

| Pattern | What to evaluate | When |
|---|---|---|
| Direct SQL for vector metadata queries | Does LibSQLVector.query() support metadata filters? If not, direct SQL needed | RAG implementation |
| MCP controller status subscriptions (Emittery) | Status events across IPC boundary for reactive connection health UI | MCP Integrations feature design |
| Chunking parameters for activity data | 512/50 defaults vs. smaller chunks for short activity strings | Embedding pipeline benchmarks |
| Per-batch pause/resume check | Chatbox checks `status='paused'` before each batch. Apply to thermal events. | Thermal management integration |

### Skip These

| Pattern | Why skip |
|---|---|
| Separate KB database file (`chatbox_kb.db`) | Single `cowork.db` for everything — simpler backup, migration, atomic operations |
| File upload UI and management | Our ingestion is automatic from capture pipeline |
| User-configurable parser selection | Our input is already text from capture — no parsing needed |
| Renderer-side MCP orchestration | All MCP connections in Agents & RAG utility process |
| Stdio-over-IPC bridge (renderer→main) | Our MCP connections aren't in the renderer — utility process spawns directly |
| Version-tracked migration framework | Additive pattern sufficient for v0.1, schema not finalized |
| `(vectorStore as any).turso` hack | Check for public API first; wrap in utility if unavoidable |

### What Chatbox Has That No Other Repo Covers (Unique Gap-Fillers)

| Gap | Other repos | Chatbox | Our approach |
|---|---|---|---|
| **Mastra LibSQLVector in Electron** | No one else uses it | Production single-client pattern with corruption avoidance | Direct copy — same stack |
| **Resumable embedding pipeline** | aime-chat has basic queue, no resume | Full state machine with chunk_count checkpoints and crash recovery | Copy state machine, adapt for capture data instead of file upload |
| **libsql packaging in Electron** | Cherry Studio uses IndexedDB/Dexie, not libsql. aime-chat uses libsql but simpler packaging | ASAR unpack + dual patching (install-time + post-pack) | Direct copy — same packaging challenge |
| **Schema migration without framework** | Others use Drizzle ORM (LobeChat) or manual SQL (aime-chat) | Additive ALTER TABLE with duplicate-column tolerance | Copy approach, add logging and per-process schema ownership |

---

## Changelog

**v1 (Feb 16, 2026):** Initial draft. All 5 sections complete: Mastra LibSQLVector in Electron, Resumable Ingestion Pipeline, libsql Schema Migrations, MCP stdio-over-IPC Bridge, libsql Packaging in Electron.

**v1.1 (Feb 17, 2026):** Fact-check corrections: LibSQLVector comes from `@mastra/libsql` not `@mastra/core`, `processing_started_at` column type is DATETIME not TIMESTAMP.
