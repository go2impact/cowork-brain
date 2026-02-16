# AnythingLLM Technical Deep-Dive (for Cowork.ai)

## Scope and Method
This document reverse-engineers the current AnythingLLM core repo at `/Users/rustancorpuz/code/anything-llm`.

Important scope constraint:
- The desktop Electron wrapper is not in this repository. `CONTRIBUTING.md:101` explicitly states desktop code is downstream and not part of core.

---

## 1) LLM Integration

### How they built it (files + patterns)
Provider abstraction and runtime factory:
- `server/utils/helpers/index.js:131` `getLLMProvider({ provider, model })`
- `server/utils/helpers/index.js:310` `getLLMProviderClass({ provider })`
- `server/utils/helpers/index.js:429` `getBaseLLMProviderModel({ provider })`
- Pattern: very large switch-based factory returning provider instances.

Embedding engine is injected into every LLM provider:
- `server/utils/helpers/index.js:254` `getEmbeddingEngineSelection()`
- Each LLM provider constructor takes `embedder` and exposes `embedTextInput/embedChunks` as wrapper methods.
- Example: `server/utils/AiProviders/openAi/index.js:285`

Provider class interface is effectively standardized by convention:
- `streamingEnabled()`
- `constructPrompt(...)`
- `getChatCompletion(...)`
- `streamGetChatCompletion(...)`
- `handleStream(...)`
- `compressMessages(...)`
- Confirmed across many provider files under `server/utils/AiProviders/*/index.js`.

Model context-window mapping strategy:
- `server/utils/AiProviders/modelMap/index.js`
- Pulls/caches LiteLLM model map for context windows (`ContextWindowFinder.remoteUrl`) with 3-day local cache.
- Falls back to legacy map when remote cache unavailable.

Streaming implementation:
- Chat endpoints are SSE (`text/event-stream`) in `server/endpoints/chat.js:45` and `server/endpoints/embed/index.js:35`.
- Chat execution path checks provider capability:
  - `server/utils/chats/stream.js:244`
  - `server/utils/chats/apiChatHandler.js:745`
- Shared stream event writer:
  - `server/utils/helpers/chat/responses.js:222` `writeResponseChunk(response, data)` emits `data: {json}\n\n`.
- Provider-specific stream handlers map provider chunk format to common chunk events:
  - Example `server/utils/AiProviders/openAi/index.js:208`.

Prompt/token fitting:
- Compression logic in `server/utils/helpers/chat/index.js:49` (`messageArrayCompressor`).
- Not summarization-based; uses "cannonball" middle truncation (`cannonball()`), preserving role budget splits.

Model switching and override behavior:
- Workspace-level provider/model: `workspace.chatProvider/chatModel` used in chat path `server/utils/chats/stream.js:53`.
- Embed-specific model/prompt/temp override gates:
  - `server/utils/chats/embed.js:22`
  - `server/models/embedConfig.js` allow_*_override flags.

### ASCII architecture/data flow
```text
[Workspace settings + ENV]
          |
          v
getLLMProvider(provider, model)  ---> getEmbeddingEngineSelection()
          |                                   |
          v                                   v
 [Provider Instance] <------------------- [Embedder]
 (OpenAI/Gemini/Ollama/...)                
          |
          v
constructPrompt + compressMessages
          |
          +--> getChatCompletion (non-stream)
          |
          +--> streamGetChatCompletion -> handleStream -> writeResponseChunk(SSE)
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy the strict provider interface shape and capability probing (`streamingEnabled`) to keep adapter complexity predictable.
- Copy embedder injection into provider runtime so retrieval and generation share one model context.

Study:
- Study their context-window map caching approach (`modelMap/index.js`) as a practical fallback system.
- Study their SSE event contract design (`writeResponseChunk`) for UI simplicity.

Skip:
- Skip giant switch statements as-is; for Cowork.ai use a registry/plugin map with Mastra tool/provider registration.
- Skip middle-truncation-only compression as your primary approach; Vercel AI SDK v6 + libsql memory can support better recency/salience strategies.

---

## 2) RAG Pipeline (core strength)

### How they built it (files + patterns)
Primary RAG orchestration (chat + query):
- `server/utils/chats/stream.js` (interactive workspace chat)
- `server/utils/chats/apiChatHandler.js` (API-compatible flows)
- `server/utils/chats/embed.js` (embed widget flow)

RAG assembly order in chat pipeline:
1. Parse slash commands / agent intercept.
2. Load pinned docs (`DocumentManager.pinnedDocs()`).
3. Load parsed-file context (upload-to-thread temporary context).
4. Vector similarity search.
5. Backfill source window from prior chat citations (`fillSourceWindow`).
6. Compress prompt to fit context window.
7. Call LLM and stream response.

Concrete code anchors:
- Pinned docs injection: `server/utils/chats/stream.js:115`
- Parsed-files injection: `server/utils/chats/stream.js:135`
- Similarity search: `server/utils/chats/stream.js:150`
- Source backfill: `server/utils/chats/stream.js:181` + `server/utils/helpers/chat/index.js:382`
- Query-mode hallucination guard: `server/utils/chats/stream.js:198`

Document ingestion pipeline:
- Upload/process endpoints:
  - `server/endpoints/workspaces.js:113` `/upload`
  - `server/endpoints/workspaces.js:163` `/upload-link`
  - `server/endpoints/workspaces.js:206` `/update-embeddings`
  - `server/endpoints/workspaces.js:873` `/upload-and-embed`
- Parse-only for chat context:
  - `server/endpoints/workspacesParsedFiles.js:115` `/parse`
  - `server/endpoints/workspacesParsedFiles.js:65` `/embed-parsed-file/:fileId`

Collector architecture:
- Collector service endpoints at `collector/index.js` (`/process`, `/parse`, `/process-link`, `/process-raw-text`).
- Dispatch by extension via `collector/processSingleFile/index.js:74` and `collector/utils/constants.js:40`.
- Security: integrity-signing between server and collector in `server/utils/collectorApi/index.js` (`X-Integrity`, `X-Payload-Signer`).

Chunking and embedding:
- `server/utils/TextSplitter/index.js`
- Uses LangChain `RecursiveCharacterTextSplitter` with configurable chunk size/overlap.
- Adds metadata header + embedder prefix into chunk payload.
- Vector providers call:
  - `textSplitter.splitText(pageContent)`
  - `EmbedderEngine.embedChunks(textChunks)`
- Example in Lance: `server/utils/vectorDbProviders/lance/index.js:341`.

Document-to-vector persistence:
- Workspace doc row + vector rows linked via `document_vectors` (`server/models/documents.js`, `server/models/vectors`).
- Vector metadata preserves chunk `text` for retriever/citation compatibility.

### ASCII architecture/data flow
```text
User Upload/Link
   |
   v
/workspace/:slug/upload or /upload-link
   |
   v
Collector API -> collector service -> normalized document JSON
   |
   v
Document.addDocuments()
   |
   v
VectorDb.addDocumentToNamespace()
   |
   +--> TextSplitter(chunkSize, overlap, metadata header)
   +--> Embedder.embedChunks()
   +--> Persist vectors + document_vectors map

Chat Request
   |
   v
Pinned docs + parsed files + similarity search
   |
   v
fillSourceWindow(backfill from prior cited chunks)
   |
   v
compressMessages -> LLM -> stream response (with sources)
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy the multi-source context assembly pattern (pins + transient files + retrieval + backfill).
- Copy query-mode refusal when no evidence exists.

Study:
- Study `fillSourceWindow` behavior deeply; this is a pragmatic fix for follow-up query failures.
- Study parsed-files workflow (`workspace_parsed_files`) for temporary contextual docs before permanent embedding.

Skip:
- Skip tightly coupling RAG orchestration into monolithic chat handlers; split into composable pipeline stages (fits Mastra better).
- Skip file-based document JSON as the long-term truth store for desktop-first sync; prefer libsql + deterministic object storage references.

---

## 3) Vector Database (LanceDB default + alternatives)

### How they built it (files + patterns)
Abstraction:
- Base interface: `server/utils/vectorDbProviders/base.js`
- Required methods include `addDocumentToNamespace`, `performSimilaritySearch`, `delete-namespace`, `reset`, `curateSources`.

Provider selection:
- `server/utils/helpers/index.js:84` `getVectorDbClass(getExactly = null)`
- Default fallback: `lancedb`.

Default LanceDB implementation:
- `server/utils/vectorDbProviders/lance/index.js`
- Storage path: `<STORAGE_DIR>/lancedb` (`uri` getter at line 22).
- Namespace = workspace slug.
- Rerank mode uses `NativeEmbeddingReranker` (`line 8`, `line 97`).

Alternative providers supported:
- `server/utils/vectorDbProviders/{chroma,chromacloud,pinecone,qdrant,weaviate,milvus,zilliz,astra,pgvector}`
- Same method contract enables drop-in switching.

PGVector-specific strategy:
- `server/utils/vectorDbProviders/pgvector/index.js`
- Dynamically creates/validates vector extension and embedding table schema.
- Handles dimension-sensitive reset cases.

Vector cache to avoid re-embedding:
- `server/utils/files/index.js:172` `cachedVectorInformation`
- `server/utils/files/index.js:191` `storeVectorResult`
- Hash key: UUIDv5(filename).

Global reset when vector config changes:
- `server/utils/helpers/updateENV.js:1062` `handleVectorStoreReset`
- Calls `resetAllVectorStores` (`server/utils/vectorStore/resetAllVectorStores.js`) to clear caches, vector mappings, docs, namespaces.

### ASCII architecture/data flow
```text
                getVectorDbClass()
                       |
      +----------------+------------------+
      |                |                  |
   LanceDB          PGVector           Pinecone...
(default)             |                  |
      +----------------+------------------+
                       |
          VectorDatabase interface contract
                       |
      addDocumentToNamespace / performSimilaritySearch
                       |
                 Chat RAG pipeline
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy the clean provider interface contract and runtime factory.
- Copy vector-cache idea to cut repeated embed cost/latency.

Study:
- Study LanceDB namespace design for desktop/offline mode.
- Study their "reset on embedding engine/vector DB change" safety handling.

Skip:
- Skip storing provider-specific behavior inside giant provider classes over time; prefer shared primitives + thin adapters.
- Skip manual ENV-driven switching UX in app logic; centralize in a typed provider config layer.

---

## 4) Agent System (no-code builder, tools, MCP)

### How they built it (files + patterns)
Invocation model:
- Agent trigger is strict prefix-based: prompt must start with `@agent`.
- `server/models/workspaceAgentInvocation.js:7` `parseAgents()`
- Chat path intercepts and swaps transport to agent channel: `server/utils/chats/agents.js`.

Execution runtimes:
- Stateful websocket agent: `server/utils/agents/index.js` + `server/endpoints/agentWebsocket.js`
- REST-compatible ephemeral agent: `server/utils/agents/ephemeral.js` used in API chat handler.
- API route detects agent invocation and runs ephemeral event listener:
  - `server/utils/chats/apiChatHandler.js:153`

Plugin loading strategy (strong point):
- In `AgentHandler.#attachPlugins` / `EphemeralAgentHandler.#attachPlugins`:
  - Parent-child plugin: `parent#child`
  - Flow plugin: `@@flow_<uuid>`
  - MCP plugin: `@@mcp_<server>`
  - Imported hub plugin: `@@<hubId>`
- This is a practical late-binding function registry.

No-code Agent Flows:
- Backend flow CRUD + storage:
  - `server/endpoints/agentFlows.js`
  - `server/utils/agentFlows/index.js` (JSON files under `storage/plugins/agent-flows`)
- Flow execution engine:
  - `server/utils/agentFlows/executor.js`
  - Supported executable block types currently: `start`, `apiCall`, `llmInstruction`, `webScraping` (`flowTypes.js`).

No-code UI builder:
- `frontend/src/pages/Admin/AgentBuilder/*`
- Block definitions and defaults in `frontend/src/pages/Admin/AgentBuilder/BlockList/index.jsx`.
- Website/file/code blocks are present but commented/disabled.

MCP compatibility:
- Hypervisor that boots/stops MCP servers and manages transports:
  - `server/utils/MCP/hypervisor/index.js`
  - Supports `stdio`, `http`, `sse` transports.
- MCP-to-agent tool conversion:
  - `server/utils/MCP/index.js` `convertServerToolsToPlugins()`.
- Admin endpoints/UI:
  - `server/endpoints/mcpServers.js`
  - `frontend/src/models/mcpServers.js`
  - `frontend/src/pages/Admin/Agents/MCPServers/*`
- Security caveat is explicit in code comment: MCP is effectively arbitrary code execution.

### ASCII architecture/data flow
```text
User prompt: "@agent ..."
      |
      v
parseAgents() -> create invocation record
      |
      +--> WebSocket AgentHandler (stateful UI)
      |        |
      |        +--> load USER + @agent defs
      |        +--> attach plugins (skills/imports/flows/MCP tools)
      |        +--> AIbitat runtime loop
      |
      +--> EphemeralAgentHandler (REST API)
               |
               +--> same plugin model, HTTP event listener

MCP server config JSON -> MCPHypervisor boots servers -> tools listed -> converted to agent functions
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy the plugin naming and late-binding strategy (`#`, `@@flow_`, `@@mcp_`, `@@hubId`).
- Copy ephemeral vs stateful runtime split.

Study:
- Study the flow-executor variable substitution (`${var.path}`) and direct-output controls.
- Study MCP hypervisor lifecycle management and transport abstraction.

Skip:
- Skip hardcoding `@agent` prefix as the only invocation mode.
- Skip JSON-file flow persistence if Cowork.ai already uses libsql; use DB-backed versioned flow specs.

---

## 5) Electron Architecture (frontend/backend/IPC)

### How they built it (files + patterns)
Repository boundary finding:
- `CONTRIBUTING.md:101` states desktop Electron wrapper is downstream and not in this repo.
- So true `main/preload/ipcMain/ipcRenderer` implementation cannot be audited here.

What is present in core repo:
- Frontend: Vite + React app
  - `frontend/package.json`
  - `frontend/vite.config.js`
  - `frontend/src/main.jsx`
- Backend: Express API + websocket routes
  - `server/index.js`
  - SSE chat endpoints in `server/endpoints/chat.js` and `server/endpoints/embed/index.js`
  - Agent websocket endpoint `server/endpoints/agentWebsocket.js`

Observed communication patterns (core app):
- Frontend -> backend over REST + SSE (`fetchEventSource`) + WS for agent channel.
- Example: `frontend/src/models/workspace.js:150` streams `/workspace/:slug/stream-chat`.

IPC-specific findings:
- No meaningful Electron IPC symbols found in core repo (`ipcMain`, `ipcRenderer`, `contextBridge`, `BrowserWindow`).

### ASCII architecture/data flow
```text
(Unknown downstream Electron wrapper)
          |
          v
[Renderer Web App: Vite + React]
          |
          +--> REST calls (/api/...)
          +--> SSE (/workspace/:slug/stream-chat, /embed/:id/stream-chat)
          +--> WS (/agent-invocation/:uuid)
          |
          v
[Express Server in core repo]
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy their mature SSE chunk contract for token streaming and UI state transitions.

Study:
- Study route/module organization and separation of chat vs admin vs embed endpoints.

Skip:
- Skip assuming this repo reveals their Electron IPC architecture; it does not.
- Skip adopting their unknown wrapper assumptions; design Cowork’s IPC around your Mastra runtime boundaries explicitly.

---

## 6) Workspace & Multi-user (isolation, permissions, Docker vs desktop)

### How they built it (files + patterns)
Auth/mode gate:
- `server/utils/middleware/validatedRequest.js`
- Decides single-user vs multi-user mode via `SystemSettings.isMultiUserMode()`.

Role-based access:
- `server/utils/middleware/multiUserProtected.js`
- `flexUserRoleValid()` only enforces roles in multi-user mode.

Workspace isolation:
- Middleware scopes workspace access by user membership in multi-user mode:
  - `server/utils/middleware/validWorkspace.js:6`
- Data access helpers:
  - `server/models/workspace.js:269` `getWithUser`
  - `server/models/workspace.js:390` `whereWithUser`

Data model (SQLite default, workspace-centric):
- `server/prisma/schema.prisma`
- Core entities: `workspaces`, `workspace_users`, `workspace_threads`, `workspace_chats`, `workspace_documents`, `workspace_parsed_files`, `workspace_agent_invocations`, `embed_configs`, `embed_chats`.

Workspace-level model/provider overrides:
- In `workspaces` schema: `chatProvider`, `chatModel`, `agentProvider`, `agentModel`, `vectorSearchMode`, etc.

Docker vs desktop/runtime differences in core:
- Storage-path behavior controlled by `NODE_ENV` + `STORAGE_DIR` conventions across server/collector.
- MCP hypervisor custom env handling when runtime is docker:
  - `server/utils/MCP/hypervisor/index.js:255`
- Loopback URL validation differs for docker in ENV update logic:
  - `server/utils/helpers/updateENV.js:1025` onward.

### ASCII architecture/data flow
```text
Request -> validatedRequest
            |
            +--> single-user auth path
            |
            +--> multi-user auth path -> response.locals.user
                                      |
                                      v
                            flexUserRoleValid + validWorkspaceSlug
                                      |
                                      v
                         Workspace.getWithUser(user, {slug})
                                      |
                                      v
                         Workspace-scoped chats/docs/threads
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy mode-aware middleware layering (auth mode -> role gate -> workspace scoping).
- Copy explicit workspace-user join model for clean isolation.

Study:
- Study how temporary parsed files are scoped by workspace/thread/user to prevent context leakage.
- Study their runtime storage conventions if you need hybrid desktop/docker parity.

Skip:
- Skip implicit role bypasses unless intentionally desired; keep authorization explicit in Cowork.
- Skip overloading one workspace table with too many provider-specific knobs long-term; consider normalized provider config tables in libsql.

---

## 7) Embeddable Chat Widget

### How they built it (files + patterns)
Public embed API:
- `server/endpoints/embed/index.js`
- Endpoints:
  - `POST /api/embed/:embedId/stream-chat`
  - `GET /api/embed/:embedId/:sessionId`
  - `DELETE /api/embed/:embedId/:sessionId`

Request safety controls:
- `server/utils/middleware/embedMiddleware.js`
- Checks:
  - embed config exists/enabled
  - allowlist domain check (`origin`)
  - valid UUID session
  - message validity + chat mode validity
  - per-day and per-session quotas

Embed config model:
- `server/models/embedConfig.js`
- Allows admin-controlled override gates:
  - `allow_model_override`
  - `allow_temperature_override`
  - `allow_prompt_override`

Embed runtime chat flow:
- `server/utils/chats/embed.js`
- Reuses main RAG logic (vector search, source backfill, compression, stream handling).

Widget install snippet generation:
- `frontend/src/pages/GeneralSettings/ChatEmbedWidgets/EmbedConfigs/EmbedRow/CodeSnippetModal/index.jsx`
- Injects script with:
  - `data-embed-id`
  - `data-base-api-url`
  - `src=/embed/anythingllm-chat-widget.min.js`

Widget artifact in repo:
- `frontend/public/embed/anythingllm-chat-widget.min.js`
- `frontend/public/embed/anythingllm-chat-widget.min.css`
- Source appears bundled/minified here rather than maintained in readable source form in this repo.

History source-leak prevention:
- `server/models/embedChats.js:47` `filterSources()` removes `sources` before history response to clients.

### ASCII architecture/data flow
```text
Website Owner
  |
  +--> Copies script snippet (embedId + baseApiUrl)
         |
         v
Visitor Browser loads widget JS/CSS
         |
         v
POST /api/embed/:embedId/stream-chat (SSE)
         |
         +--> embed middleware checks
         |      (enabled, allowlist, quotas, session UUID)
         |
         v
streamChatWithForEmbed() -> RAG pipeline -> SSE chunks
```

### What Cowork.ai should copy / study / skip
Copy:
- Copy allowlist + quota middleware stack before any expensive LLM call.
- Copy source-redaction on embed history APIs.

Study:
- Study override gating model (per-embed policy for prompt/model/temp).
- Study their simple script-tag installation UX for non-technical users.

Skip:
- Skip shipping only minified widget source in main app repo if you want faster iteration/debugging.
- Skip embedding without strong tenant isolation and abuse controls.

---

## 8) Summary: Copy / Study / Skip

| Area | Copy | Study | Skip |
|---|---|---|---|
| LLM Integration | Provider interface standardization; embedder injection; SSE chunk contract | Context-window cache from LiteLLM map; workspace+embed override handling | Giant switch factories; truncation-only compression strategy |
| RAG Pipeline | Multi-source context assembly; query refusal on no-evidence | Source backfill (`fillSourceWindow`); parsed-files transient context model | Monolithic chat-handler orchestration; file-JSON-heavy persistence |
| Vector DB | Strong adapter contract; cached vector reuse | Lance default patterns; reset-on-config-change safety | ENV-heavy provider toggling UX; duplicated logic across providers |
| Agent System | Plugin naming/loading protocol (`#`, `@@flow_`, `@@mcp_`); ephemeral vs stateful runtimes | Flow executor variable system; MCP lifecycle/transport layer | `@agent`-prefix-only invocation model; JSON-file flow persistence for scale |
| Electron Architecture | SSE/REST/WS communication design in core | Endpoint organization and stream semantics | Any inference about their actual Electron IPC from this repo (not present) |
| Workspace & Multi-user | Mode-aware auth + role + workspace scoping middleware | Temporary context scoping (`workspace_parsed_files`); runtime portability patterns | Overloading workspace with too many provider fields; permissive bypass patterns |
| Embeddable Widget | Domain allowlist + quota middleware; source-redacted history | Override policy model and install UX | Minified-only widget source workflow |

---

## Strategic Takeaways for Cowork.ai
1. AnythingLLM’s strongest moat in this repo is practical RAG reliability engineering: context assembly, retrieval backfill, and robust operational safeguards.
2. Their architecture favors broad provider compatibility and operational pragmatism over strict modular elegance.
3. For Cowork.ai’s stack (Electron + Mastra.ai + Vercel AI SDK v6 + libsql), the best move is to copy their proven RAG/guardrail patterns, but implement them in a more modular, typed, DB-native architecture.
