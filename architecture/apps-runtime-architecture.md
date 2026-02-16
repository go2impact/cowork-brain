# Apps Runtime Architecture

**v1 — February 16, 2026**
**Status:** Draft — open for discussion
**Audience:** Engineering (Rustan + team)
**Purpose:** Design document for how Cowork.ai renders third-party apps, how those apps communicate with the platform, and how we handle the AI Studio app format at runtime.

**Related open questions:**
- [system-architecture.md — Open Question #3](./system-architecture.md#open-architecture-questions) (App Gallery hosting)
- [aime-adaptation-guide.md — Section 3 & 4](./aime-adaptation-guide.md#3-mcp-integration) (MCP server role, Express HTTP skip)

**Context:** [product-features.md — Apps](../product/product-features.md#1-apps) defines what Apps are. This doc explores how they might run.

---

## The Problem

product-features.md says: "Apps built in Google AI Studio are uploaded to the Cowork.ai desktop app and rendered inside it. Apps get tools, not agents — they access platform capabilities through platform-provided MCPs."

**None of this is decided yet.** This doc is research and options analysis — not a decision record. The following questions are all open:

1. What rendering primitive hosts the app (webview, iframe, something else)
2. How the app calls platform MCP tools (the transport question)
3. How we handle the AI Studio export format at runtime (it's not just HTML)
4. How we sandbox untrusted app code
5. How per-app permissions are enforced

Where the doc says "Decision" (e.g., WebContentsView for rendering), it's a strong recommendation based on research — not a finalized decision. All choices here need review before committing.

---

## What Google AI Studio Apps Actually Are

Research into AI Studio exports revealed that these are **full Node.js + React web applications**, not lightweight HTML bundles. The exact file structure varies by template, but the typical pattern observed from community exports and deployment guides:

### Export contents (zip, typical)

```
exported-app/
├── src/
│   ├── App.tsx                    # React entry point (Angular is an alternative)
│   ├── components/                # UI components
│   ├── hooks/                     # React hooks
│   ├── services/
│   │   └── geminiService.ts       # Gemini API calls — the core AI logic
│   └── utils/
├── server/
│   └── server.js                  # Express proxy — injects GEMINI_API_KEY
├── public/
│   ├── service-worker.js          # Intercepts fetch to Gemini API → redirects to proxy
│   └── websocket-interceptor.js   # Monkey-patches WebSocket for streaming
├── package.json                   # Dependencies: @google/genai, express, react, vite
├── vite.config.ts
├── .env.example                   # GEMINI_API_KEY placeholder
└── tsconfig.json
```

**Note:** This structure is observed from community exports and deployment guides, not from an official spec. Google does not publish a formal export file contract — the Build mode generates project code, and its shape may vary across templates and over time.

### How they communicate with AI

Apps call the Gemini API directly via `@google/genai` SDK. The typical pattern includes an Express proxy server (`server/server.js`) that injects the API key so it never hits the browser, with a service worker that intercepts `fetch` calls to `generativelanguage.googleapis.com` and redirects them to the local proxy.

### MCP support status

MCP is supported in the `@google/genai` SDK and documented in Google's [Interactions API](https://ai.google.dev/gemini-api/docs/interactions). The SDK provides `mcpToTool()` for converting MCP clients into Gemini-compatible tools. However, the AI Studio Build mode UI does not expose MCP configuration — there's no "Add MCP Server" panel. If an app uses MCP, it's coded manually. There's no manifest declaring MCP tool requirements.

### No declarative manifest for platform capabilities

There is no `manifest.json` or equivalent declaring app capabilities, required MCP tools, or permissions. `package.json` is the only metadata. Google's Build mode generates standard web projects, not a platform-specific app format. This means our platform needs to define its own manifest format for apps to declare what they need.

### Prior art: existing Swift codebase

The [coworkai-mac-native-desktop](../repos.md) repo has a **working app runtime** that renders Google AI Studio apps on macOS native (Swift + WKWebView). This is not necessarily the correct approach for the Electron version — it's additional context showing that this particular method worked in a native macOS environment. The Electron version may need a different approach due to different rendering primitives, security models, and process architecture. What follows is the detailed reference for context, not a blueprint to copy.

**Key files** (all paths relative to `coworkai-mac-native-desktop/coworkai-mac-native-desktop/`):

| Component | File |
|---|---|
| Preprocessor | `Core/Services/AppManager/AppPreprocessor.swift` |
| App lifecycle | `Core/Services/AppManager/AppManager.swift` |
| JS SDK (source in Swift string) | `Core/Services/AppManager/CoworkAISDK.swift` |
| Bridge (Swift ↔ JS) | `Features/Apps/CoworkAIBridge.swift` |
| WebView wrapper | `Features/Apps/AppWebView.swift` |
| Mastra HTTP client | `Core/Services/Mastra/MastraClient.swift` |

#### Preprocessing pipeline (AppPreprocessor.swift)

Uses **Babel standalone** (~2.7 MB) running in **JavaScriptCore** (macOS built-in). No Node.js, no npm, no external tooling. Babel is downloaded once from CDN and cached locally.

**Babel config:**
```
presets: [
  ['react', { runtime: 'automatic' }],       // JSX → _jsx() function calls
  ['typescript', { allExtensions: true, isTSX: true/false }]
]
sourceType: 'module'
```

**Transformations performed (in order):**

| Step | What | Example |
|---|---|---|
| **Transpile** | TSX/TS → JS via Babel in JavaScriptCore | `App.tsx` → `App.js` (originals retained alongside) |
| **Rewrite imports** | Bare relative imports → add `.js` extension | `from './App'` → `from './App.js'`, `from './App.tsx'` → `from './App.js'`. Preserves third-party imports (`react`) and non-JS extensions (`.css`, `.json`, `.svg`). Regex: `((?:from\|import\()\s*['"])(\.\.?\/[^'"]+)(['"\)])` |
| **Patch HTML** | Rewrite script/link paths in `index.html` | `src="/xxx.tsx"` → `src="./xxx.js"`, absolute paths → relative |
| **Inject env vars** | Read `.env.local` / `.env`, inject `<script>` polyfill into `</head>` | `window.process = window.process \|\| {}; window.process.env = { GEMINI_API_KEY: "...", ... };` Also maps `GEMINI_API_KEY` → `API_KEY` for Vite compat. |
| **Mark processed** | Write `.coworkai-processed` marker file | Used to skip re-processing on subsequent loads |

**What this skips:** No `npm install`, no `node_modules`, no Vite build, no Express proxy server. The preprocessor produces browser-ready ES modules directly.

#### SDK injection (CoworkAISDK.swift)

The JS SDK is **not baked into app HTML** — it's injected at runtime via `WKUserScript` at `atDocumentEnd`. This means the SDK stays up-to-date without re-processing installed apps.

**SDK API surface (`window.coworkai.*`):**

| Method | Signature | What it does |
|---|---|---|
| `callTool` | `callTool(name, args) → Promise<object>` | Call an MCP tool via the platform |
| `chat` | `chat(agentId, message) → Promise<string>` | Chat with a Mastra agent (agent-as-tool) |
| `listTools` | `listTools() → Promise<object[]>` | Get all available MCP tools |
| `onMessage` | `onMessage(callback) → void` | Register listener for platform-pushed data |

Internal methods called by Swift: `_resolve(id, result)`, `_reject(id, error)`, `_onPush(data)`.

Each JS→Swift call gets an auto-incrementing string ID. A `_pending` Map tracks `id → {resolve, reject}` pairs. Swift evaluates JS to resolve/reject.

#### Communication bridge (CoworkAIBridge.swift)

**JS → Swift:** `window.webkit.messageHandlers.coworkai.postMessage(JSON.stringify({...}))` with message format:
```json
{ "id": "1", "action": "callTool" | "chat" | "listTools", "payload": { "name": "...", "arguments": {...} } }
```

**Swift → JS:** After async work completes:
```swift
webView?.evaluateJavaScript("window.coworkai._resolve('\(id)', \(resultJSON))")
// or
webView?.evaluateJavaScript("window.coworkai._reject('\(id)', '\(error)')")
```

#### Backend protocol (MastraClient.swift)

Mastra runs as a separate process on `localhost:4111`. The bridge calls it via HTTP:

| Action | Protocol | Endpoint | Body |
|---|---|---|---|
| `callTool` | JSON-RPC 2.0 | `POST /api/mcp/coworkai/mcp` | `{ jsonrpc: "2.0", id: UUID, method: "tools/call", params: { name, arguments } }` |
| `listTools` | JSON-RPC 2.0 | `POST /api/mcp/coworkai/mcp` | `{ jsonrpc: "2.0", id: UUID, method: "tools/list", params: {} }` |
| `chat` | REST | `POST /api/agents/{agentId}/generate` | `{ messages: [{ role: "user", content: message }] }` |

#### Installation flow (AppManager.swift)

```
User picks .zip → extract to ~/Library/Application Support/{bundleId}/InstalledApps/{UUID}/
  → read metadata.json (fallback: package.json name) → preprocess → write manifest → app appears in UI
```

Manifest stored at: `InstalledApps/coworkai_apps_manifest.json` (JSON array of `InstalledApp` records with `id`, `name`, `description`, `installedAt`, `directoryPath`).

#### What's NOT implemented in Swift

- **No per-app permissions.** All apps get full access to all MCP tools and all agents. No scoping, no capability declarations.
- **No streaming.** Agent chat uses `generate` (non-streaming), not `stream`.
- **No approval gates.** `callTool` executes immediately with no user confirmation.
- **Agent ID hardcoded** in the built-in chat widget (`'weather-agent'`).

#### What maps to Electron

| Swift | Electron equivalent |
|---|---|
| `WKWebView` | `WebContentsView` |
| `WKScriptMessageHandler` | `contextBridge` + `ipcRenderer` in preload |
| `WKUserScript` (SDK injection at documentEnd) | Preload script (injected automatically) |
| Babel in JavaScriptCore | esbuild or Babel in Node.js (faster, more options) |
| `allowFileAccessFromFileURLs` | Custom `cowork-app://` protocol (more secure) |
| `loadFileURL(indexURL, allowingReadAccessTo: directoryURL)` | `cowork-app://{app-id}/index.html` |
| HTTP to `localhost:4111` | IPC to Agents & RAG utility process (no network) |
| No permissions | Per-app permission model with manifest (see below) |

---

## Rendering: WebContentsView

### Decision

Use Electron's `WebContentsView` to render each app. One `WebContentsView` per installed app.

### Why not the alternatives

| Option | Status | Why not |
|---|---|---|
| `<webview>` tag | Discouraged by Electron team, may be removed | "Not guaranteed to remain available in future versions" |
| `BrowserView` | Deprecated since Electron 30 | Internally shimmed over `WebContentsView` already |
| `<iframe>` | Works but limited | No preload scripts, no `contextBridge`, shares parent process, insufficient for untrusted content |
| `WebContentsView` | **Recommended** | Own renderer process, full `webPreferences` control, preload support, process isolation |

VS Code (iframe-backed webviews with `postMessage`), Slack (`BrowserView` → `WebContentsView`), and Figma (original `BrowserView` inventor) all converged on the same principle: isolated content with a controlled communication bridge. The rendering primitive varies, but the security model is consistent.

### Security configuration

Each app's `WebContentsView` is locked down:

```typescript
const appView = new WebContentsView({
  webPreferences: {
    preload: path.join(__dirname, 'app-preload.js'),
    contextIsolation: true,       // preload world ≠ page world
    sandbox: true,                // OS-level process sandbox
    nodeIntegration: false,       // no require(), no process, no fs
    nodeIntegrationInSubFrames: false,
    webSecurity: true,
    allowRunningInsecureContent: false,
    webviewTag: false,            // prevent nested webviews
    partition: `persist:app-${appId}`  // isolated cookies/storage per app
  }
})

// Block navigation away from app content
appView.webContents.on('will-navigate', (details) => {
  details.preventDefault()
})

// Block window.open()
appView.webContents.setWindowOpenHandler(() => ({ action: 'deny' }))
```

### File serving: custom protocol

Load app files via a custom `cowork-app://` protocol instead of `file://`. This prevents cross-origin issues and gives us path validation against directory traversal.

```typescript
// Must run BEFORE app.whenReady() — can only be called once
protocol.registerSchemesAsPrivileged([{
  scheme: 'cowork-app',
  privileges: {
    standard: true,        // relative URLs work, localStorage available
    secure: true,          // treated as HTTPS (required for many web APIs)
    supportFetchAPI: true, // fetch() works
  }
}])

app.whenReady().then(() => {
  // Register on each app's session partition (not the default session)
  // since each WebContentsView uses partition: `persist:app-${appId}`
  function registerAppProtocol(appId: string) {
    const ses = session.fromPartition(`persist:app-${appId}`)
    ses.protocol.handle('cowork-app', (request) => {
      const { host, pathname } = new URL(request.url)
      const appDir = getAppDirectory(host)
      const filePath = path.resolve(appDir, pathname.slice(1))
      const relativePath = path.relative(appDir, filePath)

      // Prevent directory traversal
      if (!relativePath || relativePath.startsWith('..') || path.isAbsolute(relativePath)) {
        return new Response('Forbidden', { status: 403 })
      }

      return net.fetch(pathToFileURL(filePath).toString())
    })
  }
})
```

App loads as: `cowork-app://{app-id}/index.html`

---

## App Installation & Preprocessing

### The problem with running AI Studio exports as-is

Typical AI Studio exports expect a standard web deployment workflow:
1. `npm install` to resolve dependencies
2. `vite build` (or similar) to produce a production bundle
3. An Express server running to proxy Gemini API calls (injecting API key)
4. `GEMINI_API_KEY` as an environment variable

The exact requirements vary by template, but the common pattern is a Node.js project that needs a build step and a server process. Running `npm install` + a build tool + an Express server per app is heavyweight for a desktop runtime. The Swift codebase solved this with preprocessing — we should do the same, adapted for our Electron context.

### Installation flow

```
User uploads .zip from AI Studio
    ↓
1. Extract to ~/Library/Application Support/cowork-ai/apps/{uuid}/
    ↓
2. Read package.json for metadata (name, description)
    ↓
3. Preprocess (see below)
    ↓
4. Write app manifest (our format, not AI Studio's)
    ↓
5. App appears in sidesheet
```

### Preprocessing pipeline

Adapt the Swift codebase's `AppPreprocessor` approach to Node.js:

| Step | What | Why |
|---|---|---|
| **Transpile** | TSX/TS → JS via Babel or esbuild | Browser can't run TypeScript directly |
| **Rewrite imports** | `from './App'` → `from './App.js'`, bare specifiers → import map | ESM in browser needs explicit extensions |
| **Patch HTML** | Rewrite `<script src>` paths, inject env vars polyfill | Vite-specific paths won't resolve without Vite |
| **Inject import map** | Map `react`, `@google/genai`, etc. to CDN or bundled copies | No `node_modules/` — dependencies served via import map or pre-bundled |
| **Inject platform SDK** | Add `<script>` for `window.cowork` bridge (or rely on preload) | App needs access to platform MCP tools |
| **Strip Express proxy** | Remove `server/` directory, rewrite service worker | We handle API key injection at the platform level |
| **Mark processed** | Write `.cowork-processed` marker file | Skip re-processing on next load |

**Gemini API key handling:** The platform manages the API key. During preprocessing, we rewrite the service worker to route Gemini API calls through our own proxy (running in the main process or utility process), which injects the key. The app never sees the raw key.

### Open question: esbuild vs Babel for preprocessing

The Swift codebase uses Babel via JavaScriptCore. In Node.js (Electron), we have better options:
- **esbuild** — fast, handles TSX/TS natively, can bundle. Could produce a single output file.
- **Babel** — more configurable, matches the Swift codebase's approach.
- **Full Vite build** — runs the app's own build tooling. Most correct output, but slow and requires `npm install`.

Lean toward esbuild for speed. Needs testing with real AI Studio exports.

---

## Platform Communication: How Apps Call MCP Tools

This is the core design question. Three options were researched.

### Option A: Preload bridge (IPC relay through main)

```
App JS → window.cowork.callTool()
  → preload (contextBridge) → ipcRenderer.invoke('mcp:call-tool', ...)
  → main process IPC handler
  → main forwards to Agents & RAG utility process
  → Mastra agent executes MCP tool
  → result returns through same chain
```

**Pros:** Battle-tested Electron pattern (VS Code, Slack). Full security control. Per-app permission scoping is natural — each `WebContentsView` gets its own preload. No network exposure.

**Cons:** Two IPC hops (renderer→main→utility). Main process relays every message. Apps use proprietary `window.cowork.*` API, not standard MCP clients.

### Option B: Local HTTP MCP server

```
App JS → StreamableHTTPClientTransport → HTTP POST to localhost:{port}/mcp
  → Express/Streamable HTTP server in utility process
  → Mastra agent executes MCP tool
  → HTTP response back to app
```

**Pros:** Apps can use standard MCP client libraries. Clean separation. Easy to test independently.

**Cons:** Localhost port exposed. **CVE-2025-49596** (June 2025, CVSS 9.4) demonstrated this attack class — Anthropic's MCP Inspector (<0.14.1) lacked authentication between the Inspector client and proxy, allowing unauthenticated requests to invoke MCP tools via DNS rebinding. While the CVE is specific to MCP Inspector, the underlying vulnerability pattern (unauthenticated localhost HTTP server + DNS rebinding) applies to any local MCP HTTP server. Mitigations exist (per-session tokens, Origin validation, Unix domain sockets) but add complexity. Per-app permissions require auth token management.

### Option C: MessagePort direct channel

```
Setup: main creates MessageChannelMain → transfers port1 to utility, port2 to app's webview

Runtime:
App JS → MessagePortTransport.send() → port.postMessage()
  → [Electron MessagePort channel]
  → utility process port.onmessage → MCP Server
  → Mastra agent executes MCP tool
  → response flows back through same port
```

**Pros:** Single hop after setup (renderer↔utility, main not in path). Lowest latency (<1ms overhead). No network exposure. The MCP SDK's `Transport` interface is trivial to implement over MessagePort (~30 LOC). Per-app permissions natural — each app gets its own port.

**Cons:** More setup choreography (main must broker initial port transfer). Still needs a preload script to receive the port. Apps still can't use `StreamableHTTPClientTransport` directly — they'd use our `MessagePortTransport`.

### Option D: Hybrid (preload bridge + MessagePort)

```
Setup:
1. App loads in WebContentsView with preload
2. Preload exposes window.cowork.* via contextBridge
3. Under the hood, preload receives a MessagePort from main
4. Preload bridges contextBridge calls to MessagePort internally

Runtime:
App JS → window.cowork.callTool()
  → preload → MessagePort.postMessage()
  → [direct channel to utility process]
  → Mastra agent executes MCP tool
  → response returns via MessagePort → preload → Promise resolves
```

**Pros:** Combines the security of the preload bridge (apps see a clean `window.cowork.*` API) with the performance of MessagePort (main process not in the message path after setup). Per-app permissions enforced in the preload. No network exposure.

**Cons:** Most complex implementation. Two layers (preload + port) to maintain.

### Comparison

| Criterion | A: Preload | B: HTTP | C: MessagePort | D: Hybrid |
|---|---|---|---|---|
| IPC hops at runtime | 2 | 1 (HTTP) | 1 | 1 |
| Main process bottleneck | Yes | No | No | No |
| Network exposure | None | localhost port | None | None |
| DNS rebinding risk | None | Yes (CVE-2025-49596) | None | None |
| Standard MCP client | No | Yes | Custom transport | No |
| Per-app permissions | Natural | Token management | Natural | Natural |
| Implementation complexity | Low | Medium | Medium | High |
| Streaming support | Custom | Native SSE | Custom | Custom |

### Recommendation: discuss

**Option A** is simplest and most battle-tested. The main-process relay is a con but may not matter in practice — MCP tool calls are infrequent (not streaming thousands of messages per second).

**Option C** is the most elegant. The `MessagePortTransport` is trivial to build (the MCP SDK's `Transport` interface is 3 methods + 3 callbacks: `start/send/close` + `onmessage/onerror/onclose`). The SDK also ships `InMemoryTransport` for in-process use, proving the interface is designed for custom transports. Main process is out of the hot path. But setup choreography is more complex.

**Option B** is ruled out unless we use Unix domain sockets. The CVE-2025-49596 attack class is directly applicable. The "standard MCP client" advantage is undermined if we need custom auth tokens anyway.

**Option D** is the most robust but also the most complex. Worth it only if we're certain the main-process relay in Option A will be a bottleneck.

---

## Platform SDK (what apps see)

Regardless of which transport option we choose, apps see the same API surface. Injected via preload script:

```typescript
// What the app developer writes:
const tools = await window.cowork.listTools()
const result = await window.cowork.callTool('zendesk_list_tickets', { status: 'open' })
const response = await window.cowork.chat('Summarize these tickets')

window.cowork.onMessage((msg) => {
  // Streaming responses from agent
})

// App metadata
const config = await window.cowork.getAppConfig()
```

### SDK methods

| Method | What it does | Approval required |
|---|---|---|
| `listTools()` | Returns tools the app has permission to use | No |
| `callTool(name, args)` | Invoke an MCP tool | Depends on tool's `requireApproval` flag |
| `chat(message)` | Send a message to the platform agent (agent-as-tool) | No |
| `onMessage(callback)` | Subscribe to streaming agent responses | No |
| `getAppConfig()` | App's metadata, granted permissions, connected services | No |

### What the SDK does NOT expose

- Raw MCP protocol — apps don't manage connections
- OAuth tokens — platform handles auth to external services
- Agent internals — no access to routing decisions, memory, or context
- Other apps' data — each app is isolated
- Node.js / Electron APIs — sandbox enforces this

---

## Per-App Permission Model

### App manifest (our format)

Since AI Studio has no manifest, we define one. Created during installation (user grants permissions) or provided by the App Gallery listing.

```json
{
  "id": "uuid",
  "name": "Zendesk Dashboard",
  "description": "...",
  "version": "1.0.0",
  "permissions": {
    "tools": [
      "zendesk_list_tickets",
      "zendesk_get_ticket",
      "zendesk_update_ticket"
    ],
    "resources": [
      "context:activity_summary",
      "context:active_window"
    ],
    "agent": true
  }
}
```

### Enforcement

Every `callTool()` from an app is checked against its permission grant before execution:

```
App calls callTool('gmail_send_email', {...})
  → preload/bridge validates: is 'gmail_send_email' in this app's allowed tools?
  → If no: reject with PermissionDeniedError
  → If yes: forward to utility process for execution
```

Permission checks happen at the bridge layer (preload or main process), not in the utility process. The utility process trusts that incoming requests have already been authorized.

### Permission UI flow

1. User installs app from Gallery (or uploads zip)
2. App's manifest declares required tools
3. Platform shows permission prompt: "Zendesk Dashboard wants access to: list tickets, get ticket, update ticket"
4. User approves or denies per-tool
5. Granted permissions stored in cowork.db
6. App can only call approved tools

---

## Gemini API Proxy

AI Studio apps call the Gemini API. In our runtime, we need to handle this without exposing the API key to the app.

### Approach

Intercept Gemini API calls at the Electron level:

1. **During preprocessing:** Rewrite the service worker to route `generativelanguage.googleapis.com` calls to a custom protocol handler (e.g., `cowork-api://gemini/...`)
2. **Custom protocol handler in main process:** Receives the request, injects `GEMINI_API_KEY` header, forwards to Google's API, streams response back
3. **App sees:** Standard Gemini API responses, never the key

Alternative: run the app's Express proxy as a child process per app. Simpler but heavier (one Express server per app).

### API key sourcing

| User tier | Key source |
|---|---|
| Free | User's own Gemini API key (entered at app install) |
| Boost+ | Platform-managed key via OpenRouter (if Gemini is the selected cloud provider) |

---

## Process Model

How Apps fit into the existing [system architecture process model](./system-architecture.md#process-model):

```
┌─────────────────────────────────────────────────┐
│                  Main Process                     │
│  App lifecycle · IPC router · Protocol handlers   │
│  cowork-app:// handler · cowork-api:// proxy      │
└──────┬──────────────┬─────────────────────────────┘
       │              │
       │              │  MessagePort (Option C/D)
       │              │  or IPC relay (Option A)
       │              │
┌──────▼──────┐  ┌────▼──────────────────────┐
│  Renderer   │  │  Agents & RAG Utility     │
│  (main UI)  │  │  Mastra agents            │
│             │  │  MCP client connections    │
│  Contains:  │  │  Platform MCP server       │
│  - Chat     │  │  (serves tools to apps)   │
│  - Settings │  │  RAG pipeline             │
│  - etc.     │  │  Embedding queue          │
└─────────────┘  └───────────────────────────┘
       │
  ┌────▼──────────────┐
  │  App WebContentsView  │  (one per installed app)
  │  Sandboxed renderer   │
  │  preload: app-preload.js
  │  partition: persist:app-{id}
  │  loads: cowork-app://{id}/index.html
  └───────────────────────┘
```

Each app is a separate renderer process. Crash isolation: if an app crashes, the main UI and other apps are unaffected.

---

## Open Questions (for discussion)

| # | Question | Options | Impact |
|---|---|---|---|
| 1 | **Transport choice** | A (preload IPC), C (MessagePort), or D (hybrid) | Core communication architecture |
| 2 | **Preprocessing depth** | esbuild bundle vs Babel transpile vs full Vite build | Installation speed, compatibility, maintenance |
| 3 | **Gemini API proxy** | Custom protocol handler vs per-app Express child process | Resource usage, complexity |
| 4 | **Import maps vs bundling** | Serve deps via import map (CDN/local) vs bundle everything with esbuild | Offline support, bundle size, update mechanism |
| 5 | **App manifest authoring** | Auto-generate from code analysis vs require manual declaration vs Gallery provides | Developer experience, security |
| 6 | **WebContentsView layout** | Main process manages bounds vs renderer coordinates via IPC | UI integration complexity |

---

## See Also

| Topic | Document |
|---|---|
| Apps feature definition | [product-features.md — Apps](../product/product-features.md#1-apps) |
| System architecture & process model | [system-architecture.md](./system-architecture.md) |
| AIME Chat MCP patterns | [aime-adaptation-guide.md — Section 3](./aime-adaptation-guide.md#3-mcp-integration) |
| Existing Swift implementation | coworkai-mac-native-desktop (Features/Apps/) |

---

## Changelog

**v1 (Feb 16, 2026):** Initial draft. Covers AI Studio app format research, WebContentsView rendering, four transport options, platform SDK design, permission model, Gemini API proxy, and process model integration.
**v2 (Feb 17, 2026):** Expanded prior art section with detailed Swift codebase analysis: AppPreprocessor pipeline (Babel config, exact transformations), SDK injection pattern, bridge protocol (message format, resolve/reject flow), MastraClient HTTP protocol (JSON-RPC 2.0), installation flow, gaps (no permissions, no streaming, no approval gates), and Swift→Electron mapping table.
