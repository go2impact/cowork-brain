# Cowork.ai Sidecar — System Architecture

**v1.3 — February 17, 2026**
**Audience:** Engineering (Rustan + team)
**Purpose:** Single reference for how the entire system fits together — processes, data flow, IPC, and how features map to infrastructure.

**This doc consolidates decisions from:**
- [DESKTOP_FRAMEWORK_DECISION.md](../decisions/DESKTOP_FRAMEWORK_DECISION.md) — why Electron, multi-process model
- [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md) — what we keep/gut, directory structure, execution phases
- [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md) — why libsql
- [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](../decisions/NATIVE_ADDON_REPLACEMENT_RESEARCH.md) — why custom native addons
- [llm-architecture.md](./llm-architecture.md) — LLM stack, routing, memory, embeddings
- [product-features.md](../product/product-features.md) — six features that define the product surface

---

## System at a Glance

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Electron (Single Runtime)                        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                     Main Process                              │    │
│  │                                                               │    │
│  │   App lifecycle · System tray · IPC router                    │    │
│  │   Thermal monitor · Notification dispatch                     │    │
│  │   No heavy work — coordinator only                            │    │
│  └──────┬──────────────┬───────────────────┬────────────────────┘    │
│         │              │                   │                         │
│     IPC (built-in) IPC (built-in)     IPC (built-in)                │
│         │              │                   │                         │
│  ┌──────▼─────┐  ┌─────▼──────────┐  ┌────▼───────────────────┐    │
│  │  Capture   │  │  Agents & RAG  │  │  Renderer              │    │
│  │  Utility   │  │  Utility       │  │  (Chromium, sandboxed)  │    │
│  │  Process   │  │  Process       │  │                         │    │
│  │            │  │                │  │  React + Vite + Tailwind│    │
│  │  Native    │  │  Mastra.ai     │  │  M3 Design System      │    │
│  │  addons    │  │  Ollama client │  │                         │    │
│  │  libsql    │  │  Gemini direct │  │  contextIsolation: true │    │
│  │  (sync)    │  │  OpenRouter    │  │  nodeIntegration: false │    │
│  │            │  │  MCP SDK       │  │  sandbox: true          │    │
│  │            │  │  libsql (async)│  │                         │    │
│  └────────────┘  └───────┬────────┘  └─────────────────────────┘    │
│                          │                                           │
│                   ┌──────▼──────────┐                                │
│                   │  Playwright     │                                │
│                   │  Child Process  │                                │
│                   │                 │                                │
│                   │  Browser auto-  │                                │
│                   │  mation, fully  │                                │
│                   │  isolated       │                                │
│                   └─────────────────┘                                │
│                                                                      │
│                   ┌─────────────────┐                                │
│                   │  cowork.db      │ ← Single libsql file          │
│                   │  (WAL mode)     │   Capture + Mastra + vectors  │
│                   └─────────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Process Model

Five processes. Each has a single responsibility. If one crashes, the others survive.

| Process | Responsibility | Runs | Crash behavior |
|---|---|---|---|
| **Main** | App lifecycle, system tray, IPC routing between all other processes. Thermal monitoring (`ProcessInfo.thermalState`). Notification dispatch to macOS. | Always | App-fatal (but does no heavy work, so shouldn't crash) |
| **Capture Utility** | Runs native capture addons (`coworkai-keystroke-capture`, `coworkai-activity-capture`). Buffers events. Flushes to libsql. | Always | Auto-restarts via Electron. UI and agents unaffected. |
| **Agents & RAG Utility** | Mastra.ai agent orchestration. Embedding generation (Qwen3-Embedding-0.6B via Ollama). Vector search. MCP server connections. Complexity router. Cloud inference (Gemini direct + OpenRouter). | Always | Auto-restarts via Electron. Capture and UI unaffected. |
| **Renderer** | All UI. Fully sandboxed — no Node.js access. Communicates exclusively via IPC. | Always | Can reload without affecting capture or agents. |
| **Playwright Child** | Browser automation for MCP Browser feature. Spawns its own Chromium instances. | On demand | Kill and respawn. No impact on anything else. |

> **Mastra in utility process is the target architecture, not yet proven.** Each component works individually (aime-chat proves Mastra+Electron; @electron/llm proves LLM streaming in utility processes), but no one has run the full Mastra stack in an Electron utility process. A proof-of-concept spike is required before committing. Fallback options: main process (accept crash risk) or separate server process (HTTP overhead). See [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md).

**Why multi-process:** Native C++ addons can segfault. Embedding generation is CPU-heavy. Playwright can hang. None of these should freeze the UI or kill the app. Electron's built-in IPC connects all processes — no custom serialization protocol, no HTTP hop.

### Playwright Execution Model

v0.1 uses a persistent Playwright context — login state (cookies, localStorage) survives across sessions. Essential for operating inside authenticated apps (Zendesk, Gmail, Slack) without re-authentication each launch.

| Concern | How it works |
|---|---|
| **Command flow** | Agent sends browser commands via IPC → Playwright child executes → results back via IPC |
| **Process boundary** | Every browser action crosses a process boundary (~1-2ms latency). Trade-off: crash isolation — a hung page can't freeze the agent. |
| **Persistent context** | Playwright launches with persistent user data directory so login state survives restarts |
| **Crash recovery** | If Playwright child hangs or crashes, Agents & RAG utility detects broken pipe, kills the child, and respawns. No impact on capture or UI. |

**v0.2 evaluation candidates:**
- **Stagehand** (`@browserbasehq/stagehand`) — LLM-driven automation without selectors. Aligns with "coachable" MCP Browser vision. Requires vision model or DOM analysis. Not v0.1.
- **CDP remote** — attach to user's actual Chrome (logged-in sessions without re-auth). Powerful but privacy-sensitive. Requires explicit consent and visible indicator ("AI is connected to your browser").

### Boot Sequence

Three processes boot in parallel. Main waits for both utilities to signal ready before creating the renderer.

**Main process:**
1. AppManager (lifecycle, settings)
2. ThermalManager (hardware monitoring)
3. Spawn Capture Utility + Agents & RAG Utility
4. Wait for both utility processes to signal ready
5. Create BrowserWindow (renderer)

**Capture Utility:**
1. NativeAddonManager (load `coworkai-keystroke-capture`, `coworkai-activity-capture`)
2. CaptureBufferManager (activity buffer, keystroke chunker)
3. DBManager (libsql sync connection to `cowork.db`)
4. Signal ready to main

**Agents & RAG Utility:**
1. DBManager (libsql async connection to `cowork.db`)
2. ProviderRegistry (Ollama + Gemini + OpenRouter via `createProviderRegistry()`)
3. MastraManager (agent orchestration, memory)
4. MCPManager (service connections, OAuth credential loading)
5. EmbeddingManager (queue, Qwen3-0.6B via Ollama)
6. Signal ready to main

Each step within a process is sequential — later steps depend on earlier ones. The three processes are independent and boot concurrently.

---

## Data Flow

Everything flows through one database file (`cowork.db`). Two processes write to it; the renderer reads via IPC.

### Capture → Storage → Embeddings → Agents → UI

```
User's desktop activity
        │
        ▼
┌─────────────────────────────────┐
│     Capture Utility Process     │
│                                 │
│  coworkai-activity-capture      │──→ Active app, window title, browser URL (ON by default)
│  coworkai-keystroke-capture     │──→ Keystroke patterns (OFF, opt-in), clipboard (OFF, opt-in)
│                                 │
│  Activity buffer (5-slot queue) │──→ Bounded memory, auto-flush
│  Keystroke chunker (debounce)   │──→ 1000-char flush, special-key triggers
│                                 │
│  libsql (sync writes, WAL)     │──→ Writes to cowork.db
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│          cowork.db              │  Single file, WAL mode
│                                 │
│  App tables (capture data)      │  ← Capture process writes
│  LibSQLStore (agent memory)     │  ← Agents process reads/writes
│  LibSQLVector (embeddings)      │  ← Agents process reads/writes
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│    Agents & RAG Utility Process │
│                                 │
│  Embedding pipeline             │──→ Raw data → Qwen3-Embedding-0.6B → vectors
│  Mastra LibSQLVector            │──→ Stores/queries embeddings in cowork.db
│  Mastra LibSQLStore             │──→ Agent memory, threads, traces
│                                 │
│  Complexity router              │──→ Routes to local or cloud brain
│  Local brain (Ollama)           │──→ DeepSeek-R1-8B (GPU)
│  Cloud brain (Gemini direct)    │──→ Gemini 2.5 Flash (free tier)
│  Cloud brain (OpenRouter)       │──→ Gemini 3 Flash / Claude (paid tiers)
│                                 │
│  MCP SDK                        │──→ Zendesk, Gmail, Slack, etc.
│  Playwright driver              │──→ Sends commands to Playwright child
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│     Main Process (IPC router)   │
│                                 │
│  Routes agent responses → UI    │
│  Routes UI requests → agents    │
│  Dispatches macOS notifications │
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│     Renderer (sandboxed)        │
│                                 │
│  Chat, Apps, MCP Browser,       │
│  Automations, Context, Settings │
│                                 │
│  All data arrives via IPC.      │
│  Zero direct DB or Node access. │
└─────────────────────────────────┘
```

### Database Access Pattern

| Process | Package | Access | Pattern |
|---|---|---|---|
| Capture Utility | `libsql` (sync) | Read/Write | High-frequency writes (keystroke events, window changes). WAL mode. |
| Agents & RAG Utility | `@mastra/libsql` (async) | Read/Write | Reads capture data for context. Writes agent memory, embeddings. Moderate frequency. |
| Renderer | None (via IPC) | Read only | Requests data through main process IPC. Never touches DB directly. |

WAL mode allows concurrent readers + one writer. The capture process writes frequently (sync). Mastra reads frequently and writes occasionally (async). This is the ideal WAL access pattern.

---

## IPC Contract

All inter-process communication uses Electron's built-in `ipcMain` / `ipcRenderer` / `MessagePort`. No HTTP, no WebSocket, no custom protocol.

### Channel Architecture

```
Renderer ←──IPC──→ Main Process ←──IPC──→ Capture Utility
                                ←──IPC──→ Agents & RAG Utility
                                          ←──child_process──→ Playwright
```

**Direction of data flow:**

| From | To | What flows |
|---|---|---|
| Capture → Main | Capture events (buffered), status updates |
| Main → Agents | Context queries, chat messages, automation triggers |
| Agents → Main | Agent responses, action results, notification triggers |
| Main → Renderer | UI updates, chat responses, execution logs |
| Renderer → Main | User input, chat messages, settings changes, approval decisions |
| Agents → Playwright | Browser automation commands |
| Playwright → Agents | Page state, action results |

**The renderer never talks directly to capture or agents.** Everything routes through main. This keeps the sandboxed renderer clean and gives main a single point for logging, rate-limiting, and access control.

### Streaming Protocol

Chat responses stream from the Agents & RAG utility through main to the renderer:

```
Agents & RAG Utility                Main Process              Renderer
    │                                   │                        │
    │  SSE-formatted JSON chunks        │                        │
    │  via MessagePort                  │                        │
    ├──────────────────────────────────►│                        │
    │                                   │  Passthrough relay     │
    │                                   │  (zero logic added)    │
    │                                   ├───────────────────────►│
    │                                   │                        │
    │                                   │            useChat() via IpcChatTransport
    │                                   │            renders chunks as they arrive
```

**Chunk encoding:** `"data: ${JSON.stringify(chunk)}\n\n"` — SSE-formatted JSON. Same encoding as AI SDK's standard streaming protocol.

**Chunk validation:** Every chunk is validated against a Zod schema (`uiMessageChunkSchema`) at the renderer boundary. Malformed chunks fail loudly instead of corrupting UI state.

**Error recovery:** If the Agents & RAG utility crashes mid-stream, main detects the broken MessagePort and sends an `error` chunk to the renderer. The renderer surfaces this to the user ("Agent disconnected — retrying...") rather than silently hanging.

### IPC Handler Pattern (BaseManager + @channel)

All IPC handlers follow a decorator pattern adapted from AIME Chat. Managers extend `BaseManager`; methods decorated with `@channel('namespace:method')` auto-register as IPC handlers in the constructor.

```
@channel('chat:sendMessage')
async sendMessage(threadId, content) { ... }
// → auto-registers as IPC handler for 'chat:sendMessage'
```

Where handlers register depends on the process:

| Process | Registration target | Example channels |
|---|---|---|
| **Main** | `ipcMain.handle()` | `app:getSettings`, `app:setTheme` |
| **Capture Utility** | `parentPort` / `MessagePort` | `capture:getStatus`, `capture:toggleStream` |
| **Agents & RAG Utility** | `parentPort` / `MessagePort` | `chat:sendMessage`, `mcp:connect`, `context:query` |

The renderer calls `window.cowork.namespace.method()` → preload routes through `ipcRenderer.invoke()` → main either handles directly or forwards to the appropriate utility process.

### IPC Observability

**tracedInvoke** — Wrapper around IPC that propagates OpenTelemetry span context across all 4 processes. The trace payload is appended as the last IPC argument and stripped before the handler sees it — business logic never knows about tracing. Backward compatible: calls without trace context work identically.

**4-process waterfall** — A single user chat request produces a trace: `renderer.send` → `main.relay` → `agents.process` → `playwright.execute` (if browser action). Shows exactly where latency lives. Main process adds a relay span (`ipc.relay`) showing the routing hop.

**IpcChannel typed constants** — All channel names defined in `src/shared/ipc-channels.ts`. Single source of truth imported by all processes. Prevents typos and enables refactoring with type safety.

**Trace storage** — Spans stored in `cowork.db` (extends Mastra's `LibSQLStore` traces, same database). Used for: latency debugging across four processes, safety rail audit trails (every tool call logged with timing), and MCP Browser execution log (action timeline).

### Preload Namespace Design

The preload script exposes `window.cowork.*` with scoped namespaces:

| Namespace | Routes to | Purpose |
|---|---|---|
| `cowork.app` | Main | Settings, theme, lifecycle |
| `cowork.chat` | Main → Agents & RAG | Chat messages, agent responses |
| `cowork.mcp` | Main → Agents & RAG | MCP connections, tool calls |
| `cowork.browser` | Main → Agents & RAG → Playwright | Browser automation commands |
| `cowork.capture` | Main → Capture | Capture status, stream toggles |
| `cowork.context` | Main → Agents & RAG | Context Card data, activity queries |

---

## Native Capture Layer

Two custom C++ N-API addons. Battle-tested in production. No viable open-source replacements (see [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](../decisions/NATIVE_ADDON_REPLACEMENT_RESEARCH.md)).

### `coworkai-keystroke-capture`

Global keyboard and mouse event capture.

| Capability | Detail |
|---|---|
| Character mapping | Full UTF-8 via `CGEventKeyboardGetUnicodeString` (macOS) / `ToUnicodeEx` (Windows) |
| Key repeat detection | OS-provided (macOS) + atomic tracking (Windows) |
| CapsLock state | Exposed as boolean in every event |
| Electron safety | Graceful Accessibility permission handling, event tap auto-recovery |
| Threading | `Napi::ThreadSafeFunction` — events dispatched to JS without blocking native thread |

### `coworkai-activity-capture`

Active window detection, app identification, browser URL extraction.

| Capability | Detail |
|---|---|
| macOS app detection | 3-tier fallback: `NSWorkspace` → `runningApplications` → `CGWindowListCopyWindowInfo` |
| Browser URL (macOS) | AppleScript + Accessibility API fallback via `AXUIElementRef` recursive traversal |
| Browser URL (Windows) | Full COM UI Automation — 28 localized address bar names, supports Chrome/Edge/Firefox/Brave/Opera |
| Deep Chromium traversal | Recursive DFS through Accessibility tree for `AXWebArea` roles |
| Architecture | In-process N-API addon, sub-millisecond. Rebuilt against Electron ABI. |

### Capture Orchestration (from `coworkai-agent`)

Reused from the existing codebase — proven buffer management and event chunking:

| Component | What it does |
|---|---|
| Activity buffer | Bounded 5-slot queue with `autoFlushIfNeeded()`. Prevents unbounded memory growth during rapid window switching. |
| Keystroke chunker | Debounce-driven flushing + special-key triggers + 1000-char activity flush. Tuned for real-world typing patterns. |
| Clipboard capture | Hotkey detection (copy/cut/paste) in the keystroke stream → `readClipboard()` on hotkey. More efficient than polling. |

---

## Database Layer

**libsql for everything.** One SDK, one `.db` file, built-in vector search. See [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md).

```
┌──────────────────────────────────────────────────────────┐
│                      cowork.db                            │
│                                                           │
│  App Tables (capture data)                                │
│  ├── Activities (window sessions, app focus)              │
│  ├── Input events (keystroke patterns)                    │
│  └── Context streams (clipboard, browser sessions)        │
│                                                           │
│  LibSQLStore (Mastra structured data)                     │
│  ├── Conversation history (recent messages)               │
│  ├── Working memory (user profile, preferences)           │
│  ├── Observational memory (compressed summaries)          │
│  └── Agent state (workflow snapshots, traces)             │
│                                                           │
│  LibSQLVector (Mastra embeddings)                         │
│  ├── Semantic recall (embedded past conversations)        │
│  ├── Embedded activity data (window titles, URLs)         │
│  └── Embedded observational summaries                     │
└──────────────────────────────────────────────────────────┘
```

**Why libsql over better-sqlite3 + sqlite-vec:**
- Native Mastra integration (`LibSQLStore` + `LibSQLVector`) — zero adapter code
- Built-in vector search (`F32_BLOB` + `vector_distance_cos`) — no `.dylib` extension packaging
- One native module instead of two — one ABI rebuild, one ASAR unpack
- Proven in production Electron apps (Beekeeper Studio, Outerbase Studio Desktop)

**Schema is not finalized.** The new schema will be designed from [product-features.md](../product/product-features.md) — specifically the six input streams (Context), four memory layers, and Mastra agent state requirements. The old tracking-oriented schema does not carry over.

### Database Hardening

**Single-client access pattern** — Each process should use exactly one `@libsql/client` Client instance for all DB operations to prevent intra-process `SQLITE_BUSY` errors from multiple clients competing for write access. In the Agents & RAG utility process, the ideal approach is to create one client via the native libsql TypeScript SDK and inject it into both `LibSQLVector` and `LibSQLStore` — depends on whether Mastra accepts an external client (needs investigation). The Capture Utility process creates its own separate client (sync API for high-frequency writes). Cross-process write contention between Capture and Agents is handled by WAL's busy timeout.

**Vector index namespacing** — Separate `LibSQLVector` indexes per data type:

| Index | What it stores |
|---|---|
| `embed_activity` | Embedded window titles, URLs, app sessions |
| `embed_conversation` | Embedded past conversations |
| `embed_observation` | Embedded observational memory summaries |

Separate indexes enable per-type queries (e.g., "find similar activity") or merged cross-type queries with different relevance weights. Same `LibSQLVector.createIndex()` API per index.

**Schema migration strategy** — Additive `ALTER TABLE ADD COLUMN` with duplicate-column tolerance (try/catch, ignore "column already exists"). No version-tracked migration framework for v0.1. Per-process schema ownership: the Capture Utility process creates and migrates app tables (activities, input events, context streams); the Agents & RAG Utility process creates and migrates its own tables (embedding queue, agent operations, automations, mcp_servers, mcp_install_progress, tool_policies). Mastra manages `LibSQLStore` and `LibSQLVector` tables internally. Neither process ALTERs tables owned by the other.

---

## LLM Stack

Two brains. A local one (free, instant) and a cloud one (smarter, metered). User chooses at onboarding.

### Local Brain

| | |
|---|---|
| Model | DeepSeek-R1-Distill-Qwen-8B (4-bit GGUF) |
| Runtime | Ollama (GPU via Metal) |
| RAM | ~5.5 GB |
| Handles | ~70% of daily work: summaries, drafts, classification, simple Q&A |
| Can't do | >8K token context, vision, complex multi-step planning |

### Cloud Brain

All three providers (Ollama above + two cloud providers below) managed via AI SDK's `createProviderRegistry()`:

| Provider | Adapter | Role |
|---|---|---|
| **Gemini direct** | `@ai-sdk/google` | Free tier + development default — Gemini API key only, no OpenRouter middleman |
| **OpenRouter** | `@ai-sdk/openai-compatible` | Paid tiers — single gateway to multiple providers |

| Tier | Model | Provider |
|---|---|---|
| Free | Gemini 2.5 Flash | Gemini direct |
| Boost | Gemini 3 Flash | OpenRouter |
| Pro | Claude Sonnet 4.5 | OpenRouter |
| Max | Claude Opus 4.6 | OpenRouter |

### Complexity Router

Rule-based, <10ms, no ML. Lives in the Agents & RAG utility process.

```
Request arrives
    │
    Step 1: ROUTE
    ├── User prefers local + task is simple (<500 tokens, no attachments, no tools)
    │   → DeepSeek-R1 via Ollama (free)
    │
    ├── User prefers local + task is complex (>8K tokens, attachments, multi-tool)
    │   → Cloud (notify user: "This one needed cloud")
    │
    ├── User prefers cloud
    │   → Cloud always
    │
    └── "Try harder" on previous response
        → Next tier up (show cost estimate)
    │
    Step 1b: RESOLVE CLOUD PROVIDER
    ├── Free tier → Gemini direct (Gemini 2.5 Flash via @ai-sdk/google)
    └── Paid tier → OpenRouter (model by tier)
    │
    Step 2: BUDGET CHECK
    ├── Free user: remaining monthly quota? (~625K tokens/month on Gemini 2.5 Flash via Gemini direct)
    ├── Paid user: within spend caps? (per-interaction $0.50, daily $10, weekly $50, monthly $200)
    ├── Yes → proceed
    └── No → friendly message + upgrade prompt
    │
    Step 3: RESULT EVALUATION (post-response)
    └── "Retry with smarter model? (~$X)" button on every response
```

**askId grouping for "retry with smarter model"** — When the user clicks "Retry with smarter model", the new response shares `askId` with the original. The UI groups both responses for comparison (stacked: local response above, cloud response below). Sequential execution (local first, cloud on request), not parallel. Each retry shows cost estimate before execution. The `askId` group tracks cumulative cost for the entire retry chain.

### Embeddings

| | |
|---|---|
| Model | Qwen3-Embedding-0.6B via Ollama (~500MB RAM) |
| Storage | libsql built-in vector search (same `cowork.db`) |
| Dimensions | 512–1024 |
| Distance | Cosine similarity |
| Fallback | Keyword search against SQLite if embeddings can't run locally |

### Audio (The Ears)

| | |
|---|---|
| Model | Whisper Turbo (Core ML → Apple Neural Engine) |
| RAM | ~1.5 GB |
| Strategy | Keep warm on 16GB+ Macs (eliminates 2s cold start). Load on demand for 8GB. |
| Rule | Audio never leaves the device. Whisper is always local. |

**Edge case:** If hardware can't run Whisper at all, cloud transcription is allowed only with explicit consent ("Audio will be sent to a cloud service. Your recordings will leave this device."). User must actively enable — never auto-enabled. Cloud-transcribed content is flagged in the activity store. See [llm-architecture.md](./llm-architecture.md#audio-never-leaves-the-device).

**Hardware split on Apple Silicon:** Whisper on ANE and DeepSeek on GPU are separate silicon — they don't compete. User can be on a call (Whisper transcribing on ANE) while the agent drafts a response (DeepSeek generating on GPU).

### Model Lifecycle

Before loading a local model, the system runs a preflight check to prevent OOM crashes. Download integrity ensures model files aren't corrupted. Preflight checks (lightweight metadata reads) run in Main. Download + SHA256 hash validation (CPU/IO-heavy) run in the Agents & RAG utility process to respect Main's "no heavy work" constraint.

**Preflight fit check** — RED/YELLOW/GREEN classification before model load:

```
Inputs:
  total_required = model_size + kv_cache_size
  usable_memory  = total_RAM - RESERVE_BYTES

  RESERVE_BYTES ≈ 3 GB (OS baseline + our multi-process footprint, see Hardware Requirements)

Classification (Apple Silicon unified memory):
  RED    = total_required > usable_memory     → disable local brain, cloud-only
  YELLOW = total_required > 80% usable_memory → warn: "Local brain may be slow"
  GREEN  = total_required ≤ 80% usable_memory → enable local brain
```

**KV cache estimation** — Uses GGUF metadata (layer count, KV head count, key/value dimensions, context length) or Ollama API model info. Checks against our actual context budget (set by the complexity router), not the model's max context — this produces a smaller, more accurate estimate.

**Download integrity** — Resumable downloads (`.tmp` + `.url` sidecar files), SHA256 validation (size-first O(1) check, hash-second O(n) check), per-task cancellation via `AbortController`. Applies to direct GGUF imports; Ollama manages its own downloads for `ollama pull`.

**Path safety** — All download paths normalized via `path.resolve()` + prefix check against the Cowork data directory. Prevents path traversal from malicious manifests.

---

## Memory System

Four layers, all on-device. Both activity capture data and conversation data feed the same pipeline.

```
Activity Data (capture)              Conversation Data (chat)
        │                                      │
        ▼                                      ▼
┌──────────────────────────────────────────────────┐
│            Local Storage (cowork.db)              │
└────────────────────┬─────────────────────────────┘
                     │
     ┌───────────────┼──────────────┬───────────────┐
     ▼               ▼              ▼               ▼
Conversation    Semantic        Observational    Working
History         Recall          Memory           Memory
     │               │              │               │
     ▼               ▼              ▼               ▼
Recent N msgs   Embedded via    Background       Structured user
in context      Qwen3-0.6B →   agents compress  profile updated
for short-term  vector store,  raw history →    from conversation
continuity      RAG retrieval  dense logs       + activity signals
```

| Layer | What it stores | Lifespan | Retrieval |
|---|---|---|---|
| **Conversation History** | Recent messages from current chat thread | Current session | Direct — last N messages loaded into context |
| **Working Memory** | User profile, preferences, goals, communication style | Persistent (until user edits/deletes) | Direct — always loaded into context |
| **Semantic Recall** | Past conversations + captured activity, embedded as vectors | Long-term | RAG — query embedded, vector similarity search |
| **Observational Memory** | Dense compressed logs from raw history | Long-term | Background compression; retrievable via semantic recall |

**RAG retrieval flow:** Query arrives → embedded via Qwen3-0.6B → vector similarity search against `cowork.db` → context assembled with priority: conversation history (highest) → working memory → semantic recall → observational summaries (lowest) → fed to model alongside query.

### Mastra Memory Configuration

The four layers map to a single Mastra Memory config object:

| Layer | Mastra primitive | Setting | What it provides |
|---|---|---|---|
| **Conversation History** | `lastMessages` | Enabled (configurable N) | Recent messages loaded into context for short-term continuity |
| **Working Memory** | `workingMemory` | `{ enabled: true }` | Persistent user profile, preferences, communication style — always in context |
| **Semantic Recall** | `semanticRecall` | `{ enabled: true }` | RAG retrieval from embedded conversations + capture data via LibSQLVector |
| **Observational Memory** | `observationalMemory` | `true` | Background two-stage compression (Observer → observations, Reflector → reflections). 5-40x compression ratio. |

Storage backend: `LibSQLStore` + `LibSQLVector` from `@mastra/libsql`, both pointing at `cowork.db`.

**Open question — Observer model:** Observational Memory's Observer/Reflector defaults to Gemini 2.5 Flash (1M context helps). Can DeepSeek-R1-8B serve as Observer locally, or must this go through cloud? Needs benchmarking. Mastra docs warn that Claude 4.5 models perform poorly as Observer — model choice matters here.

---

## Embedding Pipeline

Raw data from the capture process and conversation history must be embedded before it's useful for RAG retrieval. The embedding pipeline runs in the Agents & RAG utility process, processing a continuous stream of data with full crash resilience.

### Resumable Ingestion State Machine

Each embedding job tracks its state explicitly:

```
pending → processing → done
              │
              ├──→ paused (crash recovery or thermal shutdown) → pending (auto-resume)
              │
              └──→ failed (timeout > 5 min, terminal)
```

Schema fields per embedding job: `embed_status`, `chunks_embedded` (progress checkpoint), `total_chunks`, `embed_started_at` (timeout detection), `embed_error`.

### Batch Processing with Checkpoints

Embeddings are generated in batches via `embedMany()` (Qwen3-Embedding-0.6B through Ollama). After each batch, `chunks_embedded` is persisted to libsql. If the app crashes after batch 3 of 10, it resumes from chunk 150 — not chunk 0.

### Startup Crash Recovery

When the Agents & RAG utility process starts (or auto-restarts after a crash):

1. Move any jobs stuck in `processing` → `paused`
2. Check `embed_started_at` — mark as `failed` if older than 5 minutes (hung job detection)
3. `paused` jobs re-enter the queue as `pending` for automatic retry

### Continuous Capture Trigger

Unlike file-upload apps, our pipeline triggers when the Capture Utility process flushes data to `cowork.db`. The Agents & RAG utility process polls for new `pending` rows. No user-initiated upload, no file parsing — capture data is already text.

---

## RAG Context Assembly

When the agent needs grounded context for a response, the RAG pipeline assembles material from multiple sources in priority order. This section describes the retrieval sub-pipeline within Mastra's `semanticRecall` layer — how grounded material is fetched and ranked. The overall context window composition (conversation history + working memory + semantic recall + observational memory) is handled by Mastra's Memory system (see [Memory System](#memory-system)).

### Multi-Source Priority Order

```
1. Observation anchors    → user-pinned captures, always included first
2. Recent capture buffer  → latest capture data not yet embedded (transient)
3. Vector similarity search → standard RAG retrieval across embed_* indexes
4. History backfill       → fillSourceWindow (prior-cited chunks, see below)
5. Compress to fit        → recency-weighted truncation to stay within token budget
```

### fillSourceWindow Backfill

Solves the follow-up query problem: user asks "tell me more about that ticket" but vector search returns nothing because the follow-up lacks original keywords.

When vector search returns fewer results than needed, the pipeline walks conversation history in reverse (most recent first), extracts previously cited chunk IDs from each response's `citations` metadata, and re-injects those chunks into context. Deduplication by chunk ID prevents the same chunk appearing twice. Pinned observation anchors are excluded from backfill (already injected). Window cap (`nDocs` default of 4) prevents backfill from overwhelming fresh search results. Only a limited number of recent messages are scanned (not the entire conversation).

### Query Refusal

Two-checkpoint pattern prevents hallucination in grounded-query scenarios:

**Checkpoint 1 — No embeddings exist at all:** If the relevant embedding indexes are empty, return a deterministic refusal message (no LLM call, saves tokens and latency).

**Checkpoint 2 — Embeddings exist but context assembly returned nothing:** If the full pipeline (anchors + buffer + vector search + backfill) produces zero context, return a refusal message.

Refusal messages must be visible to the user but excluded from future context windows (prevents "I don't know" answers from polluting subsequent queries). The exclusion mechanism depends on whether Mastra's `lastMessages` supports metadata filtering — see Open Architecture Questions #9.

Applies to agents configured with grounded-query mode (e.g., capture summary agent). General chat agents skip refusal checkpoints and can respond using conversation history alone.

---

## Agent Orchestration

**Mastra.ai** runs in the Agents & RAG utility process. Embedded directly in Electron (not client-server) — Electron 37.1.0 ships Node 22.16.0, satisfying `@mastra/libsql`'s requirement.

### Agent Execution Model

Agent executions are tracked as **operations** — persistent execution units stored in the libsql `agent_operations` table. An operation captures the full state of an agent execution: status, step count, accumulated cost, and any tools awaiting human approval. Operations survive crashes (improvement over reference apps that store in-memory).

**Operation status:** `idle` → `running` → `done` or `error`. May transition to `waiting_for_human` when a tool requires approval, then resume on user decision.

**Instruction executor** — The agent loop separates decisions from side effects. The agent's runner function returns instructions; executors handle side effects:

| Instruction | Purpose |
|---|---|
| `call_llm` | Send messages to LLM, get response |
| `call_tool` | Execute single MCP tool |
| `call_tools_batch` | Execute multiple tools in parallel |
| `request_human_approve` | Pause for user approval of tool call |
| `finish` | End execution normally |

**Max-step safety limit** — Step counter per operation, incremented on each execution step. Prevents infinite tool-call loops (LLM calls tool → tool output triggers another tool → repeat). Complements the existing token budget safety from the complexity router.

**Step scheduling** — In-process event loop (`setImmediate`), not HTTP queue. Operations paused by `waiting_for_human` resume via IPC from the renderer when the user approves or rejects.

### Execution Paths

Agents use two execution paths, often in the same task:

**Path 1: Agent → Tools → MCP (background)**
- Fast, invisible. Agent calls tools that connect to services via MCP.
- For: data retrieval, bulk operations, API endpoints.
- Example: reading 50 Zendesk tickets, categorizing emails.

**Path 2: Agent → Tools → Browser (visible)**
- Agent drives Playwright for visible, coachable actions.
- For: complex forms, visual verification, apps without full API support.
- Example: composing a nuanced reply in Zendesk's actual UI.

**Combined example (Zendesk ticket reply):**
1. MCP — read ticket + customer history (background)
2. MCP — draft reply (background)
3. User reviews draft
4. Browser — open ticket in Zendesk UI, paste reply (visible)
5. User coaches ("add refund timeline note")
6. Browser — click Send (visible)
7. MCP — log to audit trail (background)

### Sub-Agent Delegation

Context isolation via disposable sub-agents. When the platform agent needs to process large volumes of data, it delegates to a sub-agent rather than loading everything into its own context.

**Pattern:** Parent agent delegates heavy work → sub-agent runs in its own context → sub-agent returns summary → raw data never pollutes parent context.

**Example:** Agent needs to find a pattern across 50 Zendesk tickets. Instead of loading all 50 tickets into its context (blowing the context window), it spawns a ticket-analysis sub-agent. The sub-agent reads tickets, identifies the pattern, returns a 200-token summary. Parent continues with the summary.

Key properties:
- **Stateless spawning** — each sub-agent invocation is independent. No follow-up messages, no memory of previous calls.
- **Dynamic description** — available sub-agents are auto-described to the parent. Adding a new specialist only requires registration.
- **Work-oriented specialists** — not code-oriented (like AIME's Explore/Plan). Specialists emerge from actual usage patterns: ticket analysis, email batch processing, context summarization.
- **Same process** — sub-agents run inside the Agents & RAG utility process. No extra IPC hop for delegation.

### MCP Connections

Services connect via MCP servers. The Agents & RAG utility process manages all connections.

| Concern | How it works |
|---|---|
| Connection management | OAuth flows, health monitoring, auth expiry detection |
| Tool registry | Each service exposes tools (list_tickets, send_email, etc.) |
| Scoped access | Apps declare what MCP tools they need. Platform grants per-app. |
| Disconnect handling | Action pauses, user notified with exact blocker + reconnect action |

**MCPClient status lifecycle:**

```
starting ──→ running ──→ stopped (user toggled off)
                │
                └──→ error (connection lost, auth expired)
                       │
                       └──→ starting (reconnect attempt)
```

| Status | UX |
|---|---|
| `starting` | "Connecting to Zendesk..." |
| `running` | Green dot, tools available |
| `stopped` | Grey dot, user toggled off |
| `error` | Red dot + error message + reconnect action |

**OAuth flow:**
- System browser opens for login (renderer can't handle OAuth popups — sandboxed)
- PKCE flow via `OAuthClientProvider` from `@modelcontextprotocol/sdk`
- Credentials stored on-device: `{userData}/.mcp/{serverUrlHash}_tokens.json`
- Token refresh handled automatically; expiry detection triggers "reconnect" prompt in UI
- Main process opens browser and captures OAuth callback, forwards tokens to Agents & RAG utility

**MCP orphan cleanup** — Each stdio MCP server gets a lockfile (`{cowork_data_dir}/mcp_lock_{serverNameHash}.json` with child PID). On app startup, Main process runs a sweep before Agents boots: glob lockfiles → check process alive (signal 0) → kill stale process (SIGTERM, 3s grace, then SIGKILL) → delete lockfile. Prevents orphan MCP servers from consuming ports and memory after a crash.

**Client cache with dedup** — `pendingClients` Map prevents duplicate connections when the agent fires multiple tool calls in rapid succession during startup. Before reuse, a health ping (1s timeout) verifies the client is still alive.

**Notification-driven invalidation** — MCP servers push `ToolListChangedNotification` → client cache cleared → next access fetches fresh tool list. Handles external changes (e.g., user adds a Zendesk macro via the web UI → Zendesk MCP server's tool list changes).

**Per-call abort** — `AbortController` per tool call with unique `callId`. Enables per-call cancellation for safety rails (cost threshold exceeded) and user "Stop" button. Active tool calls tracked in a Map; `abortTool(callId)` signals the controller.

**MCP install state machine** — Guided 7-step flow for installing new MCP servers:

```
1. FETCHING_MANIFEST  → Fetch server manifest from registry or manual config
2. CHECKING_DEPS      → Run dependency checks (Node.js version, npm/python)
3. DEPS_REQUIRED      → [PAUSE] Show what's needed + install instructions
4. CONFIG_REQUIRED    → [PAUSE] Show config form (API keys, URLs) from JSON schema
5. STARTING_SERVER    → Start MCP server + list tools (validate working)
6. SAVING_CONFIG      → Save connection config to libsql mcp_servers table
7. COMPLETED          → Show success, clean up install state
```

Install state persisted in libsql (survives app restart — user can install Node.js, relaunch, and continue). Platform-specific dependency checking with actionable error messages ("node >= 18.0.0 required — install with `brew install node`"). Abort/cancel with cleanup at any step via `AbortController`.

### Safety Rails

#### Multi-Phase Tool Approval Policy

When the LLM requests a tool call, a 6-phase hierarchy decides whether to execute automatically or pause for approval. Security phases override user preferences:

| Phase | Check | Result |
|---|---|---|
| 1. Security blacklist | Hardcoded dangerous-action list (e.g., `delete_*`, `send_email`, `execute_command`) | → Always require approval |
| 2. Always-approve tools | Tool config says `requiresApproval: 'always'` | → Always require approval |
| 3. Automation context | Running inside an automation flow | → Auto-execute (unless Phase 1/2 blocked) |
| 4. Per-server policy | MCP server has custom approval policy | → Follow server policy |
| 5. Auto-run mode | User setting: "Trust this integration" | → Allow all remaining tools |
| 6. Default mode | Allow-list or manual mode | → Allow-list: tool in list → allow, else block. Manual: show approval dialog |

**Mixed execution** — When the LLM requests multiple tools in a single step, safe tools execute immediately while dangerous ones await approval. The agent stays responsive — the user sees partial results while reviewing the approval request.

**Approval state** — Operation transitions to `waiting_for_human`, stores `pendingToolsCalling` (the tools awaiting approval). User approves/rejects via IPC from the renderer. Approved tools execute directly (skip runner, go straight to executor). Rejected tools inform the agent, which decides on an alternative action.

#### Guardrail Conditions

| Condition | Action |
|---|---|
| Same tool called 3x with same input | Pause: "I seem to be stuck" |
| Task chain exceeds $1.00 in 10 min | Hard pause, tap to continue |
| Confidence drops 2+ consecutive steps | Pause, offer human review |
| Task chain > 5 minutes | Check-in: "Still working on X?" |
| Error rate > 50% in chain | Abort, show error summary |

All actions logged with undo window (default 5 min). Destructive actions always confirm first time. Money actions always require explicit approval. These rails apply to all execution paths — including automations. An automation runs its routine steps unattended but pauses at approval gates and notifies the user.

### Proactive Notifications

The platform watches captured activity and connected service signals, then surfaces native macOS notifications when something is worth acting on. See [product-features.md — Proactive notifications](../product/product-features.md#proactive-notifications) for trigger types, throttling rules, and UX behavior.

**Trigger detection** runs in the Agents & RAG utility process on a polling loop:

```
Agents & RAG Utility (polling loop, configurable interval)
    │
    ├── Query capture data from cowork.db
    │   └── Activity patterns (same app >10min, state transitions, focus sessions)
    │
    ├── Poll connected MCP services
    │   └── Incoming signals (new tickets, unread emails, calendar proximity)
    │
    └── Check time-based conditions
        └── Scheduled triggers, meeting proximity from calendar MCP
    │
    ▼
Throttling engine (in-memory + cowork.db)
    │
    ├── Priority tier check (urgent / helpful / informational)
    ├── Hourly cap check per tier
    ├── Flow-state suppression (query capture data for deep work signal)
    ├── Dismissal history check (cowork.db — deprioritize repeatedly dismissed triggers)
    ├── Bundling buffer (hold 30s, merge related notifications)
    └── Cooldown check (recent dismissal of same trigger type)
    │
    ▼ passes all gates
    │
IPC to Main → Main dispatches via Electron Notification API → macOS
```

**Process boundaries:**

| Step | Process | How |
|---|---|---|
| Trigger detection + throttling | Agents & RAG Utility | Polling loop queries cowork.db (capture data) + MCP services. Throttling state: delivery counts and cooldowns in-memory, dismissal history in cowork.db. |
| Notification dispatch | Main | Receives notification payload via IPC from Agents. Creates `Notification` via Electron API. No logic — just dispatch. |
| User response | Main → Agents & RAG / Renderer | **Accept** → Main routes to Agents → Agents begins execution (MCP/Browser path). **Dismiss** → Main routes to Agents → Agents logs dismissal to cowork.db. **Expand** → Main focuses Renderer → creates chat thread with pre-loaded context. |

**Flow-state suppression:** Capture data already tracks extended single-app sessions (focus detection stream, ON by default). The trigger engine queries this before dispatching non-urgent notifications. If the user is in a deep work session, helpful and informational notifications are held in a queue and released on the next context switch.

**Dismissal learning:** Each dismissal is logged to cowork.db with the trigger type. The trigger engine checks dismissal frequency when evaluating whether to surface a notification — consistently dismissed trigger types get deprioritized over time.

### Automations Engine

The Automations feature (product-features.md §5) uses a flow executor for sequential step execution with variable passing between steps. Runs in the Agents & RAG utility process.

**Flow executor** — Sequential step execution with fail-fast on error. Each step's output is stored in a variables map, available to all subsequent steps. A step can set `directOutput: true` to bypass remaining steps and return immediately (early-return pattern).

**Variable substitution** — Dot notation + bracket support for referencing prior step outputs: `${step1.data.ticketId}`, `${items[0].title}`. Unresolved variables are kept as `${...}` literals (allows partial resolution). Deep replacement walks every string in the step config recursively.

**Block types:**

| Block | Purpose |
|---|---|
| `trigger` | Event-driven: capture event, schedule, MCP notification |
| `mcpToolCall` | Call an MCP tool (auth/transport handled by MCP layer) |
| `agentStep` | Route to a Mastra agent (model selected via complexity router) |
| `condition` | If/else branching based on variable values |
| `delay` | Wait N seconds (for rate-limited tool calls) |
| `notification` | Send result to user via system notification |

**Flow storage** — libsql `automations` table with JSON config column. Atomic updates, queryable metadata (name, trigger type, enabled/disabled), included in single-DB backup.

**Flow-as-tool** — Automations registered as Mastra tools so the agent can invoke flows mid-conversation. The agent calls `flow_{uuid}` as a tool, passing variables as arguments.

**Safety rails apply** — Automation steps go through the same multi-phase tool approval policy. Approval gates pause the flow and notify the user. This prevents automations from taking destructive actions without consent.

---

## Feature → Infrastructure Map

How each product feature maps to the underlying infrastructure:

| Feature | Processes involved | Key infrastructure |
|---|---|---|
| **Apps** | Renderer + Agents & RAG | Apps rendered in renderer. Apps access platform via MCP tools (agent-as-tool pattern). App Gallery in renderer. |
| **MCP Integrations** | Agents & RAG | MCP SDK manages connections. Health monitoring. OAuth. |
| **Chat** (on-demand) | Renderer → Main → Agents & RAG | User message → IPC → complexity router → local/cloud brain → RAG context assembly → response → IPC → renderer |
| **Chat** (proactive) | Agents & RAG → Main → macOS | Trigger engine polls capture data + MCP services → throttling gates → main dispatches macOS notification → user accepts/dismisses/expands. See [Proactive Notifications](#proactive-notifications). |
| **MCP Browser** | Renderer + Agents & RAG + Playwright | Execution log in renderer. Agent sends commands to Playwright child. Live browser view streamed to renderer. Approval gates. |
| **Automations** | Agents & RAG | Trigger engine (time/event/activity-pattern). Runs agent tasks. Logs to cowork.db. |
| **Context** | Capture Utility → cowork.db → Agents & RAG → Renderer | Six input streams captured → stored → embedded → queryable via RAG → Context Card in renderer. |

**Context stream defaults:** Not all streams are active by default. Per [product-features.md](../product/product-features.md#what-the-ai-observes):

| Stream | Default |
|---|---|
| Window & app tracking | ON |
| Focus detection | ON |
| Browser activity (agent sessions) | ON during sessions |
| Keystroke & input capture | **OFF** (opt-in) |
| Screen recording | **OFF** (opt-in) |
| Clipboard monitoring | **OFF** (opt-in) |

Granular per-stream consent is required. Users can enable/disable each stream independently.

---

## Thermal Management

Silent. Users never see the word "thermal."

```
macOS ProcessInfo.thermalState (polled every 30s by main process)

nominal / fair  → normal operation
serious         → kill DeepSeek (GPU), route to cloud
                  keep Whisper on ANE (separate hardware)
                  notify: "Switched to cloud — machine was getting hot"
critical        → kill ALL local inference including Whisper
                  cloud-only until recovery
                  notify: "All AI moved to cloud"

On recovery (back to nominal/fair):
  → Do NOT auto-restore (cold restart is expensive)
  → notify: "Cooled down. Tap to restore local AI."
  → User taps → models reload
```

---

## Directory Structure

Business logic in `core/` with zero Electron imports. Electron wiring in `electron/`. UI in `renderer/`.

```
src/
├── core/                       # Pure TypeScript — no Electron dependency
│   ├── agents/                 # Mastra.ai agent definitions + orchestration
│   ├── automations/            # Rule/workflow engine
│   ├── capture/                # coworkai-keystroke-capture, coworkai-activity-capture wrappers
│   ├── chat/                   # Chat logic (on-demand + proactive)
│   ├── mcp/                    # MCP server connections + tool registry
│   ├── memory/                 # Embedding, vector store, RAG pipeline
│   └── store/                  # libsql data layer (capture + vectors)
├── electron/                   # Electron-specific wiring only
│   ├── main.ts                 # Main process — lifecycle, IPC routing, tray
│   ├── preload.ts              # Preload script for renderer
│   ├── capture.worker.ts       # Utility process entry — data capture
│   └── agents.worker.ts        # Utility process entry — agents, RAG, MCP
├── renderer/                   # Frontend UI (sandboxed, no Node.js)
│   ├── views/
│   │   ├── Chat/
│   │   ├── Apps/
│   │   ├── AppGallery/
│   │   ├── MCPBrowser/
│   │   ├── Automations/
│   │   ├── Context/
│   │   └── Settings/
│   └── components/
└── shared/                     # Types, IPC channel definitions, constants
```

**Why this split matters:** If we ever need to migrate off Electron, `core/` lifts out entirely — it has no Electron dependency. The `electron/` directory is ~4 files of wiring.

### Build Configuration

**ASAR unpack** — libsql and native capture addons (`coworkai-keystroke-capture`, `coworkai-activity-capture`) must be unpacked from the ASAR archive for `dlopen()`. Native `.node` files can't be loaded from inside ASAR — they need real filesystem paths.

**Post-pack patching** — libsql native module patching at both install-time (pnpm patch) and post-pack (script after electron-builder). Belt-and-suspenders approach ensures patches survive regardless of how the build pipeline transforms `node_modules`.

**Process-specific path aliases** — Separate build aliases prevent accidental cross-process imports:

| Alias | Resolves to | Available in |
|---|---|---|
| `@main` | `src/electron` | Main process build |
| `@core` | `src/core` | Main, workers |
| `@renderer` | `src/renderer` | Renderer build |
| `@shared` | `src/shared` | All builds |
| `@workers` | `src/electron/*.worker.ts` | Worker builds |

A renderer file can't `import '@main/...'` because that alias doesn't exist in the renderer build.

**Main process chunk inlining** — Single output file for the main process build (`inlineDynamicImports: true`). No async chunk loading, no race conditions on startup.

---

## Hardware Requirements (v0.1, Mac only)

| Machine | What runs | Default mode |
|---|---|---|
| **16GB+ Apple Silicon** | Whisper on ANE + DeepSeek on GPU + full capture | User chooses local or cloud at onboarding |
| **8GB Apple Silicon** | Whisper on ANE + full capture. No local LLM (not enough RAM). | Cloud-only (no choice needed) |

**Multi-process memory footprint** (~3GB reserve for preflight fit check: ~2GB OS baseline + ~1GB our processes):

| Process | Estimated memory |
|---|---|
| Electron Main | ~100MB |
| Renderer | ~200MB |
| Capture Utility | ~150MB |
| Agents & RAG Utility (Mastra + Ollama client) | ~300MB |
| Playwright Child (when active) | ~200MB |

On 16GB → ~13GB usable → GREEN for DeepSeek-R1-8B (~5.5GB model + ~2GB KV cache). On 8GB → ~5GB usable → RED for 8B model → confirms cloud-only design.

---

## Key Technical Decisions (Summary)

| Decision | Choice | Why | Reference |
|---|---|---|---|
| Desktop framework | Electron | All dependencies are Node.js. Swift/Tauri = two runtimes for zero benefit. | [DESKTOP_FRAMEWORK_DECISION.md](../decisions/DESKTOP_FRAMEWORK_DECISION.md) |
| Database | libsql | Native Mastra integration, built-in vectors, one SDK, proven in Electron | [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md) |
| Native addons | Keep custom `coworkai-*` | Critical capability gaps in open-source alternatives. Electron deadlock in uiohook-napi. | [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](../decisions/NATIVE_ADDON_REPLACEMENT_RESEARCH.md) |
| Local LLM | DeepSeek-R1-8B via Ollama | Reasoning capability, Qwen base (Tagalog+EN), fits 16GB Mac | [llm-architecture.md](./llm-architecture.md) |
| AI SDK version | Vercel AI SDK v6 (`ai` ^6.0.x, `@ai-sdk/react` ^3.0.x) | Interface contract between agents and providers, and between main process and React frontend. v6 over v5: mechanical renames (automated codemod), `@ai-sdk/react` v3 in renderer. Mastra 1.0+ supports both natively. | [decision-log.md](../decisions/decision-log.md#2026-02-16--ai-sdk-version-v6-not-v5) |
| Cloud providers | Gemini direct (`@ai-sdk/google`) + OpenRouter (`@ai-sdk/openai-compatible`) | Gemini direct for free tier (no middleman). OpenRouter for paid tiers (multi-provider gateway). Managed via AI SDK `createProviderRegistry()`. | [llm-architecture.md](./llm-architecture.md) |
| Agent orchestration | Mastra.ai via `@mastra/libsql` | Native libsql integration. Target: embedded in Electron utility process (needs spike). | [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md) |
| Embeddings | Qwen3-Embedding-0.6B via Ollama | ~500MB RAM, strong Tagalog+EN, stored in libsql built-in vector search | [llm-architecture.md](./llm-architecture.md) |
| Audio | Whisper via Core ML (ANE) | Dedicated hardware, doesn't compete with GPU/CPU | [llm-architecture.md](./llm-architecture.md) |
| Codebase strategy | Gut existing repos in place | No old product to maintain. Keep packaging, capture, orchestration. Kill tracking. | [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md) |

---

## Open Architecture Questions

Unresolved questions that affect implementation. Carried forward from [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md) and [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md).

| # | Question | Impact |
|---|---|---|
| 1 | **Mastra in utility process** — needs proof-of-concept spike. Can `@mastra/core` + `@mastra/libsql` initialize and run agents in an Electron utility process? | Architecture of Agents & RAG process. Fallback: main process or separate server. |
| 2 | **MCP server packaging** — bundled with the app, installed on demand, or running remotely? | Affects app bundle size, update mechanism, and offline capability. |
| 3 | **App Gallery hosting** — where do Google AI Studio apps get stored? Local filesystem? Bundled? Cloud service? | Affects Apps feature architecture and distribution. |
| 4 | **Bundle ID rename** — existing bundle ID is coworkai-branded. When do we rename to Cowork.ai Sidecar? | Affects signing, notarization, and auto-updater. |
| 5 | **Database schema** — not finalized. New schema designed from product-features.md, not carried over from old tracking tables. | Blocks Phase 1 implementation. |
| 6 | **IPC contract** — partially answered: typed `IpcChannel` constants in `src/shared/ipc-channels.ts` + `tracedInvoke` pattern for observability. Remaining: full channel inventory and Zod schemas for each channel's payload. | Blocks inter-process communication implementation. |
| 7 | **App-to-platform MCP transport** — how do third-party apps (rendered in Electron) access platform MCP tools? IPC bridge in preload (more secure, custom transport) vs. local HTTP server (reuses standard MCP client, opens a port). | Affects Apps feature security model and implementation. |
| 8 | **MCP install registry** — where do MCP server manifests come from? Own registry, community standard (e.g., MCP marketplace), or manual config only for v0.1? | Affects MCP install state machine (step 1: fetch manifest). |
| 9 | **Refusal message exclusion** — query refusal messages must be visible in the UI but excluded from Mastra's `lastMessages` context loading. Options: store refusals outside Mastra's message store (separate UI-only table), use Mastra metadata filtering if supported, or post-filter Mastra's context output. | Affects RAG query refusal implementation. |

---

## See Also

| Topic | Document |
|---|---|
| LLM stack deep-dive (routing, budgets, billing) | [llm-architecture.md](./llm-architecture.md) |
| Product features & capabilities | [product-features.md](../product/product-features.md) |
| Salvage plan & execution phases | [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md) |
| Mastra + Electron viability research | [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md) |
| Design system & interaction model | [design-system.md](../design/design-system.md) |
| Cost models & pricing | [llm-strategy.md](../strategy/llm-strategy.md) |

---

## Changelog

**v1 (Feb 16, 2026):** Initial draft. Consolidates process model, data flow, IPC contract, capture layer, database layer, LLM stack, memory system, agent orchestration, feature-to-infrastructure mapping, thermal management, and directory structure from six existing decision/architecture docs into a single reference.
**v1.1 (Feb 17, 2026):** Three-provider model: Gemini direct (`@ai-sdk/google`) for free tier, OpenRouter for paid tiers, Ollama for local. Managed via AI SDK `createProviderRegistry()`. Updated system diagram, data flow, cloud brain section, complexity router, and key decisions table.
**v1.2 (Feb 17, 2026):** Integrated adaptation guide decisions. Added: boot sequence for three processes, Playwright execution model, IPC streaming protocol with error handling, BaseManager + @channel IPC handler pattern, preload namespace design, Mastra Memory configuration mapping, sub-agent delegation pattern, MCP connection lifecycle and OAuth flow, AI SDK v6 in key decisions table.
**v1.3 (Feb 17, 2026):** Integrated 5 remaining adaptation guides (Jan, Chatbox, LobeChat, AnythingLLM, Cherry Studio). Added: database hardening (single-client access, vector index namespacing, schema migration strategy), model lifecycle (preflight fit check, download integrity, path safety), embedding pipeline (resumable ingestion, batch checkpoints, crash recovery), RAG context assembly (multi-source priority order, fillSourceWindow backfill, query refusal), agent execution model (persistent operations, instruction executor, max-step safety), multi-phase tool approval policy (6-phase hierarchy, mixed execution, approval state), MCP enhancements (orphan cleanup, client cache with dedup, notification-driven invalidation, per-call abort, 7-step install state machine), IPC observability (tracedInvoke, 4-process waterfall, typed IpcChannel constants, trace storage), automations engine (flow executor, variable substitution, block types, flow-as-tool), build configuration (ASAR unpack, post-pack patching, process-specific aliases, chunk inlining), complexity router askId grouping, explicit hardware reserve breakdown.
