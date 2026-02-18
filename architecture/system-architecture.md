# Cowork.ai Sidecar — System Architecture

**v1.8 — February 18, 2026**
**Audience:** Engineering (Rustan + team)
**Purpose:** Single reference for how the entire system fits together — processes, data flow, IPC, and how features map to infrastructure.

**Open items (0 questions, 1 inline):**

| # | Question | Blocks v0.1? | What it takes to resolve |
|---|---|---|---|
| 2 | **Database schema** | **Resolved — Draft** | Full table design, ownership map, retention policy, and types in [database-schema.md](./database-schema.md) |
| 3 | **IPC contract** | **Resolved — Final Draft** | 35 channels, 9 namespaces, full Zod schemas with locked Phase 1B decisions. See [ipc-contract.md](./ipc-contract.md) |

Inline: **Observer model choice** (line 612, needs benchmarking)

**This doc consolidates decisions from:**
- DESKTOP_SALVAGE_PLAN.md (consolidated into this doc — see [Codebase Origin](#codebase-origin) and [Execution Phases](#execution-phases))
- [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md) — why libsql
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

> **Mastra in utility process is proven.** PoC spike (Feb 17, 2026) passed all 7 steps: Mastra + libsql init, `agent.generate()` via IPC, `agent.stream()` via MessagePort, crash isolation + restart, and packaged runtime with ASAR-unpacked native modules. See [phase-1b-sprint-plan.md Sprint 0](./phase-1b-sprint-plan.md#sprint-0-complete-mastra-utility-process-spike) for evidence links.

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

Two utility processes boot in parallel. Main waits up to 10s for utilities to signal ready, then creates the renderer regardless (see [Startup Failure Policy](#startup-failure-policy)).

**Main process:**
0. MCP orphan cleanup — glob lockfiles, kill stale processes (see [MCP Connections](#mcp-connections))
1. AppManager (lifecycle, settings)
2. ThermalManager (hardware monitoring)
3. Spawn Capture Utility + Agents & RAG Utility
4. Wait up to 10s for both utility processes to signal ready (timeout → degraded mode)
5. Create BrowserWindow (renderer)

**Capture Utility:**
1. DBManager (libsql sync connection to `cowork.db`)
2. NativeAddonManager (load `coworkai-keystroke-capture`, `coworkai-activity-capture`)
3. CaptureBufferManager (activity buffer, keystroke chunker)
4. Signal ready to main

**Agents & RAG Utility:**
1. DBManager (libsql async connection to `cowork.db`)
2. ProviderRegistry (Ollama + Gemini + OpenRouter via `createProviderRegistry()`)
3. MastraManager (agent orchestration, memory)
4. MCPManager (service connections, OAuth credential loading)
5. EmbeddingManager (queue, Qwen3-Embedding-0.6B via Ollama)
6. Signal ready to main

Each step within a process is sequential — later steps depend on earlier ones. The two utility processes are independent and boot concurrently.

### Startup Failure Policy

The boot sequence above describes the happy path. If a utility process fails to start, the app must still be usable — not a black screen.

**Service states** — Each utility process is tracked by Main with one of five states:

| State | Meaning |
|---|---|
| `starting` | Process spawned, waiting for ready signal |
| `running` | Ready signal received, fully operational |
| `degraded` | Ready signal not received within timeout, or partial init failure (e.g., Ollama unreachable but DB connected) |
| `restarting` | Crashed or timed out, auto-restart in progress |
| `failed` | Max retries exhausted, manual intervention required |

**Timeout + degraded boot** — Main waits up to 10 seconds for each utility's ready signal. If a utility doesn't respond:

1. Main creates the BrowserWindow anyway (renderer always launches)
2. The timed-out utility is marked `degraded` and Main begins retry
3. Renderer receives health status via IPC and shows a degraded banner ("Agent features unavailable — reconnecting...")

**Retry behavior** — Exponential backoff: 2s → 4s → 8s → 16s → 32s, max 5 retries. After max retries, state transitions to `failed` and the user sees "Agent services couldn't start. Check Settings for details." with a manual retry button.

**What works in degraded mode:**

| Agents & RAG down | Capture down |
|---|---|
| Settings, theme, app lifecycle: work | Settings, theme, app lifecycle: work |
| Chat, automations, MCP tools: disabled | Chat, MCP tools: work (no fresh context) |
| Context Card: shows stale data | Context Card: shows stale data |
| Proactive notifications: disabled | Proactive notifications: degraded (MCP signals only, no capture triggers) |

**Capture failure is less severe** — agent features still work, just without fresh activity context. Agents failure is more severe — all AI features are disabled.

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
│  coworkai-keystroke-capture     │──→ Keystroke patterns, clipboard events
│                                 │
│  Activity buffer (5-slot queue) │──→ Bounded memory, auto-flush on focus change
│  Keystroke chunker (debounce)   │──→ 1000-char chunk cap, special-key triggers
│  Independent streams            │──→ Each self-contained, correlated by time
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
| Agents & RAG Utility | `@libsql/client` + `@mastra/libsql` (async) | Read/Write | Reads capture data for context. Writes agent memory, embeddings. Moderate frequency. |
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
| Main → Renderer | UI updates, chat responses, execution logs, service health events |
| Renderer → Main | User input, chat messages, settings changes, approval decisions |
| Agents → Playwright | Browser automation commands |
| Playwright → Agents | Page state, action results |

**The renderer never talks directly to capture or agents.** Everything routes through main. This keeps the sandboxed renderer clean and gives main a single point for logging, rate-limiting, and access control.

**Capture and Agents never communicate via IPC either** — they share data through `cowork.db`. Capture writes; Agents polls and reads. No direct IPC channel between utility processes.

### Service Health IPC

Main tracks the health of each utility process (see [Startup Failure Policy](#startup-failure-policy)) and pushes status changes to the renderer:

| Channel | Direction | Payload | When |
|---|---|---|---|
| `system:health` | Main → Renderer | `{ service: string, state: ServiceState, message?: string }` | On any state transition (starting → running, running → restarting, etc.) |
| `system:retry` | Renderer → Main | `{ service: string }` | User clicks manual retry button in degraded/failed banner |

The renderer uses `system:health` events to show/hide degraded banners and disable UI sections that depend on unavailable services. No polling — purely event-driven.

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

**tracedInvoke** — Wrapper around IPC that propagates OpenTelemetry span context across all resident processes. The trace payload is appended as the last IPC argument and stripped before the handler sees it — business logic never knows about tracing. Backward compatible: calls without trace context work identically.

**Cross-process waterfall** — A single user chat request produces a trace: `renderer.send` → `main.relay` → `agents.process` → `playwright.execute` (if browser action). Shows exactly where latency lives. Main process adds a relay span (`ipc.relay`) showing the routing hop.

**IpcChannel typed constants** — All channel names defined in `src/shared/ipc-channels.ts`. Single source of truth imported by all processes. Prevents typos and enables refactoring with type safety.

**Trace storage** — Spans stored in `cowork.db` (extends Mastra's `LibSQLStore` traces, same database). Used for: latency debugging across processes, safety rail audit trails (every tool call logged with timing), and MCP Browser execution log (action timeline).

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
| `cowork.apps` | Main → Agents & RAG | App install/remove, list installed apps |
| `cowork.system` | Main | Service health events, manual retry, app diagnostics |

---

## Native Capture Layer

Two custom C++ N-API addons. Battle-tested in production. No viable open-source replacements.

### `coworkai-keystroke-capture`

Global keyboard and mouse event capture.

| Capability | Detail |
|---|---|
| Character mapping | Full UTF-8 via `CGEventKeyboardGetUnicodeString` (macOS) / `ToUnicodeEx` (Windows) |
| Key repeat detection | OS-provided (macOS) + atomic tracking (Windows) |
| CapsLock state | Exposed as boolean in every event |
| Electron safety | Graceful Accessibility permission handling, event tap auto-recovery |
| Threading | `Napi::ThreadSafeFunction` — events dispatched to JS without blocking native thread |

### Active Window Detection

Primary source: `@miniben90/x-win` — polls native API at 100ms intervals via `subscribeActiveWindow()`. Provides app name, window title, and browser URL.

### `coworkai-activity-capture`

Enrichment fallback — supplements `x-win` when window title or browser URL is missing. Called synchronously on each x-win event; enrichment is skipped if both fields are already present. App name must match x-win's app name (case-insensitive) to prevent cross-app data contamination.

| Capability | Detail |
|---|---|
| macOS app detection | 3-tier fallback: `NSWorkspace` → `runningApplications` → `CGWindowListCopyWindowInfo` |
| Browser URL (macOS) | AppleScript + Accessibility API fallback via `AXUIElementRef` recursive traversal |
| Browser URL (Windows) | Full COM UI Automation — 28 localized address bar names, supports Chrome/Edge/Firefox/Brave/Opera |
| Deep Chromium traversal | Recursive DFS through Accessibility tree for `AXWebArea` roles |
| Architecture | In-process N-API addon, sub-millisecond. Rebuilt against Electron ABI. |

### Capture Orchestration (adapted from `coworkai-agent`)

Core buffering/chunking patterns are reused from the existing codebase. Key semantic change from the old architecture: **capture streams are independent peers, not parent-child.** Each stream writes self-contained rows (keystroke chunks carry their own `app_name`/`bundle_id`, clipboard events carry `source_app`). The `activity_session_id` on child tables is an optional correlation hint, not a structural dependency. Streams are correlated by overlapping timestamps at read time.

| Component | What it does |
|---|---|
| Activity buffer | Bounded 5-slot queue with `autoFlushIfNeeded()`. Prevents unbounded memory growth during rapid window switching. |
| Keystroke chunker | Debounce-driven flushing + special-key triggers + 1000-char chunk cap. Each chunk carries denormalized app context (`app_name`, `bundle_id`). Activity session ending flushes any open chunk (orchestration-level coordination, not a DB constraint). |
| Clipboard capture | Hotkey detection (copy/cut/paste) in the keystroke stream → `readClipboard()` on hotkey. Each event carries `source_app`. More efficient than polling. |

See [capture-pipeline-architecture.md](./capture-pipeline-architecture.md) for the complete implementation-level specification including supervisor lifecycle, IPC protocol, flush triggers, constants, and simulation mode.

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

**Schema:** Draft for implementation. See [database-schema.md](./database-schema.md) for the full table design, ownership map, retention policy, and types.

### Database Hardening

**Single-client access pattern** — Each process should use exactly one `@libsql/client` Client instance for all DB operations to prevent intra-process `SQLITE_BUSY` errors from multiple clients competing for write access. In the Agents & RAG utility process, the ideal approach is to create one client via the native libsql TypeScript SDK and inject it into both `LibSQLVector` and `LibSQLStore` — depends on whether Mastra accepts an external client (needs investigation). The Capture Utility process creates its own separate client (sync API for high-frequency writes). Cross-process write contention between Capture and Agents is handled by WAL's busy timeout.

**Vector index namespacing** — Separate `LibSQLVector` indexes per data type:

| Index | What it stores |
|---|---|
| `embed_activity` | Embedded window titles, URLs, app sessions |
| `embed_conversation` | Embedded past conversations |
| `embed_observation` | Embedded observational memory summaries |

Separate indexes enable per-type queries (e.g., "find similar activity") or merged cross-type queries with different relevance weights. Same `LibSQLVector.createIndex()` API per index.

**Schema migration strategy** — Additive `ALTER TABLE ADD COLUMN` with duplicate-column tolerance (try/catch, ignore "column already exists"). No version-tracked migration framework for v0.1. Per-process schema ownership: the Capture Utility process creates and migrates app tables (activities, input events, context streams); the Agents & RAG Utility process creates and migrates its own tables (embedding queue, agent operations, automations, mcp_servers, mcp_connection_state, tool_policies, notification_log, installed_apps, app_permissions). Mastra manages `LibSQLStore` and `LibSQLVector` tables internally. Neither process ALTERs tables owned by the other.

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
| Dimensions | 1024 |
| Distance | Cosine similarity |
| Fallback | Keyword search against SQLite if embeddings can't run locally |

### Audio (The Ears)

| | |
|---|---|
| Model (16GB+) | Whisper Turbo (Core ML → Apple Neural Engine) |
| Model (8GB) | Whisper Small (Core ML, ~244MB, ~0.5GB RAM) — loads on demand |
| RAM (16GB+) | ~1.5 GB |
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

Once embedded, this data is retrievable via the [RAG Context Assembly](#rag-context-assembly) pipeline.

---

## Context Pipeline

How data reaches the Cowork agent and third-party apps at runtime. Two distinct lanes — the agent path (full reasoning) and the app read lane (data only).

### Agent Path: Two-Layer Runtime

The agent path handles user chat via `chat:sendMessage`. Two layers work together — they are complementary, not alternatives. Layer 1 gives the agent awareness of what's happening. Layer 2 gives it the ability to take action.

**Layer 1: Automatic Context Injection**

| | |
|---|---|
| Trigger | Every `chat:sendMessage` request |
| Location | Agents utility request path, before model invocation |
| Responsibility | Build a lightweight runtime context snapshot and attach it to agent instructions |
| Property | Deterministic application behavior (not model-decided) |

Layer 1 runs on every message. The agents utility queries recent capture data from `cowork.db` and injects it as system context before the model sees the user's message. This is how chat becomes activity-aware without embeddings — direct SQL before each agent call.

**Layer 2: Tool Execution**

| | |
|---|---|
| Trigger | Model emits a tool call during agent reasoning |
| Location | Agents utility tool execution layer |
| Responsibility | Execute platform and MCP tools, return structured results |
| Property | On-demand, driven by agent reasoning only |

Layer 2 fires only when the agent decides it needs to act. The agent can invoke platform tools (context queries, notifications) and MCP tools (Zendesk, Gmail, Slack) during a single reasoning turn.

**Agent chat flow:**

```
Renderer (Main UI)
  → ipcRenderer.invoke('chat:sendMessage', payload)
  → Main process relay
  → Agents utility:
       1) Apply Layer 1 context injection (query capture data → system context)
       2) Run agent invocation (model reasoning)
       3) Agent may invoke Layer 2 tools (platform + MCP)
       4) Stream response chunks back via MessagePort
  → Renderer receives streamed output
```

Transport: `chat:sendMessage` returns immediately with `{ threadId }`. Response content streams separately via `chat:streamPort` MessagePort with ACK gate. See [ipc-contract.md](./ipc-contract.md) for channel schemas and streaming protocol.

### App Read Lane

Third-party apps use a separate typed read lane through `window.cowork.{context,user,data}.*` mapped to a single `cowork:read` IPC channel. No agent reasoning, no tool execution, just deterministic data retrieval. See [Apps Runtime](#apps-runtime) for the full SDK surface, access controls, and installation flow.

**Policy: apps get data, not tools.** Tool execution is reserved for the agent. App write/action capabilities are out of scope for Phase 1B — if needed later, the design will be driven by concrete app use cases.

### Why Two Separate Lanes

Agent-mediated app access was evaluated and rejected — non-deterministic responses, slow (LLM round-trip per call), and expensive (tokens for every data fetch). Direct tool calls (`callTool`) were rejected — tight coupling to internal tool surface, conflates reads with writes. MCP protocol for apps was rejected — designed for LLM-to-service communication, adds unnecessary protocol overhead for simple data lookups. The typed read lane gives apps a normal JavaScript API: discoverable via TypeScript types, fast (no LLM), predictable (deterministic queries), and frictionless (no permission grants for reads).

---

## RAG Context Assembly

When the agent needs grounded context for a response, the RAG pipeline assembles material from multiple sources in priority order. Source data is ingested by the [Embedding Pipeline](#embedding-pipeline). This section describes the retrieval sub-pipeline within Mastra's `semanticRecall` layer — how grounded material is fetched and ranked. The overall context window composition (conversation history + working memory + semantic recall + observational memory) is handled by Mastra's Memory system (see [Memory System](#memory-system)).

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

Refusal messages must be visible to the user but excluded from future context windows (prevents "I don't know" answers from polluting subsequent queries). Solved via a custom Mastra `MemoryProcessor`: a `RefusalFilter` strips refusal messages from the context window before sending to the LLM, while leaving them in storage for UI display. Same pattern as Mastra's built-in `ToolCallFilter` processor.

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

MCP servers are **bundled** with the app — developers (us) build and ship each integration. Users don't install or discover MCP servers; they connect to bundled ones by providing credentials. The Agents & RAG utility process manages all connections.

> **Phase 1B implements:** connection management, tool registry, API key auth, health monitoring. **Phase 3 adds:** OAuth auth, orphan cleanup, notification-driven invalidation, per-call abort, per-server approval policies.

| Concern | How it works |
|---|---|
| Connection management | OAuth flows, health monitoring, auth expiry detection |
| Tool registry | Each service exposes tools (list_tickets, send_email, etc.) |
| Scoped access | Apps declare what MCP tools they need. Platform grants per-app. |
| Disconnect handling | Action pauses, user notified with exact blocker + reconnect action |

**MCPClient status lifecycle:**

```
starting ──→ running ──→ stopped (user toggled off)
                │              │
                │              └──→ starting (user re-enabled)
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

**OAuth flow:** *(Phase 3 scope — Phase 1B uses API key auth only)*
- System browser opens for login (renderer can't handle OAuth popups — sandboxed)
- PKCE flow via `OAuthClientProvider` from `@modelcontextprotocol/sdk`
- Credentials (OAuth tokens, API keys) encrypted via Electron `safeStorage` API (uses macOS Keychain / Windows DPAPI under the hood), persisted to `cowork.db` keyed by `cowork-mcp-{serverUrlHash}`. Non-secret metadata (server URL, scopes, expiry timestamp) stored in `{userData}/.mcp/{serverUrlHash}.json`. Tokens never written to plaintext files. (`keytar` is deprecated and archived — `safeStorage` is Electron-native, no additional native bindings needed.)
- Token refresh handled automatically; expiry detection triggers "reconnect" prompt in UI
- Main process opens browser and captures OAuth callback, forwards tokens to Agents & RAG utility

**MCP orphan cleanup** — Each stdio MCP server gets a lockfile (`{cowork_data_dir}/mcp_lock_{serverNameHash}.json`) with `childPid`, normalized `executablePath`, and `argvHash` (SHA256 of normalized args). On app startup, Main process runs a sweep before Agents boots: glob lockfiles → check process alive (signal 0) → fetch live command line via `ps -p <pid> -o command=` → parse + normalize live executable path and args → compare `(executablePath, argvHash)` from lockfile vs live process → kill only on exact match (SIGTERM, 3s grace, then SIGKILL) → delete lockfile. If PID is dead or identity doesn't match, just delete the stale lockfile. Prevents orphan MCP servers from consuming ports and memory after a crash without risking PID-reuse false kills.

**Client cache with dedup** — `pendingClients` Map prevents duplicate connections when the agent fires multiple tool calls in rapid succession during startup. Before reuse, a health ping (1s timeout) verifies the client is still alive.

**Notification-driven invalidation** — MCP servers push `ToolListChangedNotification` → client cache cleared → next access fetches fresh tool list. Handles external changes (e.g., user adds a Zendesk macro via the web UI → Zendesk MCP server's tool list changes).

**Per-call abort** — `AbortController` per tool call with unique `callId`. Enables per-call cancellation for safety rails (cost threshold exceeded) and user "Stop" button. Active tool calls tracked in a Map; `abortTool(callId)` signals the controller.

**MCP connection flow** — Since MCP servers are bundled (dependencies ship with the app), the user-facing flow is connection, not installation:

```
1. SELECT_SERVICE    → User picks from bundled integrations (Zendesk, Gmail, Slack, etc.)
2. CONFIG_REQUIRED   → [PAUSE] Show config form (API keys, URLs) from bundled JSON schema
3. AUTHENTICATING    → OAuth flow or API key validation
4. STARTING_SERVER   → Start bundled MCP server + list tools (validate working)
5. SAVING_CONFIG     → Save connection config to libsql mcp_servers table
6. CONNECTED         → Show success, tools available
```

Connection state persisted in libsql (survives app restart). Abort/cancel with cleanup at any step via `AbortController`.

### Safety Rails

#### Multi-Phase Tool Approval Policy

When the LLM requests a tool call, a 6-phase hierarchy decides whether to execute automatically or pause for approval. Security phases override user preferences:

| Phase | Check | Result |
|---|---|---|
| 1. Security blacklist | Hardcoded dangerous-action list (e.g., `delete_*`, `send_email`, `execute_command`) | → Always require approval |
| 2. Mandatory approval tools | Tool config says `requiresApproval: 'always'` | → Always require approval |
| 3. Automation context | Running inside an automation flow | → Auto-execute (unless Phase 1/2 blocked) |
| 4. Per-server policy | MCP server declares approval policy in its `mcp_servers` config row (e.g., `approval_policy: 'auto'` or `'manual'`). v0.1: not implemented — all servers use Phase 6 default. | → Follow server policy |
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
Throttling engine (cowork.db — all state persisted)
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
| Trigger detection + throttling | Agents & RAG Utility | Polling loop queries cowork.db (capture data) + MCP services. All throttling state persisted in cowork.db: delivery counts, cooldowns, and dismissal history. Survives process crashes — a restart doesn't bypass hourly caps or cooldown periods. |
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

**Safety rails apply** — Automation steps go through the same multi-phase tool approval policy. Approval gates pause the flow and notify the user. This prevents automations from taking destructive actions without consent. When an agent invokes a flow via `flow_{uuid}` tool call, the flow runs in automation context (Phase 3) — its steps auto-execute unless blocked by Phase 1 (security blacklist) or Phase 2 (mandatory approval).

---

## Apps Runtime

Third-party apps (Google AI Studio exports) run inside the Electron shell. Each app is an isolated renderer process with no Node.js access. The runtime handles rendering, preprocessing, platform communication, and read-lane access controls.

### Rendering

Each installed app runs in its own `WebContentsView` — Electron's recommended primitive for isolated content (replaces deprecated `BrowserView`, avoids discouraged `<webview>` tag).

| Setting | Value | Why |
|---|---|---|
| `contextIsolation` | `true` | Preload world separated from page world |
| `sandbox` | `true` | OS-level process sandbox |
| `nodeIntegration` | `false` | No `require()`, `process`, or `fs` |
| `webviewTag` | `false` | Prevent nested webviews |
| `partition` | `persist:app-${appId}` | Isolated cookies/storage per app |

Navigation blocked (`will-navigate` → `preventDefault()`). `window.open()` blocked. Each app is a separate renderer process — if one crashes, main UI and other apps are unaffected.

**File serving** — Custom `cowork-app://` protocol (registered as privileged scheme: `standard`, `secure`, `supportFetchAPI`). App loads as `cowork-app://{app-id}/index.html`. Path validation prevents directory traversal (`path.relative()` + prefix check).

### Platform Communication (Decided)

**Preload IPC relay** — same pattern as the main renderer's `window.cowork.*`:

```
App JS → window.cowork.context.activeWindow()
  → preload (contextBridge) → ipcRenderer.invoke('cowork:read', { appId, ns, method, args })
  → main process relays to Agents & RAG utility
  → agents read handler executes deterministic data query
  → result returns through same chain
```

No custom port, no auth tokens for platform communication. Main remains a transport relay only.

### Platform SDK

Apps see `window.cowork.*` injected via the app's preload script. In Phase 1B, apps get read-only platform context/data methods routed through `cowork:read`.

**Apps can make their own external API calls** (Gemini, OpenAI, etc.) using standard web APIs (`fetch`, `XMLHttpRequest`) with their own API keys. The sandbox blocks Node.js/Electron APIs and direct DB access, not network access — apps are web pages. A Google AI Studio export calling the Gemini API with its own key works as-is.

**Apps cannot access platform internals:** MCP server connections, platform-managed API keys/OAuth tokens, agent reasoning, database, cross-app data, or Node.js/Electron APIs.

| Namespace | Methods | What it does |
|---|---|---|
| `context` | `activeWindow()`, `recentActivity(opts?)`, `currentTime()` | Focused app/window, recent activity summary, local time |
| `user` | `preferences()`, `profile()` | User settings/profile used for app personalization |
| `data` | `conversations(opts?)`, `search(query, opts?)` | App-scoped conversation history and semantic search over activity |

### App Access Controls

Read-lane methods are default-allowed in Phase 1B (no per-method runtime permission prompts), but the model is not "no security."

- **Install-time disclosure is required:** before enabling an app, the installer shows the read-lane data envelope (activity/window context, user profile/preferences, app-scoped conversations, semantic search results).
- **Trusted app identity:** preload derives `appId` from the `persist:app-{appId}` session partition and injects it into every `cowork:read` call. App JS cannot spoof identity.
- **Scoped reads:** app-owned data methods (for example `data.conversations`) are always scoped by trusted `appId`.
- **Process isolation:** app renderers remain sandboxed (`nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`) with no direct DB or utility-process access.

Flow:

```
App calls window.cowork.data.conversations({ limit: 50 })
  → preload injects trusted appId
  → ipcRenderer.invoke('cowork:read', { appId, ns: 'data', method: 'conversations', args })
  → main relay
  → agents utility enforces appId scoping
  → result returns
```

### Two-Track App Strategy

**Track 1 — Template apps (happy path):** We publish a Cowork.ai-compatible Google AI Studio template. Apps built from it are guaranteed to work. The template:
- Uses `window.cowork.{context,user,data}.*` for platform data access
- May include its own Gemini API calls with its own key (app-managed, not platform-managed)
- Can include optional `cowork.manifest.json` metadata for installer/display copy
- Has a clean React structure that esbuild bundles without issues

**Track 2 — Generic AI Studio exports (best-effort):** Users can upload any AI Studio export. esbuild attempts to bundle it. If it fails, show error + link to compatibility guidelines. Generic exports work as standalone web apps (their own Gemini API calls function normally in the sandbox). Adding platform context via the read-lane SDK is optional but recommended.

### Design Decisions (Resolved)

| Question | Decision | Reasoning |
|---|---|---|
| Preprocessing pipeline | **esbuild bundle** | Fast (~10-100ms). Handles TSX/TS natively. Produces single output file that resolves all imports. Solves preprocessing + import resolution in one step. |
| Gemini API proxy | **Not needed.** Apps make their own Gemini API calls directly with their own keys. | No proxy layer needed — apps are web pages with normal network access. Platform doesn't manage or meter app LLM usage. |
| Import resolution | **esbuild bundles deps** — no import maps needed. esbuild resolves bare specifiers during bundling. | Falls out of the esbuild decision. No runtime import resolution, no CDN dependency, works offline. |
| Manifest authoring | **Optional metadata manifest.** No app tool-permission grants in Phase 1B read lane. | Keeps install UX simple while still allowing template metadata and disclosure copy improvements. |

### Installation Flow

```
User uploads .zip
    ↓
1. Extract to ~/Library/Application Support/cowork-ai/apps/{uuid}/
    ↓
2. Read cowork.manifest.json (if present) or package.json for metadata
    ↓
3. esbuild bundle: TSX/TS → single JS output, all deps resolved
    ↓
4. If esbuild fails → show error + link to compatibility guidelines
    ↓
5. Show read-lane disclosure (what this app can read in Phase 1B); require explicit user confirmation
    ↓
6. Store app metadata + install record in cowork.db
    ↓
7. App appears in sidesheet
```

### cowork.manifest.json Format

```json
{
  "name": "Zendesk Dashboard",
  "description": "View ticket context from your current activity",
  "category": "CRM",
  "dataUseDescription": "Uses your current window, recent activity, and app-scoped conversations to prioritize tickets."
}
```

Used for installer/app metadata only in Phase 1B. It does not grant runtime write/action permissions.

---

## Feature → Infrastructure Map

How each product feature maps to the underlying infrastructure:

| Feature | Processes involved | Key infrastructure |
|---|---|---|
| **Apps** | Renderer + Agents & RAG | Each app in its own `WebContentsView` (sandboxed renderer process). Platform communication via preload IPC read lane (`cowork:read`). Install-time disclosure is required before enabling app access to read-lane data. v0.1: user uploads zip file, Cowork renders it locally. No hosted gallery. See [Apps Runtime](#apps-runtime). |
| **MCP Integrations** | Agents & RAG | MCP SDK manages connections. Health monitoring. OAuth. |
| **Chat** (on-demand) | Renderer → Main → Agents & RAG | User message → IPC → complexity router → local/cloud brain → RAG context assembly → response → IPC → renderer |
| **Proactive Notifications** | Agents & RAG → Main → macOS | Trigger engine polls capture data + MCP services → throttling gates → main dispatches macOS notification → user accepts/dismisses/expands. See [Proactive Notifications](#proactive-notifications). |
| **MCP Browser** | Renderer + Agents & RAG + Playwright | Execution log in renderer. Agent sends commands to Playwright child. Live browser view streamed to renderer. Approval gates. |
| **Automations** | Agents & RAG | Trigger engine (time/event/activity-pattern). Runs agent tasks. Logs to cowork.db. |
| **Context** | Capture Utility → cowork.db → Agents & RAG → Renderer | Five input streams in v0.1 (screen recording reserved for v0.2) → stored → embedded → queryable via RAG → Context Card in renderer. |

**Context stream defaults:** All five v0.1 streams are ON by default. Per [capture-pipeline-architecture.md](./capture-pipeline-architecture.md):

| Stream | Default | v0.1 |
|---|---|---|
| Window & app tracking | ON | Yes |
| Focus detection | ON | Yes |
| Browser activity (agent sessions) | ON | Yes |
| Keystroke & input capture | ON | Yes |
| Screen recording | — | No (reserved for v0.2) |
| Clipboard monitoring | ON | Yes |

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
│   │   ├── Context/
│   │   ├── Settings/
│   │   ├── Integrations/
│   │   └── Apps/
│   │   # Phase 4/5: MCPBrowser/, Automations/, AppGallery/
│   └── components/
└── shared/                     # Types, IPC channel definitions, constants
```

**Why this split matters:** If we ever need to migrate off Electron, `core/` lifts out entirely — it has no Electron dependency. The `electron/` directory is ~4 files of wiring.

### Build Configuration

**ASAR unpack** — libsql and native capture addons (`coworkai-keystroke-capture`, `coworkai-activity-capture`) must be unpacked from the ASAR archive for `dlopen()`. Native `.node` files can't be loaded from inside ASAR — they need real filesystem paths. (Credential encryption uses Electron's built-in `safeStorage` API — no additional native bindings needed.)

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
| Desktop framework | Electron | All dependencies are Node.js. Swift/Tauri = two runtimes for zero benefit. | [decision-log.md](../decisions/decision-log.md) |
| Database | libsql | Native Mastra integration, built-in vectors, one SDK, proven in Electron | [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md) |
| Native addons | Keep custom `coworkai-*` | Critical capability gaps in open-source alternatives. Electron deadlock in uiohook-napi. | [decision-log.md](../decisions/decision-log.md) |
| Local LLM | DeepSeek-R1-8B via Ollama | Reasoning capability, Qwen base (Tagalog+EN), fits 16GB Mac | [llm-architecture.md](./llm-architecture.md) |
| AI SDK version | Vercel AI SDK v6 (`ai` ^6.0.x, `@ai-sdk/react` ^3.0.x) | Interface contract between agents and providers, and between main process and React frontend. v6 over v5: mechanical renames (automated codemod), `@ai-sdk/react` v3 in renderer. Mastra 1.0+ supports both natively. | [decision-log.md](../decisions/decision-log.md#2026-02-16--ai-sdk-version-v6-not-v5) |
| Cloud providers | Gemini direct (`@ai-sdk/google`) + OpenRouter (`@ai-sdk/openai-compatible`) | Gemini direct for free tier (no middleman). OpenRouter for paid tiers (multi-provider gateway). Managed via AI SDK `createProviderRegistry()`. | [llm-architecture.md](./llm-architecture.md) |
| Agent orchestration | Mastra.ai via `@mastra/libsql` | Native libsql integration. Embedded in Electron utility process (proven via PoC spike, Feb 17 2026). | [phase-1b-sprint-plan.md — Sprint 0](./phase-1b-sprint-plan.md#sprint-0-complete-mastra-utility-process-spike) |
| Embeddings | Qwen3-Embedding-0.6B via Ollama | ~500MB RAM, strong Tagalog+EN, stored in libsql built-in vector search | [llm-architecture.md](./llm-architecture.md) |
| Audio | Whisper via Core ML (ANE) | Dedicated hardware, doesn't compete with GPU/CPU | [llm-architecture.md](./llm-architecture.md) |
| Codebase strategy | Gut existing repos in place | No old product to maintain. Keep packaging, capture, orchestration. Kill tracking. | [Codebase Origin](#codebase-origin) (this doc) |

---

## Codebase Origin

Cowork.ai Sidecar is built on top of four existing repositories from a previous employee activity monitoring product. The product direction reversed — from employer surveillance to worker-owned AI assistant — but the infrastructure (packaging, native addons, capture orchestration) carries over.

### Repo Disposition

| Repo | Action | What survived |
|---|---|---|
| `coworkai-desktop` | **Gutted in place** | Electron Forge config (packaging, signing, publishing, auto-updater), macOS/Windows code signing, S3 publisher, environment profiles, ASAR unpacking, native resolution logic, React + Vite + Tailwind build tooling. Killed: tracker IPC, auth IPC, timer IPC, sync config, media capture configs, entire renderer. |
| `coworkai-agent` | **Abandoned (reference only)** | Repo is no longer used. Capture orchestration patterns were ported into `coworkai-desktop`: activity buffer management (bounded 5-slot queue), keystroke chunking (debounce + special-key triggers + 1000-char cap), clipboard hotkey detection, WAL setup pattern, config-driven capture toggling. Everything else (timer, sync, media, employer identity) was dropped. |
| `coworkai-activity-capture` | **Kept as-is** | Custom C++ N-API addon. 3-tier macOS app detection, Accessibility API browser URL fallback, deep Chromium traversal, full Windows COM UI Automation. No viable open-source replacement. |
| `coworkai-keystroke-capture` | **Kept as-is** | Custom C++ N-API addon. Full UTF-8 character mapping, key repeat detection, CapsLock state, Electron-safe threading. No viable open-source replacement (uiohook-napi has an unresolved Electron deadlock). |

No fork was needed — there is no old product to maintain. CI/CD pipelines, signing certificates, bundle ID, team access, and git history all carry over.

### What was killed and why

The previous product sent activity data to an employer cloud. Everything related to that model was removed:

- **Sync pipeline** — Employer cloud sync scheduling, batch retry, `sync` flag columns
- **Timer module** — Midnight splitting, daily aggregation, session tracking ("not a time tracker")
- **Employer auth** — `AUTH/SET_USER` channel, employer-scoped `user_id` isolation
- **Media capture** — Screen, audio, video modules (screen recording deferred to v0.2)
- **Tracker IPC** — `TRACKER/START`, `TRACKER/STOP` channels
- **Entire renderer** — Complete rewrite for sidecar product surface

---

## Execution Phases

Phased delivery from salvage to full product. Phase 1A is complete. Phase 1B is active. Phases 2-5 are rough scope buckets, not detailed sprint plans.

### Phase 1A: Local Gut/Unblock (Complete)

Gutted `coworkai-desktop` enough to restore local build/run loop. Removed blocked dependencies, tracker/auth/agent runtime coupling, replaced renderer with minimal shell. Validated `npm install`, `tsc`, `rebuild`, `start`. PR #8.

### Phase 1B: Core Runtime Foundation (Active)

Full sprint plan: [phase-1b-sprint-plan.md](./phase-1b-sprint-plan.md)

- Target folder architecture (`core/`, `electron/`, `renderer/`, `shared/`)
- Typed IPC contract and process boundaries
- Capture utility process with native addons writing to libsql
- Agents utility process with Mastra.ai runtime
- Basic chat with activity context (direct SQL, no embeddings)
- Basic MCP connection (API key auth, agent calls tools)
- Apps runtime (sandboxed WebContentsView, template apps, read lane SDK)
- Renderer foundation: SideSheet + detail canvas, Chat/Context/Apps/Settings views

### Phase 2: Context + Data Layer

- Embedding/vector storage pipeline
- Full RAG retrieval (vector search, backfill, observation anchors)
- Full complexity router (local/cloud brain routing logic)
- Observational memory compression
- Retention enforcement

### Phase 3: MCP Advanced + Agent Runtime Hardening

- OAuth MCP auth (Phase 1B uses API key auth only)
- MCP advanced features: orphan cleanup, notification-driven invalidation, per-call abort
- Approval gate framework and execution audit model
- Proactive notifications
- `onMessage()` push channel from platform to apps

### Phase 4: MCP Browser + Automations

- Playwright execution service in isolated child process
- Unified execution timeline (MCP + browser + user interventions)
- Automation trigger engine and run logs

### Phase 5: Product Surfaces + App Ecosystem

- Full App Gallery UI
- Generic AI Studio export support (Phase 1B supports template apps only)
- Privacy controls and per-stream consent surfaces
- Onboarding flow (hardware detection & brain choice)

---

## Open Architecture Questions

All blocking items below are now resolved.

| # | Question | Impact |
|---|---|---|
| 2 | **Database schema** — finalized as draft for implementation: [database-schema.md](./database-schema.md). | Unblocks Phase 1 implementation. |
| 3 | **IPC contract** — finalized as Sprint 2 final draft: [ipc-contract.md](./ipc-contract.md). | Unblocks inter-process communication implementation. |

**Resolved:**
- **Mastra in utility process** → GO. PoC spike (Feb 17, 2026) passed all 7 steps. `@mastra/core` + `@mastra/libsql` initialize and run agents in Electron utility process. Streaming via MessagePort with ACK gate pattern. Crash isolation works (exit → restart → resumed). Packaged runtime resolves native modules from `app.asar.unpacked`. See [phase-1b-sprint-plan.md Sprint 0](./phase-1b-sprint-plan.md#sprint-0-complete-mastra-utility-process-spike).
- **MCP server packaging** → Bundled. Developers ship each integration with the app. Users connect, not install.
- **App Gallery hosting** → No hosted gallery for v0.1. Users upload zip files, Cowork renders locally.
- **Bundle ID rename** → No rename. Keep existing coworkai bundle ID.
- **MCP install registry** → N/A. MCP servers are bundled, not user-installed. No registry needed.
- **App-to-platform transport** → Preload IPC relay with read lane. Apps call `window.cowork.{context,user,data}.*` → preload injects trusted `appId` → `ipcRenderer.invoke('cowork:read', ...)` → main relays to Agents & RAG utility. Same IPC pattern as the main renderer. Apps can make their own external API calls (Gemini, etc.) with their own keys — the sandbox blocks platform internals, not network access.
- **Refusal message exclusion** → Custom Mastra `MemoryProcessor`. A `RefusalFilter` processor strips refusal messages from the context window before sending to the LLM, while leaving them in storage for UI display. Mastra's `lastMessages` has no metadata filtering, but the `MemoryProcessor` interface (same pattern as the built-in `ToolCallFilter`) runs after fetch and before LLM — exactly the right hook.
- **Apps runtime design** → Two-track strategy: template apps (guaranteed compatible) + generic AI Studio exports (best-effort via esbuild). esbuild bundles TSX/TS + resolves all imports (~10-100ms). Phase 1B app runtime is read-lane-only (`cowork:read`) with install-time disclosure and no app-level tool execution.

---

## See Also

| Topic | Document |
|---|---|
| LLM stack deep-dive (routing, budgets, billing) | [llm-architecture.md](./llm-architecture.md) |
| Product features & capabilities | [product-features.md](../product/product-features.md) |
| Execution phases & codebase origin | [Execution Phases](#execution-phases) / [Codebase Origin](#codebase-origin) (this doc) |
| Context pipeline (how data reaches the agent and apps) | [Context Pipeline](#context-pipeline) (this doc) |
| Design system & interaction model | [design-system.md](../design/design-system.md) |
| Cost models & pricing | [llm-strategy.md](../strategy/llm-strategy.md) |

---

## Changelog

**v1.8 (Feb 18, 2026):** Review pass — 17 fixes. Header version and IPC channel count corrected. Schema paragraph replaced with reference to `database-schema.md`. Boot sequence: "three processes" → "two utility processes"; capture boot order fixed (DB first, then addons). Data flow diagram: removed stale "(OFF, opt-in)" from keystroke/clipboard. Capture stream defaults: all five v0.1 streams now ON by default (aligned with `capture-pipeline-architecture.md`; updated `product-features.md` to match). Active window detection: added `@miniben90/x-win` as primary source, reframed `coworkai-activity-capture` as enrichment fallback. IPC observability: fixed "4 processes" → "resident processes" / "cross-process". Embedding dimensions: 512–1024 → 1024. Audio: added Whisper Small (8GB fallback). Directory structure: replaced Phase 4/5 views with Phase 1B set (Chat, Context, Settings, Integrations, Apps). MCP: added Phase 1B/3 scope markers for OAuth and advanced features. Changelog reordered to strict reverse-chronological. `repos.md`: desktop build status FAIL → PASS, dependency graph updated (coworkai-agent → Mastra).

**v1.7 (Feb 18, 2026):** Consolidated `DESKTOP_SALVAGE_PLAN.md` into this doc. Added Codebase Origin section (repo disposition, what survived from each repo, what was killed and why) and Execution Phases section (Phase 1A-5 definitions). Corrected Apps Runtime network access model: apps CAN make their own external API calls (Gemini, etc.) with their own keys — the sandbox blocks platform internals, not network access. Updated Platform SDK, Two-Track Strategy, Gemini API proxy decision, and resolved items to reflect this. `DESKTOP_SALVAGE_PLAN.md` deleted.

**v1.6 (Feb 18, 2026):** Consolidated `agent-context-pipeline.md` into this doc. Expanded Context Pipeline section with full two-layer runtime detail (Layer 1 automatic context injection + Layer 2 tool execution), agent chat flow diagram, app read lane summary with cross-reference to Apps Runtime, rejected alternatives rationale (why not agent-mediated, why not callTool, why not MCP), and "apps get data, not tools" policy. `agent-context-pipeline.md` deleted — all content now lives here or in ipc-contract.md.

**v1.5.9 (Feb 17, 2026):** Aligned Apps Runtime and Context Pipeline sections with the read-lane architecture in `agent-context-pipeline.md` v8 and `ipc-contract.md` v6. Replaced app `callTool`/permission-gate contract with typed read-only SDK (`window.cowork.{context,user,data}.*`) over `cowork:read`. Added explicit install-time disclosure requirement, trusted preload `appId` injection/scoping notes, updated feature-to-infrastructure mapping, and updated resolved-items wording for app transport/runtime design.

**v1.5.8 (Feb 17, 2026):** Wording alignment pass for app permission flow. Updated preload permission-check example to reflect in-memory grants projection (`app_permissions` → preload grants) rather than per-call DB reads. Added explicit note that permission updates must refresh preload grants before subsequent tool calls (implementation mechanism intentionally unspecified).

**v1.5.7 (Feb 17, 2026):** Independent capture streams. Updated data flow diagram and capture orchestration section to reflect streams as independent peers with denormalized app context. Each chunk/event carries its own `app_name`; `activity_session_id` is an optional correlation hint, not a structural dependency. Capture streams are independent peers correlated by overlapping timestamps.

**v1.5.6 (Feb 17, 2026):** Capture flush semantics alignment. Replaced "1000-char activity flush" with decoupled chunk behavior: 1000-char cap flushes keystroke chunk only; activity session boundaries remain focus/idle/sleep/shutdown driven. Added explicit one-directional dependency wording (activity end flushes open chunk; chunk max-length never ends activity).

**v1.5.5 (Feb 17, 2026):** Synced database and IPC blocker status with completed Sprint 1/2 design docs. Top tracker now shows 0 open blocker questions (inline observer benchmark still open). Updated "Open Architecture Questions" table language to reflect resolved state with direct links to `database-schema.md` and `ipc-contract.md`.

**v1.5.4 (Feb 17, 2026):** Synced IPC contract status from "Resolved — Draft" to "Resolved — Final Draft" after final Sprint 2 contract lock. No contract-surface changes; status alignment only.

**v1.5.3 (Feb 17, 2026):** Apps runtime permission model alignment pass. Removed direct app SDK `chat(message)` method to match product rule "Apps get tools, not agents." Apps now reach platform reasoning through `callTool('platform_chat', ...)` (agent-as-tool), with normal tool permission checks. Updated Platform SDK table, two-track generic-export guidance, Gemini proxy decision row, installation flow permission step, and the resolved decisions summary.

**v1.5.2 (Feb 17, 2026):** Replaced `keytar` with Electron `safeStorage` API for MCP credential storage — `keytar` is deprecated and archived since 2022. `safeStorage` is built-in (Electron 15+), no additional native bindings. Updated OAuth flow section, ASAR unpack list, and added `cowork.apps` namespace to preload table. Decision log entry updated.

**v1.5.1 (Feb 17, 2026):** Deferred `onMessage(callback)` from Platform SDK table — push channel moves to Phase 3. Sprint 8 / Phase 1B SDK is request/response only (`listTools`, `callTool`, `chat`, `getAppConfig`). Added cross-reference to phase-1b-sprint-plan.md exclusions list.

**v1.5 (Feb 17, 2026):** Resolved Q#1 (Mastra in utility process) — marked as GO after PoC spike passed all 7 steps (Feb 17). Moved from open items to resolved list. Updated process model blockquote from "not yet proven" to "proven." Open questions reduced from 3 to 2.

**v1.4.1 (Feb 17, 2026):** Resolved all 4 Apps runtime design questions. Two-track strategy: template apps (Cowork.ai-published Google AI Studio template, guaranteed compatible) + generic AI Studio exports (best-effort esbuild bundling). esbuild handles preprocessing + import resolution in one step. No Gemini proxy — apps use `window.cowork.chat()`, platform handles model selection + API keys. Template includes `cowork.manifest.json` for scoped permissions; generic exports fall back to user-granted permissions. Added installation flow (7 steps from zip upload to sidesheet). Removed Q#4 from open items. Open questions reduced from 4 to 3.

**v1.4 (Feb 17, 2026):** Added Apps Runtime section — was missing from the doc despite Apps being in v0.1 scope. Covers: WebContentsView rendering (sandboxed, partition-isolated, custom `cowork-app://` protocol), platform communication (preload IPC relay, decided), platform SDK API surface, per-app permission model. Added Q#4 (Apps runtime design) to open items — remaining decisions: preprocessing pipeline, Gemini API proxy, import resolution, manifest authoring. Updated Feature → Infrastructure Map with Apps Runtime cross-reference.

**v1.3.4 (Feb 17, 2026):** Resolved refusal message exclusion question. Mastra's `lastMessages` has no metadata filtering, but the `MemoryProcessor` interface solves it: a custom `RefusalFilter` processor strips refusal messages from context before sending to the LLM while leaving them in storage for UI display. Same pattern as Mastra's built-in `ToolCallFilter`. Updated Query Refusal section with resolved mechanism. Open questions reduced from 4 to 3.

**v1.3.3 (Feb 17, 2026):** Resolved App-to-platform MCP transport question. Decision: preload IPC relay — apps use `window.cowork.*` → preload → main → Agents & RAG utility, same pattern as the main renderer. Local HTTP server ruled out (unnecessary attack surface; CVE-2025-49596 demonstrated the DNS rebinding risk class). MessagePort direct channel deferred — adds setup complexity for ~1ms savings on infrequent calls. Open questions reduced from 5 to 4. Fixed stale `#9` cross-reference to refusal message exclusion (now `#4`).

**v1.3.2 (Feb 17, 2026):** Resolved 4 open questions. MCP servers are bundled (developers ship integrations, users connect) — replaced 7-step install state machine with 6-step connection flow. App Gallery: no hosted gallery for v0.1, zip upload + local render. Bundle ID: no rename, keep coworkai. MCP install registry: N/A (bundled servers, no user installation). Open questions reduced from 9 to 5.

**v1.3.1 (Feb 17, 2026):** Normative behavior pass. Added: startup failure policy (timeout + degraded boot, service states, retry with backoff, degraded mode capabilities), service health IPC contract (`system:health` + `system:retry` channels), Keychain-backed MCP credential storage (replaces plaintext token files), crash-safe notification throttling (all counters persisted to cowork.db), PID-reuse safety for MCP orphan cleanup (executable path + argv hash identity check before kill). Review fixes: renamed "Chat (proactive)" to "Proactive Notifications", screen recording marked not-in-v0.1, MCPClient lifecycle added stopped→starting transition, Phase 2 renamed to "Mandatory approval tools", Phase 4 per-server policy defined, utility-to-utility DB-only communication documented, embedding model name normalized, DB access table updated to show both SDK layers, cross-references between Embedding Pipeline and RAG Context Assembly, flow-as-tool automation context clarified.

**v1.3 (Feb 17, 2026):** Integrated 5 remaining adaptation guides (Jan, Chatbox, LobeChat, AnythingLLM, Cherry Studio). Added: database hardening (single-client access, vector index namespacing, schema migration strategy), model lifecycle (preflight fit check, download integrity, path safety), embedding pipeline (resumable ingestion, batch checkpoints, crash recovery), RAG context assembly (multi-source priority order, fillSourceWindow backfill, query refusal), agent execution model (persistent operations, instruction executor, max-step safety), multi-phase tool approval policy (6-phase hierarchy, mixed execution, approval state), MCP enhancements (orphan cleanup, client cache with dedup, notification-driven invalidation, per-call abort, 7-step install state machine), IPC observability (tracedInvoke, 4-process waterfall, typed IpcChannel constants, trace storage), automations engine (flow executor, variable substitution, block types, flow-as-tool), build configuration (ASAR unpack, post-pack patching, process-specific aliases, chunk inlining), complexity router askId grouping, explicit hardware reserve breakdown.

**v1.2 (Feb 17, 2026):** Integrated adaptation guide decisions. Added: boot sequence for three processes, Playwright execution model, IPC streaming protocol with error handling, BaseManager + @channel IPC handler pattern, preload namespace design, Mastra Memory configuration mapping, sub-agent delegation pattern, MCP connection lifecycle and OAuth flow, AI SDK v6 in key decisions table.

**v1.1 (Feb 17, 2026):** Three-provider model: Gemini direct (`@ai-sdk/google`) for free tier, OpenRouter for paid tiers, Ollama for local. Managed via AI SDK `createProviderRegistry()`. Updated system diagram, data flow, cloud brain section, complexity router, and key decisions table.

**v1 (Feb 16, 2026):** Initial draft. Consolidates process model, data flow, IPC contract, capture layer, database layer, LLM stack, memory system, agent orchestration, feature-to-infrastructure mapping, thermal management, and directory structure from six existing decision/architecture docs into a single reference.
