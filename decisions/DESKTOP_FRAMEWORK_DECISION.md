# Desktop Framework Decision

| | |
|---|---|
| **Status** | Final decision |
| **Last Updated** | 2026-02-13 |
| **Decision** | **Electron** |
| **Alternatives Evaluated** | Native Swift (AppKit/SwiftUI), Tauri 2.0 |

---

## Context

We are building a greenfield desktop application that combines:

- **Browser automation** via Playwright
- **AI agents** via Mastra.ai
- **Local RAG pipeline** (embeddings, vector store, document loaders)
- **System-wide data capture** (global keystrokes, mouse events, active window tracking)
- **Browser URL extraction** without a browser extension (via AppleScript / UI Automation)
- **Activity logging** across all applications on the host machine
- **Google AI Studio app ecosystem** rendered inside the desktop app

The target audience is general consumers, not developers.

---

## The Core Argument: The Stack Is Node.js

Every major dependency in the application is a Node.js library or TypeScript framework:

| Component | Runtime | Runs in Electron? | Runs in Swift? | Runs in Tauri (Rust)? |
|---|---|---|---|---|
| Playwright | Node.js | Native | Needs sidecar | Needs sidecar |
| Mastra.ai | Node.js | Native | Needs sidecar | Needs sidecar |
| RAG pipeline (loaders, embeddings, vector DB) | Node.js | Native | Needs sidecar | Needs sidecar |
| uiohook-napi | Node.js | Native | CGEvent taps (same API) | Needs sidecar |
| active-win | Node.js | Native | NSWorkspace (same API) | Needs sidecar |
| AppleScript (URL grab) | Node.js | child_process | NSAppleScript | Needs sidecar |
| better-sqlite3 | Node.js | Native | GRDB (good equivalent) | Rust has own SQLite |

In both Swift and Tauri, **100% of the application logic would live in a Node.js sidecar**. The native layer would manage windows and the system tray — nothing more.

---

## Why Not Native Swift

### The Architecture Problem

Building this application in native Swift produces two runtimes with a hand-rolled bridge:

```
┌──────────────────────────────────────────────────────────┐
│               NATIVE SWIFT (Two Runtimes)                │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │            Swift Application (AppKit/SwiftUI)      │  │
│  │                                                    │  │
│  │  ┌──────────────┐  ┌───────────────────────────┐  │  │
│  │  │ Window Mgmt  │  │ Native UI (SwiftUI)       │  │  │
│  │  │ Menu Bar     │  │ ...but dashboards, data   │  │  │
│  │  │ System Tray  │  │ tables, config panels are │  │  │
│  │  │ Permissions  │  │ painful here, so you end  │  │  │
│  │  │              │  │ up embedding WKWebView    │  │  │
│  │  │              │  │ anyway.                   │  │  │
│  │  └──────────────┘  └───────────────────────────┘  │  │
│  │                                                    │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │ CGEvent Taps (global input capture)          │  │  │
│  │  │ NSWorkspace (active window detection)        │  │  │
│  │  │ NSAppleScript (browser URL extraction)       │  │  │
│  │  │                                              │  │  │
│  │  │ Same APIs, same permissions as Node.js       │  │  │
│  │  │ No native advantage for any of this          │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └──────────────┬─────────────────────────────────────┘  │
│                 │                                        │
│                 │ stdin/stdout or local HTTP              │
│                 │ No standard bridge pattern              │
│                 │ No tooling support                      │
│                 │ Entirely hand-rolled                    │
│                 │                                        │
│  ┌──────────────▼─────────────────────────────────────┐  │
│  │          Embedded Node.js Runtime                   │  │
│  │                                                     │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │  │
│  │  │  Playwright  │  │ Mastra.ai   │  │ Local RAG  │ │  │
│  │  │  Browser     │  │ Agents      │  │ Pipeline   │ │  │
│  │  │  Automation  │  │ Orchestrate │  │ Embeddings │ │  │
│  │  │              │  │ Tools       │  │ VectorDB   │ │  │
│  │  └─────────────┘  └─────────────┘  └────────────┘ │  │
│  │                                                     │  │
│  │  100% of the business logic lives here.             │  │
│  │  Swift does nothing except host this process.       │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

  The Swift layer becomes an expensive wrapper around a Node.js
  application. All product logic crosses a language boundary.
```

### No Ecosystem for AI Agents and Automation

The Swift/macOS ecosystem is built for consumer apps: photo editors, note-taking apps, calendar apps. For this product's core domains:

| Domain | JavaScript/TypeScript Ecosystem | Swift Ecosystem |
|---|---|---|
| Agent orchestration | Mastra.ai, LangChain.js, Vercel AI SDK | Nothing |
| Browser automation | Playwright, Puppeteer, Selenium | Nothing |
| RAG pipeline | LangChain loaders, embedding SDKs, vector clients | Fragments, unmaintained |
| Vector store clients | Mature, well-documented | Sparse, poorly maintained |
| Embedding SDKs | Full coverage (OpenAI, Google, Ollama) | Partial (OpenAI Swift only) |

Choosing Swift means rebuilding foundational infrastructure that already exists and is battle-tested in TypeScript.

### Global Input Capture Has No Native Advantage

Global input capture in Swift uses `CGEvent.tapCreate()`, a C-level Core Graphics API bridged into Swift. It requires the same Accessibility permission that Node.js packages require. The `uiohook-napi` package calls the same underlying macOS APIs (via `libuiohook`), wraps them in a clean event emitter interface, and works identically on macOS, Windows, and Linux.

The same applies to active window detection (`active-win` calls `NSWorkspace` via native bindings) and browser URL extraction (`osascript` in Node produces the same result as `NSAppleScript` in Swift).

Going native buys nothing for the capture layer.

### The UI Demands Web Technologies

The application's UI consists of real-time dashboards, filterable data tables, configuration panels, rich log viewers with syntax highlighting, and interactive activity timelines. This is exactly where web technologies excel and native frameworks struggle.

Applications with similar UI complexity — Notion, Slack, Discord, Figma, Linear — use web rendering for their content areas. If you go native Swift and then embed WKWebView for the complex UI portions, you end up with two rendering paradigms and a bridge between them — strictly worse than Electron.

### Development Velocity

- **Build times**: Swift compilation is notoriously slow. Incremental builds in Xcode can take 30–60 seconds. Vite/esbuild hot reloading is near-instant.
- **Debugging**: VS Code and Chrome DevTools provide unified debugging across Electron processes. Xcode only covers Swift — debugging the embedded Node.js sidecar requires a separate toolchain.
- **Package ecosystem**: npm has over 2 million packages. SPM has a fraction, concentrated in iOS/macOS UI libraries.

### Hiring

TypeScript developers who can work across frontend UI, Playwright automations, Mastra.ai agents, and RAG pipelines are abundant. Finding Swift developers with experience in browser automation, AI agents, or data pipeline orchestration is exceptionally difficult because those domains barely exist in Swift.

---

## Why Not Tauri

### The Sidecar Problem

In Tauri, the Rust backend would manage windows and the system tray — nothing more. All real work lives in a Node.js sidecar:

```
┌──────────────────────────────────────────────────────────┐
│                   TAURI (Two Runtimes)                    │
│                                                          │
│  ┌────────────────────────────────────┐                  │
│  │        Rust Backend (Tauri Core)   │                  │
│  │                                    │                  │
│  │  ┌──────────────┐                 │                  │
│  │  │ Window Mgmt  │  That's         │                  │
│  │  │ System Tray  │  basically it.  │                  │
│  │  │ File I/O     │                 │                  │
│  │  └──────────────┘                 │                  │
│  └──────────────┬─────────────────────┘                  │
│                 │                                        │
│                 │ IPC (stdin/stdout, HTTP, or WebSocket)  │
│                 │ Must serialize EVERYTHING               │
│                 │ Cross-process debugging                 │
│                 │ Error handling across boundary          │
│                 │                                        │
│  ┌──────────────▼─────────────────────┐                  │
│  │     Node.js Sidecar Process        │                  │
│  │                                    │                  │
│  │  ┌─────────────┐ ┌──────────────┐ │                  │
│  │  │  Playwright  │ │ Mastra.ai    │ │                  │
│  │  │  Automations │ │ Agents       │ │                  │
│  │  └─────────────┘ └──────────────┘ │                  │
│  │  ┌─────────────┐ ┌──────────────┐ │                  │
│  │  │  Local RAG   │ │ uiohook-napi │ │                  │
│  │  │  Pipeline    │ │ active-win   │ │                  │
│  │  └─────────────┘ │ AppleScript  │ │                  │
│  │                   └──────────────┘ │                  │
│  └────────────────────────────────────┘                  │
│                 │                                        │
│  ┌──────────────▼─────────────────────┐                  │
│  │        Webview (OS native)         │                  │
│  │   React/Svelte UI — Dashboard      │                  │
│  └────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────┘

  Rust backend does almost nothing. All real work is in the
  Node sidecar. Two runtimes for no meaningful benefit.
```

### Sidecar Complexity Is Real Engineering Cost

The Tauri sidecar pattern requires:

1. Bundling a Node.js runtime alongside the Tauri app
2. Implementing an IPC protocol between Rust and Node
3. Managing the lifecycle of the sidecar process (start, restart on crash, graceful shutdown)
4. Serializing all data crossing the boundary (keystroke events, agent outputs, RAG results)
5. Handling errors across the process boundary with no shared stack traces
6. Testing across two runtimes

This is weeks of infrastructure work that produces zero user-facing value.

### Debugging Across Runtimes

With Electron, all processes are Node.js. A single VS Code launch configuration can attach to the main process, and `console.log` in utility processes routes to the same terminal. Chrome DevTools covers the renderer. All stack traces are JavaScript/TypeScript — one language, one set of tools, one mental model.

With Tauri + sidecar, debugging requires monitoring a Rust process and a Node.js process simultaneously, correlating logs across runtimes, and potentially running two debuggers with different toolchains. When something breaks in the IPC layer, root cause analysis becomes significantly harder because the stack trace ends at the runtime boundary.

### No Meaningful Rust Benefit

Tauri's Rust backend shines when the application needs CPU-intensive computation (image processing, compression, cryptography), low-level OS access that Node.js can't do, or memory-safe concurrent data processing.

Our application needs none of these from the desktop shell. The heavy lifting (browser automation, AI inference, embedding generation) all happens through Node.js libraries calling external APIs or spawning managed processes. Rust adds nothing to this workload.

### Tauri's Advantages Don't Apply Here

**Binary size**: Tauri apps are typically 5–15 MB vs Electron's 150–300 MB. But this application also bundles Playwright browser binaries (~200+ MB for Chromium), embedding models or API clients, vector store data, and SQLite databases. The Electron overhead is rounding error in the total application size.

**Memory usage**: Tauri's base memory footprint is lower (~30 MB vs ~80–150 MB). But this application runs Playwright browser instances (hundreds of MB each), AI agent execution, embedding generation, continuous global input capture, and in-memory vector indices. The Electron base overhead is negligible compared to actual workload consumption.

**Native feel**: Tauri uses the OS native webview, which can feel slightly more native. But the application's UI is primarily dashboards, data tables, and configuration panels — functional screens where clarity matters more than mimicking native macOS controls.

---

## Decision: Electron

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│          ELECTRON (Single Runtime, Multi-Process)         │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Main Process (Node.js)                 │  │
│  │                                                     │  │
│  │  - App lifecycle management                         │  │
│  │  - IPC routing between processes                    │  │
│  │  - System tray                                      │  │
│  │  - better-sqlite3 (shared local DB)                 │  │
│  │  - Light coordination only — no heavy work here     │  │
│  └────┬──────────────┬──────────────┬─────────────────┘  │
│       │              │              │                     │
│       │ IPC          │ IPC          │ IPC                 │
│       │ (built-in)   │ (built-in)   │ (built-in)         │
│       │              │              │                     │
│  ┌────▼───────────┐ ┌▼───────────┐ ┌▼──────────────────┐ │
│  │ Utility Process │ │  Utility   │ │ Renderer Process  │ │
│  │                 │ │  Process   │ │ (Chromium)        │ │
│  │ DATA CAPTURE    │ │            │ │                   │ │
│  │                 │ │ AGENTS &   │ │ React/Svelte UI   │ │
│  │ ┌─────────────┐│ │ RAG        │ │ Dashboard         │ │
│  │ │ uiohook-napi││ │            │ │ Config, Logs      │ │
│  │ │ (global keys ││ │ ┌────────┐│ │                   │ │
│  │ │  + mouse)    ││ │ │Mastra  ││ │ Fully sandboxed.  │ │
│  │ ├─────────────┤│ │ │.ai     ││ │ contextIsolation  │ │
│  │ │ active-win   ││ │ │Agents  ││ │ nodeIntegration:  │ │
│  │ │ (focused app)││ │ ├────────┤│ │ false             │ │
│  │ ├─────────────┤│ │ │Embed-  ││ │ sandbox: true     │ │
│  │ │ AppleScript  ││ │ │dings   ││ │                   │ │
│  │ │ (browser URL)││ │ ├────────┤│ │ No Node.js access.│ │
│  │ └─────────────┘│ │ │Vector  ││ │ All privileged    │ │
│  │                 │ │ │Store   ││ │ ops go through    │ │
│  │ Crash-isolated. │ │ └────────┘│ │ explicit IPC      │ │
│  │ Auto-restarts.  │ │           │ │ channels.         │ │
│  └─────────────────┘ └─────┬─────┘ └───────────────────┘ │
│                             │                             │
│                      ┌──────▼──────────┐                  │
│                      │ Child Process   │                  │
│                      │                 │                  │
│                      │ Playwright      │                  │
│                      │ (spawns its own │                  │
│                      │ browser — fully │                  │
│                      │ isolated from   │                  │
│                      │ app UI)         │                  │
│                      └─────────────────┘                  │
└──────────────────────────────────────────────────────────┘

  One runtime (Node.js). One language (TypeScript). Multiple
  processes for fault isolation. All IPC is Electron built-in
  — no custom protocol needed.
```

### Why Multi-Process Matters

Each workload runs in its own process for fault isolation and performance:

- **Capture layer** is crash-isolated — if uiohook-napi's native addon segfaults, only the capture utility process dies and auto-restarts. The UI, agents, and automations are unaffected.
- **Agents and RAG** are isolated from the UI — CPU-heavy embedding generation and LLM calls won't block the renderer or freeze the dashboard.
- **Playwright** runs as a child process and spawns its own browser instances — a hung automation cannot take down the app.
- **Renderer is fully sandboxed** — `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`. Zero Node.js access. All privileged operations go through explicitly defined IPC channels. This is functionally the same security model as Tauri's command system.

All processes communicate through Electron's built-in IPC — fast, battle-tested, no custom serialization protocol.

### What Electron Gives Us

- Every dependency runs natively in the Node.js runtime
- Multiple processes for fault isolation, all using built-in IPC
- Crash-isolated capture, agents, and rendering
- Fully sandboxed renderer with the same security model as Tauri
- Battle-tested ecosystem for desktop apps with heavy Node.js backends
- One language, one debugger, one mental model across the entire application
- Focus stays on the product, not the plumbing

---

## Anti-Patterns to Avoid

Do NOT run everything in the main process. Specifically:

- **uiohook-napi in the main process** — it is a native C addon. A segfault in the native layer kills the entire app with no recovery. Isolate it in a utility process that can auto-restart.
- **Playwright in the main process** — a hung automation or browser crash blocks the event loop. Run it as a child process. Playwright already spawns its own browser instances, so this is natural.
- **Embedding generation in the main process** — CPU-heavy computation blocks IPC routing and window management. Isolate in a utility process.
- **nodeIntegration: true in the renderer** — any XSS vulnerability in the UI becomes full system access. Never enable this.

---

## Risk Mitigation

### Avoiding Electron Lock-in

Structure the codebase so that all business logic is in pure TypeScript modules with zero Electron imports:

```
src/
├── core/                  # Pure TypeScript — no Electron dependency
│   ├── automations/       # Playwright flows
│   ├── agents/            # Mastra.ai agent bundles
│   ├── rag/               # RAG pipeline
│   ├── capture/           # uiohook-napi, active-win, URL extraction
│   └── store/             # better-sqlite3 data layer
├── electron/              # Electron-specific wiring
│   ├── main.ts            # Main process — lifecycle, IPC routing
│   ├── preload.ts         # Preload script for renderer
│   ├── capture.worker.ts  # Utility process entry — data capture
│   └── agents.worker.ts   # Utility process entry — agents & RAG
├── renderer/              # Frontend UI (sandboxed, no Node.js)
└── shared/                # Types, IPC channel definitions, constants
```

The `electron/` directory contains only process entry points and IPC wiring. The `core/` directory contains all business logic with zero Electron imports, so it lifts out cleanly. If migration is ever needed (to Tauri + sidecar, to a CLI tool, to a headless service), the business logic moves without rewriting.

### Security

Electron's security has improved substantially. The multi-process architecture reinforces it:

- Renderer runs fully sandboxed with `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`
- All privileged operations go through explicitly defined IPC channels — functionally the same security model as Tauri's command system
- Process boundaries between capture, agents, and rendering are real security boundaries

---

## Summary

| Dimension | Native Swift | Tauri | Electron |
|---|---|---|---|
| Playwright | Needs embedded Node.js | Needs Node sidecar | Runs natively |
| Mastra.ai agents | Needs embedded Node.js | Needs Node sidecar | Runs natively |
| RAG ecosystem | Near-empty in Swift | Needs Node sidecar | Rich JS/TS ecosystem |
| Global input capture | Same APIs, no advantage | Rust crates exist, but Node sidecar likely | uiohook-napi, clean API |
| Browser URL extraction | Same AppleScript | Same AppleScript via sidecar | Same AppleScript |
| Dashboard / data table UI | Painful in SwiftUI/AppKit | Web (good) | Web (good) |
| AI/automation ecosystem | Nonexistent | Needs Node sidecar | Largest ecosystem |
| Development velocity | Slow builds, thin tooling | Good for Rust, but Node sidecar adds overhead | Hot reload, instant iteration |
| Hiring pool | Niche | Medium (Rust + TS) | Large (TypeScript only) |
| Runtime architecture | Swift shell + Node sidecar + bridge | Rust shell + Node sidecar + bridge | Single runtime, built-in IPC |
| Runtimes to manage | 2 | 2 | 1 |
| Debugger count | 2 | 2 | 1 |
| IPC complexity | Hand-rolled, no community support | Tauri sidecar pattern (documented but still custom) | Built-in, battle-tested |
| Binary size | Small shell + Node runtime | Small shell + Node runtime | ~150–300 MB (negligible given Playwright) |
| Memory footprint | Lower base, same workload | Lower base, same workload | Higher base, same workload |

**Electron is the correct choice. Ship the product.**
