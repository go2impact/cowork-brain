# Research: Database Stack — better-sqlite3 + sqlite-vec vs libsql

| | |
|---|---|
| **Status** | Research complete — recommendation pending team discussion |
| **Last Updated** | 2026-02-16 |
| **Related** | [DESKTOP_SALVAGE_PLAN.md](./DESKTOP_SALVAGE_PLAN.md), [decision-log.md](./decision-log.md) (SQLite-vec over LanceDB entry) |

---

## Context

The [Desktop Salvage Plan](./DESKTOP_SALVAGE_PLAN.md) specifies `better-sqlite3` + `sqlite-vec` as the database stack. However, our AI agent orchestration layer (Mastra.ai) natively supports libsql via `@mastra/libsql`. If libsql can replace both `better-sqlite3` and `sqlite-vec`, we unify the database stack into a single dependency that Mastra agents can directly read/write.

---

## The Two Options

### Option A: better-sqlite3 + sqlite-vec (Current Plan)

- `better-sqlite3` for structured storage (activity, context, conversations, working memory)
- `sqlite-vec` extension for vector embeddings (RAG retrieval)
- Custom Mastra storage adapter needed for agent memory

### Option B: libsql (Unified)

- Single `libsql` dependency for structured storage AND built-in vector search
- Native Mastra integration via `@mastra/libsql` (`LibSQLStore` + `LibSQLVector`)
- One `.db` file for everything

---

## better-sqlite3

### Strengths

| Dimension | Detail |
|---|---|
| **API** | Synchronous. `db.prepare(sql).run()` returns immediately. Simple, deterministic, no async complexity. |
| **Electron maturity** | Years of production use. Pre-built binaries for Electron (including v39+, Apple Silicon). Known workarounds for every edge case. |
| **Performance** | Widely regarded as the fastest SQLite library for Node.js. Direct C API bindings. |
| **WAL mode** | Full support. Already in use in the existing codebase. |
| **Maintenance** | v12.6.2 (Jan 16, 2026). Releases every 1-2 weeks. ~1.5M weekly downloads. 3,400+ npm dependents. |
| **Extension loading** | `db.loadExtension(path)` — works with sqlite-vec. |

### Weaknesses

| Dimension | Detail |
|---|---|
| **Mastra integration** | None built-in. Would need a custom `MastraStorageAdapter` wrapping better-sqlite3 to expose it to Mastra agents. This is a documented extension point in Mastra but requires engineering effort. |
| **Vector search** | Requires sqlite-vec as a separate extension. sqlite-vec has a known macOS Electron issue (`.dylib.dylib` double-extension bug in electron-builder). Needs `asarUnpack` config for `.dylib` files. |
| **Two native modules** | Both better-sqlite3 and sqlite-vec need Electron ABI rebuilds and ASAR unpacking. More packaging complexity. |

---

## libsql

### What It Is

Open-source fork of SQLite by Turso. Adds native vector search (DiskANN algorithm), embedded replicas, and ALTER TABLE extensions. Fully SQLite-compatible for standard operations.

### Two Different npm Packages (Critical Distinction)

| Package | API | Mastra Uses It? | better-sqlite3 Compatible? |
|---|---|---|---|
| **`libsql`** (npm) | **Synchronous** (with opt-in async) | No | Yes — designed as drop-in replacement |
| **`@libsql/client`** (npm) | **Async only** (all operations return Promises) | **Yes** — `@mastra/libsql` depends on this | No |

This is the key tension. The sync API lives in one package; the Mastra integration lives in the other.

### Strengths

| Dimension | Detail |
|---|---|
| **Mastra integration** | Native. `LibSQLStore` for structured data, `LibSQLVector` for embeddings. Both point to the same `.db` file. Agents directly read/write context, memory, and embeddings with zero bridge code. |
| **Built-in vector search** | No extension needed. Supports F32 vectors up to 65,536 dimensions. Uses DiskANN (approximate nearest neighbor) which scales better than sqlite-vec's brute-force KNN. |
| **Single dependency** | Replaces both better-sqlite3 and sqlite-vec. One native module, one `.db` file, no extension loading. |
| **Eliminates sqlite-vec macOS issues** | No `.dylib` packaging, no `loadExtension()` call path, no ASAR unpacking for extensions. |
| **Local-only mode** | Fully supported. `file:./path/to/db.db` — no Turso account, no network, no cloud dependency. |
| **Maintenance** | Turso is a funded company with a team. libsql core: 18.7K GitHub stars, actively developed. |

### Weaknesses

| Dimension | Detail |
|---|---|
| **Async API via Mastra** | `@libsql/client` (what Mastra uses) is async-only. The entire data layer becomes `await client.execute(...)` instead of `db.prepare(...).run()`. Non-trivial refactor for high-frequency capture writes. |
| **Electron maturity** | **Unproven.** No Electron-specific prebuilds. No documented production use in Electron. Known cross-platform build issues (`@libsql/darwin-arm64` not pulled correctly in some build scenarios). |
| **Pre-release status** | The `libsql` npm package (sync API) is v0.6.0-pre.29 — still pre-release. `@libsql/client` is stable. |
| **Vector search limitations** | Currently only supports cosine distance. sqlite-vec supports L2, L1, cosine, and hamming. (Cosine is sufficient for our use case — embedding similarity.) |
| **No head-to-head benchmarks** | No published local-only performance comparison vs better-sqlite3. |
| **New territory** | Zero existing references to libsql in the cowork-brain repo. All architecture docs assume better-sqlite3 + sqlite-vec. |

---

## Vector Search Comparison

| Feature | sqlite-vec | libsql Native Vector |
|---|---|---|
| **Installation** | Loadable extension (`.dylib`/`.so`/`.dll`) | Built-in, zero setup |
| **Vector types** | float32, int8, bit | F1BIT, F8, FB16, F16, **F32**, F64 (6 types) |
| **Max dimensions** | ~2,000-4,000 practical | 65,536 |
| **Distance metrics** | L2, L1, cosine, hamming | Cosine only (currently) |
| **Index type** | Exact KNN (brute-force) | DiskANN (approximate NN) |
| **Scale** | Degrades linearly with dataset size | Full-scan OK to ~10K; DiskANN index for larger |
| **SIMD** | Yes (AVX/NEON) | Not documented |
| **Electron packaging** | Known `.dylib.dylib` bug on macOS | No extension packaging needed |

**For our use case** (Qwen3-Embedding-0.6B at 512-1024 dimensions, cosine similarity, thousands to tens of thousands of embeddings per user): both are sufficient. libsql eliminates the packaging friction.

---

## Mastra Integration Detail

With libsql, all four memory layers from the architecture doc map directly:

```
┌─────────────────────────────────────────────────────────┐
│                    Single .db File                        │
│                                                           │
│  LibSQLStore (structured data)                           │
│  ├── Conversation History (recent messages)              │
│  ├── Working Memory (user profile, preferences)          │
│  ├── Observational Memory (compressed summaries)         │
│  └── Agent state (workflow snapshots, traces)            │
│                                                           │
│  LibSQLVector (embeddings)                               │
│  ├── Semantic Recall (embedded past conversations)       │
│  ├── Embedded activity data (window titles, URLs)        │
│  └── Embedded observational summaries                    │
│                                                           │
│  App Tables (capture data)                               │
│  ├── Activities (window sessions)                        │
│  ├── Input events (keystroke patterns)                   │
│  └── Context streams                                     │
└─────────────────────────────────────────────────────────┘
```

With better-sqlite3, Mastra agents need a custom bridge to access app tables. With libsql, they share the database natively.

---

## Three Possible Paths

### Path 1: Stay with better-sqlite3 + sqlite-vec

- **Pros:** Proven, synchronous, zero Electron risk.
- **Cons:** Need custom Mastra adapter. Two native modules to package. sqlite-vec macOS packaging friction.
- **Effort:** Custom Mastra adapter (~2-3 days), sqlite-vec packaging workaround (~1 day).

### Path 2: Full libsql (via @mastra/libsql)

- **Pros:** One dependency, native Mastra integration, built-in vectors, no extension packaging.
- **Cons:** Async API for everything (capture pipeline refactor). Unproven in Electron. Pre-release sync package.
- **Effort:** Async refactor of data layer (~3-5 days), Electron integration testing (~2-3 days), risk of unknown unknowns.

### Path 3: Hybrid — `libsql` (sync) for capture + `@mastra/libsql` for agents

- **Pros:** Synchronous writes for capture (like better-sqlite3). Native Mastra integration for agents. Built-in vectors. One `.db` file.
- **Cons:** Two client libraries accessing one database (SQLite WAL handles this, but needs testing). `libsql` npm package is pre-release. Electron integration still unproven for both.
- **Effort:** Similar to Path 2, plus WAL concurrency testing.

---

## Open Questions

1. **Has anyone shipped libsql in a production Electron app?** No documented cases found. We would be early adopters.
2. **Can the `libsql` sync package and `@libsql/client` safely share a WAL-mode database?** SQLite supports multiple readers + one writer in WAL mode. This should work but needs verification.
3. **What is the actual Electron ABI rebuild story for `@libsql/client`?** Platform packages (`@libsql/darwin-arm64`) exist but are not Electron-specific. Rebuild behavior is undocumented.
4. **How complex is a custom Mastra storage adapter for better-sqlite3?** If simple, Path 1 becomes more attractive.

---

## Recommendation

**Start with Path 1 (better-sqlite3 + sqlite-vec) for v0.1. Evaluate libsql migration for v0.2.**

Reasoning:

1. **v0.1 is Mac-only, shipping fast matters.** better-sqlite3 has zero Electron unknowns. libsql in Electron is uncharted territory that could consume engineering time during the critical Phase 1 build.
2. **The Mastra adapter is bounded work.** Mastra documents custom storage adapters as an extension point. Building one for better-sqlite3 is ~2-3 days, not weeks.
3. **sqlite-vec packaging friction is solvable.** The `.dylib` issue has documented workarounds (`asarUnpack` config). Annoying, not blocking.
4. **libsql matures fast.** Turso is a funded company. By v0.2, the `libsql` sync package may be out of pre-release, Electron compatibility may be documented, and we'll have real usage data to inform the migration decision.
5. **Migration from better-sqlite3 to libsql is not a rewrite.** The `libsql` sync package is designed as a drop-in replacement. If we decide to switch later, the data layer migration is mechanical.

**If Mastra integration proves to be a significant engineering bottleneck during Phase 3 (MCP + Agent Runtime), re-evaluate and migrate to libsql at that point.** The architectural decision to use a single `.db` file for everything holds regardless of which SQLite library backs it.

---

## Changelog

**v1 (Feb 16, 2026):** Initial research. Comprehensive comparison of better-sqlite3 + sqlite-vec vs libsql for Electron desktop app with Mastra.ai agent integration. Recommendation: stay with better-sqlite3 for v0.1, evaluate libsql for v0.2.
