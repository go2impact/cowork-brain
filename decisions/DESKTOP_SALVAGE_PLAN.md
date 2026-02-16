# Desktop Salvage Plan: coworkai → Cowork.ai Sidecar

| | |
|---|---|
| **Status** | Decision |
| **Last Updated** | 2026-02-16 |
| **Decision** | **Gut all four repos in place. Keep packaging, capture addons, and capture orchestration. Kill tracking-specific code.** |
| **Related** | [DESKTOP_FRAMEWORK_DECISION.md](./DESKTOP_FRAMEWORK_DECISION.md), [product-features.md](../product/product-features.md) |
| **Inputs** | Salvage plans from Gemini CLI, Codex, and Claude Opus — synthesized below |

---

## Context

We have four repositories from the existing employee activity monitoring product:

- `coworkai-desktop` — Electron app (packaging, signing, updater, renderer, main process)
- `coworkai-agent` — Tracking engine (capture orchestration, SQLite persistence, timer, sync)
- `coworkai-activity-capture` — Custom C++ native addon (window detection, browser URL extraction)
- `coworkai-keystroke-capture` — Custom C++ native addon (global keystroke hooks)

We are building Cowork.ai Sidecar — a fundamentally different product. The question: **what do we keep, what do we replace, and how do we structure the transition?**

Three independent salvage analyses were produced (Gemini CLI, Codex, Claude Opus). This document records the consensus decisions and the reasoning behind them.

---

## Product Paradigm Shift

This is not an incremental upgrade. The product direction reverses:

| Dimension | Existing (coworkai) | New (Cowork.ai Sidecar) |
|---|---|---|
| **Purpose** | Employee activity monitoring | AI assistant / sidecar |
| **Data flow** | Capture → sync to employer cloud | Capture → local memory → AI acts on your behalf |
| **Who sees data** | Employer dashboard | Worker only (local-first) |
| **Core value** | Activity tracking and reporting | AI that observes, remembers, and acts |
| **Agent layer** | None (tracking engine) | Mastra.ai orchestration, RAG, MCP, Playwright |

---

## Consensus Decisions

### 1. Gut `coworkai-desktop` in place

There is no old product to maintain separately. We continue in the existing repo — no fork needed. CI/CD pipelines, signing certificates, bundle ID, team access, and git history all carry over.

**What we keep:**

| What | Source | Why | Action |
|---|---|---|---|
| Electron Forge config | `forge.config.ts` | Packaging pipeline is weeks of work to rebuild. Plugins, makers, platform targets all configured. | Update `rebuildConfig`: remove `coworkai-video-capture`, `coworkai-audio-capture`, `clipboard-capture`, `better-sqlite3`. Add `libsql`. Keep `coworkai-activity-capture`, `coworkai-keystroke-capture`. |
| macOS code signing | `forge.config.ts` (osxSign, osxNotarize) | Hardened runtime + notarization required for macOS distribution. Already configured with Apple ID/Team ID. | No changes. |
| Windows code signing | `forge.config.ts` (windowsSign) | Required for Windows distribution. Azure Trusted Signing already integrated. | No changes. |
| S3 publisher | `forge.config.ts` (publisher-s3) | Distribution infrastructure. Profile-based key structure handles alpha/beta/production channels. | No changes. |
| Auto-updater | `src/app/main/modules/updater/` | Users need OTA updates. `update-electron-app` with platform-specific manifest URLs already works. | No changes. |
| Environment profiles | `.env.*`, `utils/forge-args.ts` | Multi-environment deployment (local/alpha/beta/production) carries over to sidecar. | Remove employer-specific env vars (auth URLs, API endpoints). Keep `FORGE_PROFILE` mechanism. |
| ASAR unpacking | `AutoUnpackNativesPlugin` | Native `.node` files can't load from ASAR archives. Already configured. | No changes. |
| Native resolution logic | Dev: `node_modules`. Packaged: `asar-unpacked`. | Native addons need different resolution paths in dev vs packaged. Already solved. | No changes. |
| Main process entry | `src/main.ts` | Electron lifecycle init, tray, IPC setup are foundational. The wiring is reusable even though what it wires changes. | Refactor: remove `coworkaiAgent.sync.start()`, Sentry (replace later). Keep lifecycle + tray + IPC init. |
| DB path config | `configs/database.ts`, `src/app/main/config.ts` | `app.getPath('userData')` path resolution is the correct local storage pattern. Same pattern needed for sidecar. | No changes. |
| React + Vite + Tailwind | Build tooling | Renderer build pipeline (bundler, CSS framework) is reusable. Only the UI components change. | No changes. |

**What we gut:**

| What | Source | Why |
|---|---|---|
| Tracker IPC | `src/app/main/ipc/tracker/index.ts`, `TRACKER/START`, `TRACKER/STOP` channels | Employer tracking concept. Replace with new capture IPC. |
| Auth IPC | `src/app/main/ipc/auth/index.ts`, `AUTH/SET_USER` channel | Employer auth. Local-first has no employer identity. |
| Timer IPC | `TIMER` channel | "Not a time tracker." |
| Sync config | `src/app/main/coworkai-agent/configs/sync.ts` | Employer cloud sync. Stub anyway. |
| Media capture configs | `configs/agent/screencapture.ts`, `configs/agent/audio.ts`, `configs/agent/video.ts` | Not in v0.1 scope. |
| Timer config | `configs/agent/timer.ts` | "Not a time tracker." |
| Entire renderer | `src/app/renderer/` (all views, components) | Complete rewrite for sidecar product surface. |
| IPC channel definitions | `channel.ts` — `AUTH`, `TRACKER`, `TIMER`, `SETUP` channels | Replace with new typed IPC contract for sidecar. |

### 2. Keep custom native addons — move to utility process

**Original decision was to replace with open-source equivalents (Opus's recommendation). Reversed after deep research.** See [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](./NATIVE_ADDON_REPLACEMENT_RESEARCH.md) for the full analysis.

**Gemini was right.** The original Opus analysis compared syscalls but not capabilities. Deep research found critical gaps in both proposed replacements:

#### Keystroke capture: keep `coworkai-keystroke-capture`

| Capability | Custom addon | uiohook-napi |
|---|---|---|
| Character mapping | Full UTF-8 via `CGEventKeyboardGetUnicodeString` / `ToUnicodeEx` | **Keycodes only** — no character/glyph mapping |
| Key repeat detection | OS-provided (macOS) + atomic tracking (Windows) | **None** — held keys generate identical streams |
| CapsLock state | Exposed as boolean in every event | **Not exposed** |
| Electron safety | Graceful permission handling, event tap auto-recovery | **Deadlock** in Electron (issue #23, unresolved) |
| Maintenance | Active | **Dormant** — last release March 2024, single maintainer |

The Electron deadlock alone is a showstopper. uiohook-napi calls `dispatch_sync_f` to the main dispatch queue for Unicode key lookup, which deadlocks because Electron's main thread is blocked in Node.js's event loop.

#### Activity capture: keep `coworkai-activity-capture`

| Capability | Custom addon | get-windows (formerly active-win) |
|---|---|---|
| macOS app detection | 3-tier fallback (NSWorkspace → runningApplications → CGWindowList) | Single method only |
| Browser URL (macOS) | AppleScript + Accessibility API fallback | AppleScript only, fails silently |
| Deep Chromium traversal | Recursive DFS through Accessibility tree for `AXWebArea` | **None** |
| Browser URL (Windows) | Full COM UI Automation with 28 localized address bar names | **Not supported** |
| Firefox support | Supported on all platforms | **Not supported** |
| Architecture | In-process N-API addon, sub-millisecond | Spawns Swift CLI as child process |
| Electron integration | Purpose-built, rebuilt against Electron ABI | Known permission inheritance issues |

The "no Windows URL extraction" gap would require building a separate solution for v0.2. The child process spawn architecture creates permission inheritance problems in signed Electron apps.

#### Capture layer structure

```
src/core/capture/
├── input.ts            ← coworkai-keystroke-capture wrapper (keystroke + mouse events)
├── window.ts           ← coworkai-activity-capture wrapper (active app, window title, browser URL)
└── types.ts            ← Shared capture event types
```

Wraps existing addons in the capture utility process. Addons are rebuilt against Electron ABI via `forge.config.ts` and unpacked from ASAR.

### 3. Gut `coworkai-agent` — keep capture orchestration, kill tracking

The agent is the orchestrator that sits between `coworkai-desktop` and the native addons. It manages the capture lifecycle, buffers events, and flushes to SQLite. That orchestration is reusable.

**What we keep:**

| What | Source | Why | Action |
|---|---|---|---|
| Agent lifecycle | `src/main.ts` | `init(config)` + SQLite WAL setup is the correct pattern for high-frequency writes. Proven in production. | Refactor: remove `sync.start()` call. Replace `better-sqlite3` with `libsql` (sync package). Keep WAL pragmas. |
| Feature flags | `src/config.ts` | Config-driven capture toggling lets us enable/disable capture streams without code changes. Needed for user privacy controls. | Remove `timer`, `screen`, `audio`, `video` flags. Keep `activity`, `keyboard`, `clipboard`. Flip defaults: `keyboard` and `clipboard` to OFF (opt-in) per privacy-first model. |
| Activity orchestration | `src/agent/activity/index.ts`, `src/agent/activity/lib/x-win.ts` | `subscribeActiveWindow()` + browser enrichment is the core capture loop. Rebuilding this means re-solving window change detection, title/URL extraction, and addon coordination. | No changes to orchestration logic. New schema underneath. |
| Activity buffer management | `src/database/activity/manager.ts` | Bounded 5-buffer queue with `autoFlushIfNeeded()` prevents unbounded memory growth during rapid window switching. This pattern is correct for sidecar too. | Refactor: point at new table schema. Keep buffer/flush logic. |
| Keystroke chunking | `src/database/keystroke/buffer.ts` | Debounce-driven flushing + special-key triggers + 1000-char activity flush are tuned for real-world typing patterns. Rebuilding means re-discovering these thresholds. | Refactor: point at new table schema. Keep chunking logic. |
| Keyboard + clipboard | `src/agent/keyboard/index.ts`, `src/database/clipboard/buffer.ts` | Copy/cut/paste hotkey detection in the keystroke stream is how clipboard capture works without polling. `readClipboard()` on hotkey is more efficient than polling. | Refactor: point at new table schema. Keep detection logic. |

**What we kill:**

| What | Source | Why |
|---|---|---|
| Timer module | `src/agent/timer/index.ts` | Midnight splitting, daily aggregation, session tracking. "Not a time tracker." |
| TimeLog buffer | `src/database/timelog/buffer.ts` | `timelogs` table. Part of timer. |
| SyncManager | `src/database/index.ts` (SyncManager wiring) | Employer cloud sync scheduling. Remove, keep DB init. |
| SyncQueue | `src/database/sync/sync.ts` | Batch sync, retry logic, `sync` flag columns. All employer sync. |
| Media modules | Screen, audio, video agent modules | Not in v0.1 scope. |
| Employer identity | `agent.setUserId(user.id)` pattern | Employer-scoped `user_id` isolation. Local-first has no employer identity. |
| SQLite schema | `activities`, `keystrokes`, `clipboards`, `timelogs` tables | **Replace entirely** with new schema designed from [product-features.md](../product/product-features.md). Old tables are tracking-oriented; new tables serve context/memory/embeddings/agent state. Keep WAL patterns and buffer flush approach. Migrate from `better-sqlite3` to `libsql`. |

### 4. New SQLite schema for context/memory/embeddings

**All three agree** on SQLite as the data layer. After [database stack research](./DATABASE_STACK_RESEARCH.md), the decision is **libsql for everything** — one SDK for both capture data and Mastra agent memory. The schema itself is entirely new:

- Designed for context retrieval and RAG, not activity reporting
- Embedding storage via libsql's built-in vector search (no extension needed)
- Retention enforcement and privacy controls built in
- No timer aggregation tables, no sync queue tables
- Native Mastra integration via `@mastra/libsql` (Mastra runs in Agents & RAG utility process)

> **Schema is not finalized.** The existing schema from [COWORKAI_TECHNICAL_REFERENCE.md](./COWORKAI_TECHNICAL_REFERENCE.md) documents the old tracking-oriented tables (`activities`, `keystrokes`, `clipboards`, `timelogs`). The new schema will be designed from scratch based on the feature set in [product-features.md](../product/product-features.md) — specifically the six input streams (Context), four memory layers (Memory Architecture), and Mastra agent state requirements. Do not assume the old schema carries over.

### 5. Multi-process architecture

**Unanimous across all three plans.** Per `DESKTOP_FRAMEWORK_DECISION.md`:

| Process | Role |
|---|---|
| **Main Process** | Lifecycle, IPC routing, system tray. No heavy work. |
| **Capture Utility Process** | `coworkai-keystroke-capture`, `coworkai-activity-capture`. Crash-isolated, auto-restarts. |
| **Agents & RAG Utility Process** | Mastra.ai, embeddings, vector store, MCP runtime. Isolated from UI. |
| **Playwright Child Process** | Browser automation. Fully isolated from app. |
| **Renderer** | Sandboxed (`contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`). |

### 6. Complete renderer rewrite

**Unanimous.** The existing tracker-oriented UI has no evolutionary path to the sidecar product surface:

- Chat (on-demand + proactive)
- Apps + App Gallery
- MCP Integrations
- MCP Browser (live execution view)
- Automations
- Context

### 7. Local-first privacy model is mandatory

**Unanimous, but worth stating explicitly.** All activity and context data remains on-device by default. There is no employer-style activity sync pipeline. The user — not an employer — controls what data exists and what gets shared.

This isn't just a technical constraint; it's the product's trust model. The existing `coworkai` product sent data to an employer cloud. The sidecar inverts that: data stays local, the AI acts on the user's behalf, and cloud inference (when used) processes queries — not raw activity logs.

### 8. Kill sync pipeline, timer module, employer auth

**Unanimous.** These are artifacts of the monitoring product and have no place in the sidecar.

---

## Resolved Conflicts

Three independent analyses disagreed on three points. The resolutions:

| Conflict | Gemini | Codex | Opus | Resolution |
|---|---|---|---|---|
| **Custom native addons** | Keep 100% ("battle-tested") | Keep / integrate | Replace with `uiohook-napi` + `active-win` | **Keep.** Deep research found critical capability gaps in replacements. Gemini was right. See [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](./NATIVE_ADDON_REPLACEMENT_RESEARCH.md). |
| **Timer module** | Keep midnight-split logic | Not addressed | Eliminate ("not a time tracker") | **Eliminate.** Contradicts product positioning. See decision #8. |
| **Tracker SQLite schema** | Keep `activities`, `keystrokes`, `clipboards` tables | Normalize to new event schema | New schema entirely | **New schema.** Designed for AI retrieval, not employer reporting. See decision #4. |

---

## Repo Disposition

| Repo | Action | Reasoning |
|---|---|---|
| `coworkai-desktop` | **Gut in place** | Packaging, signing, publishing, updater infrastructure is valuable. Kill tracker IPC, auth, sync, timer, renderer. No fork needed — there is no old product to maintain. |
| `coworkai-agent` | **Gut in place** | Keep capture orchestration (activity buffers, keystroke chunking, clipboard capture, SQLite persistence). Kill timer, sync, media, employer auth. |
| `coworkai-activity-capture` | **Keep as-is** | 3-tier detection, Accessibility fallback, deep Chromium traversal, full Windows UI Automation. No viable replacement. |
| `coworkai-keystroke-capture` | **Keep as-is** | UTF-8 character mapping, repeat detection, CapsLock state, Electron-safe threading. No viable replacement. |

---

## New Dependencies

| Dependency | Purpose | Where it runs |
|---|---|---|
| libsql (sync) | Database — capture writes, structured storage, built-in vector search | Capture utility process |
| @mastra/libsql | Database — agent memory, embeddings, semantic recall (same .db file) | Agents & RAG utility process |
| Mastra.ai | Agent orchestration | Agents & RAG utility process |
| Playwright | Browser automation (MCP Browser) | Child process |
| Ollama client | Local LLM (DeepSeek-R1-8B) + embeddings (Qwen3-Embedding-0.6B) | Agents & RAG utility process |
| MCP SDK | Connect to Zendesk, Gmail, Slack, etc. | Agents & RAG utility process |
| OpenRouter client | Cloud inference gateway | Agents & RAG utility process |

Existing dependencies carried over: `coworkai-keystroke-capture`, `coworkai-activity-capture`. `better-sqlite3` replaced by `libsql`.

---

## Target Directory Structure

Per `DESKTOP_FRAMEWORK_DECISION.md`, business logic stays in `core/` with zero Electron imports:

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

---

## Execution Phases

### Phase 1: Core Runtime Foundation

- Gut `coworkai-desktop` and `coworkai-agent` in place
- Create target folder architecture (`core/`, `electron/`, `renderer/`, `shared/`)
- Define typed IPC contract and process boundaries
- Stand up capture utility process with `coworkai-keystroke-capture` + `coworkai-activity-capture`
- Move main process to coordinator-only role
- Integrate `libsql` into local database with new schema (replaces `better-sqlite3` + `sqlite-vec`)

### Phase 2: Context + Data Layer

- Normalize event schema for context streams
- Implement local SQLite retention enforcement
- Add embedding/vector storage pipeline
- Expose context retrieval API to agents/chat

### Phase 3: MCP + Agent Runtime

- Add MCP integration manager (connections, scopes, health)
- Add Mastra.ai agent runtime orchestration (local/cloud routing)
- Add approval gate framework and execution audit model

### Phase 4: MCP Browser + Automations

- Add Playwright execution service in isolated child process
- Build unified execution timeline (MCP + browser + user interventions)
- Implement automation trigger engine and run logs

### Phase 5: Product Surfaces

- Build SideSheet and detail canvas aligned to feature model
- Implement Apps, Integrations, Chat, MCP Browser, Automations, Context UX
- Add privacy controls and per-stream consent surfaces
- Finalize onboarding flow (hardware detection & brain choice)

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Native module crash in capture layer | App-wide crash | Isolate in restartable utility process |
| Main-process overload | UI freezes | Strict worker/process isolation; main is coordinator only |
| Trying to evolve legacy renderer in place | Slow velocity, regressions | Greenfield UI architecture |
| Scope creep across six features | Delayed delivery | Phase by capability layer, not by feature count |
| Data/privacy mismatch | Product trust risk | Ship privacy boundaries and retention controls early |
| Custom addon maintenance burden | Two extra repos + prebuild CI | Cost of maintenance < cost of rebuilding capabilities. Addons are battle-tested. |

---

## Open Questions

1. **Bundle ID rename** — Existing bundle ID is coworkai-branded. When do we rename to Cowork.ai Sidecar?
2. **Electron version** — Existing uses Electron 37.1.0. Confirm target version.
3. **Native module rebuild list** — Existing `forge.config.ts` rebuilds 6 native modules (includes video, audio, clipboard captures being killed). Trim to: `libsql`, `coworkai-keystroke-capture`, `coworkai-activity-capture`. Confirm no others needed.
4. **MCP server packaging** — Bundled with the app, installed on demand, or running remotely?
5. **App Gallery hosting** — Where do Google AI Studio apps get stored? Local filesystem? Bundled? Cloud service?

---

## Changelog

**v4 (Feb 16, 2026):** Updated database stack from `better-sqlite3` + `sqlite-vec` to `libsql` for everything. One SDK for both capture data and Mastra agent memory. Built-in vector search eliminates sqlite-vec extension. Mastra runs in Agents & RAG utility process (Electron 37.1.0 ships Node 22.16.0, satisfying `@mastra/libsql` requirement). See [DATABASE_STACK_RESEARCH.md](./DATABASE_STACK_RESEARCH.md).

**v3 (Feb 16, 2026):** Replaced fork-and-gut with gut-in-place for all repos. No old product to maintain — forking adds complexity for zero benefit. Reversed decision #3 — keep `coworkai-agent` active, gut tracking-specific code, retain capture orchestration (activity buffers, keystroke chunking, SQLite persistence). Added file-level specificity to decisions #1 and #3 (exact source files to keep vs gut). Updated repo disposition, dependencies, execution phases, and open questions.

**v2 (Feb 16, 2026):** Reversed decision #2 — keep custom native addons instead of replacing with open-source equivalents. Deep research found critical capability gaps in uiohook-napi (no character mapping, Electron deadlock) and get-windows (no Windows URLs, no fallback strategy). Updated repo disposition, dependencies, directory structure, and execution phases. See [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](./NATIVE_ADDON_REPLACEMENT_RESEARCH.md).

**v1 (Feb 16, 2026):** Initial decision. Synthesized from three independent salvage analyses (Gemini CLI, Codex, Claude Opus). Fork-and-gut strategy, native addon replacement, repo disposition, phased execution plan.
