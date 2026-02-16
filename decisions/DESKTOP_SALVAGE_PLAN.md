# Desktop Salvage Plan: coworkai-desktop → Cowork.ai Sidecar

| | |
|---|---|
| **Status** | Decision |
| **Last Updated** | 2026-02-16 |
| **Decision** | **Fork and gut `coworkai-desktop`. Replace custom native addons. Archive legacy repos.** |
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

### 1. Fork `coworkai-desktop` as new project base

**All three plans agree:** The Electron packaging, signing, and distribution infrastructure is the highest-value salvage. It represents weeks of zero-user-facing-value work if rebuilt from scratch.

**What we keep directly:**

| Component | Notes |
|---|---|
| Electron Forge config (`forge.config.ts`) | Plugins, makers, full packaging pipeline |
| macOS code signing | osxSign, osxNotarize, hardened runtime, Apple ID/Team ID setup |
| Windows code signing | Azure Trusted Signing integration |
| S3 publisher | Profile-based key structure |
| Auto-updater | `update-electron-app` with platform-specific manifest URLs |
| Environment profiles | local, alpha, beta, production |
| ASAR unpacking for native modules | `AutoUnpackNativesPlugin` |
| React + Vite + Tailwind setup | Renderer tooling carries over as starting point |

**What we gut:**

- All tracker IPC channels (`TRACKER/START`, `TRACKER/STOP`)
- Auth flow tied to employer service
- Sync pipeline (stub anyway — eliminated entirely for local-first)
- Timer module (product explicitly says "not a time tracker")
- Renderer UI (complete rewrite for sidecar: Chat, Apps, MCP Browser, Automations, Context)

**Why fork, not greenfield (Codex's alternative):** Forking preserves the packaging bootstrap. A fully new repo means re-implementing Forge config, signing, S3 publishing, and updater from scratch. The contamination risk Codex raises is mitigated by gutting internals immediately.

**Why fork, not evolve-in-place (Gemini's approach):** In-place evolution risks carrying legacy patterns. A fork creates a clean git history with an intentional starting point.

### 2. Replace custom native addons with open-source equivalents

**Gemini dissented here (keep 100%).** Codex was neutral (keep/integrate). **Opus argued for replacement.** We go with Opus.

#### Keystroke capture: `coworkai-keystroke-capture` → `uiohook-napi`

Both call the exact same OS APIs:

| | Custom addon | uiohook-napi |
|---|---|---|
| macOS hook | `CGEventTap` + dedicated `CFRunLoop` thread | `CGEventTap` + `CFRunLoop` thread (via libuiohook) |
| Windows hook | `SetWindowsHookExW` + message pump | `SetWindowsHookExW` + message pump (via libuiohook) |
| Node.js bridge | Hand-rolled `Napi::ThreadSafeFunction` | `ThreadSafeFunction` (same pattern) |
| Cross-platform | macOS + Windows | macOS + Windows + Linux |
| Maintenance | Us (prebuild scripts, per-platform CI, Electron ABI rebuilds) | Community + libuiohook maintainers |

The custom addon adds zero unique capability. Same syscalls, different wrapper. Dropping it eliminates the `coworkai-keystroke-capture` repo and its prebuild/CI infrastructure.

#### Activity capture: `coworkai-activity-capture` → `active-win` + AppleScript wrapper

Two sub-problems:

**Window detection:** The custom addon's 3-tier fallback (`NSWorkspace` → `runningApplications` → `CGWindowList`) was built for employer monitoring where missing a single window switch affects reporting accuracy. For AI context, an occasional miss is invisible. `active-win` calls `NSWorkspace` (the primary path) and handles the common case.

**Browser URL extraction:** The custom addon's AppleScript-primary + Accessibility-fallback approach has genuine value. However:

1. v0.1 is Mac-only — the entire Windows COM UI Automation implementation is irrelevant.
2. On macOS, plain `child_process.execFile('osascript', ...)` gives the same primary path without a native addon.
3. During MCP Browser sessions, Playwright already knows the URL — activity capture URL extraction only matters for the user's own browsing.
4. AppleScript works for all major macOS browsers (Safari, Chrome, Firefox, Arc, Brave, Edge) without Accessibility permissions.

**If AppleScript proves unreliable in practice, add the Accessibility fallback later as a focused module.** Don't carry that complexity from day one.

**Why we disagree with Gemini:** "Battle-tested" doesn't hold when the replacement calls the same APIs. The custom addons are wrappers around identical OS syscalls. Keeping them means maintaining two repos, their prebuild infrastructure, and per-platform CI for zero unique capability.

#### New capture layer structure

```
src/core/capture/
├── input.ts            ← uiohook-napi wrapper (keystroke + mouse events)
├── window.ts           ← active-win wrapper (active app, window title)
└── browser-url.ts      ← AppleScript via child_process (~50 lines)
```

Replaces two entire repos and their prebuild/CI infrastructure.

### 3. Archive `coworkai-agent` — no salvage

**Gemini wanted to salvage the SQLite schema and timer module. Codex suggested keeping as source/fork. Opus said archive entirely.** We go with Opus.

**Why no salvage:**

1. The agent is a tracking engine. Mastra.ai replaces its purpose entirely — there is no evolutionary path from "capture → sync to employer cloud" to "capture → local memory → AI acts on your behalf."
2. The timer module contradicts product positioning. The product is explicitly "not a time tracker." Keeping midnight-split aggregation logic imports the wrong mental model.
3. The SQLite schema (`activities`, `keystrokes`, `clipboards`) was designed for tracking. The new schema needs context/memory tables, embedding storage (sqlite-vec), and a fundamentally different data model oriented around AI retrieval, not employer reporting.

**What has reference value:** The AppleScript commands and browser-specific queries in the capture repos are useful reference for edge case handling. Archive as read-only, not deleted.

### 4. New SQLite schema for context/memory/embeddings

**All three agree** on using `better-sqlite3` + `sqlite-vec`. The schema itself is entirely new:

- Designed for context retrieval and RAG, not activity reporting
- Embedding storage via `sqlite-vec` extension
- Retention enforcement and privacy controls built in
- No timer aggregation tables, no sync queue tables

### 5. Multi-process architecture

**Unanimous across all three plans.** Per `DESKTOP_FRAMEWORK_DECISION.md`:

| Process | Role |
|---|---|
| **Main Process** | Lifecycle, IPC routing, system tray. No heavy work. |
| **Capture Utility Process** | `uiohook-napi`, `active-win`, AppleScript URL extraction. Crash-isolated, auto-restarts. |
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
| **Custom native addons** | Keep 100% ("battle-tested") | Keep / integrate | Replace with `uiohook-napi` + `active-win` | **Replace.** Same OS syscalls, less maintenance. See decision #2. |
| **Timer module** | Keep midnight-split logic | Not addressed | Eliminate ("not a time tracker") | **Eliminate.** Contradicts product positioning. See decision #8. |
| **Tracker SQLite schema** | Keep `activities`, `keystrokes`, `clipboards` tables | Normalize to new event schema | New schema entirely | **New schema.** Designed for AI retrieval, not employer reporting. See decision #4. |

---

## Repo Disposition

| Repo | Action | Reasoning |
|---|---|---|
| `coworkai-desktop` | **Fork as new project base** | Packaging, signing, publishing, updater infrastructure is valuable. Gut internals. |
| `coworkai-agent` | **Archive (read-only)** | Tracking engine with no evolutionary path to AI agent orchestrator. Mastra.ai replaces entirely. |
| `coworkai-activity-capture` | **Archive (read-only)** | AppleScript commands have reference value. Addon replaced by `active-win` + thin wrapper. |
| `coworkai-keystroke-capture` | **Archive (read-only)** | Same underlying APIs as `uiohook-napi`. No reason to maintain custom implementation. |

---

## New Dependencies

| Dependency | Purpose | Where it runs |
|---|---|---|
| Mastra.ai | Agent orchestration | Utility process (agents & RAG) |
| Playwright | Browser automation (MCP Browser) | Child process |
| uiohook-napi | Global keystroke + mouse capture | Utility process (capture) |
| active-win | Active window detection | Utility process (capture) |
| Ollama client | Local LLM (DeepSeek-R1-8B) + embeddings (Qwen3-Embedding-0.6B) | Utility process (agents & RAG) |
| sqlite-vec | Vector store for RAG | Utility process (agents & RAG) |
| MCP SDK | Connect to Zendesk, Gmail, Slack, etc. | Utility process (agents & RAG) |
| OpenRouter client | Cloud inference gateway | Utility process (agents & RAG) |

---

## Target Directory Structure

Per `DESKTOP_FRAMEWORK_DECISION.md`, business logic stays in `core/` with zero Electron imports:

```
src/
├── core/                       # Pure TypeScript — no Electron dependency
│   ├── agents/                 # Mastra.ai agent definitions + orchestration
│   ├── automations/            # Rule/workflow engine
│   ├── capture/                # uiohook-napi, active-win, browser URL extraction
│   ├── chat/                   # Chat logic (on-demand + proactive)
│   ├── mcp/                    # MCP server connections + tool registry
│   ├── memory/                 # Embedding, vector store, RAG pipeline
│   └── store/                  # better-sqlite3 + sqlite-vec data layer
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

- Fork `coworkai-desktop`, gut internals
- Create target folder architecture (`core/`, `electron/`, `renderer/`, `shared/`)
- Define typed IPC contract and process boundaries
- Stand up capture utility process using `uiohook-napi` + `active-win` + AppleScript
- Move main process to coordinator-only role
- Integrate `sqlite-vec` into local database with new schema

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
| AppleScript URL extraction unreliable | Missing browser context | Add Accessibility fallback as focused module if needed |

---

## Open Questions

1. **Bundle ID continuity** — Do we want the new product to share the existing app's bundle ID (for update continuity) or start fresh (clean break)?
2. **Electron version** — Existing uses Electron 37.1.0. Confirm target version for the new project.
3. **Native module rebuild list** — Existing `forge.config.ts` rebuilds 6 native modules. New list is shorter (`better-sqlite3`, `uiohook-napi`). Confirm no others needed.
4. **MCP server packaging** — Bundled with the app, installed on demand, or running remotely?
5. **App Gallery hosting** — Where do Google AI Studio apps get stored? Local filesystem? Bundled? Cloud service?

---

## Changelog

**v1 (Feb 16, 2026):** Initial decision. Synthesized from three independent salvage analyses (Gemini CLI, Codex, Claude Opus). Fork-and-gut strategy, native addon replacement, repo disposition, phased execution plan.
