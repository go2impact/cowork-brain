# Jan → Cowork.ai Adaptation Guide

**Purpose:** Maps the key patterns from the [Jan deep-dive](../strategy/jan-deep-dive/codex-output.md) to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our architecture.

**Audience:** Engineering (Rustan + team)

**Source material:** `strategy/jan-deep-dive/` (reverse-engineering of Jan's codebase)
**Target architecture:** `architecture/system-architecture.md`

**Why Jan?** Jan is the most mature local-LLM desktop app in our competitive set. It chose Tauri+Rust (we chose Electron — see `decisions/DESKTOP_FRAMEWORK_DECISION.md`), but its production patterns for model lifecycle, download integrity, MCP process management, and hardware preflight checks are framework-agnostic. These patterns fill gaps that aime-chat, Cherry Studio, and Chatbox don't cover.

**Key difference from other guides:** Jan runs its native logic in Rust via Tauri commands. We run ours in Node.js via Electron utility processes. The patterns translate cleanly — the algorithms, data structures, and lifecycle flows are the same regardless of language. We note where Rust-specific idioms need TypeScript equivalents.

---

## Sections

| # | Section | Status |
|---|---|---|
| 0 | [Model Download Manager](#0-model-download-manager) | Done |
| 1 | [MCP Lockfile & Orphan Cleanup](#1-mcp-lockfile--orphan-cleanup) | Done |
| 2 | [GGUF Preflight Fit Checks](#2-gguf-preflight-fit-checks) | Done |
| 3 | [Local API Gateway](#3-local-api-gateway) | Done |
| 4 | [Summary](#4-summary) | Done |

---

## 0. Model Download Manager

**Source:** Jan's `src-tauri/src/core/downloads/helpers.rs` (main pipeline) + `commands.rs` (cancellation) + `models.rs` (progress tracking)

Cowork.ai needs to download Ollama models (DeepSeek-R1-Distill-Qwen-8B, Qwen3-Embedding-0.6B) and potentially GGUF files for direct llama.cpp usage. Jan has the most robust download pipeline in our competitive set: parallel, resumable, cancellable, path-sandboxed, and hash-verified. These are table-stakes for enterprise trust.

### Copy

#### Resumable downloads via `.tmp` + `.url` sidecar

Jan's resume strategy uses two sidecar files per download:

```
model.gguf.tmp  — partial download payload (append-mode writes)
model.gguf.url  — stores the URL this partial belongs to
```

**Resume decision logic:**

```
Resume only if ALL three conditions are true:
  1. .tmp file exists (partial payload)
  2. .url file exists (URL marker)
  3. URL in .url matches current download URL (same file, not a mirror swap)

On resume:
  → Read .tmp file size (= bytes already downloaded)
  → Send HTTP Range header: bytes={downloaded_size}-
  → Open .tmp in append mode, continue writing

On completion:
  → rename .tmp → final filename (atomic on most filesystems)
  → delete .url sidecar
```

**Why copy:** Without resume, a user downloading a 4GB model on flaky WiFi loses all progress on disconnect. The `.url` sidecar is clever — it prevents resuming a stale partial from a different mirror URL, which would produce a corrupt file.

**Caveat:** Jan's main `download_files` command currently calls the internal download with `resume=false`, meaning the resume infrastructure exists but isn't active in the normal command path. The pattern itself is sound — we'd enable it from the start.

**Our adaptation:** Ollama handles its own downloads for `ollama pull`, but if we support direct GGUF import (for users who want specific quantizations), this pattern applies directly. Implement in the Main process (downloads are coordinated centrally, not per-utility-process).

#### Path safety — sandbox enforcement

```
For each download item:
  1. save_path = jan_data_folder.join(item.save_path)
  2. save_path = normalize_path(save_path)  // resolve .., . components (no symlink resolution)
  3. if !save_path.starts_with(jan_data_folder):
       → return Error("Path is outside of Jan data folder")
       → file never created
```

**Why copy:** A malicious MCP server or crafted model manifest could specify `../../.ssh/authorized_keys` as a download path. Path normalization + prefix check is the defense. This is a hard security requirement. Note: Jan's `normalize_path` only resolves `.` and `..` components — it does not resolve symlinks. Our implementation should use `path.resolve()` for component normalization and consider `fs.realpath()` for symlink resolution as an additional hardening step.

**Our mapping:** In Cowork.ai, all model downloads go to the Ollama data directory or a Cowork-managed `models/` subdirectory. The same normalize-then-check pattern applies:

```typescript
const normalizedPath = path.resolve(savePath);
if (!normalizedPath.startsWith(COWORK_DATA_DIR)) {
  throw new Error(`Path ${normalizedPath} escapes data directory`);
}
```

#### SHA256 validation — two-phase (size first, hash second)

```
Phase 1: Size check (O(1), instant)
  → fs.stat(file).size vs expected size
  → mismatch → fail early, skip expensive hash

Phase 2: Hash check (O(n), slow for large files)
  → compute SHA256 of entire file (supports cancellation token)
  → mismatch → fail

Emit "onModelValidationStarted" event before starting
```

**Why copy:** Size check is a fast pre-filter that catches truncated downloads. Hash check catches bit-rot and mirror corruption. Running size first avoids hashing a 4GB file only to discover it's the wrong size.

**Our mapping:** Same two-phase pattern. In TypeScript (Node.js crypto):

```typescript
// Phase 1: size check
const stat = await fs.stat(filePath);
if (expectedSize && stat.size !== expectedSize) throw new Error('Size mismatch');

// Phase 2: hash check (with abort signal)
const hash = crypto.createHash('sha256');
const stream = fs.createReadStream(filePath, { signal: abortController.signal });
for await (const chunk of stream) hash.update(chunk);
if (hash.digest('hex') !== expectedSha256) throw new Error('Hash mismatch');
```

#### Per-task cancellation with `CancellationToken`

```
On download start:
  → create CancellationToken for task_id
  → if existing token for task_id → cancel it first (replace)
  → store in download_manager.cancel_tokens map

On cancel:
  → look up token by task_id → call .cancel()
  → all file-download coroutines check token.is_cancelled() per chunk
  → if cancelled + not resumable → delete partial file
  → if cancelled + resumable → keep .tmp for later resume

Cleanup:
  → on download complete → remove token from map
```

**Why copy:** Without cancellation, a user can't stop a 4GB download that's going to the wrong model. The replace-existing-token pattern prevents duplicate download tasks from accumulating.

**Our mapping:** In Node.js, use `AbortController`:

```typescript
const controller = new AbortController();
downloadTokens.set(taskId, controller);
// In download loop:
if (controller.signal.aborted) throw new Error('Download cancelled');
```

#### Parallel downloads with aggregated progress

```
ProgressTracker:
  file_progress: Map<fileId, bytesTransferred>  (shared via Arc<Mutex>)
  total_size: sum of all file sizes

Per file:
  → emit progress every 10 MB (throttled to avoid UI flooding)
  → update file_progress[fileId] = current bytes
  → compute combined: sum(file_progress.values()) / total_size

All files spawned as concurrent tasks, joined at end
```

**Why copy:** Model downloads often include multiple files (model weights + mmproj for vision models). Parallel download with a single progress bar gives the user accurate total progress.

**Our mapping:** Same pattern with `Promise.all()` for concurrent downloads and a shared progress state object.

### Adapt

#### Download coordinator location

Jan runs downloads in Rust (Tauri command). In Cowork.ai:

| Component | Jan | Cowork.ai |
|---|---|---|
| Download coordinator | Rust Tauri command | Main process (Node.js) |
| Progress events | Tauri `app.emit()` | IPC to Renderer via `BrowserWindow.webContents.send()` |
| Cancel tokens | `CancellationToken` (tokio) | `AbortController` (Node.js) |
| File I/O | `tokio::fs` async | `fs/promises` + streams |
| HTTP client | `reqwest` | `undici` or `node:http` |

The architecture is the same; the implementation language differs.

#### Mirror fallback strategy

Jan uses a custom mirror (`apps.jan.ai`) with HMAC-signed requests and falls back to the original HuggingFace URL:

```
1. Try mirror URL first (faster CDN)
2. If mirror fails → try original URL
3. Mirror requests include HMAC-signed headers for auth
```

**Our adaptation:** We don't need a custom mirror. Ollama has its own registry. For direct GGUF downloads, HuggingFace is the primary source with no mirror needed initially. Skip the HMAC mirror infrastructure.

### Skip

#### HuggingFace repo parsing and quant catalog

Jan's frontend parses HF repo metadata to list available quantizations (Q4_K_M, Q8_0, etc.) and pairs model files with mmproj vision adapters. This is specific to their "model catalog" UX.

**Why skip:** Cowork.ai uses Ollama for model management. Ollama handles its own model registry, quantization selection, and download pipeline. We only need the download integrity patterns (resume, validation, cancellation) for edge cases like direct GGUF import.

#### Backend binary management

Jan manages separate llama.cpp backend binaries (versioned, per-platform, with optional CUDA bundles). This adds significant operational complexity.

**Why skip:** We use Ollama, which bundles its own llama.cpp runtime. No separate binary management needed.

---

## 1. MCP Lockfile & Orphan Cleanup

**Source:** Jan's `src-tauri/src/core/mcp/lockfile.rs` + `helpers.rs:547-559` (create) + `:906-913` (cleanup)

Cowork.ai's MCP Integrations feature spawns stdio child processes for MCP servers. If the app crashes, those child processes become orphans — consuming ports, memory, and potentially holding locks on resources. Jan's lockfile pattern is the most thorough orphan management in our competitive set.

### Copy

#### Lockfile data structure

Jan stores one lockfile per MCP server port:

```
File: {app_data_dir}/mcp_lock_{port}.json

{
  "pid": 12345,            // NOTE: this is the app (Jan) PID, not the child process PID
  "port": 17389,
  "server_name": "Jan Browser MCP",
  "created_at": "2024-01-15T10:30:45Z",
  "hostname": "MacBook-Pro.local"
}
```

**Important caveat:** Jan currently only creates lockfiles for the built-in "Jan Browser MCP" server, not for all MCP servers. Additionally, the `pid` field stores the Jan app's own process ID (`std::process::id()`), not the child MCP server's PID. The lockfile is used to detect whether the Jan app instance that spawned the server is still running — if not, the server is considered orphaned.

**Why copy the pattern (with fixes):** The lockfile is the only reliable way to track child processes across app restarts. For our implementation, we should store the **child process PID** (not the app PID) to enable direct kill-on-cleanup, and create lockfiles for **all** stdio MCP servers, not just one.

**Our mapping:** Store lockfiles in `{cowork_data_dir}/mcp_lock_{port}.json`. Modify the schema to store the child PID:

#### Stale process detection

```
is_process_alive(pid):
  macOS/Linux: kill(pid, signal=0)  → returns Ok if process exists (no signal sent)
  Windows: tasklist /FI "PID eq {pid}" → check if PID in output
```

**Why copy:** Signal 0 is the standard Unix way to check process existence without affecting it. The Windows fallback via `tasklist` is necessary because Windows doesn't have `kill -0`.

**Our mapping (Node.js):**

```typescript
function isProcessAlive(pid: number): boolean {
  try {
    process.kill(pid, 0); // signal 0 = existence check
    return true;
  } catch {
    return false; // ESRCH = no such process
  }
}
```

#### Startup cleanup sweep

```
cleanup_all_stale_locks():
  1. Glob all mcp_lock_*.json in app data dir
  2. For each:
     a. Parse port from filename
     b. Read lockfile → get pid
     c. if !is_process_alive(pid) → delete lockfile, log cleanup
     d. if process alive → leave lockfile, log as active

Note: Jan's cleanup_all_stale_locks only removes stale lockfiles — it does NOT kill
processes. Process killing is handled in a separate flow (find_process_using_port → kill).
```

**Why copy:** On app startup after a crash, orphan MCP servers may still be running. The lockfile sweep identifies stale entries (their app PID no longer running), and a separate flow kills orphan processes by port. Without this, restarting the app fails because ports are still occupied.

**Our mapping:** Call `cleanupStaleLocks()` in the Main process during app initialization, before booting MCP servers. Unlike Jan (which separates lockfile cleanup from process killing), we should combine both in one sweep: detect stale lockfiles → kill the child process by PID → delete the lockfile.

#### Graceful shutdown with SIGTERM → SIGKILL escalation

```
kill_process_by_pid(pid):
  1. Send SIGTERM (graceful shutdown request)
  2. Poll every 100ms for up to 3 seconds:
     if !is_process_alive(pid) → done
  3. If still alive after 3s → send SIGKILL (force kill)
```

**Why copy:** SIGTERM gives the MCP server time to flush state and close connections. The 3-second grace period is generous enough for cleanup but short enough to not block app shutdown.

**Our mapping (Node.js):**

```typescript
async function killProcess(pid: number): Promise<void> {
  process.kill(pid, 'SIGTERM');
  for (let i = 0; i < 30; i++) {
    await sleep(100);
    if (!isProcessAlive(pid)) return;
  }
  process.kill(pid, 'SIGKILL');
}
```

### Adapt

#### Lockfile location and process ownership

Jan creates lockfiles from the Rust Tauri backend. In Cowork.ai, MCP servers are managed by the Agents & RAG Utility Process (system-architecture.md §1.3):

| Lifecycle event | Jan | Cowork.ai |
|---|---|---|
| Create lockfile | Rust backend on `start_mcp_server` | Agents Utility on MCP server spawn |
| Read lockfile | Rust backend on startup | Main process during init (before Agents Utility starts) |
| Delete lockfile | Rust backend on `stop_mcp_server` | Agents Utility on graceful stop |
| Orphan cleanup | Rust backend on app startup | Main process on app startup |

The key difference: our Main process does the startup sweep (it starts first), then the Agents Utility manages lockfiles during normal operation. This means lockfile operations need to work from both processes — use the filesystem as the coordination mechanism (which is exactly what lockfiles are for).

#### Orphan detection beyond lockfiles

Jan has a fallback: if no lockfile exists but a port is still occupied, use `lsof`/`netstat` to find the process and verify it's an MCP server:

```
Fallback: find_process_using_port(port) → get PID
  → get_process_info_by_pid(pid)
  → is_orphaned_mcp_process(info)  // check command name
```

**Our adaptation:** Good defense-in-depth. For v0.1, lockfile-only is sufficient. Add `lsof` fallback if we see orphan reports from users.

### Skip

#### Jan Browser MCP special-casing

Jan currently only creates/manages lockfiles for "Jan Browser MCP" — a built-in MCP server. Other MCP servers don't get lockfile treatment. This appears to be incremental implementation rather than a deliberate design choice.

**Why skip the limitation:** In Cowork.ai, ALL stdio MCP servers get lockfile treatment. No special-casing by server name.

#### Windows `tasklist` fallback

Jan implements Windows orphan detection via `tasklist /FI` command parsing. Cowork.ai v0.1 is Mac-only (system-architecture.md §1).

**Why skip for now:** Mac-only launch. When we add Windows support, port this pattern.

---

## 2. GGUF Preflight Fit Checks

**Source:** Jan's `src-tauri/plugins/tauri-plugin-llamacpp/src/gguf/commands.rs:51-145` + `utils.rs`

Cowork.ai's local brain requires DeepSeek-R1-Distill-Qwen-8B (4-bit, ~5GB) to run on Apple Silicon with 16GB+ RAM. Before loading the model, we need to check: will this model fit? Jan's RED/YELLOW/GREEN preflight check answers this question using GGUF metadata — no trial-and-error loading required.

### Copy

#### RED/YELLOW/GREEN classification formula

```
Inputs:
  model_size = file size on disk (weights)
  kv_cache_size = estimated KV cache memory (from GGUF metadata)
  total_required = model_size + kv_cache_size

  RESERVE_BYTES = 2.13 GB  (reserved for OS + other processes)

  For unified memory (Apple Silicon):
    usable_memory = total_RAM - RESERVE_BYTES

  For discrete GPU:
    usable_vram = total_VRAM - RESERVE_BYTES
    usable_total = (total_RAM - RESERVE_BYTES) + usable_vram

Classification:
  RED    = total_required > usable_total_memory     (impossible)
  GREEN  = total_required ≤ usable_vram             (fully in GPU/unified memory)
  YELLOW = fits in combined memory but not in VRAM alone (hybrid, slower)
```

**Why copy:** This is exactly the check we need for our hardware gate. On 16GB Apple Silicon, a 5GB model with 2GB KV cache = 7GB required. After 2.13GB reserve, 13.87GB usable. GREEN. On 8GB machines, 5.87GB usable — might be YELLOW depending on KV cache size, which determines whether they get local brain or cloud-only.

**Our mapping:**

| Jan | Cowork.ai |
|---|---|
| RED → "Can't run" | Disable local brain, show cloud-only mode |
| YELLOW → "Slow hybrid" | Show warning: "Local brain may be slow, consider cloud mode" |
| GREEN → "Full speed" | Enable local brain, no warnings |

#### KV cache estimation from GGUF metadata

```
kv_per_token = n_layers × n_kv_heads × (key_dim + value_dim) × 2 bytes

If sliding_window present:
  kv_cache = (context_length × kv_per_token + sliding_window × kv_per_token) / 2
Else:
  kv_cache = context_length × kv_per_token
```

**Required GGUF metadata keys:**
- `general.architecture` — model type
- `{arch}.block_count` — layer count
- `{arch}.attention.head_count_kv` — KV head count (or fallback to `head_count`)
- `{arch}.attention.key_length` / `value_length` — head dimensions (or compute from `embedding_length / head_count`)
- `{arch}.context_length` — max context window
- `{arch}.attention.sliding_window` — optional, for efficient attention

**Why copy:** KV cache can be 2-4GB for large context windows. Ignoring it makes the fit check inaccurate — a model that appears to fit by weight size alone may OOM when context fills up.

#### GGUF metadata reading via HTTP Range requests

For remote files (pre-download check):

```
1. Fetch 2MB chunks using HTTP Range: bytes={start}-{end}
2. After each chunk, attempt GGUF metadata parsing
3. If parsing succeeds → return metadata (usually within first 2-10MB)
4. If parsing fails → fetch next 2MB chunk
5. Cap at 120MB total (safety limit)
```

**Why copy:** This lets us check model compatibility BEFORE downloading the full 4-8GB file. The progressive chunking is efficient — GGUF metadata is near the start of the file, so 2-4MB usually suffices.

### Adapt

#### Ollama vs direct GGUF

Jan loads GGUF files directly via llama.cpp. Cowork.ai uses Ollama, which has its own model format (Modelfile + blobs). The fit check needs to work with both:

| Scenario | How to get metadata |
|---|---|
| Ollama model (primary) | `ollama show --modelfile` → parse parameters + model info |
| Direct GGUF import (future) | Read GGUF metadata directly (copy Jan's parser) |

For Ollama, the preflight check is simpler: Ollama exposes model parameter count and quantization in its API. We can estimate memory usage from those without parsing GGUF directly.

#### Reserve bytes for Cowork.ai's multi-process architecture

Jan reserves 2.13GB for OS. Cowork.ai runs more processes:

| Process | Estimated memory |
|---|---|
| Electron Main | ~100MB |
| Renderer | ~200MB |
| Capture Utility | ~150MB |
| Agents Utility (Mastra + Ollama client) | ~300MB |
| Playwright Child (when active) | ~200MB |

**Our reserve:** ~3GB total (vs Jan's 2.13GB). This accounts for our heavier multi-process footprint. On 16GB unified memory: 13GB usable for model + KV cache. On 8GB: 5GB usable — likely RED for our 8B model, confirming our "8GB gets cloud-only" design decision.

#### Context-length parameter

Jan checks with the model's full context length by default. Cowork.ai should check with our ACTUAL context budget (set by the complexity router), not the model's max:

```
// Jan: uses model's max context (e.g., 32K)
ctx_len = metadata.context_length

// Cowork.ai: uses our configured context budget (e.g., 4K for local brain)
ctx_len = min(COWORK_LOCAL_CONTEXT_BUDGET, metadata.context_length)
```

This makes our KV cache estimate smaller and more accurate, potentially upgrading YELLOW → GREEN for many configurations.

### Skip

#### Discrete GPU detection and multi-GPU aggregation

Jan aggregates VRAM across multiple discrete GPUs. Cowork.ai v0.1 is Mac-only (Apple Silicon = unified memory, no discrete GPUs).

**Why skip for now:** Apple Silicon has one unified memory pool. The GPU detection logic is dead code for us. Add it when we support Windows/Linux with discrete GPUs.

#### llama.cpp argument builder

Jan has a detailed `ArgumentBuilder` that maps config to llama.cpp CLI flags (GPU offload, batch size, flash attention, RoPE scaling, etc.). We use Ollama, which manages its own llama.cpp process.

**Why skip:** Ollama abstracts llama.cpp CLI flags behind its own API. We don't spawn llama.cpp directly.

---

## 3. Local API Gateway

**Source:** Jan's `src-tauri/src/core/server/proxy.rs` + `commands.rs`

Cowork.ai's complexity router (system-architecture.md §3) routes requests between local and cloud models. Jan's local API gateway is the most complete implementation of "one endpoint, many backends" in our competitive set — it multiplexes local llama.cpp sessions and remote cloud providers behind a single OpenAI-compatible API.

### Study

#### Single-endpoint model routing

```
POST /v1/chat/completions
  → parse body.model
  → if model matches active local session → forward to http://localhost:{session_port}/v1/...
  → if model matches remote provider → forward to provider base_url with auth
  → stream/return response
```

**What to learn:** The routing logic is simple: model ID → backend lookup → proxy forward. This is conceptually similar to our complexity router, but Jan routes by explicit model name while we route by complexity score.

**Our application:** We don't need a localhost proxy (our renderer talks to the Agents Utility via IPC, not HTTP). But the pattern of "one interface, backend-agnostic routing" validates our complexity router design.

#### Trusted host + dual auth

```
Security:
  1. Host allowlist: requests from trusted hosts pass (blocks port-scanning attacks)
  2. Auth accepts both: Authorization: Bearer {key} OR X-Api-Key: {key}
  3. CORS preflight with allowed methods/headers
  4. Docs paths (/, /openapi.json) bypass both auth AND host checks
```

**What to learn:** If we ever expose a local API (for MCP servers to call back, or for external tools), these security patterns are the baseline. The dual-auth pattern (Bearer + X-Api-Key) is pragmatic for compatibility with different clients. **Caution:** Jan's docs paths bypass both auth and host checks entirely — this is a design gap we should not replicate. Our local endpoints should always validate the host, even for documentation routes.

### Skip

#### Anthropic ↔ OpenAI format translation

Jan translates between Anthropic's `/messages` format and OpenAI's `/chat/completions` format. This adds significant complexity for backward compatibility.

**Why skip:** We use Vercel AI SDK v6, which handles provider format differences internally. No manual translation layer needed.

#### Swagger/OpenAPI docs endpoint

Jan serves live API docs at `GET /` with runtime host/port substitution. Nice for developer experience, but not needed for a desktop sidecar that doesn't expose a public API.

**Why skip:** Our API surface is IPC-based, not HTTP-based. Documentation lives in our architecture docs, not a live endpoint.

---

## 4. Summary

### Copy directly

| Pattern | Source | Cowork.ai target | Rationale |
|---|---|---|---|
| Resumable downloads (`.tmp` + `.url` sidecar) | `downloads/helpers.rs:556-706` | Main process — model downloader | Prevents losing progress on large model downloads |
| Path safety (normalize + prefix check) | `downloads/helpers.rs:408-424` | Main process — any file write | Hard security requirement for download paths |
| SHA256 validation (size-first, hash-second) | `downloads/helpers.rs:89-208` | Main process — post-download verification | Catches truncation and corruption efficiently |
| Per-task cancellation tokens | `downloads/commands.rs:18-69` | Main process — download manager | Users must be able to cancel downloads |
| Parallel downloads + aggregated progress | `downloads/helpers.rs:405-445` | Main process — download UI events | Accurate total progress for multi-file downloads |
| MCP lockfile data structure | `mcp/lockfile.rs:6-13` | Main + Agents Utility — MCP manager | Track child processes across app restarts |
| Stale process detection (signal 0) | `mcp/lockfile.rs:77-114` | Main process — startup sweep | Detect orphan MCP servers from crashed sessions |
| Startup lockfile sweep | `mcp/lockfile.rs:146-176` | Main process — app initialization | Clean up orphans before booting MCP servers |
| SIGTERM → SIGKILL escalation | `mcp/helpers.rs:740-761` | Agents Utility — server stop | Graceful shutdown with force-kill fallback |
| RED/YELLOW/GREEN fit classification | `gguf/commands.rs:51-145` | Main process — hardware gate | Prevent OOM by checking before model load |
| KV cache estimation formula | `gguf/utils.rs:59-192` | Main process — fit calculator | Accurate memory estimate includes KV cache |

### Study (learn from, adapt approach)

| Pattern | What to learn | How it changes for us |
|---|---|---|
| Single-endpoint model routing | One API multiplexes local + cloud | Validates complexity router design; our version uses IPC not HTTP |
| Trusted host + dual auth | Security baseline for local API | Apply if we ever expose a local HTTP endpoint |
| GGUF HTTP Range metadata reading | Check model compatibility before full download | Adapt for Ollama API metadata queries |
| Session-based per-model process registry (PID+port+key) | Clean model lifecycle tracking | Maps to Ollama process management |
| Service-hub `window.core.api` abstraction | Clean platform boundary for IPC | Similar to our typed IPC contract |
| Versioned migration bootstrap (`store.json` + startup hooks) | Migration strategy for desktop app upgrades | Apply to libsql schema migrations |

### Skip

| Pattern | Why skip |
|---|---|
| Tauri+Rust architecture | We chose Electron (see DESKTOP_FRAMEWORK_DECISION.md) |
| llama.cpp direct process management | We use Ollama, which manages its own llama.cpp |
| ArgumentBuilder for llama.cpp CLI flags | Ollama abstracts this |
| HuggingFace repo parsing + quant catalog | Ollama has its own model registry |
| Backend binary management (versioned llama.cpp) | Ollama bundles its runtime |
| Mirror infrastructure with HMAC signing | No custom CDN needed; Ollama handles downloads |
| Anthropic ↔ OpenAI format translation | Vercel AI SDK handles provider differences |
| Discrete GPU detection + multi-GPU aggregation | Mac-only v0.1 (unified memory) |
| Windows process detection (`tasklist`) | Mac-only v0.1 |
| Cortex migration scaffolding | Legacy from Jan's prior architecture |
| JSON-file thread persistence (desktop) | We use libsql for everything |
| Frontend localStorage for critical config | We store config in libsql |

---

## Changelog

**v1 (Feb 17, 2026):** Initial draft. All 4 sections complete.

**v1.1 (Feb 17, 2026):** Fact-check corrections: download resume is implemented but called with `resume=false` in normal command path, `normalize_path` doesn't resolve symlinks, lockfile stores app PID not child PID, lockfiles only for Jan Browser MCP not all servers, lockfile cleanup only removes files (doesn't kill processes), docs paths bypass both auth AND host checks.
