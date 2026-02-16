# Cherry Studio → Cowork.ai Adaptation Guide

**Purpose:** Maps the key patterns from the [Cherry Studio deep-dive](../strategy/cherry-studio-deep-dive/codex-output.md) to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our multi-process architecture.

**Audience:** Engineering (Rustan + team)

**Source material:** `strategy/cherry-studio-deep-dive/` (reverse-engineering of Cherry Studio's codebase)
**Target architecture:** `architecture/system-architecture.md`

**Why Cherry Studio?** Cherry Studio fills gaps that aime-chat doesn't cover. It's Electron + Vite + AI SDK (same foundation), but has production patterns for: IPC observability, MCP tool aggregation, multi-window state sync, multi-model UX, and multi-entry Electron builds. These five patterns are directly relevant to Cowork.ai's architecture and don't exist in aime-chat.

---

## Sections

| # | Section | Status |
|---|---|---|
| 0 | [IPC Trace Propagation](#0-ipc-trace-propagation) | Done |
| 1 | [MCP Hub Meta-Server](#1-mcp-hub-meta-server) | Done |
| 2 | [Multi-Window State Sync](#2-multi-window-state-sync) | Done |
| 3 | [Multi-Model Conversation UX](#3-multi-model-conversation-ux) | Done |
| 4 | [electron.vite Multi-Entry Build](#4-electronvite-multi-entry-build) | Done |
| 5 | [Summary](#5-summary) | Done |

---

## 0. IPC Trace Propagation

**Source:** Cherry Studio's `src/preload/index.ts` (`tracedInvoke`) + `src/main/services/NodeTraceService.ts`

Cowork.ai has more IPC hops than any other Electron app in our competitive set (Renderer → Main → Agents Utility → Playwright Child). When something goes wrong — a stream hangs, a tool call times out, a response is slow — we need to trace the full request path across all four processes. Cherry Studio solved this for two processes (renderer → main). We need to extend it to four.

### Copy

#### `tracedInvoke` — IPC-transparent trace propagation

Cherry Studio's most valuable observability pattern. A wrapper around `ipcRenderer.invoke()` that transparently propagates OpenTelemetry span context across the IPC boundary.

**How it works in Cherry Studio:**

```
Renderer creates span → extracts SpanContext → tracedInvoke() appends as last arg
    ↕ IPC
Main process monkey-patched ipcMain.handle() → detects trace payload → restores OTel context → executes handler in context.with()
```

**Cherry Studio's implementation (3 key pieces):**

1. **Preload** (`src/preload/index.ts:95-100`): `tracedInvoke(channel, spanContext, ...args)` — if `spanContext` exists, wraps it as `{ type: 'trace', context: spanContext }` and appends as the last IPC argument. If no context, falls through to normal `ipcRenderer.invoke()`.

2. **Main process** (`src/main/services/NodeTraceService.ts:33-46`): Monkey-patches `ipcMain.handle` globally. Every handler checks the last argument for `{ type: 'trace' }`. If found: `trace.wrapSpanContext(payload.context)` → `trace.setSpan(context.active(), span)` → `context.with(ctx, () => handler(event, ...cleanArgs))`. The handler executes with the renderer's span as its active parent.

3. **Argument cleanup** (line 42): `newArgs = args.slice(0, -1)` — the trace payload is stripped before the handler sees it. Business logic never knows about tracing.

**Our flow — same principle, more hops:**

```
Renderer (useChat + tracedInvoke)
    ↕ IPC (hop 1)
Main process (IPC router — trace passthrough)
    ↕ IPC (hop 2)
Agents & RAG utility process (Mastra agent → tool execution)
    ↕ child_process (hop 3)
Playwright child (browser automation)
```

What changes:
- **Main process becomes a trace relay.** Cherry Studio's main process is the final handler — it restores context and does the work. Ours restores context, then forwards to the utility process with the same context. Main adds a relay span (`ipc.relay`) showing the routing hop.
- **Utility process needs its own `tracedHandle`.** The Agents & RAG utility process receives IPC from main and needs the same monkey-patch pattern. We apply `NodeTraceService.init()` in both main and the utility process.
- **Playwright child gets trace via `child_process` message.** Different transport than Electron IPC, but same serialization: `SpanContext` is JSON-safe. We send `{ type: 'trace', context }` as part of the Playwright command message.
- **Four-process waterfall visibility.** A single user chat message produces a trace: `renderer.send` → `main.relay` → `agents.process` → `playwright.execute` (if browser action). This waterfall shows exactly where latency lives.

What stays the same:
- `tracedInvoke` function signature — identical in our preload.
- `SpanContext` serialization format — `{ traceId, spanId, traceFlags }` is JSON-safe across all transports.
- Last-argument convention — trace payload always appended last, stripped before handler.
- Backward compatibility — calls without trace context work identically.

**Reference:** [system-architecture.md — IPC Contract](./system-architecture.md#ipc-contract)

#### `IpcChannel` typed constants

Cherry Studio defines all IPC channels in `@shared/IpcChannel` — a single source of truth for channel names. Preload, main, and renderer all import from the same enum.

We adopt this directly. Our `shared/` directory (see [system-architecture.md — Directory Structure](./system-architecture.md#directory-structure)) is the same location. Channel definitions live there alongside types and constants.

### Adapt

#### SpanCacheService — persistent trace storage

Cherry Studio stores completed spans to disk (`~/.cherrystudio/trace/`) via `SpanCacheService`. Spans are cached in-memory during the conversation, then flushed to disk when the topic closes. Supports `getSpans(topicId, traceId)` for later retrieval.

We need trace storage for different reasons:
- **Cherry Studio uses it for:** A trace viewer window (`traceWindow.html`) that shows span timelines for completed conversations.
- **We use it for:** Debugging latency across four processes, safety rail audit trails (every tool call logged with timing), and the MCP Browser execution log (which already shows a timeline of actions).

What changes:
- **Storage location:** Cherry Studio uses a flat file directory. We store trace data in `cowork.db` (libsql) — it's already our single data store, and Mastra's `LibSQLStore` already persists agent traces. We extend that rather than adding a parallel file-based store.
- **Scope:** Cherry Studio traces AI SDK calls and MCP tool calls. We also trace: capture process events, complexity router decisions, budget checks, and browser automation commands. Broader surface, same span format.

### Study

#### AI SDK telemetry plugin integration

Cherry Studio's `telemetryPlugin.ts` bridges AI SDK's internal spans with the app-level OTel context. When `streamText()` runs, the plugin sets a `parentSpanContext` so AI SDK spans appear as children of the conversation span.

Worth evaluating for our Mastra integration. Mastra uses AI SDK internally — if we can set the parent context before `agent.stream()`, the entire agent execution (including tool calls, model inference, retry loops) appears as a subtree of the user's chat request span.

**Key question:** Does Mastra expose a hook for injecting OTel parent context before `agent.stream()`? If not, we may need a wrapper similar to Cherry Studio's telemetry plugin. Investigate during the Mastra-in-utility-process spike.

### Skip

#### Monkey-patching approach for production

Cherry Studio monkey-patches `ipcMain.handle` globally at app startup. This works but has fragility risks — any code that calls `ipcMain.handle` before `NodeTraceService.init()` won't be patched.

For our production build, prefer explicit `tracedHandle()` wrappers over monkey-patching. Define a `tracedHandle(channel, handler)` utility in `shared/` that both main and the utility process import. Same trace detection logic, but opt-in per handler rather than global monkey-patch.

#### `traceWindow.html` — dedicated trace viewer window

Cherry Studio has a separate HTML entry and window for viewing traces. We don't need a separate window — our MCP Browser execution log already shows the action timeline. Trace details (span waterfall, timing breakdown) integrate into the execution log's detail view.

---

## 1. MCP Hub Meta-Server

**Source:** Cherry Studio's `src/main/mcpServers/hub/` directory

Cherry Studio's MCP hub is a meta-server that aggregates tools from all active MCP servers into a single interface. An LLM can call `hub.list` to discover all available tools, `hub.inspect` to see a tool's signature, `hub.invoke` to call a single tool, or `hub.exec` to run JavaScript that orchestrates multiple tools in parallel.

This pattern is relevant to Cowork.ai's Apps feature, where third-party apps need scoped access to platform capabilities (connected services, capture context, agent reasoning) via MCP tools.

### Copy

#### Transport-agnostic client cache with dedup

Cherry Studio's `MCPService` caches `Client` instances by a hash of the full server configuration (`JSON.stringify({ baseUrl, command, args, env, id })`). If two concurrent calls request the same server, only one connection is created — `pendingClients` Map deduplicates initialization.

**Key pattern (`src/main/services/MCPService.ts`):**

```
initClient(server) →
  1. Check pendingClients[key] → if exists, return same Promise
  2. Check clients[key] → if exists, ping(1s timeout) → if healthy, return
  3. Create init Promise → store in pendingClients[key]
  4. Connect transport → store in clients[key] → delete pendingClients[key]
```

We adopt this directly for our MCP connections in the Agents & RAG utility process. Each service (Zendesk, Gmail, Slack) is a cached client with the same lifecycle. The dedup pattern is critical — our agent may fire multiple tool calls in rapid succession during startup, and we don't want parallel connection attempts to the same service.

#### Notification-driven cache invalidation

Cherry Studio registers `ToolListChangedNotification`, `ResourceListChangedNotification`, and `PromptListChangedNotification` handlers on each MCP client. When a server pushes a notification, the corresponding cache entry is cleared immediately. Next access triggers a fresh fetch.

```
MCP server sends: ToolListChangedNotification
    → client handler fires
    → CacheService.remove('mcp:list_tool:' + serverKey)
    → next listTools() call → cache miss → fresh fetch
```

We need this for service connections that change state externally (e.g., user adds a new Zendesk macro via the Zendesk web UI → the Zendesk MCP server's tool list changes → our cache needs to reflect the new macro).

#### Tool call abort semantics

Cherry Studio tracks active tool calls via `activeToolCalls: Map<string, AbortController>`. Each `callTool()` creates an `AbortController` with a unique `callId`. The `abortTool(callId)` method signals the controller, cancelling the in-flight MCP request.

This maps to our safety rails: if a tool chain exceeds the cost threshold ($1.00 in 10 min) or the user clicks "Stop", we need to abort in-flight MCP calls. The `AbortController` + `callId` pattern gives us per-call cancellation.

### Adapt

#### Hub tool aggregation → scoped tool registry for Apps

Cherry Studio's hub aggregates ALL tools from ALL active servers into one flat list. Any LLM call can access any tool. This is fine for a chat app where the user controls what servers are active.

**We need scoping.** Our Apps feature requires per-app permission grants:

| Cherry Studio (hub) | Cowork.ai (scoped registry) |
|---|---|
| All tools from all servers, flat | Per-app: only tools the app declared + platform granted |
| No access control | Permission model: app declares needs, platform grants scoped access |
| LLM sees everything | App sees only its granted tools + platform-provided resources |

What we keep from the hub pattern:
- **Tool namespacing.** Cherry Studio uses `serverId__toolName` (double underscore) to disambiguate tools from different servers. We use the same convention.
- **Tool listing with pagination.** Cherry Studio's `listTools(tools, { limit, offset })` returns paginated results. Useful when an app has access to dozens of tools.
- **JSDoc generation for tool signatures.** Cherry Studio's `schemaToJSDoc()` converts a tool's JSON Schema into a human-readable function signature. This feeds the LLM's understanding of available tools. We use the same approach for our agent's system prompt.

What changes:
- **Filtering layer.** Between "all available tools" and "what the app sees", we add a permission filter. Each app's MCP session only includes tools from its granted scopes.
- **No `exec` tool.** Cherry Studio's `hub.exec` runs arbitrary JavaScript in a worker thread with access to all tools. This is a security concern for our multi-tenant app model. Apps call tools individually via MCP, not via arbitrary code execution.

#### Worker thread timeout → agent task timeout

Cherry Studio's 60-second worker thread timeout with `AbortController`-based tool cancellation is the right pattern, wrong location. They use it for the hub's JavaScript execution sandbox.

We apply the same timeout + abort pattern to our agent task execution:
- **Agent task timeout:** Configurable per-task (default 5 min, see safety rails in system-architecture.md). If exceeded, abort in-flight tools, notify user.
- **Per-tool timeout:** Cherry Studio uses per-server timeout (`server.timeout * 1000`, default 60s) with `resetTimeoutOnProgress` for long-running tools. We adopt the same — some MCP operations (e.g., fetching 50 Zendesk tickets) take longer than a simple API call.

### Study

#### Built-in server factory pattern

Cherry Studio's `createInMemoryMCPServer(name)` instantiates built-in MCP servers (memory, filesystem, fetch, browser, hub) via `InMemoryTransport` — zero network overhead, same MCP protocol.

**Evaluate for:** Our platform-provided MCP tools. Instead of exposing platform capabilities through custom code, we could run an in-memory MCP server that exposes capture context, agent-as-tool, and connected service data as standard MCP tools/resources. Third-party apps then use the standard MCP client protocol, and we get protocol-level access control for free.

**Trade-off:** In-memory MCP server adds abstraction overhead vs. direct function calls. But it gives us a clean protocol boundary between the platform and apps, and makes the app developer experience consistent (everything is MCP).

### Skip

#### Hub `exec` tool — arbitrary JavaScript execution

Cherry Studio's hub runs user-provided JavaScript in a worker thread with access to all tools via `mcp.callTool()`. This is powerful for LLM-driven orchestration but opens a wide attack surface.

We skip this. Our agent orchestrates tool calls directly via Mastra's multi-step execution loop (see [aime-adaptation-guide.md — Section 1](./aime-adaptation-guide.md#1-mastra-agent-system)). No arbitrary code execution from external inputs.

#### Implicit trust for tool execution

Cherry Studio has no per-tool policy gates — any active tool can be called by any LLM interaction. We require explicit approval for destructive actions and per-app scoping for third-party access (see [system-architecture.md — Safety Rails](./system-architecture.md#safety-rails)).

---

## 2. Multi-Window State Sync

**Source:** Cherry Studio's `src/renderer/src/services/StoreSyncService.ts` (renderer) + `src/main/services/StoreSyncService.ts` (main)

Cherry Studio runs multiple renderer windows (main window, mini window, selection toolbar, selection action, trace window) that share Redux state. When a user changes settings in the main window, the mini window reflects the change immediately.

Cowork.ai currently has a single renderer, but the pattern is relevant for two reasons: (1) we may add auxiliary windows (quick-action toolbar, notification detail, MCP Browser detached view), and (2) the sync mechanism itself is a clean pattern for keeping any two state consumers consistent via IPC.

### Copy

#### Action-level sync with loop prevention

Cherry Studio's sync is elegant: a Redux middleware intercepts dispatched actions, checks if the action's type matches a whitelist, and forwards matching actions to main. Main rebroadcasts to all other windows. The receiving window dispatches the action locally with `meta.fromSync = true` — the middleware sees this flag and does NOT re-broadcast, preventing infinite loops.

**Cherry Studio's sync flow:**

```
Window A dispatches action (type: 'settings/setTheme')
    ↓
StoreSyncService middleware checks:
  - Is action.type in syncList? ('settings/' prefix → yes)
  - Is action.meta.fromSync === true? (no → this is a local dispatch)
    ↓
Forward to main: window.api.storeSync.onUpdate(action)
    ↓
Main StoreSyncService broker:
  - Tracks all subscribed windows (BrowserWindow IDs)
  - Rebroadcasts to all windows EXCEPT the sender
  - Adds meta.fromSync = true
    ↓
Window B receives: dispatch(action with meta.fromSync = true)
    ↓
StoreSyncService middleware checks:
  - Is action.meta.fromSync === true? (yes → do NOT rebroadcast)
  - Dispatches locally, state updates
```

**Cherry Studio's whitelist (`src/renderer/src/store/index.ts`):**

```typescript
storeSyncService.setOptions({
  syncList: ['assistants/', 'settings/', 'llm/', 'selectionStore/', 'note/']
})
```

Only these Redux slice prefixes sync across windows. Heavy slices (`messages`, `messageBlocks`) are excluded — they're per-window state.

### Adapt

#### Sync scope — our state boundaries are different

Cherry Studio syncs Redux slices between multiple renderer windows. We sync between the renderer and the Agents & RAG utility process (which needs to know user settings, active connections, and tier/budget state).

| Cherry Studio sync | Our sync |
|---|---|
| Window A ↔ Main ↔ Window B (all renderers) | Renderer ↔ Main ↔ Agents & RAG utility |
| Syncs: assistants, settings, LLM config, notes | Syncs: user preferences, connection state, budget/tier, active automations |
| Purpose: UI consistency across windows | Purpose: agent has up-to-date user context |

What changes:
- **Direction is mostly one-way.** User changes settings in the renderer → main relays to agents. Agent status changes (connection health, budget usage) → main relays to renderer. There's no two-window-same-direction case.
- **No Redux in the utility process.** The agents process doesn't run Redux. It receives state updates as plain objects via IPC and stores them in its working context. The sync middleware pattern is renderer-side only; the main-side broker adapts for the utility process.

### Study

#### Subscription model for window lifecycle

Cherry Studio's main-side `StoreSyncService` tracks window subscriptions. When a window is created, it subscribes (`subscribe(windowId)`). When destroyed, it unsubscribes. The broker only sends to live windows.

Study this for our utility process lifecycle. If the Agents & RAG utility process crashes and restarts (auto-restart via Electron), it needs to re-subscribe and receive a full state snapshot — not just incremental actions. Cherry Studio doesn't handle this case (renderer windows don't crash-restart), but we need to.

### Skip

#### Mini window's tight coupling to main window state

Cherry Studio's mini window (`miniWindow.html`) initializes `window.keyv` and `storeSyncService.subscribe()` at entry — it's tightly coupled to the main app's provider/model state (API keys, model catalog). The mini window even loads `KeyvStorage` for API key rotation.

We don't need this level of coupling for auxiliary windows. Our auxiliary windows (if any) would be lightweight — a quick-action toolbar doesn't need the full provider stack.

---

## 3. Multi-Model Conversation UX

**Source:** Cherry Studio's `src/renderer/src/store/thunk/messageThunk.ts` + `src/renderer/src/pages/home/Messages/MessageGroup.tsx`

Cherry Studio lets users @-mention multiple models in a single message. The platform sends the same prompt to each model in parallel, groups the responses by `askId`, and renders them in tabs/cards/grid. The user can compare outputs and mark the "useful" one.

This pattern is relevant to Cowork.ai's complexity router and "retry with smarter model" UX.

### Adapt

#### `askId` fan-out → "retry with smarter model" grouping

Cherry Studio creates one assistant message stub per model before any generation starts:

```
User sends: "Draft a reply" with @mentions [gemini-flash, claude-sonnet]
    ↓
dispatchMultiModelResponses():
  1. Create msg_A { askId: userMsgId, model: gemini-flash, status: pending }
  2. Create msg_B { askId: userMsgId, model: claude-sonnet, status: pending }
  3. Queue one generation task per model via topic-based PQueue
    ↓
Grouped in UI: messages with same askId render together
```

**We don't do @-mention model selection** (users pick a tier, not a model). But the `askId` grouping pattern maps to our "retry with smarter model" flow:

```
User sends: "Draft a reply"
    ↓
Complexity router → DeepSeek-R1 (local, free)
    ↓
Response arrives with "Retry with smarter model? (~$0.03)" button
    ↓
User clicks retry
    ↓
Create msg_B { askId: originalMsgId, model: gemini-flash, status: pending }
    ↓
Same prompt → cloud model → response arrives
    ↓
Grouped in UI: both responses grouped by askId, user sees comparison
```

What changes:
- **Sequential, not parallel.** Cherry Studio fires all models simultaneously. We fire the local model first, then upgrade on request. The grouping mechanism is the same, but generation is sequential.
- **Simpler UX.** Cherry Studio shows fold/horizontal/grid layouts with model switching. We show two responses stacked: "Local response" and "Cloud response" with a visual quality comparison. No grid, no tab switching.
- **Budget integration.** Each retry shows the cost before execution. The `askId` group tracks cumulative cost for the entire fan-out.

### Study

#### Topic-based queue for generation concurrency

Cherry Studio uses `PQueue` instances per topic (`getTopicQueue(topicId)`) to manage concurrent generation requests. This prevents a single topic from monopolizing all inference capacity.

**Evaluate for:** Our agent execution queue. If a user triggers multiple actions (proactive notification + manual chat + automation), each needs an execution slot. A topic-based queue prevents one long-running agent task from blocking others.

**Open question:** What's the right concurrency per topic? Cherry Studio leaves it to PQueue defaults (which may mean unlimited). We need explicit limits based on our model capacity (one local inference at a time on GPU, cloud calls can parallelize).

### Skip

#### Full multi-model fan-out UX

Cherry Studio's multi-model UI supports fold/horizontal/grid display modes, per-model tabs, "useful" message tagging, and style settings (`multiModelMessageStyle`, `foldDisplayMode`, `gridColumns`). This is overengineered for our use case.

We have exactly two models in view at most (local response + cloud retry). A simple stacked comparison with a "Use this one" button is sufficient. No grid, no tabs, no configurable display modes.

#### @-mention model selection

Cherry Studio's `MentionModelsInput` lets users type `@` to pick models from a dropdown. This is developer-friendly but wrong for our audience. Cowork.ai abstracts model selection entirely — the complexity router picks, the user sees tier labels (Free/Boost/Pro/Max), not model names.

---

## 4. electron.vite Multi-Entry Build

**Source:** Cherry Studio's `electron.vite.config.ts`

Cherry Studio uses `electron-vite` to build a single Electron app with 5 separate HTML entry points, each loading its own React app. The build produces separate bundles for main, preload, and renderer (with the renderer containing all 5 entries).

This is relevant to Cowork.ai because we'll need at least 2 entry points (main renderer + potential auxiliary windows), and the build configuration for process-specific aliases and chunk splitting is directly applicable.

### Copy

#### Process-specific path aliases

Cherry Studio's `electron.vite.config.ts` defines separate aliases per build target:

```
main:     @main → src/main, @shared → src/shared, @logger → packages/logger
preload:  @shared → src/shared
renderer: @renderer → src/renderer/src, @shared → src/shared, @logger → packages/logger
```

This prevents accidental cross-process imports. A renderer file can't `import '@main/...'` because that alias doesn't exist in the renderer build.

We adopt this for our directory structure:

```
main:     @main → src/electron, @core → src/core, @shared → src/shared
preload:  @shared → src/shared
renderer: @renderer → src/renderer, @shared → src/shared
workers:  @core → src/core, @shared → src/shared
```

Note: Cherry Studio has `@main` → `src/main`, but our main process code is in `src/electron/` (wiring only). The `@core` alias maps to our business logic directory that has zero Electron dependency.

#### Main process chunk inlining

Cherry Studio disables chunk splitting in the main process build:

```
output.manualChunks = undefined
output.inlineDynamicImports = true
```

This ensures the main process loads as a single file — no async chunk loading, no race conditions on startup. Important for Electron's main process which must be ready before creating windows.

We adopt this. Our main process is thin (~4 files of wiring), so a single output file is ideal.

### Adapt

#### Multi-entry HTML → our window types

Cherry Studio's 5 entries:

| Entry | Purpose | Our equivalent |
|---|---|---|
| `index.html` | Main app window | Main renderer (Chat, Apps, MCP Browser, etc.) |
| `miniWindow.html` | Quick assistant popup | Potential: quick-action overlay (v0.2+) |
| `selectionToolbar.html` | Text selection floating toolbar | Not needed (we don't intercept text selection) |
| `selectionAction.html` | Selection action panel | Not needed |
| `traceWindow.html` | Trace/debug viewer | Not needed (trace integrates into MCP Browser execution log) |

For v0.1, we likely have a single entry (`index.html`). The multi-entry config is ready to extend when we add auxiliary windows.

What changes:
- **Fewer entries, same build config.** electron-vite's multi-entry pattern scales from 1 to N entries without config changes — just add another HTML file and its entry point.
- **No shared state complexity in v0.1.** Cherry Studio's multi-entry necessitated `StoreSyncService` (Section 2). With one entry, we don't need cross-window sync yet.

### Study

#### electron-vite vs. our current build

Cherry Studio uses `electron-vite` (Vite-based, built-in HMR for renderer, process-aware config). Our existing codebase (from the salvage plan) may use a different build tool.

**Evaluate during Phase 1:** Is `electron-vite` the right choice for our build? Key considerations:
- **HMR for renderer:** Instant feedback during UI development. Major productivity win.
- **Process-aware config:** Separate Vite configs per process, with aliases that prevent cross-process imports.
- **Native addon compatibility:** Does electron-vite handle `.node` files (our `coworkai-*` addons) correctly in the main/utility process builds?

If the existing codebase already uses `electron-builder` + `webpack`, the migration to `electron-vite` is a separate decision. Study Cherry Studio's config as the reference implementation.

### Skip

#### Permissive CSP in HTML entries

Cherry Studio's CSP headers include `'unsafe-eval'`, `'unsafe-inline'`, and wildcard connect-src. Their `webPreferences` include `sandbox: false`, `webSecurity: false`, and `allowRunningInsecureContent: true`.

We skip all of this. Our renderer runs with `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` (see [system-architecture.md — Process Model](./system-architecture.md#process-model)). Security posture is non-negotiable for a desktop app with access to work services and activity data.

---

## 5. Summary

### Copy These Patterns

| Pattern | Cherry Studio source | Cowork.ai target | Why |
|---|---|---|---|
| `tracedInvoke` IPC trace propagation | `src/preload/index.ts:95-100` + `src/main/services/NodeTraceService.ts:33-46` | `src/shared/tracedInvoke.ts` + `src/electron/main.ts` + `src/electron/agents.worker.ts` | 4-process trace waterfall for debugging latency and failures |
| `IpcChannel` typed constants | `@shared/IpcChannel` | `src/shared/ipc-channels.ts` | Single source of truth for all IPC channel names |
| MCP client cache with dedup | `src/main/services/MCPService.ts` (clients + pendingClients Maps) | `src/core/mcp/client-cache.ts` | Prevents duplicate connections during rapid agent tool calls |
| Notification-driven cache invalidation | `MCPService.setupNotificationHandlers()` | Same pattern in our MCP connection manager | Fresh tool lists when external services change |
| Tool call abort semantics | `MCPService.activeToolCalls` + `AbortController` per call | `src/core/mcp/tool-executor.ts` | Per-call cancellation for safety rails and user "Stop" |
| Action-level multi-window sync | `StoreSyncService` (renderer middleware + main broker) | `src/core/state-sync/` | Renderer ↔ Agents utility state consistency |
| Process-specific path aliases | `electron.vite.config.ts` aliases | `electron.vite.config.ts` | Prevent cross-process import accidents |
| Main process chunk inlining | `output.inlineDynamicImports = true` | Same in our main build config | Single-file main process, no async loading races |

### Study These

| Pattern | What to evaluate | When |
|---|---|---|
| AI SDK telemetry plugin for OTel | Can we set parent span context before `agent.stream()`? | Mastra-in-utility-process spike |
| In-memory MCP server for platform tools | Platform capabilities as MCP server vs. direct function calls | Apps feature design (Phase 2+) |
| Window lifecycle subscription for crash-restart | Utility process re-subscribe + full state snapshot on restart | Agents process resilience design |
| Topic-based generation queue | Concurrency limits per task type (local=1, cloud=parallel) | Agent execution architecture |
| electron-vite for build system | Native addon compat, HMR, migration cost from current build | Phase 1 build setup |

### Skip These

| Pattern | Why skip |
|---|---|
| Monkey-patching `ipcMain.handle` globally | Fragile — prefer explicit `tracedHandle()` wrappers |
| Dedicated trace viewer window | Trace data integrates into MCP Browser execution log |
| Hub `exec` tool (arbitrary JS in worker) | Security risk for multi-tenant app model |
| Implicit trust for tool execution | We require per-tool approval and per-app scoping |
| Full multi-model fan-out UX (grid/tabs/fold) | We only compare 2 responses (local + cloud retry) |
| @-mention model selection | Users pick tiers, not models |
| `selectionToolbar.html` / `selectionAction.html` | We don't intercept text selection |
| Permissive `webPreferences` (`sandbox: false`, `webSecurity: false`) | Security posture is non-negotiable |
| Mini window's tight coupling to provider state | Auxiliary windows should be lightweight |

### What Cherry Studio Has That aime-chat Doesn't (Our Gap-Fillers)

| Gap | aime-chat | Cherry Studio | Our approach |
|---|---|---|---|
| **IPC observability** | No tracing across IPC | Full OTel propagation (renderer → main) | Extend to 4 processes |
| **MCP tool aggregation** | Basic MCPClient per server | Hub meta-server with namespacing, caching, abort | Scoped registry with per-app permissions |
| **Multi-window state** | Single window | Action-level sync with loop prevention | Renderer ↔ utility process sync |
| **Multi-model comparison** | Single model per conversation | askId fan-out, grouped rendering | "Retry with smarter model" grouping |
| **Multi-entry Electron build** | Single entry | 5 HTML entries, process-aware aliases | Start with 1, config ready for N |

---

## Changelog

**v1 (Feb 16, 2026):** Initial draft. All 5 sections complete: IPC Trace Propagation, MCP Hub Meta-Server, Multi-Window State Sync, Multi-Model Conversation UX, electron.vite Multi-Entry Build.
