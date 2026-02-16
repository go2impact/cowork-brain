# Jan Codebase Deep Dive for Cowork.ai Competitive Analysis

## Scope and Method
This analysis is based on direct inspection of Jan source code (Rust + TypeScript) in `/Users/rustancorpuz/code/jan`, with emphasis on architecture and implementation patterns relevant to Cowork.ai (Electron + Mastra.ai + Vercel AI SDK v6 + libsql).

I focused on these areas:
1. Tauri Architecture
2. Local LLM Execution
3. OpenAI-Compatible Local API
4. Provider System
5. MCP Integration
6. Storage & Data
7. Model Download & Management
8. Copy / Study / Skip summary

I explicitly call out where conclusions are **inferred** vs directly implemented.

---

## 1) Tauri Architecture

### (1) How Jan built it (with file paths and code patterns)

#### A. Native shell is Tauri, with Rust as orchestration backend
- `src-tauri/src/main.rs:1` is minimal and calls `app_lib::run()`.
- `src-tauri/src/lib.rs:20` is the real composition root:
  - Registers Tauri plugins: `os`, `opener`, `http`, `store`, `shell`, `llamacpp`, `vector_db`, `rag`, and feature-gated `mlx`, `hardware`, `deep-link`.
  - Exposes a large Rust command surface via `invoke_handler` (filesystem, app config, system, server, providers, MCP, threads, downloads, updater).
  - Initializes shared `AppState` with mutexed maps for MCP servers, download cancel tokens, provider configs, and local API server handle (`src-tauri/src/core/state.rs:35`).

#### B. Frontend is a bundled web app loaded into Tauri webview
- `src-tauri/tauri.conf.json:7` points `frontendDist` to `../web-app/dist`.
- Dev mode uses web server (`devUrl: http://localhost:1420`) at `src-tauri/tauri.conf.json:8`.
- Security posture includes explicit CSP and asset protocol scoping (`src-tauri/tauri.conf.json:21`).

#### C. IPC pattern: typed TS API facade -> snake_case invoke command mapping
- Frontend bootstraps a global bridge in `web-app/src/providers/ExtensionProvider.tsx:11`:
  - `window.core = { api: APIs }`
  - plus event bus, extension manager, engine manager, model manager.
- Route mapping in `web-app/src/lib/service.ts:40`:
  - camelCase route names are transformed to Tauri snake_case command names.
  - special compatibility shim for `start_server` config payload at `web-app/src/lib/service.ts:49`.
- Tauri invoke wrapper in `web-app/src/services/core/tauri.ts:11`.
- Core package consumes `globalThis.core.api` abstraction (`core/src/browser/core.ts:8`, `core/src/browser/fs.ts:7`, `core/src/browser/events.ts:7`).

#### D. Platform service composition: desktop/mobile/web variants with one interface
- `web-app/src/services/index.ts:108` initializes platform services and lazy-loads Tauri implementations.
- Same high-level service interfaces, different adapters (`tauri`, `mobile`, or default web fallback).

#### E. Migration/versioning strategy uses Tauri store + extension bootstrap
- `src-tauri/src/lib.rs:244` loads `store.json` in data folder.
- Compares stored app version and current version, then:
  - reinstalls extensions on version changes (`setup::install_extensions`),
  - runs MCP config migrations (`setup::migrate_mcp_servers`).

### Why Tauri instead of Electron? (inference grounded in code)
Direct “why” text is not explicitly documented in code. But implementation strongly implies these drivers:
- **Binary size and OS integration priority**: aggressive release size optimizations in `src-tauri/Cargo.toml:142` (`opt-level=z`, fat LTO, strip symbols, panic abort).
- **Rust-native control of local AI infra**: process management, proxy server, download manager, and MCP lifecycle are implemented in Rust modules.
- **Cross-platform desktop + mobile reuse**: feature flags and platform-specific dependency sets in `src-tauri/Cargo.toml:16` and `package.json` mobile scripts.

### (2) ASCII diagrams

#### Architecture overview
```text
+------------------------------ Jan App -------------------------------+
|                                                                      |
|  Web UI (React/TS)                                                   |
|  web-app/*                                                           |
|    |                                                                 |
|    | window.core.api (API facade)                                    |
|    v                                                                 |
|  Tauri Invoke Bridge                                                 |
|    |                                                                 |
|    v                                                                 |
|  Rust Backend (src-tauri/src/lib.rs)                                 |
|    |- AppState (MCP, downloads, provider configs, server handle)     |
|    |- Commands (threads, MCP, server, downloads, config, etc.)       |
|    |- Plugins: llamacpp, mlx, rag, vector_db, store, shell, http     |
|                                                                      |
+----------------------------------------------------------------------+
```

#### IPC command flow
```text
UI action
  -> web-app service call (camelCase)
  -> web-app/src/lib/service.ts maps to snake_case
  -> @tauri-apps/api/core.invoke(command, args)
  -> Rust #[tauri::command] handler
  -> Result returned to TS caller
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- `window.core.api` + service-hub abstraction pattern for clean platform boundary.
- Rust-side centralized state (`AppState`) for cross-feature coordination.
- Versioned migration bootstrap (`store.json` + startup migration hooks).

**Study**
- Jan’s command sprawl in `invoke_handler`: powerful but can become hard to reason about at scale.
- Security model around legacy filesystem commands and path guards.

**Skip**
- Overly broad legacy FS command surface if Cowork can keep stricter capability boundaries from day one.

---

## 2) Local LLM Execution (Cortex lineage, llama.cpp integration, model management)

### (1) How Jan built it (with file paths and code patterns)

### A. Current engine is direct llama.cpp; Cortex is migration legacy
- Changelog explicitly states direct llama.cpp replaced Cortex (`docs/src/pages/changelog/2025-07-31-llamacpp-tutorials.mdx:24`, `:108-110`).
- Frontend still contains migration flags and path rewrites:
  - `cortex_models_migrated` guard in `extensions/llamacpp-extension/src/index.ts:900`.
  - legacy ID normalization replacing `cortex.so|huggingface.co` prefixes at `extensions/llamacpp-extension/src/index.ts:939`.

### B. llama.cpp runs as spawned backend process per loaded model
- Plugin wiring: `src-tauri/plugins/tauri-plugin-llamacpp/src/lib.rs:20`.
- Runtime state:
  - `LlamacppState` holds PID -> session map (`src-tauri/plugins/tauri-plugin-llamacpp/src/state.rs:25`).
  - `SessionInfo` contains pid, port, model_id, model_path, embedding mode, api key, mmproj path (`state.rs:8`).
- Load model flow:
  - `load_llama_model` in `.../commands.rs:42`.
  - Builds command args from `LlamacppConfig` through `ArgumentBuilder` (`args.rs:43`).
  - Spawns process, watches stdout/stderr for readiness markers (`commands.rs:147-153`, `:186-191`).
  - Timeout and early-exit error handling with structured mapping.
  - On success, inserts session keyed by PID (`commands.rs:270`).
- Unload model flow:
  - `unload_llama_model` gracefully terminates process (`commands.rs:283`), with platform-specific termination logic in `process.rs:52` and `process.rs:80`.

### C. Rich runtime configuration maps cleanly to llama.cpp CLI
- `LlamacppConfig` in `args.rs:4` includes GPU offload, ctx size, batching, flash-attn, RoPE, KV cache types, fit settings, etc.
- `ArgumentBuilder::build` composes flags deterministically (`args.rs:68`).

### D. GGUF introspection and fit heuristics integrated into runtime UX
- Read GGUF metadata from local file or HTTP Range chunks (`gguf/utils.rs:8`, `:10-20`).
- KV cache estimator from metadata (`gguf/utils.rs:60`).
- Model support status RED/YELLOW/GREEN based on model size + KV estimate vs memory (`gguf/commands.rs:50`, `:130-144`).

### E. Model manager extension wraps lifecycle + import + chat
- Main extension: `extensions/llamacpp-extension/src/index.ts`.
- Import pipeline (`index.ts:1119`):
  - accepts URL or local paths,
  - delegates downloads,
  - validates GGUF metadata,
  - writes `model.yml` manifest,
  - emits `onModelImported`.
- Load pipeline (`index.ts:1446`):
  - ensures backend binary exists/downloaded,
  - generates per-session API key,
  - calls Rust `load_llama_model`.
- Chat path (`index.ts:1739`):
  - finds active session,
  - health checks process,
  - calls local endpoint `http://localhost:{port}/v1/chat/completions` with bearer token.

### F. Backend binaries are managed separately from models
- `extensions/llamacpp-extension/src/backend.ts`:
  - fetches supported backends from GitHub with CDN fallback,
  - installs backend binaries under `llamacpp/backends/{version}/{backend}`,
  - optionally downloads CUDA runtime bundles.

### (2) ASCII diagrams

#### Local model load lifecycle
```text
Model selected in UI
  -> llamacpp extension (performLoad)
    -> ensureBackendReady(version/backend)
      -> download backend if missing
    -> read model.yml (model path, mmproj, settings)
    -> invoke plugin:llamacpp|load_llama_model
      -> Rust spawns llama-server process
      -> wait for readiness markers
      -> register SessionInfo(pid, port, api_key)
  -> chat uses http://localhost:{port}/v1/*
```

#### Import pipeline
```text
User import (URL/local)
  -> build download items (model.gguf, optional mmproj.gguf)
  -> download-extension -> Rust downloader
  -> GGUF metadata validation
  -> compute size + embedding flags
  -> write llamacpp/models/{modelId}/model.yml
  -> emit onModelImported
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- Session-based per-model process registry (PID+port+api_key).
- Arg-builder pattern for deterministic CLI generation.
- Readiness detection from logs + timeout safety.
- GGUF metadata-driven fit checks surfaced before run.

**Study**
- Dual-process orchestration complexity (backend binaries + model files).
- Auto-unload behavior in mixed workloads (`index.ts:1459`) to avoid user surprise.

**Skip**
- Keeping Cortex-era migration scaffolding too long; aggressively remove once migration window closes.

---

## 3) OpenAI-Compatible Local API (localhost:1337)

### (1) How Jan built it (with file paths and code patterns)

### A. Local API server start/stop is a Tauri command
- Start config struct in `src-tauri/src/core/server/commands.rs:8` includes `host`, `port`, `prefix`, `api_key`, `trusted_hosts`, `proxy_timeout`.
- `start_server` passes llama + MLX session maps into proxy runtime (`commands.rs:39`).

### B. Proxy server is Hyper + Reqwest with strict host/auth/cors checks
- Core implementation: `src-tauri/src/core/server/proxy.rs`.
- CORS preflight validation with allowed method/header lists (`proxy.rs:436`, `:462`, `:510`).
- Host allowlist enforced using `trusted_hosts` except explicit docs/static paths (`proxy.rs:604`, `:616`).
- Auth accepts either:
  - `Authorization: Bearer <api_key>`
  - or `X-Api-Key: <api_key>`
  (`proxy.rs:645-662`).

### C. Routing logic supports local and remote providers on one endpoint
- Model-bearing POST routes:
  - `/chat/completions`, `/completions`, `/embeddings`, `/messages/count_tokens` (`proxy.rs:831-834`)
- Anthropic-style `/messages` route with fallback to OpenAI chat completions (`proxy.rs:700`, `:1319`).
- `GET /models` merges three sets:
  - active llama sessions,
  - active MLX sessions,
  - remote provider model IDs (`proxy.rs:1028-1088`).

### D. Anthropic/OpenAI format interop is built in
- Request transforms: Anthropic messages/tools -> OpenAI structure (`proxy.rs:18`).
- Response transforms: OpenAI -> Anthropic `/messages` style (`proxy.rs:330`).

### E. Docs/OpenAPI are served directly from the local server
- `GET /openapi.json` serves static spec with runtime host/port substitution (`proxy.rs:1119-1137`).
- `GET /` serves Swagger UI shell (`proxy.rs:1161`).

### F. How other apps connect
- README advertises local OpenAI-compatible server at localhost:1337 (`README.md:89`).
- Desktop defaults are persisted in Zustand:
  - host `127.0.0.1`, port `1337`, prefix `/v1`, timeout `600` (`web-app/src/hooks/useLocalApiServer.ts:41-47`, `:62`).
- Jan includes first-party “Claude Code env export” integration:
  - writes `ANTHROPIC_BASE_URL` and auth/model env vars (`src-tauri/src/core/system/commands.rs:181`).

### (2) ASCII diagrams

#### Request routing path
```text
External app -> http://127.0.0.1:1337/v1/chat/completions
  -> Proxy validates Host + Auth + CORS
  -> Parse JSON body.model
  -> if model mapped to remote provider: forward to provider base_url
  -> else if local session exists: forward to http://127.0.0.1:{session_port}/v1/...
  -> stream/return response
```

#### Anthropic fallback path
```text
POST /v1/messages
  -> try upstream /messages first
  -> if upstream error:
       transform request to OpenAI /chat/completions
       forward + transform response back to Anthropic shape
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- One local API plane that multiplexes local+cloud models behind OpenAI-compatible routes.
- Host and auth guardrails (trusted hosts + bearer/x-api-key compatibility).
- Built-in docs endpoint (`/openapi.json` + Swagger) for integration ergonomics.

**Study**
- Anthropic/OpenAI transform layer complexity and long-term maintenance cost.
- Security edge-cases around docs-path allowlisting and CORS behavior.

**Skip**
- Overly broad backward-compat shims if Cowork can define one canonical API surface early.

---

## 4) Provider System (local vs cloud abstraction)

### (1) How Jan built it (with file paths and code patterns)

### A. Frontend provider composition = predefined cloud + runtime engines
- Built-in provider catalog in `web-app/src/constants/providers.ts:27` (OpenAI, Azure, Anthropic, OpenRouter, Mistral, Groq, Gemini, HuggingFace, etc).
- Runtime providers are discovered from registered engines (`web-app/src/services/providers/tauri.ts:48`).
- Uses Tauri HTTP plugin fetch to avoid browser CORS restrictions (`tauri.ts:11`, `:17`, `:166`).

### B. Remote provider credentials/models are registered into Rust state for API proxy routing
- Frontend sync: `web-app/src/providers/DataProvider.tsx:31` and `:153`.
- Rust commands store provider config in-memory map:
  - `register_provider_config` (`src-tauri/src/core/server/remote_provider_commands.rs:25`)
  - state in `AppState.provider_configs` (`src-tauri/src/core/state.rs:49`).

### C. Model creation is unified through Vercel AI SDK adapters
- `web-app/src/lib/model-factory.ts`:
  - local `llamacpp`/`mlx` sessions mapped to `OpenAICompatibleChatLanguageModel` (`:191`, `:253`),
  - cloud OpenAI/Anthropic official SDK adapters (`:342`, `:369`),
  - OpenAI-compatible generic adapter for many vendors (`:396`).
- Common parameter injection via custom fetch wrapper (`model-factory.ts:122`).

### D. Legacy and migration handling in persisted provider store
- Zustand persisted provider state + migrations in `web-app/src/hooks/useModelProvider.ts:25`.
- Includes old Cortex migration flags and schema migrations (`useModelProvider.ts:57`, `:225`).

### (2) ASCII diagrams

```text
Provider sources
  +--------------------+      +------------------------+
  | predefinedProviders|      | runtime EngineManager  |
  | (cloud defaults)   |      | (llamacpp, mlx, etc.)  |
  +---------+----------+      +-----------+------------+
            \                          /
             \                        /
              +------ merged provider graph ------+
                             |
                             v
                    ModelFactory.createModel()
                             |
          +------------------+------------------+
          |                                     |
   local session adapters                 cloud adapters
   (llamacpp/mlx local URL)               (openai/anthropic/oai-compatible)
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- Clean split: provider metadata in frontend, routing credentials mirrored into backend for local API server.
- AI SDK factory abstraction that hides provider-specific differences behind a single constructor.

**Study**
- In-memory-only remote provider registration (runtime, not durable by default on Rust side).
- Duplicate source-of-truth risk across frontend persisted state and backend registered map.

**Skip**
- Excessive special-case migration logic accumulating in provider state if not aggressively retired.

---

## 5) MCP Integration (agentic features + tool calling)

### (1) How Jan built it (with file paths and code patterns)

### A. MCP is first-class in Rust core with runtime server manager
- Command layer: `src-tauri/src/core/mcp/commands.rs`.
- Startup/bootstrap: `setup_mcp` in `src-tauri/src/core/setup.rs:275`.
- Shared runtime maps in `AppState` for active services, restart monitors, PIDs, cancellation tokens (`src-tauri/src/core/state.rs:40-49`).

### B. Config persistence + migrations
- File: `mcp_config.json` under Jan data folder (`commands.rs:343`, `setup.rs:283`).
- Default template includes Jan Browser MCP + example servers and settings (`src-tauri/src/core/mcp/constants.rs:7`).
- Schema migration flow exists in `setup::migrate_mcp_servers` (`src-tauri/src/core/setup.rs:157`).

### C. Multiple transport types supported
- `start_mcp_server` supports:
  - `stdio` child process,
  - streamable HTTP,
  - SSE
  (`src-tauri/src/core/mcp/helpers.rs:281`, `:343`, and surrounding logic).

### D. Tool calling supports timeout and cancellation
- `get_tools` with timeout per server (`commands.rs:165`, `:173`).
- `call_tool` includes optional cancellation token and races timeout vs cancel signal (`commands.rs:221`, `:273`).
- `cancel_tool_call` removes/sends token (`commands.rs:319`).

### E. Process-hardening details for local MCP servers
- Port conflict cleanup for Jan Browser MCP (`helpers.rs:413`, `:419`).
- Lockfile-based stale process cleanup (`src-tauri/src/core/mcp/lockfile.rs:23`, `:116`, `:146`).
- Runtime command overrides:
  - `npx` can be swapped to bundled `bun x` (`helpers.rs:443`),
  - `uvx` can be swapped to bundled `uv tool run` (`helpers.rs:458`).

### F. Frontend tool integration into generation stack
- MCP service adapter: `web-app/src/services/mcp/tauri.ts`.
- MCP settings/state store: `web-app/src/hooks/useMCPServers.ts`.
- Tools merged into chat transport with AI SDK tool schemas (`web-app/src/lib/custom-chat-transport.ts:182-200`).

### (2) ASCII diagrams

```text
mcp_config.json
   -> setup_mcp() on app startup
   -> run_mcp_commands()
   -> start_mcp_server(name, config)
      -> transport: stdio/http/sse
      -> register RunningService in AppState
      -> emit "mcp-update"

Chat request
   -> CustomChatTransport.refreshTools()
      -> serviceHub.mcp().getTools()
      -> JSON schema -> AI SDK Tool map
   -> model streamText(... tools=...)
   -> tool call -> mcp.callTool() -> Rust call_tool() -> target MCP server
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- Runtime MCP supervisor pattern with active-server restart semantics.
- Timeout + cancellation tokens for tool calls.
- Lockfile + orphan cleanup for local sidecar processes.
- Unified tool injection into model runtime.

**Study**
- Operational complexity of managing stdio/http/sse in one module.
- Security controls around arbitrary MCP command execution.

**Skip**
- Shipping many default servers by default; safer to ship minimal defaults and explicit user opt-in.

---

## 6) Storage & Data (local persistence)

### (1) How Jan built it (with file paths and code patterns)

### A. App config and data-root indirection
- App config file path logic in `src-tauri/src/core/app/commands.rs:101`.
- Data root from `AppConfiguration.data_folder` via `get_jan_data_folder_path` (`commands.rs:71`, `:96`).
- Data folder migration copies files recursively while excluding `.uvx`, `.npx` (`commands.rs:166`, `:190-194`).

### B. Store-based migration state
- `store.json` in Jan data folder tracks app version + migration markers (`src-tauri/src/lib.rs:244`, `:268`).

### C. Conversation storage strategy: desktop file-based, mobile SQLite
- Thread constants: `threads/{threadId}/thread.json` + `messages.jsonl` (`src-tauri/src/core/threads/constants.rs:1`).
- Desktop commands read/write those files (`src-tauri/src/core/threads/commands.rs`).
- Per-thread async locks prevent concurrent message write corruption (`threads/helpers.rs:13`, `:22`).
- Mobile uses SQLite (`src-tauri/src/core/threads/db.rs:21`) with `threads` and `messages` tables.

### D. Model metadata storage
- llama model manifests persisted as YAML under provider path, typically `llamacpp/models/{modelId}/model.yml` (`extensions/llamacpp-extension/src/index.ts:1134`, `:1323`).

### E. MCP and local API settings persistence
- MCP configs in `mcp_config.json` (`mcp/commands.rs:343`).
- Local API settings persisted in frontend localStorage store (`web-app/src/hooks/useLocalApiServer.ts:36`).
- Provider preferences and migrations in localStorage via Zustand persist (`web-app/src/hooks/useModelProvider.ts:25`).

### (2) ASCII diagrams

```text
Jan Data Folder
├── settings.json (app config -> data_folder path indirection)
├── store.json (version + migration markers)
├── logs/app.log
├── mcp_config.json
├── threads/
│   └── {thread_id}/
│       ├── thread.json
│       └── messages.jsonl
└── llamacpp/
    ├── models/{model_id}/model.yml
    └── backends/{version}/{backend}/...

Mobile-only:
app_data_dir/jan.db (SQLite threads/messages)
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- Explicit data-root abstraction (`data_folder`) for portability and enterprise policy control.
- Desktop file-based thread persistence with per-thread locking (simple and debuggable).
- Mobile-specific SQLite backend split.

**Study**
- Mixed persistence surface (Rust files + frontend localStorage) and reconciliation logic.
- Data migration edge cases when users move data roots.

**Skip**
- Letting too much critical runtime config live only in frontend localStorage if backend also depends on it.

---

## 7) Model Download & Management (HuggingFace, formats, quantization)

### (1) How Jan built it (with file paths and code patterns)

### A. Download orchestration is extension-driven, Rust-executed
- Extension facade: `extensions/download-extension/src/index.ts:53`.
- For each task, frontend calls Rust `download_files` and listens to `download-{taskId}` events (`index.ts:59-74`).
- HF token support is translated to `Authorization` headers (`index.ts:92-95`).

### B. Rust downloader is parallel, cancellable, resumable, and validates artifacts
- Entry command: `src-tauri/src/core/downloads/commands.rs:10`.
- Internal pipeline: `_download_files_internal` in `downloads/helpers.rs:378`.
- Features:
  - Per-task cancellation token map (`commands.rs:18`, `models.rs:8`).
  - Parallel per-file tasks with aggregated progress (`helpers.rs:411-445`).
  - Resume support via `.tmp` and `.url` sidecar (`helpers.rs:564-573`).
  - Path safety check ensures `save_path` stays under Jan data folder (`helpers.rs:418-424`).
  - Optional size and SHA256 validation with model-validation events (`helpers.rs:88`, `:119`, `:494`).

### C. HuggingFace mirror fallback with signed requests
- Mirror conversion for HF domains (`helpers.rs:19-27`, `:57-73`).
- Download attempts mirror first, falls back to original URL (`helpers.rs:720-742`).
- Mirror requests include HMAC-signed headers (`helpers.rs:746-767`).

### D. Model catalog and HF repo parsing in frontend
- `web-app/src/services/models/default.ts`:
  - fetches HF repo metadata (`:63-101`),
  - filters `.gguf` files,
  - splits regular quants vs `mmproj` files,
  - creates direct `resolve/main/...` URLs (`:115-149`).
- Quantization is currently filename-driven in catalog/UI (e.g., `Q4_K_M`, `Q8_0` in variants), not deeply parsed into a canonical quant enum.

### E. GGUF and support checks integrated into download/import UX
- During import, GGUF metadata is validated for both main and mmproj files (`extensions/llamacpp-extension/src/index.ts:1272`, `:1285`).
- Support checks call Rust `is_model_supported` and classify RED/YELLOW/GREEN (`gguf/commands.rs:50`).

### F. Backend binary management is coupled but distinct from model download
- llama backend downloader in `extensions/llamacpp-extension/src/backend.ts:153`:
  - GitHub release query with CDN fallback,
  - backend archive + optional CUDA runtime bundle installation,
  - decompression into versioned backend folders.

### (2) ASCII diagrams

```text
HF model selected
  -> DefaultModelsService.fetchHuggingFaceRepo()
  -> Convert repo siblings to quants/mmproj URLs
  -> llamacpp extension import(modelId, opts)
      -> optional download files via download-extension
          -> Rust _download_files_internal (parallel + resume + validate)
             -> mirror fallback (apps.jan.ai) -> origin fallback
      -> GGUF metadata validation
      -> write model.yml
      -> onModelImported
```

```text
Backend binary lifecycle
  if backend executable missing:
    listSupportedBackends() -> remote+local merge
    downloadBackend(version/backend)
    decompress archive(s)
    run llama-server binary for model sessions
```

### (3) What Cowork.ai should Copy / Study / Skip

**Copy**
- Strong downloader core: cancellation + resume + path guard + hash/size verification.
- HF repo metadata-driven import flow (including mmproj pairing).
- Mirror-first with origin fallback strategy for resilience.

**Study**
- Operational cost and security implications of custom signed mirror infrastructure.
- Separation-of-concerns between “model files” and “engine binaries” in product UX.

**Skip**
- Leaving quantization semantics mostly implicit in filenames; Cowork should normalize quant metadata explicitly in schema.

---

## 8) Summary: Copy / Study / Skip (for Cowork.ai)

| Area | Copy | Study | Skip |
|---|---|---|---|
| Tauri architecture | Service-hub + `window.core.api` abstraction; centralized native state; startup migrations | Command surface growth and capability boundaries | Broad legacy FS command exposure |
| Local LLM execution | Session registry (PID/port/key); arg-builder; readiness/timeout guards; GGUF preflight checks | Auto-unload behavior and user expectations | Long-lived legacy migration scaffolding |
| Local API server | Unified OpenAI-compatible gateway across local+remote; trusted-host + dual auth checks; local OpenAPI docs | Anthropic/OpenAI transform complexity and security testing | Excess compatibility layers if greenfield API can be cleaner |
| Provider system | Factory-based model adapter layer over Vercel AI SDK; runtime provider registration pattern | Dual source-of-truth between FE state and BE runtime map | Heavy provider-specific conditional drift in core code |
| MCP integration | Runtime supervisor, timeout+cancellation, lockfile/orphan cleanup, tool injection into chat runtime | Mixed transport operational complexity and command execution risk model | Too many default active servers out-of-box |
| Storage/data | Data-root indirection; desktop file persistence with thread locks; mobile SQLite split | Cross-store consistency (files + localStorage + runtime memory) | Critical config only in localStorage without BE durability |
| Model downloads | Parallel/resumable/cancellable validated downloader with path safety; HF metadata-based import; mirror fallback | Mirror signing infra and maintenance burden | Filename-only quant semantics without normalized metadata |

---

## Cowork.ai-Specific Strategic Recommendations

1. Keep Electron, but copy Jan’s **native-service boundaries**:
   - Jan’s biggest structural advantage is not Tauri itself; it is the clean split between UI, service adapters, and native orchestration modules.
   - In Cowork (Electron), replicate this as:
     - renderer service hub,
     - strictly typed IPC contract,
     - process supervisor in main/Node side.

2. Build a Jan-like **local API gateway** early:
   - One localhost API that can route to local runtimes and cloud providers gives strong ecosystem leverage.
   - Include strict host/auth policy from day one.

3. Prioritize Jan’s **download integrity model**:
   - Path sandboxing + checksum/size verification + cancellation/resume should be table stakes for enterprise trust.

4. For MCP, start with Jan’s operational primitives, but tighter defaults:
   - keep timeout/cancel/health-check/lockfile patterns,
   - reduce default server footprint,
   - add stronger allowlist/policy controls for command-based servers.

5. Improve where Jan is still transitional:
   - normalize quant metadata structurally,
   - eliminate migration leftovers quickly,
   - avoid dual source-of-truth drift between frontend persisted config and backend runtime state.

