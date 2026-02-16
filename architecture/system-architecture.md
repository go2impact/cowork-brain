# Cowork.ai Sidecar — System Architecture

**v1.1 — February 17, 2026**
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

> **Mastra in utility process is the target architecture, not yet proven.** Each component works individually (aime-chat proves Mastra+Electron; @electron/llm proves LLM streaming in utility processes), but no one has run the full Mastra stack in an Electron utility process. A proof-of-concept spike is required before committing. Fallback options: main process (accept crash risk) or separate server process (HTTP overhead). See [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md).
| **Renderer** | All UI. Fully sandboxed — no Node.js access. Communicates exclusively via IPC. | Always | Can reload without affecting capture or agents. |
| **Playwright Child** | Browser automation for MCP Browser feature. Spawns its own Chromium instances. | On demand | Kill and respawn. No impact on anything else. |

**Why multi-process:** Native C++ addons can segfault. Embedding generation is CPU-heavy. Playwright can hang. None of these should freeze the UI or kill the app. Electron's built-in IPC connects all processes — no custom serialization protocol, no HTTP hop.

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

Three providers managed via AI SDK's `createProviderRegistry()`:

| Provider | Adapter | Role |
|---|---|---|
| **Gemini direct** | `@ai-sdk/google` | Free tier — Gemini API key only, no OpenRouter middleman |
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

---

## Agent Orchestration

**Mastra.ai** runs in the Agents & RAG utility process. Embedded directly in Electron (not client-server) — Electron 37.1.0 ships Node 22.16.0, satisfying `@mastra/libsql`'s requirement.

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

### MCP Connections

Services connect via MCP servers. The Agents & RAG utility process manages all connections.

| Concern | How it works |
|---|---|
| Connection management | OAuth flows, health monitoring, auth expiry detection |
| Tool registry | Each service exposes tools (list_tickets, send_email, etc.) |
| Scoped access | Apps declare what MCP tools they need. Platform grants per-app. |
| Disconnect handling | Action pauses, user notified with exact blocker + reconnect action |

### Safety Rails

| Condition | Action |
|---|---|
| Same tool called 3x with same input | Pause: "I seem to be stuck" |
| Task chain exceeds $1.00 in 10 min | Hard pause, tap to continue |
| Confidence drops 2+ consecutive steps | Pause, offer human review |
| Task chain > 5 minutes | Check-in: "Still working on X?" |
| Error rate > 50% in chain | Abort, show error summary |

All actions logged with undo window (default 5 min). Destructive actions always confirm first time. Money actions always require explicit approval.

---

## Feature → Infrastructure Map

How each product feature maps to the underlying infrastructure:

| Feature | Processes involved | Key infrastructure |
|---|---|---|
| **Apps** | Renderer + Agents & RAG | Apps rendered in renderer. Apps access platform via MCP tools (agent-as-tool pattern). App Gallery in renderer. |
| **MCP Integrations** | Agents & RAG | MCP SDK manages connections. Health monitoring. OAuth. |
| **Chat** (on-demand) | Renderer → Main → Agents & RAG | User message → IPC → complexity router → local/cloud brain → RAG context assembly → response → IPC → renderer |
| **Chat** (proactive) | Agents & RAG → Main → macOS | Agent detects trigger → main dispatches native notification → user accepts → execution via MCP/Browser |
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

---

## Hardware Requirements (v0.1, Mac only)

| Machine | What runs | Default mode |
|---|---|---|
| **16GB+ Apple Silicon** | Whisper on ANE + DeepSeek on GPU + full capture | User chooses local or cloud at onboarding |
| **8GB Apple Silicon** | Whisper on ANE + full capture. No local LLM (not enough RAM). | Cloud-only (no choice needed) |

---

## Key Technical Decisions (Summary)

| Decision | Choice | Why | Reference |
|---|---|---|---|
| Desktop framework | Electron | All dependencies are Node.js. Swift/Tauri = two runtimes for zero benefit. | [DESKTOP_FRAMEWORK_DECISION.md](../decisions/DESKTOP_FRAMEWORK_DECISION.md) |
| Database | libsql | Native Mastra integration, built-in vectors, one SDK, proven in Electron | [DATABASE_STACK_RESEARCH.md](../decisions/DATABASE_STACK_RESEARCH.md) |
| Native addons | Keep custom `coworkai-*` | Critical capability gaps in open-source alternatives. Electron deadlock in uiohook-napi. | [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](../decisions/NATIVE_ADDON_REPLACEMENT_RESEARCH.md) |
| Local LLM | DeepSeek-R1-8B via Ollama | Reasoning capability, Qwen base (Tagalog+EN), fits 16GB Mac | [llm-architecture.md](./llm-architecture.md) |
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
| 6 | **IPC contract** — typed channel definitions not yet designed. | Blocks inter-process communication implementation. |

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
