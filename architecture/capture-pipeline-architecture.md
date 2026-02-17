# Capture Pipeline Architecture (Native Addon -> SQLite)

| Field | Value |
|---|---|
| Status | Active |
| Last Updated | 2026-02-17 |
| Owner | Rustan |
| Sprint | Phase 1B, Sprint 4 |
| Source | `coworkai-desktop` implementation |
| Related | [system-architecture.md](./system-architecture.md) § Native Capture Layer, [database-schema.md](./database-schema.md) § Capture Tables, [CAPTURE_FLUSH_COUPLING_ANALYSIS.md](../decisions/CAPTURE_FLUSH_COUPLING_ANALYSIS.md) |

This document is the implementation-level reference for how capture data moves from native addons into `cowork.db`, including exact write/flush triggers, IPC protocol, supervisor lifecycle, and full behavioral specification.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MAIN PROCESS                                                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │  CaptureSupervisor  (supervisor.ts)                              │       │
│  │                                                                  │       │
│  │  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌──────────────┐ │       │
│  │  │ Lifecycle │  │ Permission │  │  Restart  │  │  IPC Bridge  │ │       │
│  │  │ Manager  │  │   Gate     │  │  Backoff  │  │ (req/res)    │ │       │
│  │  └──────────┘  └────────────┘  └───────────┘  └──────┬───────┘ │       │
│  └───────────────────────────────────────────────────────┼──────────┘       │
│                                                          │                  │
│          utilityProcess.fork()          postMessage / on('message')         │
│                                                          │                  │
├──────────────────────────────────────────────────────────┼──────────────────┤
│  UTILITY PROCESS                                         │                  │
│                                                          ▼                  │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │  capture.worker.ts                                               │       │
│  │                                                                  │       │
│  │  ┌───────────────────────────────────────────────────────────┐   │       │
│  │  │  Command Router                                           │   │       │
│  │  │  init │ start │ stop │ health │ shutdown │ toggleStream   │   │       │
│  │  │  getStatus │ getStreamConfig │ simulate                   │   │       │
│  │  └───────────────────────────────────────────────────────────┘   │       │
│  │                                                                  │       │
│  │  ┌─────────────────────┐   ┌──────────────────────────────────┐ │       │
│  │  │  x-win              │   │  KeystrokeCapture                │ │       │
│  │  │  subscribeActive    │   │  (@engineering-go2/              │ │       │
│  │  │  Window (100ms)     │   │   coworkai-keystroke-capture)    │ │       │
│  │  └────────┬────────────┘   └──────────────┬───────────────────┘ │       │
│  │           │                                │                     │       │
│  │           ▼                                ▼                     │       │
│  │  ┌────────────────┐              ┌──────────────────┐           │       │
│  │  │ ActivityCapture │              │ onKeystroke()    │           │       │
│  │  │ (enrichment    │              │                  │           │       │
│  │  │  only)         │              │ ┌──────────────┐ │           │       │
│  │  └───────┬────────┘              │ │ Keystroke    │ │           │       │
│  │          │                       │ │ Chunker      │ │           │       │
│  │          ▼                       │ │ (1000 chars, │ │           │       │
│  │  ┌──────────────────┐           │ │  1200ms idle)│ │           │       │
│  │  │ applyWindow      │           │ └──────┬───────┘ │           │       │
│  │  │ Context()        │           │        │         │           │       │
│  │  │                  │           │ ┌──────────────┐ │           │       │
│  │  │ session start /  │           │ │ Clipboard    │ │           │       │
│  │  │ heartbeat /      │           │ │ Detection    │ │           │       │
│  │  │ boundary change  │           │ │ (Cmd/Ctrl+   │ │           │       │
│  │  └──────┬───────────┘           │ │  C/V/X)      │ │           │       │
│  │         │                       │ └──────┬───────┘ │           │       │
│  │         │                       └────────┼─────────┘           │       │
│  │         │                                │                     │       │
│  │         ▼                                ▼                     │       │
│  │  ┌─────────────────────────────────────────────────────────┐   │       │
│  │  │  libsql (sync API)                                      │   │       │
│  │  │  cowork.db — WAL mode, busy_timeout=5000                │   │       │
│  │  │                                                         │   │       │
│  │  │  ┌──────────────────┐  ┌─────────────────┐             │   │       │
│  │  │  │ activity_sessions│  │ keystroke_chunks │             │   │       │
│  │  │  └──────────────────┘  └─────────────────┘             │   │       │
│  │  │  ┌──────────────────┐  ┌─────────────────┐             │   │       │
│  │  │  │ clipboard_events │  │ focus_sessions   │             │   │       │
│  │  │  └──────────────────┘  └─────────────────┘             │   │       │
│  │  │  ┌──────────────────┐                                   │   │       │
│  │  │  │ browser_sessions │  (schema exists, no write path)   │   │       │
│  │  │  └──────────────────┘                                   │   │       │
│  │  └─────────────────────────────────────────────────────────┘   │       │
│  └──────────────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1. End-to-End Data Path

```
Supervisor.start()
    │
    ▼
utilityProcess.fork("capture.worker.js")
    │
    ├── Worker posts  { id: "__worker_ready__" }  ◄── handshake
    │
    ▼
Supervisor sends  init  { dbPath, streamConfig }
    │
    ├── Worker opens libsql DB at dbPath
    ├── Applies pragmas: WAL, busy_timeout=5000, foreign_keys=ON
    ├── Runs CREATE TABLE IF NOT EXISTS for all 5 tables
    ├── Runs schema migrations (add columns, drop legacy columns)
    ├── Recovers stale open sessions (ended_at IS NULL)
    ├── Initializes ActivityCapture (enrichment; failure is non-fatal)
    ├── Initializes KeystrokeCapture + registers onKey callback
    │
    ▼
Supervisor sends  start
    │
    ├── Worker subscribes to x-win active window events (100ms interval)
    ├── Seeds initial activity from x-win.activeWindow() snapshot
    ├── Starts keystroke native capture (if permission granted)
    ├── Starts chunk idle flush poller (500ms interval)
    │
    ▼
Events flow continuously until  stop  or  shutdown
    │
    ├── x-win events  ──►  applyWindowContext()  ──►  activity_sessions writes
    ├── onKey events   ──►  KeystrokeChunker     ──►  keystroke_chunks writes
    ├── onKey events   ──►  clipboard detection  ──►  clipboard_events writes
    ├── activity finalize  ──►  maybeInsertFocusSession()  ──►  focus_sessions writes
    │
    ▼
Supervisor sends  stop  (or  shutdown  which calls stop first)
    │
    ├── Unsubscribes from x-win
    ├── Stops chunk idle poller
    ├── Stops native keystroke capture
    ├── Flushes open keystroke chunk
    ├── Finalizes current activity session
    └── (shutdown only) Closes DB, resets state
```

## 2. Runtime Components

| Component | File | Purpose |
|-----------|------|---------|
| Supervisor | `src/app/main/modules/capture/supervisor.ts` | Spawns/manages utility process, IPC bridge, restart backoff, permission gate |
| Worker | `src/electron/capture.worker.ts` | Capture engine — DB writes, native addon orchestration, session management |
| Keystroke chunker | `src/core/capture/keystroke-chunker.ts` | Buffers keystrokes into chunks with debounce and max-length flush |
| Clipboard events | `src/core/capture/clipboard-events.ts` | Hotkey detection (Cmd/Ctrl+C/V/X), duplicate suppression |
| Recovery | `src/core/capture/recovery.ts` | Stale session recovery — timestamp derivation and clamping |
| Stream config | `src/app/main/modules/capture/config.ts` | Reads/writes `capture-streams.json`, caches in memory |
| Stream contract | `src/shared/ipc-schemas.ts` | Stream names, defaults, IPC type definitions |
| DB constants | `src/core/store/database.ts` | `DB_FILENAME = 'cowork.db'`, `toDbUrl()` helper |
| Activity buffer | `src/core/capture/activity-buffer.ts` | Generic queue-flush buffer (not used by current worker; kept for future) |

## 3. Supervisor Lifecycle

### 3.1 Service states

```
            ┌──────────┐
            │ degraded │ ◄── initial state
            └────┬─────┘
     start()     │
                 ▼
            ┌──────────┐
            │ starting │
            └────┬─────┘
                 │
        ┌────────┴────────┐
  success│                │ failure / timeout (10s)
        ▼                 ▼
  ┌──────────┐     ┌────────────┐
  │ running  │     │  degraded  │──── scheduleRestart()
  └────┬─────┘     └────────────┘           │
       │                                     ▼
       │ child exit                   ┌────────────┐
       └─────────────────────────────►│ restarting │
                                      └──────┬─────┘
                                             │ backoff elapsed
                                             ▼
                                       spawnAndBoot()
                                             │
                            ┌────────────────┴──────────┐
                     success│                           │ 5 failures
                            ▼                           ▼
                      ┌──────────┐              ┌────────────┐
                      │ running  │              │   failed   │ (terminal)
                      └──────────┘              └────────────┘
```

### 3.2 Restart backoff

Backoff delays: `[2s, 4s, 8s, 16s, 32s]`. After 5 failed attempts the supervisor enters `failed` state permanently (until app restart).

### 3.3 Permission gate

`setPermissionGate(blocked)` is called from the setup module (macOS Accessibility permission):
- `blocked=true`: sets `permissionsBlocked=true`, state to `degraded`, calls `pauseForSystem('permissions-blocked')`.
- `blocked=false`: sets `permissionsBlocked=false`, calls `resumeFromSystem('permissions-cleared')` which clears `pausedBySystem` and resumes capture if `desiredRunning` is true.

### 3.4 System pause/resume

Power monitor events (screen lock, sleep) call `pauseForSystem(reason)` / `resumeFromSystem(reason)`. While paused, the worker stays alive but capture is stopped (worker receives `stop` command).

### 3.5 Worker path resolution

Supervisor checks these paths in order:
1. `${__dirname}/capture.worker.js` (Vite dev build)
2. `${process.cwd()}/.vite/build/capture.worker.js` (CWD-relative)
3. `${process.resourcesPath}/app.asar/.vite/build/capture.worker.js` (packaged)

### 3.6 IPC protocol

Request/response with UUID correlation:

```
Supervisor ──postMessage──► Worker
{
  id: "<uuid>",
  command: "init" | "start" | "stop" | "health" | "shutdown"
           | "toggleStream" | "getStatus" | "getStreamConfig",
  payload?: { ... }
}

Worker ──postMessage──► Supervisor
{
  id: "<uuid>",              // correlates to request
  type: "result" | "error",
  command: "<command>",
  payload: { ... }           // or error: { message: "..." }
}
```

Timeouts:
- Worker startup: `10,000ms`
- Worker ready handshake: `2,000ms`
- Normal command: `6,000ms`
- Shutdown command: `4,000ms`

## 4. Stream Defaults and Config Resolution

### 4.1 Streams

Five capture streams, all default ON:

| Stream | What it captures | Native addon |
|--------|-----------------|--------------|
| `window_tracking` | Active window app/title changes | `@miniben90/x-win` |
| `focus` | Focus sessions (activity sessions exceeding duration threshold) | Derived from `activity_sessions` |
| `browser` | Browser URL tracking | `@miniben90/x-win` (url field) |
| `keystroke` | Keystroke chunks | `@engineering-go2/coworkai-keystroke-capture` |
| `clipboard` | Copy/paste/cut events | Keystroke addon + OS clipboard read |

### 4.2 Activity monitoring group

`window_tracking`, `focus`, and `browser` share a single x-win subscription. Activity monitoring is active when **any** of these three streams is enabled (`window_tracking || focus || browser`).

### 4.3 Keystroke capture group

The native keystroke capture addon is needed when **either** `keystroke` or `clipboard` is enabled, since clipboard detection piggybacks on keystroke events.

### 4.4 Config persistence

File location:
- Packaged: `~/Library/Application Support/<App>/agent/capture-streams.json`
- Dev: `<repo>/agent/capture-streams.json`

Resolution:
1. If config file does not exist: silently return all-ON defaults (no warning).
2. If file exists: read and parse JSON.
3. Normalize: merge known stream booleans over defaults, ignore unknown keys.
4. On parse failure: log warning, fall back to all-ON defaults.
5. Cache in memory; write-through on toggle.

## 5. Activity Stream Behavior (`activity_sessions`)

### 5.1 Input source

- Primary: `@miniben90/x-win` `subscribeActiveWindow(callback, 100)` — polls native API at **100ms** intervals and fires callback on each poll.
- Enrichment: `@engineering-go2/coworkai-activity-capture` `getActiveWindowInfo()` — called synchronously on each x-win event to supplement `windowTitle` or `browserUrl` when **either** field is missing. Enrichment is skipped only if **both** `windowTitle` and `browserUrl` are already present. Additionally, the `ActivityCapture` app name must match the x-win app name (case-insensitive) — mismatches skip enrichment to prevent cross-app data contamination.

### 5.2 Window context extraction

From an `XWinWindowInfo` event, the worker extracts:
- `appName`: `info.info?.name` (falling back to `info.info?.execName`), trimmed, null if empty. Note: these are nested under `info.info`, not top-level.
- `bundleId`: always `null` (x-win does not provide macOS bundle IDs).
- `windowTitle`: `info.title` (top-level), trimmed.
- `browserUrl`: `info.url` (top-level), trimmed.

Events with no valid `appName` are silently dropped.

### 5.3 Session start

On stream start, the worker calls `x-win.activeWindow()` synchronously to seed the first session. If no current session exists and the window context is valid:
1. Generate `randomUUID()` session id.
2. INSERT `activity_sessions` row with `ended_at = NULL`, `last_observed_at = started_at`, `duration_ms = NULL`.
3. Set in-memory `currentSession` with `startedAtMs`, `lastObservedAtMs`, `lastHeartbeatWriteAtMs` all equal to `Date.now()`.

### 5.4 Session heartbeat

When an x-win event arrives with **unchanged** context (all 4 fields match: `appName`, `bundleId`, `windowTitle`, `browserUrl`):
1. Update in-memory `lastObservedAtMs` to event time.
2. If `>= 1000ms` since last DB write (`HEARTBEAT_WRITE_MS`): UPDATE `last_observed_at` in DB.

### 5.5 Session boundary change

A boundary change triggers when **any** of these 4 fields differ from the current session:
- `appName`
- `bundleId`
- `windowTitle`
- `browserUrl`

On change:
1. Flush open keystroke chunk, tagged with **previous** session id.
2. Finalize previous activity session: set `ended_at`, compute `duration_ms`, update `last_observed_at`.
3. Evaluate focus session insertion for the finalized session (if `duration_ms >= 300,000ms`).
4. Create and INSERT new activity session.

### 5.6 Finalization triggers

Activity session finalization occurs on:
- Active window context boundary change.
- Worker `stop` command.
- Worker `shutdown` command (calls stop internally).
- `toggleStream` that results in all activity monitoring streams disabled (`!window_tracking && !focus && !browser`).

### 5.7 Startup recovery

During `init`, the worker finds all `activity_sessions` rows where `ended_at IS NULL` and closes them in a single transaction:

For each stale session:
1. Gather latest signal timestamp from: `last_observed_at`, max `keystroke_chunks.ended_at`, max `clipboard_events.captured_at`, max `browser_sessions.captured_at` (matched by URL).
2. Pick the maximum of those signals.
3. Clamp: `endedAt = min(latestSignal, recoveryBootTimestamp)`, floored at `startedAt`.
4. UPDATE the row with `ended_at`, `duration_ms`, `last_observed_at`.
5. Insert a `focus_sessions` row if duration exceeds threshold (forced insert, ignoring stream toggle).

## 6. Keystroke Stream Behavior (`keystroke_chunks`)

### 6.1 Input source

Native addon: `@engineering-go2/coworkai-keystroke-capture` — `KeystrokeCapture.onKey(callback)` fires for every key event.

The `onKeystroke` handler in the worker:
1. If `keystroke` stream enabled: push event through `KeystrokeChunker`.
2. Always (regardless of keystroke stream): check for clipboard hotkey via `maybeInsertClipboardEvent()`.

### 6.2 Event acceptance filter (KeystrokeChunker)

**Rejected** by chunker (returns `null`):
- `event.state !== 'DOWN'` (UP events dropped)
- `event.key` is falsy (empty string, null, undefined)
- `event.key` is whitespace-only **and** not a single space character (after trim)

Note: a single space character `' '` is caught as **printable** before the trim check and is accepted.

**Accepted**:
- Single printable characters: `a`, `B`, `7`, `' '` (space char), etc.
- Multi-char key names `SPACE` / `SPACEBAR` -> normalized to printable space.
- Auto-repeat DOWN events (no repeat suppression in current implementation).
- Modifier key events passed through as-is (if key name is multi-char, wrapped as control token).

### 6.3 Key normalization (full mapping)

`keystroke-chunker.ts:normalizeKey()`:

| Native key name | Token | Printable count |
|----------------|-------|-----------------|
| Single character (e.g. `a`, `7`) | `a`, `7` | 1 |
| `" "` (space char) | `" "` | 1 |
| `SPACE`, `SPACEBAR` | `" "` | 1 |
| `RETURN`, `ENTER` | `<Enter>` | 0 |
| `TAB` | `<Tab>` | 0 |
| `ESC`, `ESCAPE` | `<Escape>` | 0 |
| `BACKSPACE` | `<Backspace>` | 0 |
| `DELETE`, `DEL` | `<Delete>` | 0 |
| Any other multi-char key (e.g. `Left Arrow`, `F1`, `Shift`) | `<Left Arrow>`, `<F1>`, `<Shift>` | 0 |

Comparison is case-insensitive (`toUpperCase()`). Unknown multi-char keys are preserved as `<{trimmedKey}>` — never dropped.

### 6.4 `chunk_text` format

Single ordered stream of printable characters and control tokens interleaved:

```
Hello<Backspace><Backspace>lo World<Enter>
```

### 6.5 `char_count` semantics

- Counts **printable characters only** (single-char keys and normalized spaces).
- Control tokens (`<Enter>`, `<Backspace>`, `<F1>`, etc.) contribute to `chunk_text` length but **not** to `char_count`.

### 6.6 Flush triggers

| Trigger | Condition | Mechanism |
|---------|-----------|-----------|
| Max printable length | `printableCount >= 1000` | `chunker.push()` returns the chunk |
| Idle debounce | `nowMs - lastEventAtMs >= 1200ms` | `chunker.flushIfIdle()` checked every `500ms` by poller |
| Activity boundary | Window context change detected | `flushOpenChunk(previousSessionId)` before finalize |
| Worker stop | `stop` or `shutdown` command | `flushOpenChunk()` called in stop handler |
| Keystroke stream OFF | `toggleStream('keystroke', false)` | `flushOpenChunk()` called in toggle handler |

### 6.7 Chunk idle poller

`setInterval` at **500ms** (`CHUNK_IDLE_POLL_MS`). Each tick calls `chunker.flushIfIdle()` which checks if `>= 1200ms` have passed since the last key event. If yes, flushes the buffer to DB.

### 6.8 Persistence fields

| Column | Source |
|--------|--------|
| `id` | `randomUUID()` |
| `activity_session_id` | Current activity session id (may be `null` if no session) |
| `app_name` | Current session `appName` (denormalized) |
| `bundle_id` | Current session `bundleId` (denormalized) |
| `chunk_text` | Chunker buffer content |
| `char_count` | Chunker printable count |
| `started_at` | ISO timestamp of first key event in chunk |
| `ended_at` | ISO timestamp of last key event in chunk |

## 7. Clipboard Stream Behavior (`clipboard_events`)

### 7.1 Trigger detection

Clipboard events are detected inside the keystroke `onKey` callback — **not** via a separate clipboard subscription. The `getClipboardDirectionFromKeyEvent()` function checks:

- `event.state === 'DOWN'` (required)
- `event.isRepeat` is falsy (required — if truthy, the event is ignored; `undefined` counts as non-repeat)
- `event.command === true` OR `event.control === true` (at least one modifier)
- `event.key` (case-insensitive):
  - `c` -> `copy`
  - `v` -> `paste`
  - `x` -> `cut`

### 7.2 Clipboard text read

Clipboard text is read synchronously from the OS at trigger time:
- macOS: `execSync('pbpaste', { encoding: 'utf8' })`
- Windows: `execSync('powershell -NoProfile -Command "Get-Clipboard -Raw"', { encoding: 'utf8' })`
- Linux: not supported (returns `null`)

For `copy` and `cut` directions: a delayed re-read is performed after **100ms** (`CLIPBOARD_COPY_RETRY_DELAY_MS`) to catch cases where the OS clipboard hasn't been updated yet. The delayed result is preferred if non-empty; otherwise the immediate result is used.

For `paste`: only the immediate read is used (clipboard content already exists).

### 7.3 Acceptance rules

An event is **skipped** if:
- `clipboard` stream is disabled.
- Clipboard text is empty or `null` (non-text payloads like images/files produce empty reads).
- Worker is no longer running (checked after the async delay for copy/cut).
- Duplicate: same `direction + content_hash` within **250ms** (`CLIPBOARD_EVENT_DEDUPE_MS`).

### 7.4 Content hashing

`createHash('sha256').update(contentText).digest('hex')` — full SHA-256 of the raw clipboard text (not trimmed).

### 7.5 Persistence fields

| Column | Source |
|--------|--------|
| `id` | `randomUUID()` |
| `activity_session_id` | Current activity session id (may be `null`) |
| `direction` | `copy`, `paste`, or `cut` |
| `content_type` | Always `'text'` |
| `content_text` | Raw clipboard text (not trimmed) |
| `content_hash` | SHA-256 hex of `content_text` |
| `source_app` | Current session `appName` |
| `captured_at` | ISO timestamp at detection time (before clipboard read) |

## 8. Focus Stream Behavior (`focus_sessions`)

### 8.1 What it is

Focus sessions are **derived** records — they are written when an `activity_sessions` row is finalized and the session duration meets the threshold. There is no separate input source.

### 8.2 Threshold

`FOCUS_THRESHOLD_MS = 300,000` (5 minutes). A focus session is only created if `duration_ms >= 300,000`.

### 8.3 Insertion rules

On activity session finalization:
1. If `focus` stream is disabled **and** this is not a recovery-forced insert: skip.
2. If `duration_ms < 300,000ms`: skip.
3. Check for existing `focus_sessions` row with the same `activity_session_id`: if found, skip (dedup).
4. INSERT focus session row.

During startup recovery, focus session insertion is **forced** (ignores the stream toggle) to ensure recovered long sessions get focus rows.

### 8.4 Persistence fields

| Column | Source |
|--------|--------|
| `id` | `randomUUID()` |
| `activity_session_id` | The finalized activity session id |
| `app_name` | Activity session `app_name` |
| `bundle_id` | Always `null` (not populated from activity session) |
| `started_at` | Activity session `started_at` |
| `ended_at` | Activity session `ended_at` |
| `duration_ms` | Computed `ended_at - started_at` |
| `threshold_ms` | `300000` (the threshold used) |

## 9. Browser Sessions (`browser_sessions`)

The `browser_sessions` table schema is created during migration, but **no write path exists in the current worker**. This table is reserved for future agent-driven browser action logging. Fields:

| Column | Type |
|--------|------|
| `id` | TEXT PRIMARY KEY |
| `agent_operation_id` | TEXT (nullable) |
| `url` | TEXT NOT NULL |
| `page_title` | TEXT (nullable) |
| `action_type` | TEXT NOT NULL |
| `action_detail` | TEXT (nullable) |
| `captured_at` | TEXT NOT NULL |
| `created_at` | TEXT NOT NULL DEFAULT (auto) |

The recovery system reads `browser_sessions.captured_at` as a signal for deriving stale session end times (matched by URL).

## 10. DB Schema

For the full DDL (CREATE TABLE, indexes, pragmas, migration strategy, TypeScript row types, and retention policies), see [database-schema.md](./database-schema.md) § Capture Tables. That document is the canonical schema source.

Summary of tables written by the capture pipeline:

| Table | Purpose |
|-------|---------|
| `activity_sessions` | One row per unique window context (app + title + URL) |
| `keystroke_chunks` | Buffered keystroke segments with denormalized app context |
| `clipboard_events` | Individual copy/paste/cut actions with content hash |
| `focus_sessions` | Derived from activity sessions exceeding 5-minute threshold |
| `browser_sessions` | Reserved for future agent browser actions (no write path yet) |

## 11. Constants Reference

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `XWIN_SUBSCRIBE_INTERVAL_MS` | `100` | worker | x-win active window poll interval |
| `CHUNK_IDLE_POLL_MS` | `500` | worker | Keystroke chunk idle flush check interval |
| `HEARTBEAT_WRITE_MS` | `1000` | worker | Min interval between activity heartbeat DB writes |
| `FOCUS_THRESHOLD_MS` | `300,000` (5 min) | worker | Min activity session duration to generate focus session |
| `CLIPBOARD_COPY_RETRY_DELAY_MS` | `100` | worker | Delay before re-reading clipboard on copy/cut |
| `CLIPBOARD_EVENT_DEDUPE_MS` | `250` | worker | Window for suppressing duplicate clipboard events |
| `debounceMs` | `1200` | chunker | Idle time before flushing keystroke buffer |
| `maxChunkLength` | `1000` | chunker | Max printable char count before forced flush |
| `START_TIMEOUT_MS` | `10,000` | supervisor | Max time to wait for worker boot |
| `READY_TIMEOUT_MS` | `2,000` | supervisor | Max time to wait for worker ready handshake |
| `REQUEST_TIMEOUT_MS` | `6,000` | supervisor | Default command timeout |
| `RESTART_BACKOFF_MS` | `[2000, 4000, 8000, 16000, 32000]` | supervisor | Exponential backoff delays for restart attempts |

## 12. Trigger Matrix (Quick Reference)

| Stream | Table | Write type | Trigger |
|--------|-------|-----------|---------|
| Activity | `activity_sessions` | INSERT (open row) | First valid x-win event with no current session |
| Activity | `activity_sessions` | UPDATE heartbeat | Unchanged window context, throttled to 1s |
| Activity | `activity_sessions` | UPDATE finalize | Context change, stop, shutdown, all activity streams OFF |
| Keystroke | `keystroke_chunks` | INSERT chunk | Max printable length (1000), idle debounce (1200ms), boundary flush, stop, keystroke OFF |
| Clipboard | `clipboard_events` | INSERT event | `Cmd/Ctrl+C/V/X` DOWN (non-repeat) with non-empty text clipboard, deduped at 250ms |
| Focus | `focus_sessions` | INSERT derived | Activity session finalized with `duration_ms >= 300,000ms` |
| Browser | `browser_sessions` | (none) | Table exists but no write path in current worker |

## 13. Simulation Mode

The worker always routes the `simulate` command, but the handler throws unless `CAPTURE_SIMULATION=1` is set in the environment. This is for deterministic local testing without native addons. The supervisor's `CaptureWorkerCommand` type does not include `simulate` — this command is only usable via direct child process messaging or test harnesses.

Supported actions:

| Action | What it does | Stream gate |
|--------|-------------|-------------|
| `windowSample` | Injects a synthetic window context into `applyWindowContext()` | None (always runs) |
| `text` | Injects synthetic key-down events character by character (supports `stepMs` default 8ms, `flushAtEnd` default true) | Returns `{ignored}` if `keystroke` stream disabled |
| `key` | Injects a single synthetic key event into the chunker | Skips chunker push if `keystroke` stream disabled |
| `flushChunk` | Forces a chunker flush | None |
| `clipboard` | Injects a synthetic clipboard event directly into DB (bypasses dedupe logic and clipboard read) | Returns `{ignored}` if `clipboard` stream disabled |

All actions accept `atIso` for deterministic timestamps and bypass native addon initialization. Unlike real `onKeystroke`, the `key` and `text` actions do **not** trigger clipboard detection.

## 14. Bug Lessons Captured in Current Implementation

1. **Native key naming normalization**: Addon emits names like `SPACE`, `RETURN`; chunker normalizes via explicit uppercase mapping. Case-insensitive comparison ensures `Space`, `SPACE`, `space` all work.

2. **Descriptor key retention**: Unknown descriptors are preserved as `<key>` control tokens in-order — never dropped. This means new key names from addon updates appear in data automatically.

3. **Single ordered keystroke field**: The legacy `special_keys` column is removed. All key order stays in `chunk_text` as a single interleaved stream.

4. **Clipboard correctness**:
   - Non-text clipboard payloads (images, files) produce empty `pbpaste` output and are skipped.
   - Text is **not trimmed** before hashing or storage — preserves whitespace fidelity.
   - `copy`/`cut` includes 100ms delayed re-read to reduce race with OS clipboard update.
   - Repeat key events are explicitly filtered before clipboard detection.

5. **Config resilience**: Malformed `capture-streams.json` is caught, warned, and defaults are used. No crash.

6. **Activity enrichment isolation**: `ActivityCapture.getActiveWindowInfo()` failure during init is non-fatal — the worker continues without enrichment. Enrichment is skipped if both `windowTitle` and `browserUrl` are already present (no need), and is only applied when the `ActivityCapture` app name matches the x-win app name case-insensitively (prevents cross-app data contamination).

7. **Recovery transaction safety**: Stale session recovery runs in a single `BEGIN/COMMIT` transaction with `ROLLBACK` on error, preventing partial recovery state.

## 15. Validation Commands

```bash
# deterministic local worker/db check
npm run capture:validate:local

# focused chunker tests
npm test -- tests/capture/keystroke-chunker.test.ts

# type safety
npx tsc --noEmit
```

## 16. Legacy Reference

For the pre-gut baseline from the old `coworkai-agent` codebase (activity source model, keystroke capture path, buffering semantics, clipboard capture path, and key differences between old and current implementations), see [COWORKAI_TECHNICAL_REFERENCE.md](../decisions/COWORKAI_TECHNICAL_REFERENCE.md).

---

## Changelog

**v1 (Feb 17, 2026):** Initial architecture doc. Ported from `coworkai-desktop/docs/architecture/capture-pipeline-reference.md` and adapted to cowork-brain conventions. Removed inline DB schema (canonical source: [database-schema.md](./database-schema.md)). Removed inline legacy reference (canonical source: [COWORKAI_TECHNICAL_REFERENCE.md](../decisions/COWORKAI_TECHNICAL_REFERENCE.md)). Added cross-references to system-architecture.md, database-schema.md, and CAPTURE_FLUSH_COUPLING_ANALYSIS.md.
