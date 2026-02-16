# Chatbox Technical Deep-Dive for Cowork.ai

Scope: reverse-engineered from the local Chatbox codebase in this workspace. The analysis focuses on implementation patterns Cowork.ai can reuse in an Electron + Mastra + Vercel AI SDK + libsql architecture.

## 1) Electron + Vite Architecture (main/renderer split, IPC, build)

### 1.1 How Chatbox built it (files + code patterns)

- Three explicit bundles in `electron.vite.config.ts:60`, `electron.vite.config.ts:118`, `electron.vite.config.ts:141`:
  - `main` entry at `src/main/main.ts`.
  - `preload` entry at `src/preload/index.ts`.
  - `renderer` React app with TanStack Router codegen/splitting.
- Environment-targeted build flags:
  - `CHATBOX_BUILD_PLATFORM` and `CHATBOX_BUILD_TARGET` are injected at build time in `electron.vite.config.ts:112` and `electron.vite.config.ts:233`.
  - Web build behavior (base tag injection) in `electron.vite.config.ts:57` and `electron.vite.config.ts:157`.
- Main process runtime owns OS/window/system concerns in `src/main/main.ts`:
  - Frameless BrowserWindow + preload bridge in `src/main/main.ts:250` and `src/main/main.ts:268`.
  - Single-instance lock/deep-link recovery in `src/main/main.ts:383`.
  - Tray + global shortcut orchestration in `src/main/main.ts:144` and `src/main/main.ts:157`.
  - Huge IPC surface via `ipcMain.handle(...)` in `src/main/main.ts:525`.
- Preload is a strict gateway, not business logic:
  - `contextBridge.exposeInMainWorld('electronAPI', ...)` in `src/preload/index.ts:63`.
  - Typed renderer contract in `src/shared/electron-types.ts:1`.
- Build/release pipeline is integrated with packaging:
  - Scripts in `package.json:13` and mobile/web scripts in `package.json:17`, `package.json:53`.
  - Packaging constraints in `electron-builder.yml:1`, including `asarUnpack` for native libsql modules in `electron-builder.yml:4`.

### 1.2 ASCII architecture/data flow

```text
                +-----------------------------+
                |       electron-vite         |
                |  (3 targets in one config)  |
                +--------------+--------------+
                               |
      +------------------------+-------------------------+
      |                        |                         |
+-----v------+         +-------v--------+        +------v-------+
| Main Proc  |         | Preload Proc   |        | Renderer Proc |
| main.ts    |<--IPC-->| contextBridge  |<--API->| React app     |
| OS/window  |         | electronAPI    |        | stores/routes |
| file/db    |         | thin wrapper   |        | LLM/mcp calls |
+-----+------+         +----------------+        +------+--------+
      |                                                   |
      |                                                   |
+-----v------------------+                        +------v-------+
| Electron APIs          |                        | User UI       |
| app/window/tray/shell  |                        | chat/settings |
+------------------------+                        +--------------+
```

```text
Build modes:

Desktop: electron-vite build -> electron-builder -> native app
Web:     CHATBOX_BUILD_PLATFORM=web + renderer tweaks -> static dist
Mobile:  CHATBOX_BUILD_TARGET=mobile_app -> renderer build -> npx cap sync
```

### 1.3 What Cowork.ai should copy / study / skip

- Copy:
  - The explicit main/preload/renderer separation and typed bridge (`src/shared/electron-types.ts:1`).
  - Renderer chunking strategy in `electron.vite.config.ts:201` to isolate heavy AI/UI/chart dependencies.
  - Single-instance + deep-link handshake pattern (`src/main/main.ts:383`, `src/main/deeplinks.ts:4`).
- Study:
  - The large `ipcMain.handle` surface in `src/main/main.ts:525` should be domain-modularized for long-term maintainability.
  - Security posture around `webSecurity: false` in `src/main/main.ts:266`; this is likely a tradeoff, but risky.
- Skip:
  - Monolithic IPC handler file growth; Cowork.ai should avoid one giant main-process integration file.

---

## 2) LLM Provider Abstraction (multi-provider, unified interface)

### 2.1 How Chatbox built it (files + code patterns)

- Registry-based provider model instead of hardcoded switch:
  - Provider registry map in `src/shared/providers/registry.ts:4`.
  - Provider side-effect registration by imports in `src/shared/providers/index.ts:6`.
  - `getModel(...)` resolver using `ProviderDefinition.createModel` in `src/shared/providers/index.ts:103`.
- Provider definition contract is explicit:
  - `CreateModelConfig` and `ProviderDefinition` in `src/shared/providers/types.ts:10`.
  - Provider modules call `defineProvider(...)`, example `src/shared/providers/definitions/openai.ts:5`.
- Unified runtime model interface:
  - `ModelInterface` (`chat`, `paint`, capability checks) in `src/shared/models/types.ts:12`.
- Shared model execution core:
  - `AbstractAISDKModel` in `src/shared/models/abstract-ai-sdk.ts:74`.
  - Standardized stream loop (`streamText`) at `src/shared/models/abstract-ai-sdk.ts:488`.
  - Tool-call/result/error mapping at `src/shared/models/abstract-ai-sdk.ts:210`.
  - Retry policy via `ai-retry` at `src/shared/models/abstract-ai-sdk.ts:569`.
- Custom provider compatibility layer:
  - `createCustomProviderModel(...)` switch in `src/shared/providers/utils.ts:10`.

### 2.2 ASCII architecture/data flow

```text
Session settings + global provider settings
                 |
                 v
      +-------------------------+
      | getModel(...)           |
      | providers/index.ts      |
      +-----------+-------------+
                  |
       provider id lookup
                  |
      +-----------v------------+
      | Provider registry Map  |
      | definitions/*.ts       |
      +-----------+------------+
                  |
          createModel(config)
                  |
      +-----------v-------------------------+
      | Provider model class                |
      | extends AbstractAISDKModel          |
      +-----------+-------------------------+
                  |
       streamText / tools / retries / errors
                  |
                  v
              StreamTextResult
```

### 2.3 What Cowork.ai should copy / study / skip

- Copy:
  - Registry + `defineProvider` contract pattern (`src/shared/providers/registry.ts:6`).
  - Shared abstract streaming core for all providers (`src/shared/models/abstract-ai-sdk.ts:488`).
  - Config injection object (`CreateModelConfig`) so UI/provider/runtime stay decoupled.
- Study:
  - Distinguishing OpenAI Chat Completions vs Responses provider classes (`src/shared/providers/definitions/openai.ts:5`, `src/shared/providers/definitions/openai-responses.ts:5`).
  - Tool result normalization strategy in `updateToolResultPart` (`src/shared/models/abstract-ai-sdk.ts:275`).
- Skip:
  - Side-effect import ordering for UI order (`src/shared/providers/index.ts:4`) is brittle; Cowork.ai should store order metadata separately.

---

## 3) libsql/SQLite Storage (schema design, local-first patterns)

### 3.1 How Chatbox built it (files + code patterns)

- Knowledge-base DB is local libsql file under userData:
  - `chatbox_kb.db` path in `src/main/knowledge-base/db.ts:12`.
  - Uses Mastra `LibSQLVector` in `src/main/knowledge-base/db.ts:4` and `src/main/knowledge-base/db.ts:116`.
- Important safety pattern for local sqlite file integrity:
  - Reuses the vector store's internal client (`turso`) to avoid multi-client corruption in `src/main/knowledge-base/db.ts:119`.
- Schema design:
  - `knowledge_base` table in `src/main/knowledge-base/db.ts:31`.
  - `kb_file` table in `src/main/knowledge-base/db.ts:39`.
  - Processing-state columns (`status`, `chunk_count`, `total_chunks`, `processing_started_at`) support resumable ingestion.
- Incremental migrations by additive `ALTER TABLE` with duplicate-column tolerance:
  - `document_parser`, `provider_mode`, parser flags in `src/main/knowledge-base/db.ts:79`, `src/main/knowledge-base/db.ts:93`.
- Worker-based ingestion loop:
  - Polling loop every 3s in `src/main/knowledge-base/file-loaders.ts:357`.
  - Chunking strategy recursive `size:512 overlap:50` in `src/main/knowledge-base/file-loaders.ts:102`.
  - Batch embed/upsert with `BATCH_SIZE=50` in `src/main/knowledge-base/file-loaders.ts:153`.
  - Resumes from existing `chunk_count` in `src/main/knowledge-base/file-loaders.ts:84`.
- Transaction + cleanup patterns:
  - Manual transaction wrapper in `src/main/knowledge-base/db.ts:193`.
  - Startup recovery of stuck processing files in `src/main/knowledge-base/db.ts:233`.
- IPC boundary for KB operations:
  - CRUD + upload/retry/pause/resume/search handlers in `src/main/knowledge-base/ipc-handlers.ts:16`.
- Cross-platform local-first storage beyond KB:
  - Mobile key/value sqlite via Capacitor in `src/renderer/platform/storages.ts:95` and `src/renderer/platform/storages.ts:237`.
  - Desktop mixed storage (file+IndexedDB) in `src/renderer/platform/desktop_platform.ts:96`.
  - Migration strategy and versioning in `docs/storage.md:14` and `src/renderer/stores/migration.ts:60`.
- libsql packaging hardening:
  - Runtime patch for missing native libsql module in `patches/libsql@0.5.22.patch:10`.
  - Post-pack patcher in `.erb/scripts/patch-libsql.cjs:105`.

### 3.2 ASCII architecture/data flow

```text
User uploads file
      |
      v
+----------------------+        +------------------------------+
| IPC: kb:file:upload  |------->| kb_file row (status=pending) |
| ipc-handlers.ts      |        +------------------------------+
+----------+-----------+
           |
           v (poll every 3s)
+-----------------------------+
| worker loop file-loaders.ts |
+-------------+---------------+
              |
              v
      parse -> chunk -> embedMany -> vector upsert
              |                |            |
              v                v            v
        kb_file parser_type   embeddings   index kb_<kbId>
              |
              v
      update chunk_count/status
```

```text
Storage topology by platform:

Desktop: configs/settings -> file (IPC)
         sessions/blobs   -> IndexedDB + blob store
Mobile:  key_value + image_generation -> SQLite (Capacitor)
Web:     all local -> IndexedDB/localforage
KB/RAG:  libsql file in main process (chatbox_kb.db)
```

### 3.3 What Cowork.ai should copy / study / skip

- Copy:
  - Single-client libsql access pattern to avoid sqlite corruption (`src/main/knowledge-base/db.ts:119`).
  - Resumable ingestion fields (`chunk_count`, `total_chunks`, `status`) + worker loop.
  - Additive schema migration style with graceful duplicate-column handling.
  - Transaction wrapper for multi-step delete/cleanup operations.
- Study:
  - Separate operational DB concerns (key-value/session vs RAG vector index) and where each lives.
  - Timeout/recovery handling in `checkProcessingTimeouts` (`src/main/knowledge-base/db.ts:255`).
- Skip:
  - Reaching into internal library fields (`(vectorStore as any).turso`) if a stable public API exists in Cowork.ai's libsql stack.

---

## 4) MCP Plugin System (plugin connection + tool calling)

### 4.1 How Chatbox built it (files + code patterns)

- Core orchestrator in renderer:
  - MCP server lifecycle/status in `src/renderer/packages/mcp/controller.ts:64`.
  - Supports stdio + HTTP transports in `src/renderer/packages/mcp/controller.ts:13` and `src/renderer/packages/mcp/controller.ts:34`.
- Stdio transport bridging through Electron main:
  - Renderer proxy transport in `src/renderer/packages/mcp/ipc-stdio-transport.ts:7`.
  - Main process real transport map in `src/main/mcp/ipc-stdio-transport.ts:26`.
  - Shell environment merge for command transport in `src/main/mcp/ipc-stdio-transport.ts:13`.
- Tool namespace normalization and resilience:
  - Tool names become `mcp__{server}__{tool}` in `src/renderer/packages/mcp/controller.ts:222`.
  - Tool execute errors are returned (not thrown) in `src/renderer/packages/mcp/controller.ts:205`.
- Bootstrap model:
  - Built-in MCP servers list in `src/renderer/packages/mcp/builtin.ts:12`.
  - Settings + built-ins merged at startup in `src/renderer/setup/mcp_bootstrap.ts:25`.
- Tool invocation integration with LLM calls:
  - MCP tools merged with web/kb/file tools in `src/renderer/packages/model-calls/stream-text.ts:295`.
  - Passed into `model.chat(..., { tools })` in `src/renderer/packages/model-calls/stream-text.ts:321`.
  - UI renders tool call state/results in `src/renderer/components/message-parts/ToolCallPartUI.tsx:29`.

### 4.2 ASCII architecture/data flow

```text
MCP Settings (builtin + custom)
            |
            v
 +-----------------------------+
 | mcp_bootstrap.ts            |
 | mcpController.bootstrap()   |
 +--------------+--------------+
                |
     +----------+-----------+
     |                      |
     v                      v
stdio transport        http/sse transport
(renderer proxy)       (direct MCP client)
     |                      |
     v                      v
Electron main         MCP remote endpoint
StdioClientTransport

All active toolsets -> normalize names -> merged ToolSet
                                     |
                                     v
                          model.chat(... tools ...)
                                     |
                                     v
                         tool-call/result parts in UI
```

### 4.3 What Cowork.ai should copy / study / skip

- Copy:
  - MCP controller abstraction + lifecycle/status subscriptions.
  - Stdio-over-IPC bridge pattern for Electron.
  - Normalized naming to avoid tool collision across servers.
- Study:
  - Their error strategy (returning errors as tool result) avoids full run failure but can hide systemic faults; define stricter policy by tool trust level.
  - How to persist and version server configs (`src/shared/types/mcp.ts:1`).
- Skip:
  - Silent degradation where repeated tool failures are only surfaced in result payloads, not circuit-broken.

---

## 5) Cross-Platform Strategy (Electron desktop + Capacitor mobile + web)

### 5.1 How Chatbox built it (files + code patterns)

- One renderer app, runtime-selected platform adapter:
  - Adapter selection in `src/renderer/platform/index.ts:7`.
  - Desktop if `window.electronAPI`, else web.
- Build-target flags used across runtime and build pipeline:
  - Build flags in `src/renderer/variables.ts:4`.
  - Scripts for web/mobile in `package.json:17`, `package.json:53`, `package.json:54`.
- Platform-specific implementations behind shared `Platform` interface:
  - Desktop impl in `src/renderer/platform/desktop_platform.ts:18`.
  - Web impl in `src/renderer/platform/web_platform.ts:15`.
- Mobile-specific behavior is mostly feature-flag + plugin-based:
  - iOS safe-area/keyboard integration in `src/renderer/index.tsx:52` and `src/renderer/setup/mobile_safe_area.ts:6`.
  - Mobile HTTP streaming path in `src/renderer/utils/mobile-request.ts:5` and `src/renderer/native/stream-http.ts:6`.
  - Mobile sqlite storage in `src/renderer/platform/storages.ts:237`.

Inference: there is no dedicated `MobilePlatform` class returned in current runtime selector (`src/renderer/platform/index.ts:12`). Mobile behavior appears to rely on build flags + Capacitor runtime modules rather than a first-class platform adapter object.

### 5.2 ASCII architecture/data flow

```text
                 Shared React renderer code
                            |
                +-----------+-----------+
                |                       |
        runtime: electronAPI?         no electronAPI
                |                       |
        +-------v------+         +------v------+
        | DesktopPlatform|       | WebPlatform |
        +-------+------+         +------+------+ 
                |                       |
      IPC -> main process       browser APIs/indexedDB

Build-time forks:
- web: CHATBOX_BUILD_PLATFORM=web
- mobile: CHATBOX_BUILD_TARGET=mobile_app + cap sync
- desktop: default electron package
```

### 5.3 What Cowork.ai should copy / study / skip

- Copy:
  - Single shared renderer with platform adapters for I/O boundaries.
  - Explicit build scripts per target with environment flags.
  - Mobile network and storage plugin fallbacks for WebView constraints.
- Study:
  - Whether Cowork.ai should add a dedicated `MobilePlatform` to reduce implicit mobile conditionals.
  - How much platform divergence to tolerate before splitting packages.
- Skip:
  - Ambiguous platform detection paths where mobile depends on side effects rather than one clear platform contract.

---

## 6) UI Framework (React, Tailwind, Markdown/LaTeX)

### 6.1 How Chatbox built it (files + code patterns)

- Core stack:
  - React + TanStack Router entry in `src/renderer/index.tsx:5` and route plugin config in `electron.vite.config.ts:149`.
  - State ecosystem includes Jotai/Zustand in dependencies (`package.json:205`, `package.json:282`) and app usage in `src/renderer/index.tsx:6`.
- Design system layering:
  - Tailwind tokens mapped to CSS vars in `tailwind.config.js:7`.
  - Theme CSS variables in `src/renderer/static/globals.css:5`.
  - Mixed component frameworks: Mantine and MUI used side-by-side (`src/renderer/index.tsx:2`, `package.json:119`).
- Markdown/Math/code rendering pipeline:
  - `react-markdown` + `remark-gfm` + `remark-math` + `rehype-katex` in `src/renderer/components/Markdown.tsx:14`.
  - URL sanitization in `src/renderer/components/Markdown.tsx:117`.
  - Custom LaTeX preprocessing in `src/renderer/packages/latex.ts:9`.
  - Syntax highlighting with Prism in `src/renderer/components/Markdown.tsx:15`.
  - Mermaid block detection and rendering in `src/renderer/components/Markdown.tsx:171` and `src/renderer/components/Mermaid.tsx:15`.
  - Artifact preview for renderable code blocks in `src/renderer/components/Artifact.tsx:171`.

### 6.2 ASCII architecture/data flow

```text
LLM message text
      |
      v
+-------------------------+
| processLaTeX()          |
| packages/latex.ts       |
+------------+------------+
             |
             v
+-------------------------+
| ReactMarkdown pipeline  |
| remark/rehype plugins   |
+------------+------------+
             |
   +---------+-----------+----------------+
   |                     |                |
code block          math block         links/text
   |                     |                |
   v                     v                v
SyntaxHighlighter     KaTeX          sanitized URL + render
   |
   +--> Mermaid renderer / Artifact preview
```

### 6.3 What Cowork.ai should copy / study / skip

- Copy:
  - Robust markdown pipeline with URL sanitization and math support.
  - Token-driven theming via CSS variables + Tailwind mapping.
  - Code-block UX (copy, collapse, preview, per-language treatment).
- Study:
  - Mixed Mantine+MUI increases bundle + design complexity; evaluate if Cowork.ai keeps one primary UI library.
  - Mermaid rendering safety/performance constraints for large diagrams.
- Skip:
  - Framework mixing unless there is strong migration justification.

---

## 7) Team Collaboration Features (how multi-user works)

### 7.1 How Chatbox built it (files + code patterns)

- There is no first-class real-time collaborative workspace model in this repo (no shared session backend, presence, roles, ACL, CRDT).
- Actual "team" capability implemented as shared API-key proxy setup:
  - Team sharing docs in `team-sharing/README.md:1`.
  - Caddy reverse proxy injects bearer token in `team-sharing/Caddyfile:2` and `team-sharing/Caddyfile:4`.
  - Container startup templates host/key into config in `team-sharing/main.sh:8`.
- Additional lightweight sharing feature exists for copilots (prompts), not multi-user chat collaboration:
  - `shared` toggle in copilot model/UI `src/shared/types.ts:83`, `src/renderer/routes/copilots.tsx:453`.
  - Share events posted to backend in `src/renderer/packages/remote.ts:174`.

Inference: Chatbox’s “team collaboration” is mainly shared API access economics and catalog sharing, not synchronized multi-user workflows.

### 7.2 ASCII architecture/data flow

```text
Team owner OpenAI key
        |
        v
+------------------------+
| Caddy reverse proxy    |
| adds Authorization     |
+-----------+------------+
            |
            v
     https://api.openai.com

Team members:
Chatbox API Host -> proxy URL
(no per-user API key entered)
```

```text
Copilot sharing flow:
Local copilot marked shared -> recordCopilotShare API event -> remote featured list
```

### 7.3 What Cowork.ai should copy / study / skip

- Copy:
  - Simple key-proxy deployment guide for small teams that need centralized billing control.
- Study:
  - Security/compliance risk of centralized key proxying and lack of per-user auditing.
  - Copilot/prompt sharing as a low-cost collaboration primitive.
- Skip:
  - Positioning this as true multi-user collaboration; it is not a replacement for org-grade collaboration architecture.

---

## 8) Summary: Copy / Study / Skip (table)

### 8.1 ASCII decision flow

```text
If pattern improves reliability + fits Cowork stack now -> COPY
If pattern is useful but needs adaptation/validation -> STUDY
If pattern creates long-term risk or mismatches product goals -> SKIP
```

### 8.2 Consolidated recommendations

| Area | Copy | Study | Skip |
|---|---|---|---|
| Electron + Vite Architecture | 3-process split (`main/preload/renderer`), typed preload bridge, chunked renderer build | Main IPC modularization strategy, deep-link extensibility | Monolithic IPC growth, permissive webPreferences defaults |
| LLM Provider Abstraction | Registry + `defineProvider`, shared abstract streaming core, dependency injection for model runtime | Provider-specific tool/reasoning nuances, display-name/model metadata strategy | Import-order side effects as source of truth |
| libsql/SQLite Storage | Single-client libsql pattern, resumable ingestion schema, additive migrations, transaction wrappers | Timeout/cancellation strategy for long ingestion jobs, vector index lifecycle ops | Tight coupling to internal libsql fields where avoidable |
| MCP Plugin System | Controller lifecycle, stdio-over-IPC bridge, tool-name namespacing, merged ToolSet pipeline | Tool error semantics and trust boundaries, config governance | Silent failure patterns without escalation/circuit-breaker |
| Cross-Platform Strategy | Shared renderer + adapters, build-flag target pipeline, plugin-backed mobile network/storage | Whether to formalize a dedicated MobilePlatform class | Implicit mobile behavior spread across side effects |
| UI Framework | Markdown+KaTeX+Mermaid rendering pipeline, CSS variable token system, code-block UX | Multi-library UI stack costs, renderer bundle pressure | Broad MUI+Mantine mixing unless migration-driven |
| Team Collaboration Features | Optional key-proxy pattern for cost sharing, copilot sharing as lightweight social feature | Secure audited multi-user architecture roadmap | Treating API-key sharing as enterprise collaboration |

### 8.3 Cowork.ai-specific priority

- Immediate copy candidates (high ROI, low migration cost):
  - Provider registry + abstract AI SDK execution core.
  - KB ingestion resumability schema and state-machine fields.
  - MCP stdio bridge architecture in Electron.
- High-leverage study items (design decisions needed):
  - Unified storage topology for desktop/mobile/web given Cowork.ai’s libsql-first plan.
  - Trust and safety policy for third-party MCP tools.
- Explicit skips:
  - Monolithic IPC file growth.
  - Collaboration-by-proxy as the core multi-user story.

