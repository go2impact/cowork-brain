# CoworkAI Platform Technical Reference

> **Synthesized from multiple repositories.** This document consolidates technical details across the following local sources:
>
> | Repository | Local Path |
> | --- | --- |
> | `coworkai-desktop` | `/Users/rustancorpuz/code/coworkai-desktop` |
> | `coworkai-agent` | `/Users/rustancorpuz/code/coworkai-agent` |
> | `coworkai-activity-capture` | `/Users/rustancorpuz/code/coworkai-activity-capture` |
> | `coworkai-keystroke-capture` | `/Users/rustancorpuz/code/coworkai-keystroke-capture` |

---

## 1. System Overview

CoworkAI is a desktop productivity tracking platform built as a modular system under the `@engineering-go2` namespace. It captures detailed telemetry including active windows, keystrokes, clipboard events, timer logs, and media. Structured event data is persisted in SQLite, while media artifacts are written to filesystem directories before synchronization loops process them.

The system consists of four primary repositories:

| Repository | Type | Version | Purpose |
| --- | --- | --- | --- |
| `coworkai-desktop` | Electron App | `1.0.13` | UI shell, OS integration, and orchestration host. |
| `coworkai-agent` | npm Library | `0.1.6` | Core tracking engine, SQLite persistence, and sync management. |
| `coworkai-activity-capture` | Native Addon | `0.0.5` | Native active-window and browser URL detection. |
| `coworkai-keystroke-capture` | Native Addon | `0.0.8` | Native system-wide keyboard and mouse event capture. |

---

## 2. Architecture & Boundaries

### High-Level System Map

The architecture follows a strict hierarchy where the desktop application consumes the agent library, which in turn orchestrates native modules.

```text
+-------------------------------+
| coworkai-desktop (Electron)   |
| - Main process orchestration  |
| - Renderer UI + Setup UI      |
| - IPC + Tray + Auth flow      |
+---------------+---------------+
                |
                | imports @engineering-go2/coworkai-agent
                v
+-------------------------------+
| coworkai-agent (TS library)   |
| - Agent lifecycle             |
| - Activity/keyboard/timer     |
| - Screen/audio/video modules  |
| - SQLite buffers + sync       |
+---------------+---------------+
                |
                | imports native packages via `bindings`
     +----------+------------------------------+
     |                                         |
     v                                         v
+------------------------------+    +------------------------------+
| activity/clipboard/audio/... |    | keystroke capture            |
| native addons (.node)        |    | native addon (.node)         |
| active window, media, etc.   |    | global key/mouse hooks       |
+------------------------------+    +------------------------------+

```

### Dependency Boundaries

* **coworkai-desktop**: Owns the Electron lifecycle, window management, system tray, auto-updater, and user authentication UX. It initializes the agent with host-specific configurations (paths, API clients).
* **coworkai-agent**: A UI-agnostic Node.js library. It manages the capture loop, normalizes raw signals, handles local persistence (SQLite WAL mode), and runs the sync engine.
* **Native Addons**: Expose OS-level signals (Win32 APIs, Cocoa/Carbon APIs) to Node.js via N-API.

---

## 3. Runtime Lifecycle & Data Flow

### App Boot & Identity

1. **Initialization**: `src/main.ts` initializes Sentry, environment variables, IPC handlers, and the system tray.
2. **Agent Boot Services**: On `app.ready`, the desktop calls `coworkaiAgent.agent.init()` and `coworkaiAgent.sync.start()` (starting sync schedulers, not active capture).
3. **Authentication**: The Renderer authenticates via `VITE_AUTH_URL` and sends an `AUTH/SET_USER` IPC message containing the user profile.
4. **Identity Isolation**: The Main process calls `coworkaiAgent.agent.setUserId(user.id)`, ensuring all database records and media files are isolated by user ID.
5. **Capture Activation**: Actual activity/keyboard/audio/video/screen capture begins when the app sends `TRACKER/START`, which calls `coworkaiAgent.agent.start()`.

### Data Capture Loops

* **Activity Flow**: The agent subscribes to active-window changes. On change, the `ActivityManager` ends the previous `ActivityBuffer`, flushes it to SQLite, and starts a new one. For supported browsers, native activity capture enriches title/URL fields.
* **Keystroke Flow**: Raw events from native hooks are normalized and chunked with debounce-driven flushing (and special-key flushes). If buffered keystrokes for an activity exceed 1000 characters, the activity buffer is flushed. Copy/cut/paste hotkeys trigger clipboard capture.
* **Timer Flow**: The timer tracks session duration, automatically splitting sessions at UTC midnight.

### Sync Pipeline

* **Data Contract (Local → Cloud)**: The `SyncQueue` batches records and invokes a configured `getApiClient` with a standardized payload:
  ```typescript
  {
    eventType: "activities_sync" | "keystrokes_sync" | "clipboards_sync",
    eventData: {
      records: Array<{ id: number; sync: number; [key: string]: any }>
    }
  }
  ```
* **Database Sync**: The `SyncManager` schedules per-table queues (`activities`, `keystrokes`, `clipboards`). It batches up to 500 records and sends them via the configured API client.
* **Media Sync**: Files (screenshots, audio, video) are handled by separate file-based sync loops that scan output directories and trigger callbacks.
* **Retry Logic**: Designed with exponential backoff.
  * **Base Delay**: 5,000ms (5s).
  * **Max Delay**: 300,000ms (5m).
  * **Algorithm**: `baseDelay * Math.pow(2, retryCount)` capped at `maxDelay`.

---

## 4. Component Deep Dives

### coworkai-desktop

* **Tech Stack**: Electron 37.1.0, React 19, Vite, Tailwind CSS.
* **IPC Message Schema**: IPC communication uses a unified `IPCPayload` wrapper:
  ```typescript
  type IPCPayload<T = unknown> = {
    action: IPCActions;
    data: T;
  }
  ```
* **Primary Channels**:
  * `tracker`: Controls `START`/`STOP` of the capture engine.
  * `auth`: Action `SET_USER` passes the authenticated user object; `user.id` is used to isolate database/media.
  * `timer`: Broadcasts session `START`/`STOP` events to the UI.
* **IPC Channels**: Defined in `channel.ts` (e.g., `AUTH`, `TRACKER`, `TIMER`, `SETUP`).
* **Configuration**: Supports environment profiles (`local`, `alpha`, `beta`, `production`) loaded at build time via `FORGE_PROFILE`.
* **Native Resolution**:
  * **Dev Mode**: Resolves native binaries from `node_modules`.
  * **Packaged Mode**: Resolves from `asar-unpacked` because Electron cannot load `.node` files from within ASAR archives.



### coworkai-agent

* **Buffering**: Uses a bounded in-memory queue of up to 5 `ActivityBuffer` instances; older buffers are auto-flushed when the limit is exceeded.
* **Config Defaults**: Activity, Keyboard, and Clipboard are enabled by default; Screen, Audio, Video, and Timer are disabled by default.
* **Database (SQLite)**: Uses `better-sqlite3` with specific performance pragmas:
  * **Journal Mode**: `WAL` (Write-Ahead Logging) for high-concurrency writes.
  * **Persistence**: Storage path is host-configured (desktop app uses `app.getPath('userData')/agent/database` in production and `<repo>/agent/database` in dev).
* **Database**: Uses `better-sqlite3` in WAL (Write-Ahead Logging) mode for performance.

#### Timer Module

**Source:** `coworkai-agent/src/agent/timer/index.ts`

The `Timer` class tracks work session duration with support for daily boundaries and database persistence.

* **Session Lifecycle**: `Timer.start()` records `Date.now()` as the session start, immediately flushes a zero-duration log to the database (so data isn't lost on crash), then starts a 1-second `setInterval` that updates elapsed time, checks for midnight crossings, and writes the updated end time to the `TimeLogBuffer`.
* **Midnight Splitting**: `checkAndHandleMidnight()` runs every tick and compares the current UTC date against the session start date (via `moment.utc`). On a day boundary, `splitSessionAtMidnight()` closes the current log at exactly UTC midnight, queues it, and opens a new session starting from midnight — resetting `todayCurrentTime` to the seconds elapsed in the new day.
* **Daily Aggregation**: `init()` loads all of today's existing log records from the database, sums their durations, and seeds `todayCurrentTime` so the UI counter is accurate even after an app restart.
* **Historical Retrieval**: `refreshTimeLogs()` fetches 4 weeks of time log records (from the start of the week 4 weeks ago to today, in UTC) and delivers them to the host via the `onSetup` callback, enabling the desktop UI to render historical charts.
* **Flush-on-Stop**: `stop()` clears the interval timer, writes the final `end` timestamp, and calls `flushTimeLogQueue(true)` to force-write all buffered logs to SQLite immediately.

### coworkai-activity-capture (Native)

**Purpose:** OS-level active window detection and browser URL extraction.

**macOS Implementation (Objective-C++)**
The macOS module uses a **layered extraction strategy** to guarantee data availability without blocking the main thread.

* **App Detection Hierarchy**:
1. **Primary**: `[[NSWorkspace sharedWorkspace] frontmostApplication]`.
2. **Fallback 1**: Iterates `runningApplications` checking `[app isActive]`.
3. **Fallback 2**: `CGWindowListCopyWindowInfo` to find the window at "layer 0" (main window) and resolve its PID.


* **Data Extraction**:
  * **AppleScript**: The module first attempts to execute a focused AppleScript (via `NSAppleScript`) to fetch the `title` of the front window and `URL` of the active tab. This is preferred as it requires **no Accessibility permissions**.
  * **Accessibility Fallback**: If AppleScript fails, it creates an `AXUIElementRef` for the target PID and traverses the element tree looking for `kAXTitleAttribute` and `kAXURLAttribute`.


* **Deep Browser Traversal**: For Chromium browsers, the `SearchForWebContent()` function performs a recursive depth-first traversal of the Accessibility tree, hunting for roles like `AXWebArea` to find the URL when standard attributes are missing.

**Windows Implementation (C++)**

* **Window Detection**: Uses `GetForegroundWindow()` for the handle and `GetWindowThreadProcessId` to resolve the PID.
* **Browser URL Extraction**: Uses **COM-based UI Automation**:
* Initializes COM via `CoInitializeEx` and creates an `IUIAutomation` instance.
* **The "28 Names" Heuristic**: To handle different browser versions and system locales, the module checks the `UIA_NamePropertyId` against a hardcoded list of **28 known address bar labels** (defined per browser in `src/windows/chrome/chrome.cpp:85-105` and equivalent files for Edge, Firefox, Brave, and Opera), including "Omnibox", "Address and search bar", and generic fallbacks like "Edit".



### coworkai-keystroke-capture (Native)

**Purpose:** System-wide keyboard/mouse interception and raw input processing.

**N-API Boundary Mechanics**:
Data is passed from the native thread to Node.js via `Napi::ThreadSafeFunction` (e.g., `g_onKeyTsf`). Native C++ types are serialized into JavaScript objects:
* `key`: `Napi::String` (UTF8 character or standardized name like "BACKSPACE").
* `keyCode`: `Napi::Number` (Native virtual key code).
* `state`: `Napi::String` ("UP" or "DOWN").
* `modifiers`: `shift`, `control`, `option`, `command`, `capsLock` (all `Napi::Boolean`).
* `meta`: `isRepeat`, `isKeyUp`, `isModifier`, `isNumpad`, `isFunctionKey` (all `Napi::Boolean`).

**macOS Implementation (Core Graphics)**

* **Threading**: Runs on a **dedicated background thread** that creates its own `CFRunLoop`. It blocks on `CFRunLoopRun()` waiting for OS events, ensuring the Node.js main thread is never blocked.
* **Hook Mechanism**: Uses `CGEventTapCreate` with `kCGSessionEventTap` and `kCGHeadInsertEventTap` to place the hook at the start of the OS event chain. It listens for `kCGEventKeyDown`, `kCGEventKeyUp`, and `kCGEventFlagsChanged`.
* **Async Delivery**: When an event fires, the C++ callback invokes `g_onKeyTsf.NonBlockingCall()` (a `Napi::ThreadSafeFunction`), which safely queues the JavaScript callback execution onto the main Node.js event loop.

**Windows Implementation (Low-Level Hooks)**

* **Threading**: Spawns a dedicated thread that runs a standard **Windows Message Pump** loop (`GetMessageW` -> `DispatchMessageW`) to keep the hooks alive.
* **Hook Mechanism**: Installs system-wide hooks via `SetWindowsHookExW` using `WH_KEYBOARD_LL` and `WH_MOUSE_LL`.
* **Atomic Repeat Detection**: To filter out "key hold" spam, it maintains a global `std::atomic<bool> g_keyDownState[256]` array. It checks if a key is already "down" before emitting a new press event.
* **Character Mapping**: Uses `ToUnicodeEx` combined with `GetKeyboardLayout` to translate virtual key codes (VK) into actual UTF-8 characters.

**Data Structure**
Every event contains: `key` (identity), `keyCode`, `state` ("DOWN"/"UP"), `isRepeat` (boolean), modifiers (`shift`, `control`, etc.), and mouse coordinates (`x`, `y`) if applicable.

---

## 5. Data Schema & Storage

### SQLite Schema

The agent manages a local SQLite database with the following key tables:

| Table | Key Columns | Purpose |
| --- | --- | --- |
| `activities` | `user_id`, `start`, `end`, `title`, `exec`, `url`, `os`, `sync` | Active window sessions. |
| `keystrokes` | `user_id`, `start`, `end`, `activity_id`, `data`, `sync` | Keystroke chunks linked to activity. |
| `clipboards` | `user_id`, `event` (COPY/PASTE/CUT), `timestamp`, `activity_id`, `data`, `sync` | Clipboard content events. |
| `timelogs` | `user_id`, `start`, `end`, `sync` | Work session durations. **Not included in SyncManager's sync schedule** (see `coworkai-agent/src/database/index.ts:72-86`). |

All tracked rows use a `sync` flag (`0` = pending, `1` = synced).

### Filesystem Layout

Data is stored under `app.getPath('userData')/agent/` in production or `<repo>/agent/` in dev.

* **Database**: `agent/database/<db-name>`.
* **Permissions**: `agent/permissions.json`.
* **Media**: `agent/screenshots/`, `agent/audios/`, `agent/videos/` (organized by user ID).

---

## 6. Current Implementation Notes (Gaps & Limitations)

The following behaviors reflect the current codebase state (v1.0.13):

1. **Sync Stub Behavior**: The desktop sync config (`coworkai-desktop/src/app/main/coworkai-agent/configs/sync.ts`) currently returns a stub success result (`Promise.resolve({ status: 'success' })`); records are marked `sync=1` locally without real network transmission unless `getApiClient` is replaced. No API contract or endpoint schema is documented.
2. **Disabled Features**: Audio, Video, Screen Capture, and Timer are disabled in the desktop config by default (unless `E2E=true`).
3. **Timelog Sync**: The `timelogs` table is not included in the `SyncManager`'s scheduled sync cycles.
4. **Media Sync**: Audio/Video sync callbacks currently only log to the console; there is no remote upload implementation.
5. **Backoff Logic**: The `getRetryCount()` method in `SyncQueue` (`coworkai-agent/src/database/sync/sync.ts:279-283`) always returns `0`, so the backoff formula `baseDelay * Math.pow(2, retryCount)` evaluates to `5000 * Math.pow(2, 0) = 5000ms` on every retry — effectively a fixed 5-second delay instead of exponential backoff.

### Failure Modes & Resilience

* **Native Addon Crashes**: If a native addon (activity capture or keystroke capture) throws or crashes, the error propagates to the agent layer. There is no automatic restart or watchdog for individual native modules; the Electron main process would need to re-initialize the agent.
* **Database Errors**: SQLite operations use `better-sqlite3` synchronous API. A disk-full or corrupt database would throw synchronously, potentially halting the capture loop. No automatic recovery or database migration/repair logic is implemented.
* **Sync Failures**: The `SyncQueue` catches errors during sync and schedules a retry with the (effectively fixed) backoff delay. However, there is no dead-letter queue or maximum retry limit — failed records remain in the pending state (`sync=0`) indefinitely and will be retried on every sync cycle.
* **Midnight Edge Case**: The midnight-splitting logic in `Timer` relies on a 1-second `setInterval`. If the system is asleep across midnight (e.g., laptop lid closed), the split will only occur when the timer tick resumes, which could result in a single log entry spanning two calendar days until the next tick fires.

---

## 7. Build & Distribution

### Native Modules

Native addons are built using `node-gyp` and `prebuild`. Current scripts build Windows `x64` and one macOS architecture per run (`BUILD_ARCH` or host arch), then upload prebuild artifacts. During Electron packaging, the following modules are rebuilt against the Electron ABI (see `forge.config.ts` `rebuildConfig`):

* `better-sqlite3`
* `@engineering-go2/coworkai-activity-capture`
* `@engineering-go2/coworkai-video-capture`
* `@engineering-go2/coworkai-audio-capture`
* `@engineering-go2/clipboard-capture`
* `@engineering-go2/coworkai-keystroke-capture`

The `AutoUnpackNativesPlugin` extracts `.node` files from the ASAR archive so Electron can load them at runtime.

### Environment Profiles

Build-time environment is controlled by the `FORGE_PROFILE` variable (see `utils/forge-args.ts`):

| Profile | Env File | Usage |
| --- | --- | --- |
| `local` | `.env.local` | Default for local development |
| `alpha` | `.env.alpha` | Internal alpha testing |
| `beta` | `.env.beta` | Beta release channel |
| `production` | `.env.production` | Production release |

The profile determines the env file loaded at build time and the S3 key prefix used for publishing and update URLs.

### Code Signing

* **macOS**: Uses `osxSign` (with hardened runtime and `--deep` argument) and `osxNotarize` (Apple ID + Team ID). Requires `APPLE_ID`, `APPLE_PASSWORD`, and `APPLE_TEAM_ID` environment variables in CI.
* **Windows**: Uses Azure Trusted Signing via a custom `windowsSign` function. Requires `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`, `SIGNTOOL_PATH`, `AZURE_CODE_SIGNING_DLIB`, and `AZURE_METADATA_JSON` in CI.
* **Local Builds**: Code signing is skipped (the signing config is only applied when `process.env.CI` is set).

### Publishing & Updates

* **Publisher**: `@electron-forge/publisher-s3` publishes to an AWS S3 bucket.
* **Key Structure**: `{FORGE_PROFILE}/{platform}/{arch}/{filename}` (see `keyResolver` in `forge.config.ts`).
* **Update URLs**: Platform-specific update manifests are generated at the same S3 prefix. macOS uses `macUpdateManifestBaseUrl`; Windows uses Squirrel's `remoteReleases`.
* **Auto-Updates**: The desktop app uses `update-electron-app` to check for and apply updates from the S3-hosted manifests.

---

## 8. Security & Privacy Considerations

* **User ID Isolation**: All database records and media files are scoped by `user_id`, set via `agent.setUserId()` after authentication. No data from one user can leak into another user's storage path.
* **Accessibility Permissions**: On macOS, browser URL extraction may require Accessibility access (System Preferences → Privacy & Security → Accessibility). The AppleScript path is preferred specifically because it avoids this requirement, with Accessibility used only as a fallback.
* **No Encryption-at-Rest**: The local SQLite database and media files (screenshots, audio, video) are stored unencrypted on disk. Anyone with filesystem access to the user data directory can read captured data, including keystroke logs and clipboard contents.
* **Data Sensitivity**: The system captures keystrokes (including potentially sensitive input), clipboard contents (copy/cut/paste), active window titles, and URLs. Deployments should consider data handling policies appropriate for this level of monitoring.
* **Network Transport**: The current sync implementation is a stub that does not transmit data. When a real `getApiClient` is implemented, all sync payloads should be transmitted over TLS to protect captured data in transit.

---

## 9. Claim-to-Source Index

> Use this as a maintenance map when updating this document. Paths are repository-relative to the local repos listed in the intro.

| Claim | Primary Source File(s) | Function-Level Anchors |
| --- | --- | --- |
| Repository versions (`1.0.13`, `0.1.6`, `0.0.5`, `0.0.8`) | `coworkai-desktop/package.json`, `coworkai-agent/package.json`, `coworkai-activity-capture/package.json`, `coworkai-keystroke-capture/package.json` | `version` manifest fields |
| Main boot flow calls `agent.init()` and `sync.start()` | `coworkai-desktop/src/main.ts` | `onReady()` |
| Auth user handoff sets agent user ID | `coworkai-desktop/src/app/main/ipc/auth/index.ts` | `ipcMain.on(IPCChannels.AUTH)` `IPCActions.SET_USER` branch |
| Capture actually starts on tracker start (`agent.start()`) | `coworkai-desktop/src/app/main/ipc/tracker/index.ts`, `coworkai-desktop/src/app/renderer/views/Timer/index.tsx` | `handleStart()` (main tracker IPC), `handleStart()` (renderer timer) |
| Desktop sync stub always returns success | `coworkai-desktop/src/app/main/coworkai-agent/configs/sync.ts` | `syncConfig.getApiClient()` |
| Desktop DB path resolution (prod vs dev) | `coworkai-desktop/src/app/main/coworkai-agent/configs/database.ts`, `coworkai-desktop/src/app/main/config.ts` | `databaseConfig.dir`, `appPath` constants |
| Desktop media output roots (screenshots/audios/videos) | `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/screencapture.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/audio.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/video.ts` | `agentScreenCaptureConfig.outputDir`, `agentAudioConfig.outputDir`, `agentVideoConfig.outputDir` |
| Desktop feature toggles via `E2E` | `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/activity.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/keyboard.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/clipboard.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/screencapture.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/audio.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/video.ts`, `coworkai-desktop/src/app/main/coworkai-agent/configs/agent/timer.ts` | `enabled: process.env.E2E === 'true' ? ...` |
| Agent default feature flags | `coworkai-agent/src/config.ts` | `defaultConfig.agent.*.enabled` |
| WAL mode enabled for SQLite | `coworkai-agent/src/main.ts` | `init(config)`, `database.pragma('journal_mode = WAL')` |
| Activity tracking via active-window subscription + browser enrichment | `coworkai-agent/src/agent/activity/index.ts`, `coworkai-agent/src/agent/activity/lib/x-win.ts` | `Activity.start()`, `Activity.getActivityRecordInfo()`, `subscribeActiveWindow()` |
| Activity in-memory buffer cap (`maxBuffers = 5`) and overflow flush | `coworkai-agent/src/database/activity/manager.ts` | `ActivityManager.constructor(maxBuffers = 5)`, `autoFlushIfNeeded()` |
| Keystroke chunking behavior (debounce + special-key flush) | `coworkai-agent/src/database/keystroke/buffer.ts` | `KeystrokeBuffer.addKeypress()`, `queueCurrentChunk()`, `isSpecialKey()` |
| Copy/cut/paste hotkey detection + clipboard read path | `coworkai-agent/src/agent/keyboard/index.ts` | `getCopyEvent()`, `Keyboard.readClipboard()` |
| Timer daily split at UTC midnight | `coworkai-agent/src/agent/timer/index.ts` | `Timer.checkAndHandleMidnight()`, `Timer.splitSessionAtMidnight()` |
| SyncManager scheduled tables (`activities`, `keystrokes`, `clipboards`) | `coworkai-agent/src/database/index.ts` | `setup()`, `new SyncManager({ tables: [...] })` |
| Sync payload contract and event names (`${tableName}_sync`) | `coworkai-agent/src/database/sync/sync.ts` | `SyncQueue.sendToAPI()` |
| Sync defaults (`batchSize=500`, `syncInterval=30000`) | `coworkai-agent/src/database/sync/sync.ts` | `SyncQueue.constructor()` |
| Backoff constants and current fixed-delay behavior (`getRetryCount() === 0`) | `coworkai-agent/src/database/sync/sync.ts` | `SyncQueue.handleSyncError()`, `SyncQueue.getRetryCount()` |
| SQLite table schemas (`activities`, `keystrokes`, `clipboards`, `timelogs`) | `coworkai-agent/src/database/activity/manager.ts`, `coworkai-agent/src/database/keystroke/buffer.ts`, `coworkai-agent/src/database/clipboard/buffer.ts`, `coworkai-agent/src/database/timelog/buffer.ts` | `ActivityManager.createTable()`, `KeystrokeBuffer.createTable()`, `ClipboardBuffer.createTable()`, `TimeLogBuffer.createTable()` |
| macOS app detection and AppleScript/Accessibility fallback | `coworkai-activity-capture/src/mac/app_detector.mm`, `coworkai-activity-capture/src/mac/browser_base.mm` | `GetFrontmostApplication()`, `GetAppInfoViaAppleScript()`, `GetAppInfoViaAccessibility()`, `BrowserBase::GetActiveWindowInfo()`, `BrowserBase::SearchForWebContent()` |
| Windows foreground app + UI Automation URL extraction and 28-name heuristic | `coworkai-activity-capture/src/windows/app_detector.cpp`, `coworkai-activity-capture/src/windows/helpers.cpp`, `coworkai-activity-capture/src/windows/chrome/chrome.cpp`, `coworkai-activity-capture/src/windows/edge/edge.cpp`, `coworkai-activity-capture/src/windows/firefox/firefox.cpp`, `coworkai-activity-capture/src/windows/brave/brave.cpp`, `coworkai-activity-capture/src/windows/opera/opera.cpp` | `GetForegroundWindowHandle()`, `SafeInitializeCOM()`, `Extract*URLViaUIA()`, `Get*ActiveWindowInfo()` |
| macOS keystroke capture event taps + thread-safe bridge to JS | `coworkai-keystroke-capture/src/mac/keystroke_capture.mm` | `KeyboardThreadEntry()`, `EventTapCallback()`, `OnKey()`, `g_onKeyTsf.NonBlockingCall(...)` |
| Windows low-level hooks, message pump, repeat filtering, character mapping | `coworkai-keystroke-capture/src/win/keystroke_capture.cpp` | `HookThread()`, `LowLevelKeyboardProc()`, `LowLevelMouseProc()`, `GetCharsFromVk()` |
| Native prebuild strategy and artifact upload scripts | `coworkai-activity-capture/scripts/prebuild-platform.js`, `coworkai-activity-capture/scripts/prebuild-upload.js`, `coworkai-keystroke-capture/scripts/prebuild-platform.js`, `coworkai-keystroke-capture/scripts/prebuild-upload.js` | Script entrypoints: `execSync(prebuild ...)`, `execSync(prebuild --upload-all ...)` |
| Electron packaging, native rebuild list, S3 publishing, updater base URLs | `coworkai-desktop/forge.config.ts`, `coworkai-desktop/src/app/main/modules/updater/index.ts` | `getUpdaterBaseUrl()` (forge + updater), publisher `keyResolver()`, `checkForUpdates()` |
