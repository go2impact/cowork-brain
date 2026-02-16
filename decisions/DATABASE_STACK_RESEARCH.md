# Research: Database Stack — better-sqlite3 + sqlite-vec vs libsql

| | |
|---|---|
| **Status** | Research complete (v2) — **recommends libsql for everything** |
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
| **Electron maturity** | **Proven for the sync package.** Beekeeper Studio (v4.6+, June 2024) and Outerbase Studio Desktop both ship the sync `libsql` npm package in production Electron apps. Cross-architecture packaging (Apple Silicon + Intel) is the main pain point. However, `@libsql/client` (async, what Mastra uses) has no documented Electron production use. |
| **Pre-release status** | The `libsql` npm package (sync API) is v0.6.0-pre.29 — still pre-release. `@libsql/client` is stable. |
| **Vector search limitations** | Currently only supports cosine distance. sqlite-vec supports L2, L1, cosine, and hamming. (Cosine is sufficient for our use case — embedding similarity.) |
| **No head-to-head benchmarks** | No published local-only performance comparison vs better-sqlite3. |
| **New territory** | Zero existing references to libsql in the cowork-brain repo. All architecture docs assume better-sqlite3 + sqlite-vec. |
| **Mastra in Electron** | **Unproven but feasible.** Mastra's official Electron guide uses a client-server pattern (HTTP to localhost:4111), not embedded. No one has run `@mastra/libsql` inside Electron's main or utility process. However, the native binary is the same one proven in Electron (Beekeeper Studio), so the risk is in the async wrapper + Mastra layer, not the native code. |
| **Node version** | `@mastra/libsql` requires Node >= 22.13.0. ~~This means Electron 36+ (currently unreleased).~~ **Corrected:** The existing codebase runs Electron 37.1.0 which ships Node 22.16.0 — this constraint is already satisfied. Embedded `@mastra/libsql` in a utility process is possible. |

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

### Custom Adapter Complexity (Path 1)

A custom Mastra adapter for better-sqlite3 is more complex than initially estimated:

- **Storage adapter:** Mastra's `MastraStorage` interface covers 10+ domains (threads, messages, traces, workflow snapshots, evals). Significant surface area.
- **Vector adapter:** `MastraVector` has 9 abstract methods. LibSQLVector internally uses libSQL-specific SQL functions (`F32_BLOB`, `vector_distance_cos`) that don't exist in standard SQLite. A better-sqlite3 vector adapter must use sqlite-vec's completely different API (`vec_f32()`, `vec_distance_L2()`, virtual tables). This is a full rewrite, not a port — estimated ~500-800 lines.
- **Total effort:** Likely 4-6 days, not 2-3 days as initially estimated.

### Mastra in Electron — What We Know (Fact-Checked)

Mastra.ai has an [official Electron guide](https://mastra.ai/docs/frameworks/electron), but it uses a **client-server architecture**: Mastra runs as a separate Node.js process (`localhost:4111`), and the Electron app calls it over HTTP. This is fundamentally different from embedding `@mastra/libsql` directly in Electron's main or utility process.

**No one has embedded `@mastra/libsql` inside an Electron process.** The `@libsql/client` package (which `@mastra/libsql` depends on) uses the same native binary as the sync `libsql` package for local file access, so the native code is proven in Electron — but the async wrapper layer and Mastra's usage of it are untested in that context.

**Node version constraint:** `@mastra/libsql` requires Node >= 22.13.0. ~~Current stable Electron (v35) ships Node 22.12.0.~~ **Corrected:** The existing codebase runs Electron 37.1.0, which ships Node 22.16.0. The Node version constraint is already satisfied — `@mastra/libsql` can run embedded in an Electron utility process. The sidecar pattern is a choice for isolation, not a requirement.

---

## The Decision: One Stack or Two?

### Unified: libsql for everything (one stack)

Use `libsql` as the single database SDK for both capture data and Mastra agent memory. One `.db` file, one SQL dialect, one set of vector functions.

**How it works:**
- Capture pipeline (utility process): sync `libsql` package for high-frequency writes
- Mastra agents (utility process): `@mastra/libsql` (`LibSQLStore` + `LibSQLVector`) embedded in Electron utility process
- Both access the same WAL-mode `.db` file
- Built-in vector search (`F32_BLOB` + `vector_distance_cos`) — no extension needed

**Pros:**
- One SDK, one SQL dialect, one native module
- Native Mastra integration — zero adapter code
- Built-in vector search eliminates sqlite-vec packaging friction
- Sync `libsql` is proven in Electron (Beekeeper Studio, Outerbase Studio Desktop)
- Single `.db` file for everything — agents can directly query capture data
- Electron 37.1.0 ships Node 22.16.0 — satisfies `@mastra/libsql` requirement, no sidecar forced

**Cons:**
- Sync `libsql` npm package is pre-release (v0.6.0-pre.29) — API could change
- No one has tested `@mastra/libsql` embedded in Electron (native binary is proven, async wrapper + Mastra layer is not)
- Cross-architecture Electron packaging is the main pain point (solvable — Beekeeper solved it)

**Effort:** libsql Electron packaging (~1-2 days), Mastra utility process integration (~2-3 days).

### Split: better-sqlite3 + sqlite-vec (two stacks)

Use `better-sqlite3` + `sqlite-vec` for capture data. Build a custom Mastra adapter to bridge agent memory.

**How it works:**
- Capture pipeline: `better-sqlite3` (sync) + `sqlite-vec` extension for vectors
- Mastra agents: custom `MastraStorage` + `MastraVector` adapter wrapping better-sqlite3 + sqlite-vec
- Same `.db` file, but different code paths for app data vs agent data

**Pros:**
- `better-sqlite3` is battle-tested in Electron (years of production use, pre-built binaries)
- Synchronous API everywhere — simple, deterministic
- Huge community (~1.5M weekly downloads, 3,400+ npm dependents)
- Stable releases every 1-2 weeks

**Cons:**
- Custom Mastra adapter is substantial (~500-800 lines for vector alone, 10+ domains for storage). Estimated 4-6 days.
- Two native modules to rebuild and package for Electron
- sqlite-vec has `.dylib.dylib` macOS packaging bug (needs `asarUnpack` workaround)
- Custom adapter is ongoing maintenance — must track Mastra interface changes
- Two SQL dialects: sqlite-vec uses `vec_f32()` + virtual tables; libsql uses `F32_BLOB` + `vector_distance_cos`. Cannot share vector queries between app code and agent code.

**Effort:** Custom Mastra adapter (~4-6 days), sqlite-vec packaging workaround (~1 day), ongoing adapter maintenance.

---

## Open Questions

1. ~~**Has anyone shipped libsql in a production Electron app?**~~ **Answered: YES.** Beekeeper Studio (v4.6+, June 2024) and Outerbase Studio Desktop both ship the sync `libsql` package in production. Cross-architecture packaging is the main challenge.
2. **Can the `libsql` sync package and `@libsql/client` safely share a WAL-mode database?** SQLite supports multiple readers + one writer in WAL mode. This should work but needs verification. (Both share the same native binary for local file access, which is encouraging.)
3. ~~**What is the actual Electron ABI rebuild story for `@libsql/client`?**~~ **Partially answered.** `@libsql/client` depends on platform packages (`@libsql/darwin-arm64`) that ship the same native binary as the sync `libsql` package. Since Beekeeper Studio ships the sync package successfully, the native binary works in Electron. The async wrapper layer is untested.
4. ~~**How complex is a custom Mastra storage adapter for better-sqlite3?**~~ **Answered: substantial.** Storage adapter covers 10+ domains. Vector adapter requires ~500-800 lines using sqlite-vec's API (completely different SQL from libSQL's built-in vectors). Estimated 4-6 days total.
5. ~~**Can `@mastra/libsql` run embedded in Electron, or must Mastra use client-server?**~~ **Answered: embedded is feasible.** Electron 37.1.0 ships Node 22.16.0, satisfying `@mastra/libsql`'s requirement. The native binary is proven in Electron. The async wrapper + Mastra layer is untested but the risk is in the JS layer, not native code. Recommend embedded utility process.
6. ~~**Is the client-server Mastra pattern acceptable for our architecture?**~~ **Superseded.** Sidecar is not required — embedded utility process is simpler and avoids HTTP hop.

---

## Recommendation

**Unified: libsql for everything.**

The original v1 recommendation (better-sqlite3 for v0.1, evaluate libsql for v0.2) was based on the belief that libsql was unproven in Electron. That was wrong. Beekeeper Studio and Outerbase Studio Desktop both ship it in production. With that objection removed, libsql wins on every dimension that matters:

1. **The Mastra adapter cost tips the scale.** The custom adapter for better-sqlite3 is 4-6 days of work (not 2-3 as initially estimated) plus ongoing maintenance tracking Mastra interface changes. libsql gets native Mastra integration for free. That's 4-6 days of engineering saved on day one, plus maintenance burden eliminated permanently.

2. **One stack eliminates an entire category of bugs.** Two SQL dialects (sqlite-vec's `vec_f32()` vs libsql's `F32_BLOB`), two native modules, two ABI rebuilds, two packaging configurations — every duplication is a surface for bugs. libsql collapses all of this into one.

3. **The pre-release risk is manageable.** The sync `libsql` package is pre-release (v0.6.0-pre.29), but the underlying native code is the same binary that Beekeeper Studio ships in production. Turso is a funded company with an active team. The API surface we need (prepare/run/get + vector functions) is stable in the underlying C library.

4. **Mastra runs in a utility process.** Electron 37.1.0 ships Node 22.16.0, which satisfies `@mastra/libsql`'s requirement (Node >= 22.13.0). Mastra can run embedded in an Electron utility process — no sidecar needed. This keeps the architecture simpler (Electron built-in IPC, no HTTP hop, no separate process lifecycle management) while maintaining crash isolation via the utility process boundary.

5. **No sqlite-vec packaging friction.** Built-in vector search means no `.dylib` extension, no `loadExtension()`, no `asarUnpack` workaround, no double-extension bug.

### What This Means for the Architecture

```
Electron Main Process (coordinator only)
├── Capture Utility Process
│   └── sync `libsql` package → writes to cowork.db
│
├── Agents & RAG Utility Process
│   └── @mastra/libsql → reads/writes same cowork.db
│       ├── LibSQLStore (agent memory, threads, traces)
│       └── LibSQLVector (embeddings, semantic recall)
│
├── Renderer (sandboxed)
│   └── reads via IPC
│
└── Playwright Child Process
    └── browser automation (fully isolated)
```

Both the capture utility and agents utility access the same `.db` file. WAL mode allows concurrent readers + one writer. The capture process writes frequently (sync). Mastra reads frequently and writes occasionally (async). This is the ideal WAL access pattern. Both utility processes are crash-isolated and auto-restartable via Electron's built-in IPC — no HTTP hop, no separate process lifecycle management.

---

## Changelog

**v3 (Feb 16, 2026):** Fixed three documentation inconsistencies: (1) Corrected stale Electron version constraint — existing codebase runs Electron 37.1.0 (Node 22.16.0), so `@mastra/libsql` can run embedded in a utility process, sidecar not required. (2) Renamed decision options from A/B to Unified/Split to avoid collision with the earlier Option A/B labels. (3) Updated architecture diagram to show Mastra in utility process, aligned with salvage plan's multi-process model.

**v2 (Feb 16, 2026):** Fact-checked v1 findings. Corrected "libsql unproven in Electron" — Beekeeper Studio and Outerbase ship it in production. Added Mastra-in-Electron research (client-server only, not embedded). Revised custom adapter estimate from 2-3 days to 4-6 days. Reframed as binary decision (one stack vs two). **Reversed recommendation: libsql for everything.**

**v1 (Feb 16, 2026):** Initial research. Comprehensive comparison of better-sqlite3 + sqlite-vec vs libsql for Electron desktop app with Mastra.ai agent integration. Recommendation: stay with better-sqlite3 for v0.1, evaluate libsql for v0.2.
