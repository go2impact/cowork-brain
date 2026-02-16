# Cherry Studio Reverse-Engineering Deep Dive (for Cowork.ai)

## Scope and ground truth
- Codebase analyzed: `CherryStudio` repository in `workspace`.
- Snapshot reality vs request framing:
  - Cherry Studio has **~62 built-in system provider configs** (not 300+ providers) in `src/renderer/src/config/providers.ts`.
  - It has a **large model catalog (~318 model entries, ~49 unique provider IDs referenced)** in `src/renderer/src/config/models/default.ts`.
  - It has **780 system assistant preset records per language** in `resources/data/agents-en.json` / `resources/data/agents-zh.json`, expanded into **2,076 selectable presets** via multi-group expansion in `src/renderer/src/pages/store/assistants/presets/index.ts`.
- Important context: many core files are marked `@deprecated` and note an ongoing `v2` refactor (`issue #10954`, `PR #10162`). Treat some patterns as transitional.

---

## 1) Electron + Vite Architecture

### (1) How they built it

Core split and build orchestration:
- `electron.vite.config.ts`
  - Defines separate bundles for `main`, `preload`, and `renderer`.
  - `renderer` has **multi-entry HTML targets**:
    - `src/renderer/index.html`
    - `src/renderer/miniWindow.html`
    - `src/renderer/selectionToolbar.html`
    - `src/renderer/selectionAction.html`
    - `src/renderer/traceWindow.html`
  - `main` disables chunk splitting via:
    - `output.manualChunks = undefined`
    - `output.inlineDynamicImports = true`
  - Process-specific aliases (`@main`, `@renderer`, `@shared`, `@logger`, ai-core aliases).

Bootstrap and lifecycle:
- `src/main/bootstrap.ts`
  - Early data-dir init before normal startup.
- `src/main/index.ts`
  - App singleton lock, startup flags, window creation, tray, protocol handlers, service initialization.
  - Calls `registerIpc(mainWindow, app)`.

Window creation and security/runtime posture:
- `src/main/services/WindowService.ts`
  - Uses `BrowserWindow` with preload `../preload/index.js`.
  - Notable `webPreferences`:
    - `sandbox: false`
    - `webSecurity: false`
    - `webviewTag: true`
    - `allowRunningInsecureContent: true`
  - Supports main + mini windows and webview hooks.

IPC architecture:
- `src/main/ipc.ts`
  - Central registration of many `ipcMain.handle(...)` handlers.
  - Delegates to service layer (`MCPService`, `KnowledgeService`, `BackupManager`, `StoreSyncService`, etc.).
- `src/preload/index.ts`
  - Very broad typed API surface exposed via `contextBridge.exposeInMainWorld('api', api)`.
  - Pattern: `window.api.<domain>.<method>() => ipcRenderer.invoke(IpcChannel.X, ...)`.

Tracing stitched through IPC:
- `src/preload/index.ts` defines `tracedInvoke(...)` by appending `{ type: 'trace', context }`.
- `src/main/services/NodeTraceService.ts` monkey-patches `ipcMain.handle` to detect that payload and restore OTel context for handler execution.

Code patterns worth noting:
- Channelized constants: `@shared/IpcChannel`.
- “Thin preload, fat main service” intent exists, but preload currently became a large surface.
- Main process has service-level abstractions, but `ipc.ts` is still very large.

### (2) ASCII architecture/data flow

```text
                +------------------------+
                |   electron-vite build  |
                | main | preload | render|
                +-----------+------------+
                            |
                 app startup (src/main/index.ts)
                            |
                    create BrowserWindow
                            |
                 preload injects window.api
                            |
Renderer React/Redux -----> ipcRenderer.invoke(IpcChannel.*)
                            |
                        ipcMain.handle
                            |
                     src/main/ipc.ts routes
                            |
                   Main Services (MCP, KB,
                   Backup, Files, Sync...)
                            |
                      OS/network/filesystem
```

Tracing path:

```text
Renderer tracedInvoke(channel, spanContext)
   -> invoke(..., {type:'trace', context})
      -> ipcMain.handle patched in NodeTraceService
         -> context.with(span) -> real handler
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- Multi-window Electron packaging pattern from `electron.vite.config.ts`.
- Strong channel constants and domain APIs (`IpcChannel` + namespaced preload API).
- IPC trace propagation pattern (`tracedInvoke` + main-side context restore).

Study:
- Main-process service decomposition strategy (`src/main/services/*`) and startup sequencing.
- How they keep multiple renderer windows in one bundle ecosystem.

Skip:
- Current security posture in `WindowService` (`webSecurity: false`, `allowRunningInsecureContent: true`, `sandbox: false`) for production-sensitive apps.
- Monolithic preload API growth; prefer generated typed bridges by domain.

---

## 2) LLM Provider System

### (1) How they built it

Catalog and config layers:
- Provider definitions: `src/renderer/src/config/providers.ts` (~62 provider entries).
- Model catalog: `src/renderer/src/config/models/default.ts` (~318 model IDs, ~49 distinct provider IDs referenced).

Runtime orchestration:
- Entry point for chat calls: `src/renderer/src/services/ApiService.ts`
  - Resolves assistant model/provider.
  - Rotates API keys (`provider.apiKey` split by comma, round-robin persisted in `window.keyv`).
  - Builds params via `buildStreamTextParams(...)`.
  - Instantiates `ModernAiProvider` and executes completions.

Modern provider abstraction:
- `src/renderer/src/aiCore/index_new.ts` (`ModernAiProvider`)
  - Converts app provider -> AI SDK config (`providerToAiSdkConfig`).
  - Lazily creates provider instance via `createAiSdkProvider`.
  - Builds middleware (`buildAiSdkMiddlewares`) + plugins (`buildPlugins`).
  - Executes via `createExecutor(...).streamText(...)` from `@cherrystudio/ai-core`.
  - Converts stream into app chunk protocol using `AiSdkToChunkAdapter`.

Provider adaptation and aliasing:
- `src/renderer/src/aiCore/provider/providerConfig.ts`
  - Provider-specific API-host normalization (`formatProviderApiHost`).
  - Endpoint routing and mode selection (`chat` vs `responses`).
- `src/renderer/src/aiCore/provider/factory.ts`
  - Maps app provider IDs/types -> AI SDK provider IDs.
- `src/renderer/src/aiCore/provider/providerInitialization.ts`
  - Dynamic registration of additional providers (`openrouter`, `bedrock`, `gateway`, etc.).

Core registry/model resolver internals:
- `packages/aiCore/src/core/providers/RegistryManagement.ts`
  - Provider registry built on AI SDK `createProviderRegistry`.
  - Uses separator `|` to avoid collisions (e.g. with model suffixes).
- `packages/aiCore/src/core/models/ModelResolver.ts`
  - Resolves namespaced model IDs and fallback provider+model IDs.
- `packages/aiCore/src/core/runtime/executor.ts`
  - Unified `streamText/generateText/generateObject/generateImage` with plugin engine.

Streaming and chunk adaptation:
- `src/renderer/src/aiCore/chunk/AiSdkToChunkAdapter.ts`
  - Maps AI SDK `TextStreamPart` events into app chunk types (`TEXT_DELTA`, `THINKING_DELTA`, tool chunks, web-search chunks, etc.).

Notable hybridity:
- `ModernAiProvider` still routes some image-generation endpoints to legacy provider path for advanced compatibility.

### (2) ASCII architecture/data flow

```text
Assistant + Model + Provider
        |
ApiService.fetchChatCompletion
        |
providerToAiSdkConfig + createAiSdkProvider
        |
buildMiddlewares + buildPlugins
        |
createExecutor(providerId, options, plugins)
        |
executor.streamText(...)
        |
AI SDK stream parts (text/reasoning/tool/search)
        |
AiSdkToChunkAdapter
        |
Cherry chunk events -> UI block renderer
```

Registry flow:

```text
Provider configs (built-in + dynamic)
        -> registerProviderConfig
        -> createProvider
        -> registerProvider
        -> globalRegistryManagement
        -> ModelResolver resolves provider|model
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- Layered provider adaptation (`providerConfig.ts`) decoupled from UI models.
- Registry + alias pattern (`RegistryManagement.ts`) for multi-provider extensibility.
- Stream adapter boundary (`AiSdkToChunkAdapter`) to isolate UI protocol from SDK protocol.

Study:
- Middleware/plugin composition strategy (`AiSdkMiddlewareBuilder`, `PluginBuilder`).
- Operational safeguards like API key rotation and provider-specific mode routing.

Skip:
- Legacy + modern dual-path complexity unless migration is temporary.
- AI SDK v5-specific workarounds if Cowork is standardizing on Vercel AI SDK v6.

---

## 3) MCP Server Integration

### (1) How they built it

Main MCP control plane:
- `src/main/services/MCPService.ts`
  - Client cache keyed by server config signature.
  - `pendingClients` deduplicates concurrent init races.
  - `withCache(...)` wraps tool/prompt/resource list/read calls with TTL.
  - Notification handlers invalidate cache (`ToolListChanged`, `PromptListChanged`, `ResourceListChanged`, etc.).
  - Server log buffering + push to renderer (`IpcChannel.Mcp_ServerLog`).

Transport abstraction:
- `src/main/services/MCPService.ts` chooses transport per server:
  - In-memory built-ins via `InMemoryTransport`.
  - Streamable HTTP / SSE for URL-based servers.
  - Stdio for command servers (`npx`, `uvx`, `uv`) with shell-path discovery and bundled binary fallback.
  - OAuth handling for auth-required servers.

Built-in server factory:
- `src/main/mcpServers/factory.ts`
  - Creates in-memory servers by name (`memory`, `filesystem`, `fetch`, `browser`, `hub`, etc.).

Hub meta-server (aggregation/orchestration):
- `src/main/mcpServers/hub/index.ts`
  - Exposes tools: `list`, `inspect`, `invoke`, `exec`.
  - Aggregates active tools and maps namespaced IDs to JS-friendly names.
- `src/main/mcpServers/hub/runtime.ts`
  - Runs `exec` code in worker thread with timeout (60s) and tool-call mediation.
- `src/main/mcpServers/hub/mcp-bridge.ts`
  - Bridges hub calls back into `MCPService` tool invocation.

Renderer integration:
- `src/preload/index.ts` exposes `window.api.mcp.*`.
- `src/renderer/src/store/mcp.ts`
  - Stores server configs and built-in templates.
  - Defines `hubMCPServer` for auto mode.
- `src/renderer/src/services/ApiService.ts`
  - `getMcpServersForAssistant()` supports `disabled` / `manual` / `auto` (`auto` -> hub).
- `src/renderer/src/types/mcp.ts`
  - Zod validation for server config schema and inferred transport type.

### (2) ASCII architecture/data flow

```text
Renderer assistant (mcpMode)
   |
   +-- manual -> selected active servers
   +-- auto   -> hubMCPServer (meta)
   |
window.api.mcp.* (preload)
   |
ipc -> MCPService (main)
   |
Transport select:
  - inMemory (builtin)
  - streamableHttp / sse
  - stdio (npx/uvx/uv)
   |
MCP server(s)
   |
tools/prompts/resources + notifications
   |
cache invalidation + logs + progress events -> renderer
```

Hub orchestration path:

```text
LLM calls hub.exec(code)
   -> Worker runtime
      -> mcp.callTool(name, params)
         -> resolve toolId mapping
            -> MCPService.callToolById(...)
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- Transport-agnostic MCP client initialization with config-keyed client cache.
- Pending init dedupe and list/read caching with notification-driven invalidation.
- Server log/event propagation to UI for operability.

Study:
- Hub meta-server design for tool discovery + orchestration.
- Worker-thread execution timeout and call abort controls.

Skip:
- Hardcoded special-case branches that may accumulate tech debt.
- Implicit trust defaults for tool execution without stronger per-tool policy gates.

---

## 4) Multi-Model Conversations

### (1) How they built it

Input model mentions:
- `src/renderer/src/pages/home/Inputbar/Inputbar.tsx`
  - Message send includes `mentions` if user selected models.
- `src/renderer/src/pages/home/Inputbar/MentionModelsInput.tsx`
  - Visual tags for selected models.
- `src/renderer/src/pages/home/Inputbar/tools/mentionModelsTool.tsx`
  - Tool/quick-panel for model picking.

Dispatch fan-out:
- `src/renderer/src/store/thunk/messageThunk.ts`
  - `sendMessage` checks `userMessage.mentions`.
  - `dispatchMultiModelResponses(...)`:
    - Creates one assistant message per mentioned model.
    - Sets same `askId` = triggering user message ID.
    - Queues one generation task per model.

Queueing model responses:
- `src/renderer/src/utils/queue.ts`
  - Topic-based `PQueue` instances (`getTopicQueue(topicId)`).
  - No explicit concurrency passed in this call path, so behavior may be highly parallel depending on `PQueue` defaults/options.

Grouped rendering UX:
- Grouping utility: `src/renderer/src/utils/messageUtils/filters.ts` (`getGroupedMessages`).
  - Assistant group key: `assistant + askId`.
- Rendering:
  - `src/renderer/src/pages/home/Messages/Messages.tsx`
  - `src/renderer/src/pages/home/Messages/MessageGroup.tsx`
  - `src/renderer/src/pages/home/Messages/MessageGroupModelList.tsx`
- Supports display styles: fold/horizontal/grid and “useful” message tagging within group.

Settings affecting UX:
- `src/renderer/src/store/settings.ts`
  - `multiModelMessageStyle`, `foldDisplayMode`, `gridColumns`, `gridPopoverTrigger`.

### (2) ASCII architecture/data flow

```text
User input + @model mentions
        |
Inputbar creates userMessage { mentions:[m1,m2,...] }
        |
messageThunk.sendMessage
        |
dispatchMultiModelResponses
        |
Create assistant stubs:
  msgA(askId=userMsgId, model=m1)
  msgB(askId=userMsgId, model=m2)
  ...
        |
Queue per topic -> fetch/process per model
        |
messages grouped by askId in UI
        |
MessageGroup renders tabs/cards/grid of model outputs
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- `askId` fan-out model for multi-response synchronization.
- Grouped-response UI with quick switching and “best/useful” selection.

Study:
- Queue tuning strategy for concurrency vs provider rate-limits.
- Persist/retry semantics when one model in the group fails.

Skip:
- Keeping this logic in a very large deprecated thunk file long-term.
- Implicit queue defaults; make concurrency explicit per provider or account tier.

---

## 5) Assistant System (Preset Library)

### (1) How they built it

Preset data sources:
- Local JSON catalogs:
  - `resources/data/agents-en.json`
  - `resources/data/agents-zh.json`
- Loader hook: `src/renderer/src/pages/store/assistants/presets/index.ts`
  - If `agentssubscribeUrl` is HTTP, attempts remote fetch first.
  - Falls back to local resources path (`resourcesPath/data/agents-*.json`).

Preset shaping:
- `getAgentsFromSystemAgents(...)` in `src/renderer/src/pages/store/assistants/presets/index.ts`
  - Expands each preset by each group tag (one record per group assignment).

State management:
- `src/renderer/src/store/assistants.ts`
  - `presets` array lives in Redux and is persisted.
- `src/renderer/src/hooks/useAssistantPresets.ts`
  - CRUD wrapper hooks.

Creation UX:
- `src/renderer/src/components/Popups/AddAssistantPopup.tsx`
  - Merges user presets + system presets + default assistant.
  - Filters out already-created assistants.
  - Keyboard navigation and max display chunking.

Observed scale:
- Raw system preset records: 780 per language JSON.
- Expanded selectable entries via group expansion: ~2,076.

### (2) ASCII architecture/data flow

```text
resources/data/agents-*.json  OR  remote subscribe URL
                 |
      useSystemAssistantPresets()
                 |
     getAgentsFromSystemAgents() expansion
                 |
      merged with user presets (Redux)
                 |
            AddAssistantPopup
                 |
   createAssistantFromAgent -> assistants store
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- Local-first preset bundle with optional remote override/fallback.
- Fast preset-to-assistant instantiation flow.

Study:
- Preset quality controls and taxonomy governance at scale.
- How to version presets independently from app release.

Skip:
- Unbounded static prompt catalog growth without lifecycle management.
- Duplicative expansion strategy if search/ranking can do better than multiplying entries.

---

## 6) Storage & Sync (Local persistence + WebDAV pattern)

### (1) How they built it

Local state layers:
- Redux persistence:
  - `src/renderer/src/store/index.ts`
  - Uses `redux-persist` + `localStorage` key `cherry-studio`.
  - Blacklists runtime-heavy slices (`messages`, `messageBlocks`, etc.).
- IndexedDB/Dexie:
  - `src/renderer/src/databases/index.ts`
  - DB: `CherryStudio` with versioned migrations up to v10.
  - Tables include `topics`, `message_blocks`, `files`, `knowledge_notes`, etc.

Knowledge/vector storage:
- `src/main/services/KnowledgeService.ts`
  - Uses `@cherrystudio/embedjs` with `LibSqlDb` under `Data/KnowledgeBase` path.

Main file persistence:
- `src/main/utils/file.ts` and `src/main/services/FileStorage.ts`
  - Stores binary/user files under `app.getPath('userData')/Data/Files`.
  - Notes under `.../Data/Notes`.

Multi-window sync:
- Renderer middleware: `src/renderer/src/services/StoreSyncService.ts`
  - Broadcasts whitelisted slice actions to main (`window.api.storeSync.onUpdate`).
- Main broker: `src/main/services/StoreSyncService.ts`
  - Tracks subscribed windows and rebroadcasts with `meta.fromSync = true` to prevent loops.

Backup + restore (local/WebDAV/S3):
- Renderer orchestration: `src/renderer/src/services/BackupService.ts`
  - `getBackupData()` captures `localStorage` + IndexedDB snapshot.
  - Auto-sync scheduler with intervals, retries, exponential backoff, and retention cleanup.
- Main execution: `src/main/services/BackupManager.ts`
  - Creates ZIP backup (data.json + optional `Data/` directory).
  - Upload/download for WebDAV/S3/local dir.
  - Caches client instances by core connection params.
- WebDAV client wrapper: `src/main/services/WebDav.ts`
  - Creates client with `rejectUnauthorized: false`.
  - Supports stream and buffer upload modes (toggle via `webdavDisableStream`).

### (2) ASCII architecture/data flow

```text
Renderer state
  - Redux persisted slices (localStorage)
  - Dexie tables (IndexedDB)
         |
BackupService.getBackupData()
         |
ipc -> BackupManager.backup()
         |
ZIP [data.json + optional Data/]
         |
+-------------------------------+
| local dir | WebDAV | S3 target|
+-------------------------------+
```

Window sync:

```text
Renderer A dispatch (whitelisted)
   -> StoreSync middleware
   -> IPC OnUpdate
   -> Main StoreSyncService broker
   -> BroadcastSync to Renderer B/C
   -> dispatch(action.meta.fromSync=true)
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- Hybrid persistence split (Redux UI state vs DB content state).
- Action-level multi-window sync via IPC broker.
- Backup scheduling + retries + retention policy scaffolding.

Study:
- Snapshot compatibility/migrations over long product lifetime.
- Operational UX around backup progress and failure recovery.

Skip:
- `rejectUnauthorized: false` for WebDAV in production-grade sync.
- Overreliance on large localStorage snapshots for critical state.

---

## 7) UI Framework (component library, theming, document processing)

### (1) How they built it

Foundation stack:
- `package.json`
  - React 19, Ant Design 5, styled-components 6, Tailwind 4, motion/framer-motion.
  - Editors/renderers: TipTap 3, CodeMirror 6, ReactMarkdown, KaTeX/MathJax, Mermaid, Shiki.

Theming system:
- Theme context: `src/renderer/src/context/ThemeProvider.tsx`
  - Tracks system/user mode and updates `theme-mode` attributes.
- AntD bridge: `src/renderer/src/context/AntdProvider.tsx`
  - Uses AntD `ConfigProvider` with CSS variable mode and component token overrides.
- CSS variable palette: `src/renderer/src/assets/styles/color.css`.
- User theme injection (primary color + fonts): `src/renderer/src/hooks/useUserTheme.ts`.

Chat/document rendering:
- Markdown renderer: `src/renderer/src/pages/home/Markdown/Markdown.tsx`
  - Pipeline includes `remark-gfm`, `remark-math`, `rehype-katex` / `rehype-mathjax`, raw HTML gating, custom code/table/link/image renderers.
- Rich text editor: `src/renderer/src/components/RichEditor/index.tsx`
  - TipTap-based editing with custom commands, drag handles, table actions, ToC, content search.

Document preprocessing/OCR for knowledge ingestion:
- OCR provider registry: `src/main/services/ocr/OcrService.ts`.
- Preprocess provider factory/pipeline:
  - `src/main/knowledge/preprocess/PreprocessProviderFactory.ts`
  - `src/main/knowledge/preprocess/PreprocessingService.ts`
  - Example provider: `src/main/knowledge/preprocess/PaddleocrPreprocessProvider.ts`
- Knowledge queue and preprocess status updates:
  - `src/renderer/src/queue/KnowledgeQueue.ts`
  - `src/renderer/src/store/preprocess.ts`

### (2) ASCII architecture/data flow

```text
ThemeProvider + AntdProvider + CSS vars
            |
     React component tree
   (AntD + styled-components + Tailwind)
            |
   Chat renderers / editors
   - Markdown pipeline (remark/rehype)
   - TipTap RichEditor
   - Code blocks / math / mermaid
```

Doc processing path:

```text
User uploads document (e.g. PDF)
        |
KnowledgeQueue -> window.api.knowledgeBase.add
        |
Main KnowledgeService preprocessing hook
        |
PreprocessProviderFactory (doc2x/mineru/mistral/paddleocr/...)
        |
processed markdown/text artifact
        |
embedding + storage in knowledge base
```

### (3) What Cowork.ai should copy / study / skip

Copy:
- CSS-variable-first theming plus AntD token binding.
- Clear markdown rendering pipeline with pluggable components.
- Provider-factory approach for OCR/preprocess adapters.

Study:
- RichEditor extension architecture and command registry.
- Dependency governance for heavy frontend/editor stacks.

Skip:
- Mixing too many styling paradigms unless you have strict style governance.
- Carrying large UI dependency surface without bundle/perf budgets.

---

## 8) Summary: Copy / Study / Skip (for Cowork.ai)

| Area | Copy | Study | Skip |
|---|---|---|---|
| Electron + Vite | Multi-entry Electron-Vite build split; typed preload API namespaces; IPC channel constants | Trace context propagation over IPC; service startup sequencing | Insecure `webPreferences` defaults; monolithic preload surface |
| LLM Provider System | Adapter layer (`provider -> ai-sdk config`), provider alias registry, stream adapter boundary | Middleware/plugin composition and provider-specific behavior routing | Long-lived legacy+modern dual path; SDK-v5-specific compatibility hacks |
| MCP Integration | Transport abstraction + pending-init dedupe + cache invalidation hooks | Hub meta-server (`list/inspect/invoke/exec`) and worker timeout control | Hardcoded special cases and weak trust defaults for tool execution |
| Multi-Model Conversations | `askId` fan-out model grouping, grouped response UX modes | Queue tuning, cancellation, and per-provider concurrency policy | Implicit queue defaults and oversized thunk-based orchestration |
| Assistant System | Local bundled presets with remote override fallback | Prompt catalog governance, quality/versioning pipeline | Unbounded static preset expansion and duplicated entries |
| Storage & Sync | Hybrid state persistence split; IPC action sync across windows; backup retention/retry scaffolding | Long-term migration/version strategy and backup UX resilience | TLS bypass in WebDAV client; localStorage-heavy critical snapshots |
| UI Framework | CSS variable theming + AntD tokens; markdown/render pipeline; preprocess provider factory | TipTap extension governance and dependency budget management | Overly broad UI dependency stack without strict ownership/perf controls |

---

## Cowork.ai-specific strategic recommendations

1. Implement Cherry-like provider abstraction, but target **Vercel AI SDK v6-first** with no legacy provider path.
2. Adopt Cherry’s MCP transport/cache patterns, but add stricter permissioning and policy enforcement per tool/server.
3. Copy the `askId` grouped multi-model UX pattern, but explicitly cap per-topic concurrency and add per-provider rate-limit policies.
4. Use Cherry’s hybrid persistence pattern, but anchor conversation/knowledge state in your `libsql` backbone and keep localStorage minimal.
5. Keep assistant preset catalogs curated and telemetry-ranked; avoid static catalog sprawl without lifecycle controls.
