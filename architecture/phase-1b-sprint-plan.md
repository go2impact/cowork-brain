# Phase 1B Sprint Plan: Core Runtime Foundation

| | |
|---|---|
| **Status** | Active |
| **Last Updated** | 2026-02-17 |
| **Owner** | Rustan |
| **Prerequisite** | Phase 1A complete (coworkai-desktop PR #8) |
| **Goal** | Working Electron app with two utility processes, capture writing to libsql, basic chat with recent-activity context (direct SQL, no embeddings), basic MCP connection (agent can call tools), basic apps runtime (render an app in sandboxed WebContentsView), sidecar folder structure |
| **Related** | [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md), [system-architecture.md](./system-architecture.md), [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md), [Spike PR (DO NOT MERGE)](https://github.com/go2impact/coworkai-desktop/pull/9) |

---

## Blocking Open Questions

From [system-architecture.md](./system-architecture.md) — Phase 1B cannot ship without resolving all three:

| # | Question | Status | Resolution path |
|---|---|---|---|
| 1 | Mastra in utility process | **Resolved — GO** | All 7 spike steps passed (Feb 17). Utility process viable for Mastra + libsql + MessagePort streaming. See `coworkai-desktop/docs/plans/mastra-utility-process-viability-spike-result.md` |
| 2 | Database schema | **Resolved — Draft** | Full DDL, table ownership map, rationale, and TypeScript row types in [database-schema.md](./database-schema.md) |
| 3 | IPC contract | **Resolved — Final Draft** | 34 channels, 9 namespaces, full Zod schemas, and locked Phase 1B design decisions. See [ipc-contract.md](./ipc-contract.md) |

---

## Phase Acceleration Note

This sprint plan pulls the agents utility skeleton, basic chat, basic MCP, basic apps runtime, and renderer foundation forward into Phase 1B. [DESKTOP_SALVAGE_PLAN.md](../decisions/DESKTOP_SALVAGE_PLAN.md) originally placed agent runtime in Phase 3 and product surfaces in Phase 5. The rationale: without a basic chat round-trip and working app runtime, there's no way to validate that the multi-process architecture actually works end-to-end. A capture process writing to a database with no consumer isn't a useful milestone. The salvage plan's phase definitions have been updated to reflect this expanded Phase 1B scope.

Embedding pipeline, full RAG, complexity router, advanced MCP features, and automations remain in later phases.

---

## Renderer Stack Decisions

UI foundation decisions derived from reverse-engineering all 6 reference apps in [research/deep-dives/](../research/deep-dives/) (AIME Chat, AnythingLLM, Chatbox, Cherry Studio, Jan, LobeChat). These are locked — a frontend dev can build against them independently while Sprints 0-5 run in parallel.

| Layer | Decision | Rationale |
|---|---|---|
| **Framework** | React 19 + TypeScript | Already decided. 6/6 reference apps use React. |
| **Bundler** | Vite (electron-vite) | Already in coworkai-desktop. AIME Chat, Chatbox, Cherry Studio all use electron-vite. |
| **Component library** | shadcn/ui (Radix primitives) + M3 token overrides | Radix provides accessible behavior (keyboard nav, focus management, ARIA). shadcn wraps Radix in Tailwind classes you own. M3 tokens replace shadcn's defaults via Tailwind config. Only approach that doesn't fight M3 — AntD apps (Cherry Studio, LobeChat) all struggle with token conflicts. |
| **CSS** | Tailwind 4 (M3 CSS variables → Tailwind config) | Already decided. 5/5 reference apps with real UI use Tailwind. M3 color roles, radii, elevation, and type scale defined as CSS variables, mapped to Tailwind utilities. |
| **Theming** | CSS variables + Tailwind config, dark theme first | M3 color roles as CSS vars, toggled via `data-theme` attribute. Per [design-system.md](../design/design-system.md). Cherry Studio + Chatbox validate this pattern. |
| **State management** | Zustand v5 | 3/6 reference apps (AIME Chat, Chatbox, Jan). Lightweight, no boilerplate, works naturally with Electron IPC. |
| **Chat state** | `@ai-sdk/react` v3 `useChat()` with IPC transport | Already decided. Interface contract between renderer and agents process. |
| **Async/cache** | TanStack Query (React Query) | For caching IPC responses (capture status, settings, health). Avoids manual cache invalidation. LobeChat validates this pattern for desktop sync. |
| **Routing** | React Router v7 | 4/6 reference apps use React Router. 4 views (Chat, Context, Apps, Settings) — no need for TanStack Router complexity. |
| **Icons** | Lucide React | Already stated in [prototype-brief.md](../design/prototype-brief.md). AIME Chat uses it. |
| **Animation** | Motion v12 (Framer Motion) | AIME Chat + Cherry Studio use it. Needed for M3 motion specs. |
| **Message rendering** | react-markdown + remark-gfm | Renders AI responses: markdown → formatted text, tables, lists. Standard across all reference apps. |
| **Math rendering** | remark-math + rehype-katex | Renders math equations in AI responses (e.g. `$E = mc^2$`). Chatbox + Cherry Studio both use this. |
| **Code highlighting** | Shiki | Syntax-highlights code blocks in AI responses. AIME Chat + Cherry Studio use it. More accurate than Prism, theme-aware. |
| **IPC bridge** | `window.cowork.*` namespaced preload API | Already designed in [system-architecture.md](./system-architecture.md). Typed via Zod schemas from IPC contract. |

### Full stack tree

```
React 19 + TypeScript
├── Vite (electron-vite)
├── shadcn/ui (Radix) + M3 token overrides
├── Tailwind 4 (M3 CSS variables → Tailwind config)
├── Zustand v5 (app state)
├── @ai-sdk/react v3 useChat() (chat state)
├── TanStack Query (IPC response caching)
├── React Router v7 (4 views: Chat, Context, Apps, Settings)
├── Lucide React (icons)
├── Motion v12 (animations)
├── react-markdown + remark-gfm (message rendering)
├── remark-math + rehype-katex (math in AI responses)
├── Shiki (code highlighting in AI responses)
└── window.cowork.* (typed IPC bridge)
```

### Parallelizable frontend task

The shadcn/ui + M3 token override work can run **fully in parallel** with Sprints 0-5. A frontend dev copies 19 shadcn components into the project and re-skins them with M3 token classes per [ui-component-task-brief.md](./ui-component-task-brief.md). Deliverable: `src/renderer/components/ui/` folder with Tailwind config, CSS variable file (dark + light), and restyled components. Integrated into the Electron renderer in Sprint 4.5.

---

## Sprint Breakdown

Each sprint is estimated at 1-2 days for a single developer. The label "Sprint N" replaces "Day N" — these are work units, not calendar promises.

### Sprint 0 (Complete): Mastra Utility Process Spike

**Repo:** coworkai-desktop (`src/app/main/spikes/mastraUtilitySpike/`)
**Blocks:** Everything else
**Result: GO** — all 7 steps passed (Feb 17, 2026).

The 7-step PoC from [MASTRA_ELECTRON_VIABILITY.md](../decisions/MASTRA_ELECTRON_VIABILITY.md):

1. Minimal Electron app + utility process — **PASS**
2. Import `@mastra/core` + `@mastra/libsql` in utility process — **PASS**
3. Init Mastra with one agent, LibSQLStore, LibSQLVector pointing at local .db file — **PASS**
4. `agent.generate()` from main via IPC, get response back — **PASS**
5. `agent.stream()` from utility to renderer via MessagePort — **PASS** (81 chunks, renderer ACK via MessagePort bridge + ACK gate)
6. Kill utility process — verify app survives, re-fork and resume — **PASS** (exit 99 isolated, restart + second generate returned `RECOVERED`)
7. Package with Electron Forge — verify ASAR unpack + native module resolution — **PASS** (packaged runtime re-ran full spike, `@libsql/darwin-arm64/index.node` resolved from `app.asar.unpacked`)

**Key finding for Sprint 5:** Streaming requires a MessagePort bridge with ACK gate between utility and renderer. Pattern proven in spike — port directly into `agents.worker.ts`.

**Evidence:**
- Spike PR: [go2impact/coworkai-desktop#9](https://github.com/go2impact/coworkai-desktop/pull/9) (DO NOT MERGE — spike code only, production implementation built fresh in Sprints 3-5)
- Result doc: `coworkai-desktop/docs/plans/mastra-utility-process-viability-spike-result.md`
- Direct run log: `coworkai-desktop/logs/mastra-utility-spike-20260217-0441-renderer-port-direct.log`
- Packaged runtime log: `coworkai-desktop/logs/mastra-utility-spike-20260217-0442-packaged-runtime.log`

---

### Sprint 1 (Complete): Database Schema Design

**Repo:** cowork-brain (design doc)
**Blocks:** Sprint 4, Sprint 5
**Result: DONE** — Draft schema document delivered at [database-schema.md](./database-schema.md).

Design the new schema from [product-features.md](../product/product-features.md). Three ownership zones per [system-architecture.md § Database Hardening](./system-architecture.md#database-hardening):

| Owner | Tables |
|---|---|
| **Capture process** | `activity_sessions`, `keystroke_chunks`, `clipboard_events`, `focus_sessions`, `browser_sessions` |
| **Agents process** | `embedding_queue`, `agent_operations`, `automations`, `mcp_servers`, `mcp_connection_state`, `tool_policies`, `notification_log`, `installed_apps`, `app_permissions` |
| **Mastra (internal)** | LibSQLStore tables (conversation, working memory, observations), LibSQLVector indexes (`embed_activity`, `embed_conversation`, `embed_observation`) |

**Work:**
- SQL DDL for capture tables (5 v0.1 input streams)
- SQL DDL for agent tables (agent orchestration + MCP + automations from system-architecture.md)
- TypeScript row types for each table
- Retention policy per table (capture data has TTL, agent state is permanent)
- Process ownership map (no cross-process ALTER)

**Does NOT include:** Mastra's internal tables — managed by LibSQLStore/LibSQLVector. We define the DB file path and vector index names only.

**Output:** `architecture/database-schema.md` — DDL + types + ownership map. Reference for all implementation.

---

### Sprint 2 (Complete): IPC Contract + Shared Types

**Repo:** cowork-brain (design doc)
**Blocks:** Sprint 4, Sprint 5
**Result: DONE** — Final draft contract delivered at [ipc-contract.md](./ipc-contract.md).

Full channel inventory + Zod schemas. [system-architecture.md § IPC Contract](./system-architecture.md#ipc-contract) defines the patterns — this sprint enumerates every channel.

**Channels by namespace** (full inventory — Phase 1B implements all except `browser:*` and `automation:*`):

| Namespace | Channels | Direction | Phase 1B? |
|---|---|---|---|
| `capture:*` | `getStatus`, `toggleStream`, `getStreamConfig` | Main ↔ Capture | Yes (Sprint 4) |
| `chat:*` | `sendMessage`, `streamResponse`, `getThreads`, `getThread` | Renderer → Main → Agents | Yes (Sprint 5) |
| `context:*` | `query`, `getRecentActivity`, `getContextCard` | Renderer → Main → Agents | Yes (Sprint 5) |
| `mcp:*` | `connect`, `disconnect`, `listConnections`, `listTools`, `getStatus` | Renderer → Main → Agents | Yes (Sprint 7) |
| `apps:*` | `install`, `list`, `remove`, `callTool`, `listTools`, `getAppConfig` | Renderer → Main → Agents | Yes (Sprint 8) |
| `app:*` | `getSettings`, `setTheme` | Renderer → Main | Yes (Sprint 6) |
| `system:*` | `health`, `retry` | Main → Renderer (health), Renderer → Main (retry) | Yes (Sprint 6) |
| `browser:*` | `execute`, `getState`, `stop`, `getExecutionLog` | Renderer → Main → Agents → Playwright | Phase 4 |
| `automation:*` | `create`, `update`, `delete`, `trigger`, `getRunLog` | Renderer → Main → Agents | Phase 4 |

**For each channel:** direction, Zod schema for request payload, Zod schema for response, which process handles it.

**Output:** Design for `src/shared/ipc-channels.ts` (typed constants) and `src/shared/ipc-schemas.ts` (Zod schemas). Documented in `architecture/ipc-contract.md` (new doc — keeps system-architecture.md from growing further).

---

### Sprint 3 (Complete): Folder Restructure + libsql Swap

**Repo:** coworkai-desktop
**Blocks:** Sprint 4, Sprint 5
**Depends on:** Spike decision (resolved — GO)
**Result: DONE** — PR [#10](https://github.com/go2impact/coworkai-desktop/pull/10) (Feb 17, 2026). All validation gates passed.

Transform coworkai-desktop from gutted-tracker to sidecar folder structure. Structural foundation — no business logic yet.

**Work:**

1. **Create Phase 1B directory structure** (additional directories added in later phases as needed):
   ```
   src/core/{agents,automations,capture,chat,mcp,memory,store}/
   src/electron/{main.ts,preload.ts,capture.worker.ts,agents.worker.ts}
   src/renderer/views/{Chat,Apps,Context,Settings}/
   src/shared/{ipc-channels.ts,ipc-schemas.ts,types/}
   ```

2. **Move existing main process code** into `src/electron/main.ts`

3. **Replace better-sqlite3 with libsql:**
   - `npm uninstall better-sqlite3`
   - `npm install libsql` (sync API for capture process — high-frequency writes)
   - `npm install @libsql/client` (async API for agents process)
   - Update `forge.config.ts` rebuild list: remove `better-sqlite3`, add `libsql`
   - Update DB path config (`configs/database.ts`) to use libsql client

4. **Create empty entry points** for utility processes (`capture.worker.ts`, `agents.worker.ts`)

5. **Validate:** `npm install` → `tsc --noEmit` → `npm start`

**Output:** Clean folder structure, libsql installed, build passes, app launches (does nothing new yet).

**What was actually implemented (deviations from plan):**
- `src/renderer/views/` directories deferred to Sprint 6 (collides with existing renderer HTML entry points)
- Mastra utility spike removed from build path (files kept on disk for reference)
- Logger `fs-extra` → native `fs`; `fs-extra` added as explicit devDependency for CSS generator
- `src/core/store/database.ts` created with `DB_FILENAME` and `toDbUrl()` helper
- `src/shared/types/index.ts` barrel re-exports from `src/app/common/types/`

---

### Sprint 4 (Complete): Capture Utility Process

**Repo:** coworkai-desktop
**Blocks:** Sprint 4.5
**Depends on:** Sprint 1 (schema) + Sprint 2 (IPC contract) + Sprint 3 (folder structure + libsql)
**Result: DONE** — PR [#11](https://github.com/go2impact/coworkai-desktop/pull/11) (Feb 17, 2026).
**Reference:** [capture-pipeline-architecture.md](./capture-pipeline-architecture.md) — full behavioral specification for the capture pipeline

Stand up the capture worker — native addons loading, event-driven activity tracking, keystroke chunking, clipboard detection, supervisor lifecycle with restart backoff, permission gating, and crash recovery. Scope expanded significantly beyond initial plan to include supervisor state machine, setup permission wiring, flush-coupling alignment, and simulation mode for deterministic testing.

**Work:**

1. **Native addon dependencies and packaging:**
   - `@engineering-go2/coworkai-activity-capture` (vendored tarball for packaging-safe install)
   - `@engineering-go2/coworkai-keystroke-capture` (vendored tarball)
   - `forge.config.ts` rebuild list: `libsql` + both capture addons
   - Verified against Electron 37 ABI

2. **Capture worker (`capture.worker.ts`) — full command lifecycle:**
   - Command router: `init`, `start`, `stop`, `health`, `shutdown`, `toggleStream`, `getStatus`, `getStreamConfig`, `simulate`
   - Init: open libsql sync client → `cowork.db`, apply pragmas (WAL, busy_timeout=5000, foreign_keys=ON), run CREATE TABLE IF NOT EXISTS for all 5 capture tables, run additive migrations, recover stale open sessions, init native addons
   - Start: subscribe to x-win active window events (100ms interval), seed initial activity session, start keystroke capture, start chunk idle flush poller (500ms)
   - Stop/shutdown: unsubscribe, flush open chunk, finalize current session, (shutdown only) close DB

3. **Activity stream — event-driven, not poll-driven:**
   - Primary source: `@miniben90/x-win` `subscribeActiveWindow()` fires on each poll
   - Enrichment: `@engineering-go2/coworkai-activity-capture` supplements missing window title/URL (app name match required, case-insensitive)
   - Parent-first session durability: INSERT `activity_sessions` row at session start (`ended_at = NULL`) before any dependent chunk/clipboard writes
   - Heartbeat: update `last_observed_at` while session active (throttled to 1s)
   - Session boundary: any change in (appName, bundleId, windowTitle, browserUrl) triggers finalize → flush chunk → new session
   - Finalization triggers: context change, stop, shutdown, all activity streams toggled OFF

4. **Keystroke stream — independent flush semantics:**
   - Keystroke chunker with debounce (1200ms idle) + max printable length (1000 chars)
   - Single ordered `chunk_text` field with interleaved printable chars and `<Control>` tokens — no separate `special_keys` column
   - Full key normalization: SPACE/SPACEBAR → space char, RETURN/ENTER → `<Enter>`, unknown multi-char keys preserved as `<key>` tokens (never dropped)
   - Chunk flush never ends activity session (one-directional: activity end flushes chunk, not vice versa)
   - Idle poller at 500ms checks for 1200ms since last key event

5. **Clipboard stream — hotkey-triggered with OS clipboard read:**
   - Detection inside keystroke `onKey` callback (Cmd/Ctrl+C/V/X, DOWN only, non-repeat)
   - OS clipboard read: `pbpaste` (macOS), `powershell Get-Clipboard` (Windows)
   - Copy/cut: 100ms delayed re-read to catch OS clipboard update race
   - SHA-256 content hashing, 250ms dedup window
   - Non-text payloads silently skipped

6. **Focus sessions — derived from activity:**
   - On activity session finalization, if `duration_ms >= 300,000ms` (5 min), INSERT `focus_sessions` row
   - Dedup by `activity_session_id`
   - During recovery, focus insertion is forced (ignores stream toggle)

7. **Startup crash recovery:**
   - During `init`, find all `activity_sessions` where `ended_at IS NULL`
   - Derive close time from latest signal: `max(last_observed_at, max(keystroke_chunks.ended_at), max(clipboard_events.captured_at), max(browser_sessions.captured_at))`
   - Clamp to recovery boot timestamp, floor at `started_at`
   - Single transaction with ROLLBACK on error

8. **Stream config and toggles:**
   - Five streams (window_tracking, focus, browser, keystroke, clipboard), all default ON
   - Activity monitoring group: any of (window_tracking, focus, browser) enables x-win subscription
   - Keystroke capture group: either (keystroke, clipboard) enables native addon
   - Config persisted to `capture-streams.json`, cached in memory, write-through on toggle

9. **Supervisor lifecycle (`src/app/main/modules/capture/supervisor.ts`):**
   - State machine: `degraded` → `starting` → `running` / `degraded` → `restarting` → `running` / `failed`
   - Restart backoff: `[2s, 4s, 8s, 16s, 32s]`, 5 attempts max → `failed` (terminal)
   - Permission gate: `setPermissionGate(blocked)` from setup module (macOS Accessibility)
   - System pause/resume: power monitor events (lock/sleep) stop capture without killing worker
   - Worker path resolution: dev → CWD-relative → packaged (3-path fallback)
   - IPC protocol: request/response with UUID correlation, per-command timeouts

10. **Setup permission wiring:**
    - Replaced mocked `setup:grant` with runtime-backed permission checks
    - Accessibility: native `KeystrokeCapture.checkPermission()` / `requestPermission()`
    - Microphone: Electron `systemPreferences` API
    - Screen: status check + System Settings deep link (no `tccutil reset`)
    - Permission outcomes persisted based on actual runtime results
    - Setup completion triggers capture supervisor permission gate update

11. **Simulation mode (deterministic testing):**
    - `CAPTURE_SIMULATION=1` env flag enables synthetic event injection without native addons
    - Actions: `windowSample`, `text`, `key`, `flushChunk`, `clipboard`
    - Deterministic timestamps via `atIso` parameter
    - `npm run capture:validate:local` for local worker/DB validation

12. **Main process integration and IPC relay:**
    - Spawn capture utility process from `src/electron/main.ts`
    - Register IPC relay handlers: `capture:getStatus`, `capture:toggleStream`, `capture:getStreamConfig`
    - Enforce 10s startup timeout without blocking renderer
    - Surface service state transitions for renderer health UX

13. **Validation:**
    - Automated tests for buffer/chunker/flush, worker message contract, IPC handlers, crash recovery
    - Dev runtime smoke: worker boot, migrations, capture row writes
    - Toggle stream ON/OFF, crash/restart recovery
    - Packaged runtime: `electron-forge package`, native module load, artifact layout

**Output:** Capture utility process running, writing real data to libsql. Full supervisor with restart backoff and permission gating. Addons working in utility process context. Complete behavioral specification documented in [capture-pipeline-architecture.md](./capture-pipeline-architecture.md).

---

### Sprint 4.5: Renderer Foundation + Component Library Integration

**Repo:** coworkai-desktop
**Blocks:** Sprint 6
**Depends on:** Sprint 4 + cowork-ui component library (complete)
**Reference:** [phase-1b-sprint-4.5-renderer-foundation.md](https://github.com/go2impact/coworkai-desktop/blob/main/docs/plans/phase-1b-sprint-4.5-renderer-foundation.md) — full implementation plan in desktop repo

Split from Sprint 6 to unblock frontend development. Component integration, theme system, and routing shell can proceed immediately after Sprint 4 merges — in parallel with Sprint 5 (agents process). Without this split, no frontend work could happen until Sprint 5 ships.

**Work:**

1. **Install missing dependencies:** 10 Radix primitives, `class-variance-authority`, `lucide-react`, `tailwind-merge`, `sonner`, `@fontsource-variable/inter`, `tw-animate-css`
2. **Configure `@/` path alias** in both `tsconfig.json` and Vite config — cowork-ui components import via `@/lib/utils`
3. **Replace theme system:** Delete old M3 v1 generator/CSS, copy new `globals.css` from cowork-ui (M3 CSS variables, `@theme inline` block, dark + light themes), add `tailwind.config.ts` with M3 token mappings
4. **Copy component library:** 19 M3-styled shadcn/ui components from cowork-ui → `src/app/renderer/components/ui/`, plus `cn()` utility. Move 25 legacy components to `_legacy/` (setup window still imports them)
5. **Wire routing shell:** MemoryRouter with 4 placeholder views (Chat, Context, Apps, Settings), `AppLayout` with sidebar navigation using new M3 components, dark theme default
6. **Update Vite chunk splitting:** Replace `react-icons` references with Radix packages
7. **Import Inter font:** Self-hosted via `@fontsource-variable/inter`

**Validate:** `npm install` → `tsc --noEmit` → `npm start` → routing shell renders with M3 dark theme → all 4 view stubs navigate correctly → setup window still works with legacy components

**Output:** Production M3 component library integrated, routing shell with 4 placeholder views, path aliases working, theme system replaced. Frontend developers can build views against real components while Sprint 5 runs in parallel.

---

### Sprint 5: Agents & RAG Utility Process Skeleton

**Repo:** coworkai-desktop
**Blocks:** Sprint 6
**Depends on:** Sprint 1 (schema) + Sprint 2 (IPC contract) + Sprint 3 (folder structure)

Wire the agents worker based on Sprint 0 spike learnings. Skeleton — not full agent logic.

**Work:**

1. **Install agent dependencies:**
   - `npm install @mastra/core @mastra/libsql`
   - Update forge rebuild list if `@mastra/libsql` includes native bindings

2. **Wire `agents.worker.ts`:**
   - Init libsql async client → same `cowork.db`
   - Init Mastra with LibSQLStore + LibSQLVector (pattern proven by spike)
   - Init ProviderRegistry (Ollama + Gemini + OpenRouter via `createProviderRegistry()`)
   - Run schema migration for agent-owned tables
   - Signal ready to main

3. **Wire main ↔ agents IPC:**
   - Main spawns agents utility process on boot
   - Implement `chat:sendMessage` → query recent capture rows → inject as system context → `agent.generate()` → response back. This is how chat becomes activity-aware without embeddings — direct SQL before each agent call.
   - Implement streaming path: `agent.stream()` → MessagePort → renderer (ACK gate pattern from spike)
   - Implement `context:getRecentActivity` → direct SQL query against capture tables → return recent activity for Context view (no embeddings, no RAG — just recent rows)

4. **Boot sequence:**
   - Main spawns both utilities in parallel
   - 10s timeout → degraded mode if either fails
   - `system:health` events to renderer

5. **Validate:** App launches → both utility processes boot → send a test chat message → get agent response

**Output:** Two utility processes running. Basic chat round-trip working end-to-end (main → agents → LLM → response → renderer).

---

### Sprint 6: End-to-End Smoke Test

**Repo:** coworkai-desktop
**Depends on:** Sprint 4.5 (renderer foundation) + Sprint 5 (agents process)

Wire the placeholder views from Sprint 4.5 to live data and validate the full capture → chat pipeline.

**Work:**

1. **Preload script:** Expose `window.cowork.*` namespaces (chat, capture, context, system, app)
2. **Install remaining renderer deps:** `zustand`, `react-markdown`, `remark-gfm`, `remark-math`, `rehype-katex`, `shiki`
3. **Chat view:** Wire placeholder to `@ai-sdk/react` v3 `useChat()` with IPC transport, markdown rendering
4. **Context view:** Query capture data from cowork.db via agents process, display recent activity
5. **Settings view:** Wire stream toggles (on/off for each capture stream), service health status
6. **Apps route:** Placeholder shell until Sprint 8 populates it

**End-to-end validation — the full pipeline:**

```
User types in app
  → capture records window/app/URL
  → flush to cowork.db
  → user opens chat
  → asks "what was I just doing?"
  → agents process queries recent capture data (direct SQL, no embeddings)
  → injects activity context into agent prompt
  → routes to local/cloud brain
  → streams response back
  → renders in chat UI
```

Phase 1B uses direct SQL queries against capture tables for activity context. The full embedding pipeline and RAG retrieval (vector search, backfill, observation anchors) come in Phase 2.

**Output:** Working end-to-end prototype. Capture → storage → retrieval → AI response → UI.

---

### Sprint 7: Basic MCP Connection

**Repo:** coworkai-desktop
**Depends on:** Sprint 5 (agents process) + Sprint 6 (renderer for UI)

Connect one MCP server, expose its tools to the agent, show connection status in the UI. This is the minimum needed for the agent to interact with external services.

**What's in scope:**

| In | Out (later phases) |
|---|---|
| `@modelcontextprotocol/sdk` integration in agents worker | OAuth flow (use API key auth only for v0.1) |
| MCPManager: start/stop stdio MCP server, list tools, health ping | Orphan cleanup, client cache dedup, notification-driven invalidation |
| Agent can call MCP tools during chat (Mastra tool integration) | Per-call abort, cost-based safety rails |
| `mcp:connect`, `disconnect`, `listConnections`, `listTools`, `getStatus` IPC handlers | Full 6-step connection flow UI |
| Connection state persisted in libsql `mcp_servers` table | Token refresh, expiry detection |
| Integrations view in renderer: list connected services, connect/disconnect, status dots | Per-server approval policies |
| Electron `safeStorage` API for credential encryption (replaces deprecated `keytar`) | |

**Work:**

1. **Install MCP dependencies:**
   - `npm install @modelcontextprotocol/sdk`
   - No native bindings needed — Electron `safeStorage` is built-in (available since Electron 15, we're on 37.1.0)

2. **Wire MCPManager in `agents.worker.ts`:**
   - Start bundled stdio MCP server with config from `mcp_servers` table
   - List tools from connected server → register as Mastra tools
   - Health ping on startup, status lifecycle (`starting` → `running` → `error`)
   - Credential encryption/decryption via Electron `safeStorage` API (uses macOS Keychain / Windows DPAPI under the hood)

3. **Wire MCP IPC handlers:**
   - `mcp:connect` → start server, validate, save config to libsql
   - `mcp:disconnect` → stop server, update status
   - `mcp:listConnections` → return all servers with status
   - `mcp:listTools` → return tools from connected servers
   - `mcp:getStatus` → return connection health

4. **Integrations view in renderer:**
   - List bundled integrations (hardcoded for v0.1 — e.g., one test server)
   - Connect: API key input → encrypt via `safeStorage` → persist → start server
   - Show status dots (green/red/grey)
   - Show available tools per connection

5. **Validate:** Connect to one MCP server → agent uses its tools in a chat response → tools appear in Integrations view

**Output:** Agent can call external service tools during chat. One MCP server connected and working end-to-end.

---

### Sprint 8: Basic Apps Runtime

**Repo:** coworkai-desktop
**Depends on:** Sprint 6 (renderer) + Sprint 7 (MCP tools available for apps to call)

Render a third-party app (Google AI Studio export) in a sandboxed WebContentsView. The app can call MCP tools through the platform SDK, including a platform-provided `platform_chat` tool for agent-as-tool use cases.

**What's in scope:**

| In | Out (later phases) |
|---|---|
| WebContentsView per app (sandboxed, partition-isolated) | Generic AI Studio export support (template apps only) |
| Custom `cowork-app://` protocol with path traversal protection | Full App Gallery UI |
| App preload: `window.cowork.callTool()`, `window.cowork.listTools()`, `window.cowork.getAppConfig()` | `onMessage()` push channel (platform → app event subscription) |
| esbuild bundling of uploaded zip → single JS output | Advanced permission management UI |
| `cowork.manifest.json` parsing → auto-grant declared permissions | |
| Per-app permission check at preload layer (before IPC) | |
| App metadata + permissions stored in libsql | |
| Apps view in renderer: list installed apps, upload zip, launch app | |

**Work:**

1. **Register `cowork-app://` protocol:**
   - Privileged scheme (`standard`, `secure`, `supportFetchAPI`)
   - Serves files from `~/Library/Application Support/cowork-ai/apps/{appId}/`
   - Path validation: `path.relative()` + prefix check (no traversal)

2. **App installation pipeline:**
   - Upload zip → extract to temp directory first
   - **Archive hardening before extraction reaches apps directory:**
     - Reject entries with absolute paths or `..` path segments (zip-slip)
     - Reject symlinks entirely (symlink escape)
     - Validate total uncompressed size against limit (zip bomb)
     - Only move to `apps/{appId}/` after all entries pass validation
   - Read `cowork.manifest.json` (or `package.json` for metadata)
   - esbuild bundle: TSX/TS → single JS output, deps resolved
   - If esbuild fails → show error with details
   - Auto-grant manifest permissions, store metadata in libsql

3. **App rendering:**
   - Create `WebContentsView` per app with sandbox settings (`contextIsolation`, `sandbox`, `nodeIntegration: false`, `partition: persist:app-${appId}`)
   - Block navigation (`will-navigate` → `preventDefault()`) and `window.open()`
   - Load `cowork-app://{appId}/index.html`

4. **App preload + platform SDK:**
   - `window.cowork.callTool(name, args)` → permission check → `ipcRenderer.invoke()` → main → agents
   - Agent reasoning path: `window.cowork.callTool('platform_chat', { message })` (same tool permission path as any other MCP tool)
   - `window.cowork.listTools()` → return tools the app has permission to use
   - `window.cowork.getAppConfig()` → app metadata, granted permissions

5. **Apps view in renderer:**
   - List installed apps with status
   - Upload zip button → install flow
   - Click app → open in WebContentsView alongside main renderer
   - Remove app option

6. **Validate:** Upload a template app zip → archive hardening passes → esbuild bundles → app renders in WebContentsView → app calls `window.cowork.callTool()` → MCP tool executes → result returned to app → app calls `window.cowork.callTool('platform_chat', ...)` → agent response returns

**Output:** One app running in sandboxed WebContentsView, calling MCP tools through the platform. **This completes Phase 1B.**

---

## Dependency Graph

```
Sprint 0: Mastra Spike
  (COMPLETE — GO)
        │
        ▼
Sprint 1: Schema Design ──────────┬──────────────┐
  (COMPLETE)                      │              │
                                  │              │
Sprint 2: IPC Contract ───────────┤              │
  (COMPLETE)                      │              │
                                  │              │
Sprint 3: Folder Restructure ─────┤              │
  (COMPLETE — PR #10)             │              │
                                  ▼              ▼
                    Sprint 4: Capture    Sprint 5: Agents
                    (COMPLETE — PR #11)       │
                         │                    │
                         ▼                    │
                    Sprint 4.5: Renderer      │
                    Foundation                │
                         │                    │
                         └────────┬───────────┘
                                  ▼
                    Sprint 6: E2E Smoke Test
                                  │
                                  ▼
                    Sprint 7: Basic MCP Connection
                                  │
                                  ▼
                    Sprint 8: Basic Apps Runtime
```

Sprints 0, 1, 2, 3, and 4 are complete.
Sprint 4.5 is unblocked and ready for implementation. Sprint 5 can run in parallel with Sprint 4.5.
Sprint 6 validates the core runtime end-to-end. Sprint 7 adds external service connectivity. Sprint 8 adds the apps runtime.

**Parallel track:** M3 component library built independently, integrated in Sprint 4.5.

---

## What Phase 1B Does NOT Include

Explicitly deferred to later phases:

- Playwright / MCP Browser (Phase 4)
- Automations engine (Phase 4)
- Proactive notifications (Phase 3)
- Screen recording (v0.2)
- Embedding pipeline (Phase 2 — capture writes raw data, embeddings come later)
- Full RAG retrieval (Phase 2 — vector search, backfill, observation anchors)
- Full complexity router (Phase 2 — Sprint 5 registers providers but hardcodes a single default, routing logic comes later)
- Observational memory compression (Phase 2)
- OAuth MCP auth (Phase 3 — Sprint 7 uses API key auth only)
- Generic AI Studio export support (Phase 5 — Sprint 8 supports template apps only)
- Full App Gallery UI (Phase 5)
- MCP advanced features: orphan cleanup, notification-driven invalidation, per-call abort (Phase 3)
- App `onMessage()` push channel — platform-to-app event subscription (Phase 3 — Sprint 8 SDK is request/response only: `callTool`, `listTools`, `getAppConfig`)

Phase 1B delivers the **full runtime skeleton** — processes, IPC, capture, storage, basic chat, MCP tool calling, and sandboxed apps. Embedding/RAG, automations, browser automation, and advanced MCP features layer on top.

---

## Changelog

**v16 (Feb 17, 2026):** Marked Sprint 4 (Capture Utility Process) as complete — PR #11 merged. Added Sprint 4.5 (Renderer Foundation + Component Library Integration) — split from Sprint 6 to unblock frontend development in parallel with Sprint 5. Sprint 6 renamed to "E2E Smoke Test" with component integration work removed (now in Sprint 4.5). Updated dependency graph, status summary, and parallel track references. Component count corrected from 15 to 19.

**v15 (Feb 17, 2026):** Sprint 4 scope expansion to match implementation reality. Replaced 5-item summary with 13-item detailed work breakdown covering: event-driven activity stream (not poll-driven), full keystroke normalization and chunk format, clipboard hotkey detection with OS read and dedup, focus session derivation, startup crash recovery, stream config persistence, supervisor state machine with restart backoff and permission gate, setup permission wiring (replaced mocked checks with runtime APIs), simulation mode for deterministic testing, and remaining work (main process integration + validation). Added cross-reference to new capture-pipeline-architecture.md. Marked sprint as "In Progress" with branch reference.

**v14 (Feb 17, 2026):** Sprint 4 capture semantics lock. Expanded Task 3 with explicit decoupled flush behavior (chunk max-length never ends activity), parent-first session persistence, `last_observed_at` heartbeat updates, one-directional end ordering (activity end flushes chunk first), deterministic startup recovery for stale open sessions, and read-path-only short-session filtering.

**v13 (Feb 17, 2026):** Marked Sprint 2 (IPC Contract + Shared Types) as complete. Updated blocking-open-questions row #3 from "Resolved — Draft" to "Resolved — Final Draft" and refreshed dependency summary to reflect that Sprints 1-3 are complete and implementation can proceed into Sprints 4-5.

**v12 (Feb 17, 2026):** Marked Sprint 1 (Database Schema Design) as complete. Updated blocking-open-questions row #2 from "Not started" to "Resolved — Draft" with direct link to [database-schema.md](./database-schema.md). Updated dependency summary to reflect that Sprints 4-5 are now unblocked by completed Sprint 1 and drafted Sprint 2 contract.

**v11 (Feb 17, 2026):** Apps runtime permission model alignment pass. Removed direct app SDK `window.cowork.chat()` references to match product rule "Apps get tools, not agents." Sprint 8 now routes agent-style requests through `window.cowork.callTool('platform_chat', ...)` (agent-as-tool), and the request/response SDK list was updated accordingly.

**v10 (Feb 17, 2026):** Review fixes. (1) Fixed dependency graph — Sprint 5 now has incoming arrows from Sprints 1/2/3 matching prose. (2) Restored changelog dates to Feb 17 (v8/v9 were incorrectly changed to Feb 16). (3) Replaced `keytar` with Electron `safeStorage` API across Sprint 7 — zero additional native bindings, removes ASAR unpack entry. Updated system-architecture.md and decision log. (4) Added `cowork.apps` namespace to system-architecture.md preload table. (5) Resolved Sprint 2 output location — `architecture/ipc-contract.md` (new doc). (6) Clarified Sprint 6 Apps route is a placeholder shell until Sprint 8. (7) Marked 3 resolved open questions in DESKTOP_SALVAGE_PLAN.md with cross-references. (8) Updated MASTRA_ELECTRON_VIABILITY.md status from "Research" to "Resolved — GO".

**v9 (Feb 17, 2026):** Added `listTools` and `getConfig` to `apps:*` IPC channel table (were in Sprint 8 preload SDK but missing from Sprint 2 inventory). Synced system-architecture.md Platform SDK table — `onMessage()` moved to deferred note with cross-reference. Fixed metadata date.

**v8 (Feb 17, 2026):** Review fixes. (1) Fixed Apps scope contradiction — Sprint 6 now references Sprint 8 for Apps view instead of deferring to Phase 5. (2) Added zip-slip/symlink/zip-bomb hardening to Sprint 8 app installation pipeline. (3) Removed `onMessage()` from Sprint 8 SDK surface (push channel deferred to Phase 3 — Sprint 8 is request/response only). (4) Synced DESKTOP_SALVAGE_PLAN.md phase definitions to reflect expanded Phase 1B scope. (5) Fixed component count: ~11 → 15 per ui-component-task-brief.md. (6) Updated phase acceleration note (salvage plan now synced, not "should be updated").

**v7 (Feb 17, 2026):** Added Sprint 7 (Basic MCP Connection) and Sprint 8 (Basic Apps Runtime) to Phase 1B scope. MCP: `@modelcontextprotocol/sdk` + `keytar`, MCPManager in agents worker, agent calls MCP tools during chat, API key auth only (OAuth deferred), Integrations view. Apps: WebContentsView rendering, `cowork-app://` protocol, app preload SDK (`callTool`, `chat`, `listTools`), esbuild bundling, manifest permissions, template apps only (generic exports deferred). Updated goal, IPC channel table (`mcp:*` and `apps:*` now Phase 1B), dependency graph, and exclusions list.

**v6 (Feb 17, 2026):** Second Codex review fixes. Added `context` to Sprint 6 preload namespace list. Wired context injection into `chat:sendMessage` handler (Sprint 5 — query recent capture rows before agent call). Rewrote Sprint 6 nav description to match design-system.md SideSheet model. Clarified Apps stub scope. Fixed "single provider" wording in exclusions (registers all, hardcodes default). Cleaned stale Sprint 0 dependency language across Sprints 1-3, 5, and dependency graph.

**v5 (Feb 17, 2026):** Updated Sprint 6 with explicit component library integration steps (copy from standalone cowork-ui project, install renderer deps, verify build). Sprint 6 now depends on ui-component-task-brief.md deliverable.

**v4 (Feb 17, 2026):** Added Renderer Stack Decisions section. Locked UI foundation: shadcn/ui (Radix) + M3 token overrides, Zustand v5, React Router v7, TanStack Query, react-markdown + remark-gfm + Shiki, Motion v12. Derived from analysis of all 6 reference app deep dives. Added parallelizable frontend task note for shadcn + M3 component work.

**v3 (Feb 17, 2026):** Marked Sprint 0 (Mastra spike) as complete — GO. All 7 PoC steps passed. Added evidence links to spike result doc and run logs. Added key finding re: MessagePort bridge pattern for Sprint 5.

**v2 (Feb 17, 2026):** Review fixes from Codex cross-model review. Corrected goal from "activity-aware RAG" to "recent-activity context (direct SQL, no embeddings)." Fixed libsql package split (sync `libsql` for capture + async `@libsql/client` for agents). Added Sprint 1/2 as explicit dependencies for Sprints 4/5. Added `context:getRecentActivity` handler to Sprint 5. Added `@mastra/core` + `@mastra/libsql` install step to Sprint 5. Annotated IPC channel table with Phase 1B scope flags. Added phase acceleration note explaining why agents/chat/renderer pulled forward from Phases 3/5. Relabeled "Day" to "Sprint" (1-2 dev-days each, not calendar promises).

**v1 (Feb 17, 2026):** Initial sprint plan for Phase 1B. Six implementation sprints plus ongoing Mastra spike. Scoped from DESKTOP_SALVAGE_PLAN.md Phase 1B definition and system-architecture.md open questions.
