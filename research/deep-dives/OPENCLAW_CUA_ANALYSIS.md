# OpenClaw CUA — Technical Deep Dive

> How OpenClaw achieves computer-use-like automation without vision models, and what this means for Cowork.ai.

**Date:** 2026-02-17
**Source repo:** `~/code/openclaw/` (version 2026.2.9)
**Author:** Rustan (via code analysis)

---

## Executive Summary

OpenClaw does **not** use Computer Use Agents (CUA). It uses a **tool-call architecture** where the LLM never sees pixels — it reads structured text (accessibility trees, DOM snapshots) and issues named actions against element refs. Browser automation runs through CDP + Playwright. Desktop automation is macOS-only via the Peekaboo CLI skill. There is no vision model in the loop.

This doc maps the exact technical implementation for our reference.

---

## Architecture Stack

```
┌─────────────────────────────────────────────────┐
│  LLM (Claude, GPT, Gemini, DeepSeek, etc.)      │
│  Sees: text snapshots, tool schemas              │
│  Emits: tool calls with action + ref             │
└──────────────────┬──────────────────────────────┘
                   │ tool call: browser(action, ref)
┌──────────────────▼──────────────────────────────┐
│  Agent Runtime (pi-embedded-runner)              │
│  Tool dispatch, policy filtering, error wrap     │
└──────────────────┬──────────────────────────────┘
                   │ HTTP POST to localhost
┌──────────────────▼──────────────────────────────┐
│  Browser Control Server (Express, 127.0.0.1)     │
│  Routes: /snapshot, /act, /screenshot, /tabs     │
└──────────┬───────────────────┬──────────────────┘
           │                   │
┌──────────▼─────────┐ ┌──────▼──────────────────┐
│  Playwright         │ │  Raw CDP (one-shot)     │
│  Persistent browser │ │  Screenshots, AX tree   │
│  connection via CDP │ │  Stateless WebSocket     │
└──────────┬─────────┘ └──────┬──────────────────┘
           │                   │
┌──────────▼───────────────────▼──────────────────┐
│  Chrome Process (--remote-debugging-port)         │
│  OR Chrome Extension Relay (CDP multiplexer)      │
└─────────────────────────────────────────────────┘
```

For desktop automation (macOS only):
```
LLM → exec tool → `peekaboo see --annotate` → macOS Accessibility API
LLM → exec tool → `peekaboo click --on B3`  → CGEvent / AXUIElement
```

---

## 1. The Browser Tool (Single Multiplexed Tool)

OpenClaw exposes **one tool called `browser`** to the LLM, not separate tools per action. The `action` parameter discriminates:

```typescript
// src/agents/tools/browser-tool.schema.ts
action: enum [
  "status", "start", "stop",       // lifecycle
  "profiles", "tabs",              // tab management
  "open", "focus", "close",        // tab ops
  "snapshot",                      // accessibility tree → text
  "screenshot",                    // pixels → image content block
  "navigate",                      // go to URL
  "console", "pdf",               // observability
  "upload", "dialog",             // file/dialog handling
  "act"                           // click, type, press, hover, drag, etc.
]
```

For `action="act"`, a nested `request` object carries the specific action:
```typescript
request: {
  kind: "click" | "type" | "press" | "hover" | "drag" |
        "select" | "fill" | "resize" | "wait" | "evaluate" | "close"
  ref?: string      // element ref from prior snapshot (e.g. "e23")
  text?: string     // for type
  key?: string      // for press
  fn?: string       // for evaluate (raw JavaScript)
  submit?: boolean  // press Enter after typing
  // ... modifiers, timing, geometry
}
```

**Why one tool?** Keeps total tool count low for LLM context efficiency. Some providers (Gemini, OpenAI) reject `anyOf`/`oneOf` in schemas, so the schema is a flat object.

---

## 2. The Snapshot → Act Loop (Core Automation Pattern)

This is the fundamental pattern. The LLM never gets raw HTML or pixel coordinates.

### Step 1: Snapshot

LLM calls `browser(action="snapshot")`. Three code paths exist:

#### Path A: `_snapshotForAI` (default, primary)
```
src/browser/pw-tools-core.snapshot.ts → snapshotAiViaPlaywright()
  → page._snapshotForAI({ timeout: 5000, track: "response" })
  → Returns YAML-like accessibility tree with Playwright's own [ref=eN] markers
  → buildRoleSnapshotFromAiSnapshot() parses refs into a map
  → Refs stored as mode="aria", resolved later via page.locator("aria-ref=e5")
```

**Output the LLM sees:**
```
- navigation "Main":
  - link [ref=e1] "Home"
  - link [ref=e2] "Products"
  - link [ref=e3] "Contact"
- main:
  - heading "Welcome" [level=1]
  - textbox [ref=e4] "Search..."
  - button [ref=e5] "Submit"
  - table:
    - row:
      - cell [ref=e6] "Item A"
      - cell [ref=e7] "$29.99"
```

#### Path B: `ariaSnapshot` (with options: selector, frame, compact)
```
src/browser/pw-tools-core.snapshot.ts → snapshotRoleViaPlaywright()
  → locator.ariaSnapshot()  (Playwright public API)
  → buildRoleSnapshotFromAriaSnapshot() assigns refs to interactive elements
  → Refs stored as mode="role", resolved via page.getByRole("button", { name })
```

#### Path C: Raw CDP Accessibility Tree (legacy, `format=aria`)
```
src/browser/pw-tools-core.snapshot.ts → snapshotAriaViaPlaywright()
  → CDP: Accessibility.getFullAXTree
  → formatAriaSnapshot() DFS walk → ax1, ax2, ... refs
  → These refs are informational only, NOT usable in actions
```

### Ref Assignment Rules

Only interactive and named content elements get refs:

```typescript
// src/browser/pw-role-snapshot.ts
INTERACTIVE_ROLES = new Set([
  "button", "link", "textbox", "checkbox", "radio", "combobox",
  "listbox", "menuitem", "searchbox", "slider", "spinbutton",
  "switch", "tab", "treeitem", ...
]);

CONTENT_ROLES = new Set(["heading", "cell", "listitem", "article", ...]);

// Ref assigned when:
// - role is INTERACTIVE, OR
// - role is CONTENT and element has a name
// Structural roles (generic, group, list, table) never get refs
```

Duplicate role+name combos get `nth` disambiguation (e.g., two buttons named "Delete" → `nth=0`, `nth=1`).

### Step 2: Act

LLM calls `browser(action="act", request={ kind: "click", ref: "e5" })`.

```
src/browser/pw-tools-core.interactions.ts → clickViaPlaywright()
  → getPageForTargetId() → cached Playwright Browser → Page
  → restoreRoleRefsForTarget() → load ref map if page reconnected
  → refLocator(page, "e5"):
      If mode="aria": page.locator("aria-ref=e5")
      If mode="role": refs["e5"] = { role:"button", name:"Submit" }
                      → page.getByRole("button", { name:"Submit", exact:true })
  → locator.click({ timeout: 8000 })
  → Playwright sends CDP: Input.dispatchMouseEvent to Chrome
```

### Ref Persistence

```typescript
// Module-level cache in pw-session.ts
roleRefsByTarget: Map<string, RoleRefsCacheEntry>  // max 50 entries
// Keyed by "cdpUrl::targetId"
// Survives Playwright reconnections within the same process
```

If the Playwright `Page` object is recycled (e.g., after reconnect), `restoreRoleRefsForTarget` copies cached refs onto the new page state. This makes multi-step agent flows resilient.

---

## 3. CDP Connection Layer

Two parallel CDP pathways, cleanly separated:

### Raw CDP (stateless, one-shot)
```typescript
// src/browser/cdp.helpers.ts → withCdpSocket()
// Opens WebSocket → sends commands → closes WebSocket
// Used for: screenshots, AX tree dumps, JS evaluation
// Protocol: { id: N, method, params } → { id: N, result }
```

### Playwright (stateful, persistent singleton)
```typescript
// src/browser/pw-session.ts
let cached: ConnectedBrowser | null = null;  // one per cdpUrl

connectBrowser(cdpUrl):
  → Double-checked locking (cached check + connecting promise)
  → connectWithRetry(): 3 attempts, escalating timeouts (5s, 7s, 9s)
  → chromium.connectOverCDP(wsEndpoint, { timeout, headers })
  → browser.on("disconnected", () => cached = null)
```

### Chrome Process Management
```typescript
// src/browser/chrome.ts → launchOpenClawChrome()
// Phase 1: Bootstrap Chrome profile (spawn → wait for prefs → kill)
// Phase 2: Decorate profile (custom name + color in Preferences JSON)
// Phase 3: Launch with flags:
//   --remote-debugging-port=<N>
//   --user-data-dir=~/.openclaw/browser/<name>/user-data
//   --no-first-run --no-default-browser-check --disable-sync
//   --headless=new (optional)
// Phase 4: Poll isChromeReachable() every 200ms up to 15s
```

---

## 4. Chrome Extension Relay

For controlling the user's **existing Chrome browser** (not a separate managed instance):

```
Chrome Extension (chrome.debugger API)
  ↕ ws://127.0.0.1:<port>/extension
Extension Relay Server (Node.js, two WSS endpoints)
  ↕ ws://127.0.0.1:<port>/cdp (requires x-openclaw-relay-token)
Playwright / CDP clients
```

The relay **emulates a standard CDP endpoint**. Playwright doesn't know it's talking to an extension.

### Commands handled locally (no extension round-trip):
- `Browser.getVersion` → fake `"Chrome/OpenClaw-Extension-Relay"`
- `Target.setAutoAttach` / `setDiscoverTargets` → synthetic events
- `Target.getTargets` / `getTargetInfo` → from local `connectedTargets` map

### Commands forwarded to extension:
Everything else (DOM, CSS, JS eval, screenshots, Input events).

### Security:
- Extension connection: must be loopback + `chrome-extension://` origin
- CDP client: must supply 32-byte random token in header
- HTTP endpoints (`/json/version`, `/json/list`): same token required

---

## 5. Labeled Screenshots

When `snapshot` is called with `labels=true`:

```typescript
// src/browser/pw-tools-core.interactions.ts → screenshotWithLabelsViaPlaywright()
for each ref in refs:
  → refLocator(page, ref).boundingBox() → { x, y, width, height }
  → filter off-viewport elements

page.evaluate(() => {
  // Inject overlay div: position:fixed, z-index:2147483647
  // Orange border boxes + ref label tags on each element
})

page.screenshot() → buffer

page.evaluate(() => {
  // Remove [data-openclaw-labels] overlay
})
```

This produces a screenshot where every interactive element is highlighted with its ref label — useful for vision-capable models. But the **primary** automation path is text snapshots, not screenshots.

---

## 6. Desktop Automation via Peekaboo (macOS Only)

Peekaboo is a standalone macOS CLI + MCP server built by Peter Steinberger (steipete). It is written in **Swift 6.2**, requires **macOS 15+**, and provides full GUI automation for any macOS application through the Accessibility API. OpenClaw integrates it as a **skill** (prompt injection + shell exec), not as embedded code.

Source: [github.com/steipete/Peekaboo](https://github.com/steipete/Peekaboo)

### 6.1 Peekaboo Architecture

```
┌─────────────────────────────────────────────────┐
│  PeekabooAgentRuntime                            │
│  MCP tools + agent orchestration                 │
│  Natural language → action plan                  │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│  PeekabooAutomation                              │
│  UIAutomationService (@MainActor orchestrator)   │
│  ├─ ElementDetectionService (AXorcist)           │
│  ├─ ClickService                                 │
│  ├─ TypeService                                  │
│  ├─ ScreenCaptureService                         │
│  ├─ ApplicationService                           │
│  ├─ WindowManagementService                      │
│  ├─ MenuService                                  │
│  └─ Snapshot cache (in-memory, auto-expiring)    │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│  AXorcist (Swift wrapper for AXUIElement)        │
│  Chainable, fuzzy-matched accessibility queries  │
│  @MainActor singleton, command-based dispatch    │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│  macOS APIs                                      │
│  ├─ Accessibility Framework (AXUIElement)        │
│  ├─ CoreGraphics (CGEvent, CGWindowList)         │
│  ├─ ScreenCaptureKit / CGWindowListCreateImage   │
│  └─ AppKit (NSWorkspace, window management)      │
└─────────────────────────────────────────────────┘
```

Three frameworks stacked:
1. **PeekabooAutomation** — all on-device operations: accessibility, screen capture, config, typed models
2. **PeekabooVisualizer** — standalone visual feedback layer (overlay animations), decoupled from core
3. **PeekabooAgentRuntime** — MCP tools + agent orchestration, depends on both above

### 6.2 AXorcist: The Accessibility Engine

AXorcist is the heart of Peekaboo's element detection. It wraps macOS `AXUIElement` with modern Swift abstractions:

**Element class** — primary wrapper around `AXUIElement`:
- Type-safe property access (role, title, value, enabled, focused, etc.)
- Automatic CF ↔ Swift type conversion
- Hierarchy navigation with caching
- Batch attribute fetching for performance

**Query system** — flexible criteria-based element search:
- Match strategies: **exact** (default), **contains** (case-insensitive), **regex**, **prefix/suffix**, **containsAny**
- Searchable attributes: role, title, value, identifier, DOM class, enabled, focused, hidden
- Chainable path navigation with depth hints
- Fuzzy matching for resilience against minor UI changes

**Command architecture** — all operations route through `runCommand()`:
- `Query` — element discovery
- `PerformAction` — UI interaction (click, press, etc.)
- `GetAttributes` — property retrieval
- Batch operations for efficiency
- All return standardized `AXResponse` objects

**Threading** — `@MainActor`-isolated singleton. macOS Accessibility APIs (`AXUIElement*` functions), AppKit, and CoreGraphics all require main-thread execution. Swift 6.2's concurrency model enforces this at the type level.

### 6.3 The Snapshot System

UI element detection via Accessibility API traversal is expensive (200–800ms). Peekaboo solves this with snapshot caching:

```
peekaboo see --app Safari --annotate --path /tmp/ui.png
  │
  ├─ 1. ElementDetectionService queries the accessibility tree
  │     → AXorcist traverses AXUIElement hierarchy
  │     → Collects: role, title, label, description, help text, identifier, shortcuts
  │     → Assigns shorthand element IDs by role: B1 (button 1), T2 (text field 2), etc.
  │     → 200–800ms for full traversal
  │
  ├─ 2. ScreenCaptureService captures screenshot
  │     → ScreenCaptureKit (modern) or CGWindowListCreateImage (legacy)
  │     → Optional 2x Retina scaling
  │     → 20–100ms
  │
  ├─ 3. SmartLabelPlacer annotates the screenshot
  │     → Generates external label candidates (above/below/sides/corners)
  │     → Filters overlaps, scores by text density
  │     → Positions labels to avoid obscuring UI elements
  │
  ├─ 4. Snapshot persisted
  │     → ~/.peekaboo/snapshots/<id>/snapshot.json
  │     → Contains: snapshot_id, ui_elements[], element counts, metadata
  │     → Cached in memory for subsequent commands (240x speedup)
  │
  └─ 5. Output
       → Annotated PNG written to --path
       → JSON output (--json): snapshot_id, ui_map path, ui_elements[]
       → Each element: { title, label, description, role_description,
                         help, identifier, keyboard_shortcuts }
```

**Element IDs** use a role-prefix shorthand: `B` = button, `T` = text field, `L` = link, etc. IDs are stable within a snapshot session. Subsequent `click`/`type` commands reference these IDs (e.g., `--on B3`) without re-traversing the accessibility tree.

**Auto web focus fallback**: If zero text fields are detected, Peekaboo locates the dominant `AXWebArea`, performs a synthetic focus action, and retries traversal. This handles cases where browser keyboard focus prevents embedded login forms from exposing input fields.

### 6.4 Interaction Commands

All interactions use the cached snapshot. Performance: **10–50ms per action** (vs 200–800ms for initial detection).

| Command | macOS API | What It Does |
|---------|-----------|-------------|
| `click --on B3` | CGEvent (`kCGEventLeftMouseDown/Up`) | Resolves element from snapshot → gets bounding box → synthesizes mouse event at center |
| `type "text" --on T2` | CGEvent (`kCGEventKeyDown/Up`) + AX | Focuses element via Accessibility API → synthesizes key events, supports `--clear`, `--delay`, `--wpm` |
| `press tab --count 2` | CGEvent | Synthesizes special key presses with repeat count |
| `hotkey --keys "cmd,shift,t"` | CGEvent | Modifier combo synthesis |
| `drag --from B1 --to T2` | CGEvent | Mouse down → move → mouse up between elements |
| `scroll --direction down --amount 6` | CGEvent (`kCGScrollEventWheel`) | Directional scroll with optional `--smooth` |
| `swipe --from-coords --to-coords` | CGEvent | Gesture-style drag between coordinates |

### 6.5 System Control Commands

Beyond element interaction, Peekaboo wraps full macOS system control:

**App management** (`peekaboo app`):
- `launch`, `quit`, `relaunch`, `hide`, `unhide`, `switch` — via `NSWorkspace` / `NSRunningApplication`

**Window management** (`peekaboo window`):
- `close`, `minimize`, `maximize`, `move`, `resize`, `focus`, `list` — via AXUIElement window attributes

**Menu automation** (`peekaboo menu`):
- `click --app Safari --item "New Window"` — traverses AXMenuBar → AXMenuItem hierarchy
- `click --path "Format > Font > Show Fonts"` — deep menu path traversal
- `click-extra --title "WiFi"` — menu bar extras (status bar items)

**Dock** (`peekaboo dock`):
- `launch`, `right-click`, `hide`, `show`, `list` — via AXDockItem elements

**Dialog** (`peekaboo dialog`):
- `click`, `input`, `file`, `dismiss`, `list` — handles system alerts, file choosers, permission dialogs

**Clipboard** (`peekaboo clipboard`):
- `read`, `write` — text, images, files via `NSPasteboard`

**Spaces** (`peekaboo space`):
- `list`, `switch`, `move-window` — macOS Mission Control / Spaces integration

### 6.6 PeekabooBridge: Privilege Separation

Peekaboo uses a **bridge model** for privilege separation:

```
peekaboo CLI (unprivileged)
    │ UNIX socket: bridge.sock
    │ JSON protocol
    ▼
Bridge Host (holds TCC permissions)
    ├─ Peekaboo.app (full UX, preferred)
    ├─ Claude.app (if installed)
    └─ OpenClaw.app (thin broker)
```

**Why?** macOS TCC (Transparency, Consent, Control) requires Screen Recording and Accessibility permissions to be granted to a specific signed application bundle. The CLI runs unprivileged; the bridge host holds the permissions.

**Discovery order**: Client tries Peekaboo.app → Claude.app → OpenClaw.app → local execution.

**OpenClaw as bridge host**: When "Enable Peekaboo Bridge" is toggled in the macOS app settings, OpenClaw starts a UNIX socket server. The CLI connects to it for all automation requests, reusing the app's TCC grants.

**Security**:
- Bridge validates caller **code signatures** against a TeamID allowlist
- Requests time out after ~10 seconds
- Local-only communication (UNIX socket, no network)
- `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` debug escape hatch

### 6.7 AI Integration (Peekaboo's Own Agent)

Peekaboo has its own built-in AI agent (separate from OpenClaw's agent):

**Tachikoma** — AI provider surface:
- Injectable `AIModelProvider` pattern (not singleton)
- `AIConfiguration.fromEnvironment()` auto-discovers credentials
- Supports: OpenAI (GPT-5.1), Anthropic (Claude 4.x), xAI (Grok 4), Google (Gemini 2.5), Ollama (local)

**Agent flow**:
```
Natural language: "Open Notes and create a TODO list"
  → PeekabooAgentService receives input
  → AI model interprets intent, generates action plan
  → UIAutomationService.detectElements() queries accessibility tree
  → Specialized services execute (click, type, navigate)
  → VisualizationClient fires async overlay animations
  → State cached in snapshot for next step
```

**MCP server mode**: Peekaboo also runs as an MCP server for Claude Desktop and Cursor, exposing the same tools via Model Context Protocol.

### 6.8 How OpenClaw Integrates Peekaboo

OpenClaw does **not** use Peekaboo's built-in agent or MCP server. It integrates purely through the **exec tool** (shell commands):

```yaml
# skills/peekaboo/SKILL.md frontmatter
openclaw:
  os: ["darwin"]                    # macOS only, hard-gated
  requires: { bins: ["peekaboo"] }  # must be on PATH
  install:
    - kind: brew
      formula: steipete/tap/peekaboo
```

**Integration model**:
1. Skills loader checks `os: ["darwin"]` → skip on non-Mac
2. Checks `peekaboo` binary exists on PATH
3. If missing, offers `brew install steipete/tap/peekaboo`
4. SKILL.md content injected into agent's system prompt (teaches usage patterns)
5. Agent reads SKILL.md via `read` tool, learns available commands
6. Agent invokes peekaboo via `exec` tool (shell command execution)
7. Each peekaboo invocation is stateless from OpenClaw's perspective (but peekaboo caches snapshots internally)

**The agent's workflow**:
```
exec: "peekaboo see --app Safari --annotate --path /tmp/ui.png"
  → Agent sees annotated screenshot with role-prefixed IDs (B1=button, T2=textbox, etc.)
  → Agent reasons about which element to interact with

exec: "peekaboo click --on B3 --app Safari"
  → Peekaboo resolves B3 (button 3) from cached snapshot
  → Synthesizes click via CGEvent
  → Returns confirmation

exec: "peekaboo type 'hello@example.com' --on T2 --app Safari --return"
  → Types into T2 (text field 2), presses Enter
```

### 6.9 Performance Characteristics

| Operation | Latency | Notes |
|-----------|---------|-------|
| Element detection (`see`) | 200–800ms | Full a11y tree traversal, cached after first run |
| Cached element lookup | ~3ms | 240x faster than re-detection |
| Click | 10–50ms | CGEvent synthesis |
| Type | Varies by length | Default WPM or character delay |
| Screen capture | 20–100ms | ScreenCaptureKit or CGWindowList |
| Application discovery | 20–200ms | NSWorkspace query |
| Window management | 10–200ms | AXUIElement attribute set |

### 6.10 Critical Limitation: macOS Only

Peekaboo is fundamentally macOS-only. Every layer depends on Apple-specific APIs:

| Dependency | macOS API | Windows/Linux Equivalent |
|------------|-----------|-------------------------|
| Element detection | `AXUIElement` (Accessibility Framework) | Win: UI Automation / Linux: AT-SPI (different APIs, different behavior) |
| Input synthesis | `CGEvent` (CoreGraphics) | Win: `SendInput` / Linux: `xdotool`/`libxdo` |
| Screen capture | ScreenCaptureKit / `CGWindowListCreateImage` | Win: Desktop Duplication API / Linux: X11/PipeWire |
| App management | `NSWorkspace` / `NSRunningApplication` | No direct equivalent |
| Window management | AXUIElement window attributes | Win: `SetWindowPos` / Linux: `wmctrl` |
| Permissions | TCC (Screen Recording + Accessibility) | Different permission models |
| Bridge IPC | UNIX socket + code signature validation | Different IPC + no code signing |

**There is no Peekaboo for Windows or Linux.** On those platforms, OpenClaw has zero desktop automation — only browser automation via Playwright/CDP. This is the key gap that true vision-based CUA (UI-TARS, Cua framework) fills: they work on any OS by seeing pixels.

---

## 7. Agent Runtime: How Tool Calls Flow

### Tool Assembly Pipeline (8 stages)

```typescript
// src/agents/pi-tools.ts → createOpenClawCodingTools()

Stage 1: Coding tools (read, write, edit, grep, find, ls)
Stage 2: Runtime tools (exec, process, apply_patch)
Stage 3: Channel plugin tools (whatsapp_login, etc.)
Stage 4: OpenClaw platform tools:
         browser, canvas, nodes, cron, message, tts, gateway,
         agents_list, sessions_list, sessions_history,
         sessions_send, sessions_spawn, session_status,
         web_search, web_fetch, image
         + all registered plugin tools
Stage 5: Authorization (owner-only gating)
Stage 6: Policy filtering (8 layers, most restrictive wins):
         profile → providerProfile → global → globalProvider →
         agent → agentProvider → group → sandbox → subagent
Stage 7: Schema normalization (Gemini/OpenAI compat)
Stage 8: Lifecycle wrappers (before_tool_call hooks, abort signal)
```

### Execution Flow (Full Round-Trip)

```
User: "Click the Submit button on that page"

1. runEmbeddedPiAgent(prompt, sessionId, provider, model)
2. → Lane serialization (session + global)
3. → Auth profile resolution
4. → runEmbeddedAttempt:
     a. createOpenClawCodingTools() → filtered tool list
     b. buildEmbeddedSystemPrompt() → includes "browser: Control web browser"
     c. SessionManager.open(sessionFile) → load history
     d. All tools passed as customTools (never builtInTools)
     e. activeSession.prompt("Click the Submit button")

5. LLM Turn 1: decides to snapshot first
   → tool call: browser(action="snapshot")
   → HTTP GET localhost:<port>/snapshot?format=ai
   → Playwright: page._snapshotForAI()
   → Returns: "- button [ref=e23] Submit\n..."

6. LLM Turn 2: sees e23 = Submit button
   → tool call: browser(action="act", request={kind:"click", ref:"e23"})
   → HTTP POST localhost:<port>/act body:{kind:"click", ref:"e23"}
   → Playwright: refLocator(page, "e23") → page.getByRole("button", {name:"Submit"})
   → locator.click() → CDP: Input.dispatchMouseEvent
   → Returns: { ok: true }

7. LLM Turn 3: "I clicked the Submit button."
   → No more tool calls → session ends

8. Result delivered to user's channel
```

---

## 8. Three Routing Backends

The browser tool routes to one of three backends based on `target`:

| Target | When | How |
|--------|------|-----|
| **Host** (default) | Local machine | HTTP to `localhost:<controlPort>` → Playwright |
| **Sandbox** | Docker env | HTTP to `sandboxBridgeUrl` → Playwright inside container |
| **Node** | Remote paired device | Gateway tool → `node.invoke` → proxy to remote browser server |

Remote node proxying handles file path remapping (screenshots saved on remote → base64 transferred → saved locally).

---

## 9. Comparison: OpenClaw CUA vs True CUA

| Aspect | OpenClaw | True CUA (UI-TARS, Cua) |
|--------|----------|------------------------|
| **Input to LLM** | Text (a11y tree, ~2KB) | Screenshot image (~500KB) |
| **Element targeting** | Ref IDs from a11y tree | Pixel coordinates (x, y) |
| **Vision model required?** | No | Yes (VLM like UI-TARS-7B) |
| **Works on apps without a11y?** | No — degrades to raw DOM | Yes — sees pixels |
| **Browser automation** | CDP + Playwright (100% accuracy) | Screenshot → VLM → click (85-95%) |
| **Desktop automation** | macOS only (Peekaboo/Accessibility API) | Any OS (just sees screen) |
| **Windows support** | Browser only (no desktop) | Full (screenshot-based) |
| **Linux support** | Browser only (no desktop) | Full (screenshot-based) |
| **Speed** | Fast — direct DOM manipulation | Slow — screenshot + VLM inference per step |
| **Token cost** | Low — small text snapshots | High — image tokens per step |
| **Robustness** | Breaks if a11y tree is bad | Breaks if UI is visually ambiguous |
| **Model agnostic?** | Yes — any text LLM works | No — needs vision-capable model |

---

## 10. Key Takeaways for Cowork.ai

1. **OpenClaw validates tool-call-first architecture.** 180k stars, no vision model in the automation loop. For SaaS tools with APIs/MCP, this is the right approach — and it's what we're already building with Mastra + MCP.

2. **The snapshot → ref → act pattern is the standard.** Both OpenClaw and the Playwright MCP server (which we already use in Claude Code) follow this exact pattern. It works well for browser automation.

3. **Desktop automation is the gap.** OpenClaw's desktop story is macOS-only Peekaboo, invoked via shell exec. There is no cross-platform desktop automation. This is where true CUA (vision + mouse/keyboard) has an advantage.

4. **Their Chrome Extension Relay is clever.** Lets Playwright control the user's actual browser session (logged-in state, cookies, extensions) by multiplexing CDP through the extension's `chrome.debugger` API. Worth noting for when we need to automate web apps where the user is already logged in.

5. **Single-tool design is intentional.** One `browser` tool with an `action` discriminator, not 15 separate tools. Keeps LLM context efficient and avoids schema compatibility issues across providers.

6. **No Windows desktop automation at all.** If Cowork.ai ever targets Windows, we'd need either true CUA or a Windows-specific accessibility bridge. OpenClaw has nothing here.

---

## Changelog

| Date | Change |
|------|--------|
| 2026-02-17 | Initial analysis from openclaw v2026.2.9 source code |
| 2026-02-17 | Added full Peekaboo deep dive (architecture, AXorcist, bridge, snapshot system) |
