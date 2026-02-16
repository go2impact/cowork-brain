# 6. Electron Architecture & Local-First Storage

[< Back to index](README.md)

**Their claim:** "Built on Electron for cross-platform native experience" + "Data stored locally for privacy protection"

**Maps to Cowork.ai:** Our entire desktop app. Same framework (Electron), same local-first philosophy, similar process model.

## How It Works

Standard Electron architecture: main process (Node.js) manages all heavy logic via Manager singletons, renderer process (React) communicates exclusively through context-isolated IPC. Database is better-sqlite3 via TypeORM for structured data, libsql for vectors.

**Key files:**
- `src/main/main.ts` — App bootstrap
- `src/main/preload.ts` — IPC bridge (1,384 lines)
- `src/main/BaseManager.ts` — Manager base class (35 lines)
- `src/main/ipc/IpcController.ts` — `@channel` decorator
- `src/main/db/index.ts` — DBManager (TypeORM setup)

## Boot Sequence

```
app.whenReady()
       │
       ▼
  Manager Initialization (sequential):
  ┌──────────────────────────────────────────────────────┐
  │  1. dbManager.init()         → TypeORM + SQLite      │
  │  2. providersManager.init()  → LLM provider registry │
  │  3. appManager.init()        → Settings, proxy, theme│
  │  4. mastraManager.init()     → Mastra + Express API  │
  │  5. knowledgeBaseManager.init() → RAG setup          │
  │  6. toolsManager.init()      → MCP + skills + tools  │
  │  7. localModelManager.init() → Ollama model cache    │
  │  8. agentManager.init()      → Agent persistence     │
  │  9. projectManager.init()    → Project management    │
  │ 10. updateManager.init()     → electron-updater      │
  │ 11. instancesManager.init()  → Playwright pool       │
  │ 12. taskQueueManager.init()  → Background jobs       │
  └──────────────────────────────────────────────────────┘
       │
       ▼
  BrowserWindow({
    contextIsolation: true,
    nodeIntegration: false,
    preload: 'preload.js'
  })
       │
       ▼
  loadURL('index.html')  →  React app
```

## IPC Architecture (Decorator Pattern)

```
MAIN PROCESS                                    RENDERER
────────────                                    ────────

class SomeManager extends BaseManager {         window.electron.some.doThing(args)
                                                       │
  @channel('some:doThing')                             │
  async doThing(args) {  ◀────── ipcMain.handle ──────┘
    return result;       ────── ipcRenderer.invoke ──────▶ result
  }

  constructor() {
    super();  // ◀── calls registerIpcChannels()
              //     reads _ipcChannels metadata
              //     binds ipcMain.handle() per method
  }
}

BaseManager.registerIpcChannels():
  channels.forEach(item => {
    if (mode === 'invoke')  → ipcMain.handle(channel, method)
    if (mode === 'on')      → ipcMain.on(channel, method)
  })
```

## Database Architecture

```
┌──────────────────────── DATA STORAGE ────────────────────────┐
│                                                                │
│  ~/Library/Application Support/aime-chat/                     │
│  ├── data/                                                     │
│  │   └── main.db          ◀── better-sqlite3 via TypeORM      │
│  │       ├─ providers           (API keys, endpoints)          │
│  │       ├─ settings            (theme, proxy, language)       │
│  │       ├─ knowledge_base      (KB definitions)               │
│  │       ├─ knowledge_base_item (KB documents)                 │
│  │       ├─ secrets             (encrypted credentials)        │
│  │       ├─ tools               (MCP + skill definitions)     │
│  │       ├─ mastra_history_messages (chat history)             │
│  │       ├─ mastra_threads_usage    (token/cost tracking)     │
│  │       ├─ agents              (agent definitions)            │
│  │       ├─ projects            (project metadata)             │
│  │       ├─ translations        (translation cache)            │
│  │       ├─ instances           (browser configs)              │
│  │       ├─ mastra_history_messages (chat history — Mastra)    │
│  │       ├─ mastra_threads_usage    (token/cost — Mastra)     │
│  │       └─ [kb_{id}_{dim}] tables  (vectors — Mastra)        │
│  │                                                              │
│  │   NOTE: Mastra's LibSQLStore also uses this same main.db   │
│  │   via getDbPath(). Two access patterns (TypeORM + libsql), │
│  │   one physical database file.                                │
│  │                                                              │
│  ├── models/               ◀── Local LLM files                 │
│  └── instances/            ◀── Playwright browser profiles     │
│                                                                │
│  KEY DESIGN DECISIONS:                                         │
│  • ONE database file (main.db), two access patterns:          │
│    TypeORM+better-sqlite3 (app data) + libsql (Mastra/vectors)│
│  • TypeORM with synchronize:true (auto-migrate, no files)     │
│  • PRAGMA foreign_keys = ON (explicitly enabled)              │
│  • Express API on port 41100 (optional, localhost only)       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Electron Process Model (Full)

```
┌─────────────────────── MAIN PROCESS ─────────────────────────┐
│                                                                │
│  12 Manager Singletons                                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐│
│  │ DBManager  │ │ Providers  │ │ AppManager │ │  Mastra    ││
│  │ (TypeORM)  │ │ Manager    │ │ (settings, │ │  Manager   ││
│  │            │ │ (15+ LLMs) │ │  capture)  │ │ (+Express) ││
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘│
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐│
│  │ Knowledge  │ │   Tools    │ │ LocalModel │ │  Agent     ││
│  │ Base Mgr   │ │  Manager   │ │  Manager   │ │  Manager   ││
│  │ (RAG)      │ │ (MCP+25+) │ │ (Ollama)   │ │            ││
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘│
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐│
│  │  Project   │ │  Update    │ │ Instances  │ │ TaskQueue  ││
│  │  Manager   │ │  Manager   │ │ Manager    │ │ Manager    ││
│  │            │ │(auto-upd.) │ │(Playwright)│ │(background)││
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘│
│                                                                │
│  Express API Server (localhost:41100)                          │
│  └─ POST /mcp (MCP bridge for external tool access)          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                          │
                   Context-Isolated IPC
                   (@channel decorators)
                          │
┌─────────────────────── PRELOAD ──────────────────────────────┐
│  contextBridge.exposeInMainWorld('electron', {               │
│    app: { ... },      providers: { ... },                    │
│    mastra: { ... },   knowledgeBase: { ... },                │
│    tools: { ... },    agents: { ... },                       │
│    projects: { ... }, taskQueue: { ... },                    │
│    instances: { ... },localModel: { ... }                    │
│  })                                                           │
└──────────────────────────────────────────────────────────────┘
                          │
                   window.electron.*
                          │
┌─────────────────────── RENDERER ─────────────────────────────┐
│  React 19 + React Router 7 + Zustand 5                       │
│  shadcn/ui (Radix) + Tailwind CSS 4.1                        │
│  Webpack HMR (dev) / code-split (prod)                       │
└──────────────────────────────────────────────────────────────┘
```

## What We Should Do

| Action | Detail |
|--------|--------|
| **Copy pattern** | The `BaseManager` + `@channel` decorator pattern for IPC. Clean, type-safe, auto-registered. Saves boilerplate. **Adapt for our utility-process model** — their managers all live in main process; ours split across Main, Capture Utility, and Agents & RAG Utility (see [system-architecture.md](../../../architecture/system-architecture.md#process-model)). |
| **Copy pattern** | Sequential manager initialization order with clear dependencies. Adapt for multi-process: each utility process has its own boot sequence. |
| **Study** | Their single-file, two-access-pattern approach (TypeORM/better-sqlite3 + libsql both pointing at `main.db`). We chose to unify on libsql only — simpler, but their TypeORM layer adds auto-migration and entity relations we'll need to handle ourselves. |
| **Study** | The preload script structure — 383 lines of typed IPC bindings organized by namespace. Good reference for our API surface. |
| **Study** | `TaskQueueManager` for background jobs with concurrency groups. We need this for automations and embedding jobs. |
| **Copy pattern** | Screen capture implementation: macOS `screencapture` CLI for native quality, `desktopCapturer` fallback for Windows/Linux. |
| **Skip** | `synchronize: true` in TypeORM (auto-migrate). Fine for a solo dev, risky for production. We should use proper migrations. |
| **Note** | They run Electron 35 (Node 22.16.0). We're on Electron 37. Close enough that their patterns apply directly. |
