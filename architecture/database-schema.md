# Database Schema: cowork.db

| | |
|---|---|
| **Status** | Draft v9 (reviewed) |
| **Last Updated** | 2026-02-17 |
| **Owner** | Rustan |
| **Sprint** | Phase 1B, Sprint 1 |
| **Source** | [product-features.md](../product/product-features.md), [system-architecture.md](./system-architecture.md) § Database Hardening |

---

## Design Notes

How we got here. This section traces the reasoning so future readers (or future us) know why the schema looks the way it does.

### Source documents

The schema is designed entirely from three sources — no carryover from the old tracking-oriented schema:

1. **[product-features.md](../product/product-features.md)** — Defines the 5 capture streams (window tracking, keystroke, clipboard, focus detection, browser activity — 4 populated in v0.1, browser_sessions in Phase 4) and 4 memory layers (conversation history, working memory, semantic recall, observational memory). The capture streams map 1:1 to capture tables. The memory layers are all Mastra-managed — we don't define DDL for them.

2. **[system-architecture.md](./system-architecture.md) § Database Hardening** — Defines the three ownership zones (capture process, agents process, Mastra), the table inventory per zone, WAL contention strategy, vector index namespacing, and additive migration approach. The agents-process table list came directly from here: `embedding_queue`, `agent_operations`, `automations`, `mcp_servers`, `mcp_connection_state`, `tool_policies`, `notification_log`, `installed_apps`, `app_permissions`.

3. **[phase-1b-sprint-plan.md](./phase-1b-sprint-plan.md)** — Sprint 1 scope definition. Gave the final capture table names (`activity_sessions`, `keystroke_chunks`, `clipboard_events`, `focus_sessions`, `browser_sessions`) which are more specific than system-architecture.md's generic "activities, input events, context streams."

### Why one table per capture stream

product-features.md lists 5 streams with different defaults (on/off), different data shapes, and different retention needs. One table per stream means:
- Toggle a stream on/off without touching other data
- Different retention windows per stream (keystroke data is more sensitive than window titles)
- Clean query patterns — `context:getRecentActivity` can select from `activity_sessions` without filtering out keystroke data

Alternative considered: single `capture_events` table with a `stream_type` discriminator column. Rejected — the columns differ too much between streams (keystroke chunks have `char_count` and `special_keys`, clipboard events have `direction` and `content_hash`, activity sessions have `bundle_id` and `browser_url`). A wide table with mostly-NULL columns would be harder to query and index.

### Why TEXT UUIDs over INTEGER autoincrement

Both capture and agents processes write to the same DB file. If both used `INTEGER PRIMARY KEY AUTOINCREMENT`, there's no collision risk (different tables), but UUIDs are still better because:
- IPC passes IDs between processes — UUIDs are globally unique, no "table + id" compound key needed
- Easier to reference in logs, debug output, and agent context injection
- No rowid fragmentation concerns after retention cleanup deletes old rows
- `crypto.randomUUID()` is fast enough — negligible overhead even for high-frequency capture writes

### Why ISO 8601 TEXT over integer timestamps

SQLite has no native datetime type. Two options: integer Unix timestamps (ms) or ISO 8601 strings. We went with strings because:
- Human-readable in DB browsers (debugging capture data)
- Sort correctly as strings (`2026-02-17T10:00:00Z` < `2026-02-17T11:00:00Z`)
- SQLite `datetime()` functions work on them natively
- Consistent with how Mastra stores its own timestamps

Tradeoff: 24 bytes per timestamp vs 8 bytes for integer. Acceptable at our data volume.

### Why capture streams are independent peers (not parent-child)

product-features.md describes five independently toggleable capture streams. The database mirrors this: each capture table is a self-contained timeline. There is no enforced FK between capture tables — `activity_session_id` columns are optional correlation hints set by the orchestration layer when both streams are active, nullable when not.

Each row carries its own app context (keystroke chunks have `app_name`/`bundle_id`, clipboard events have `source_app`). This means "what was typed in which app?" is answerable from the `keystroke_chunks` table alone — no JOIN to `activity_sessions` required.

The old `coworkai-agent` codebase used a parent-child model (keystrokes were children of activities, linked by FK) because the sync pipeline shipped activities with nested keystrokes as payload containers. That sync pipeline is gone. The new architecture stores data locally — streams are peers correlated by overlapping timestamps, not parent-child entities.

See [CAPTURE_FLUSH_COUPLING_ANALYSIS.md](../decisions/CAPTURE_FLUSH_COUPLING_ANALYSIS.md) for the full reasoning.

### Why focus_sessions is a separate table

Focus sessions are derived from activity_sessions — an extended single-app session above a configurable threshold. Could be a computed view instead of a table. Separate table because:
- The capture process detects and writes focus sessions as they happen (real-time, not retroactive)
- `threshold_ms` is stored per-row so historical focus sessions remain valid if the threshold changes
- Simpler query for the Context view — `SELECT * FROM focus_sessions` vs a window function over activity_sessions

### Config vs runtime state (mcp_servers + mcp_connection_state)

`mcp_servers` holds user-configured connection info (command, args, credentials). `mcp_connection_state` holds process-lifetime runtime state (health check times, failure counts, tools discovered) — persisted to disk but reset on process restart. Split because:
- Connection state resets on process restart — wiping `mcp_connection_state` rows is safe
- Config survives restarts — user shouldn't have to re-enter credentials after a crash
- Different update frequencies — config changes rarely, connection state updates on every health check

### Why COALESCE in the app_permissions unique index

SQLite treats each NULL as distinct in UNIQUE constraints. The schema allows `permission_target = NULL` for broad type-level grants (e.g., "this app can use any tool"). Without COALESCE, `UNIQUE (app_id, permission_type, permission_target)` would allow unlimited `(app_1, tool, NULL)` rows — duplicate broad grants that make permission evaluation ambiguous. `COALESCE(permission_target, '')` maps NULL to empty string for uniqueness checks only, preserving NULL storage for application reads.

### Cross-process reads

The agents process reads capture tables (for `chat:sendMessage` context injection and `context:getRecentActivity`). This is a read-only cross-boundary access — it never writes to capture tables. WAL mode supports concurrent readers without blocking the capture process's writes.

### Tables defined now, populated later

`browser_sessions` (Phase 4), `embedding_queue` (Phase 2), `automations` (Phase 4), `notification_log` (Phase 3) are all defined in this schema but won't be populated until their respective phases. Rationale: each owning process runs all its `CREATE TABLE IF NOT EXISTS` on startup. Defining them now means the migration code is already in place when those features ship — no schema change PR needed, just the feature code.

---

## Overview

Single libsql database file (`cowork.db`) with WAL mode. Three ownership zones — each process creates and migrates only its own tables. No process ALTERs tables owned by another.

| Owner | API | Tables |
|---|---|---|
| **Capture process** | `libsql` (sync) | `activity_sessions`, `keystroke_chunks`, `clipboard_events`, `focus_sessions`, `browser_sessions` |
| **Agents process** | `@libsql/client` (async) | `embedding_queue`, `agent_operations`, `automations`, `mcp_servers`, `mcp_connection_state`, `tool_policies`, `notification_log`, `installed_apps`, `app_permissions` |
| **Mastra** | `LibSQLStore` + `LibSQLVector` | Internal tables (conversations, threads, working memory, observational memory, traces, embeddings) |

**Conventions:**
- Primary keys: TEXT (UUID v4 via `crypto.randomUUID()`)
- Timestamps: TEXT in ISO 8601 (`new Date().toISOString()`). All SQLite functions that produce timestamps must use `strftime('%Y-%m-%dT%H:%M:%fZ', ...)` — never `datetime()` — to match application-code format
- JSON data: TEXT columns with JSON content
- Booleans: INTEGER (0/1). Data access layer converts to `boolean` — TypeScript types below reflect the mapped domain object, not raw SQLite rows
- `updated_at` columns: DEFAULT handles INSERT; application code must set explicitly on UPDATE (SQLite has no `ON UPDATE` clause — no triggers, write-path responsibility)
- No cross-process foreign key constraints (documented in comments, not enforced)
- User preferences (stream toggles, focus threshold, retention windows) live in Electron's config store (`electron-store`), not in libsql — only runtime and captured data lives in the database

---

## Capture Tables

Owned by the Capture Utility process. Written via sync `libsql` API for high-frequency inserts.

### activity_sessions

**Product feature:** [Context § What the AI observes](../product/product-features.md#what-the-ai-observes) — Window & app tracking stream (default: on).

The primary work-context table. Records which app and window the user is in, enabling the AI to answer "what was I just doing?" and power the Context Card's current-state display. Also the base data for focus detection and the `context:getRecentActivity` IPC handler that injects recent activity into every chat prompt. One row per continuous period in a single app/window — new row when user switches focus.

Write semantics: insert row on session start (`ended_at = NULL`), keep `last_observed_at` updated while active, then finalize `ended_at` + `duration_ms` when the session ends.

```sql
CREATE TABLE IF NOT EXISTS activity_sessions (
  id            TEXT PRIMARY KEY,
  app_name      TEXT NOT NULL,           -- e.g. 'Slack', 'VS Code'
  bundle_id     TEXT,                    -- macOS bundle ID, e.g. 'com.tinyspeck.slackmacgap'
  window_title  TEXT,                    -- active window title at session start
  browser_url   TEXT,                    -- extracted URL if app is a browser
  started_at    TEXT NOT NULL,           -- ISO 8601
  ended_at      TEXT,                    -- NULL while active
  last_observed_at TEXT,                 -- heartbeat while active (window still present)
  duration_ms   INTEGER,                -- computed on end: ended_at - started_at
  created_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_activity_sessions_started_at ON activity_sessions (started_at);
CREATE INDEX IF NOT EXISTS idx_activity_sessions_app_name ON activity_sessions (app_name);
```

Crash recovery note: on capture startup, any stale open `activity_sessions` rows (`ended_at IS NULL`) must be closed deterministically. Derive `ended_at` from the latest available signal (`last_observed_at`, `keystroke_chunks.ended_at`, `clipboard_events.captured_at`, `browser_sessions.ended_at`) clamped to recovery boot time; if no signals exist, close at `started_at`.

### keystroke_chunks

**Product feature:** [Context § What the AI observes](../product/product-features.md#what-the-ai-observes) — Keystroke & input capture stream (default: off, opt-in).

Captures raw input patterns to build communication profiles — "Draft a reply in my tone" requires the AI to know how the user writes. Opt-in because keystroke data is the most sensitive capture stream. Chunked (not per-keystroke) to reduce write frequency and limit raw-text exposure window. Flushed on debounce timeout, special-key trigger, 1000-char limit, or parent activity end.

```sql
CREATE TABLE IF NOT EXISTS keystroke_chunks (
  id                    TEXT PRIMARY KEY,
  activity_session_id   TEXT,            -- optional correlation hint (not enforced as FK, nullable)
  app_name              TEXT,            -- denormalized: which app was active when this was typed
  bundle_id             TEXT,            -- denormalized: macOS bundle ID of the active app
  chunk_text            TEXT NOT NULL,   -- accumulated text for this chunk
  char_count            INTEGER NOT NULL,
  special_keys          TEXT,            -- JSON array: ['Enter', 'Tab', 'Cmd+C']
  started_at            TEXT NOT NULL,   -- first keystroke in chunk
  ended_at              TEXT NOT NULL,   -- last keystroke / flush trigger
  created_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_keystroke_chunks_started_at ON keystroke_chunks (started_at);
CREATE INDEX IF NOT EXISTS idx_keystroke_chunks_activity ON keystroke_chunks (activity_session_id);
CREATE INDEX IF NOT EXISTS idx_keystroke_chunks_app_name ON keystroke_chunks (app_name);
```

### clipboard_events

**Product feature:** [Context § What the AI observes](../product/product-features.md#what-the-ai-observes) — Clipboard monitoring stream (default: off, opt-in).

Enriches work context with what the user copies between apps. When a user copies a Zendesk ticket number in one app and pastes it in Slack, the AI can connect those activities. Opt-in because clipboard content can contain sensitive data (credentials, personal messages). `content_hash` enables dedup without storing duplicate content. Captured on copy/paste hotkey detection.

```sql
CREATE TABLE IF NOT EXISTS clipboard_events (
  id                    TEXT PRIMARY KEY,
  activity_session_id   TEXT,            -- optional correlation hint (not enforced as FK, nullable)
  direction             TEXT NOT NULL,   -- 'copy' or 'paste'
  content_type          TEXT NOT NULL,   -- 'text', 'image', 'file'
  content_text          TEXT,            -- text content (NULL for non-text)
  content_hash          TEXT,            -- SHA-256 for dedup
  source_app            TEXT,            -- app where copy/paste occurred
  captured_at           TEXT NOT NULL,   -- when the event happened
  created_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_clipboard_events_captured_at ON clipboard_events (captured_at);
```

### focus_sessions

**Product feature:** [Context § What the AI observes](../product/product-features.md#what-the-ai-observes) — Focus detection stream (default: on). Also drives [Chat § Proactive notifications → Throttling](../product/product-features.md#proactive-notifications).

Enables flow-state detection — "You've been in VS Code for 2 hours working on the auth module" in the Context Card. Also drives notification throttling: when a focus session is active, non-urgent proactive notifications are held until context switch. Derived from activity_sessions — an extended single-app session above a configurable threshold.

```sql
CREATE TABLE IF NOT EXISTS focus_sessions (
  id                    TEXT PRIMARY KEY,
  activity_session_id   TEXT,            -- optional correlation hint (triggering session, not enforced as FK)
  app_name              TEXT NOT NULL,
  bundle_id             TEXT,
  started_at            TEXT NOT NULL,
  ended_at              TEXT,                    -- NULL while active
  duration_ms           INTEGER,                -- computed on end
  threshold_ms          INTEGER NOT NULL,        -- minimum duration to qualify (configurable)
  created_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_focus_sessions_started_at ON focus_sessions (started_at);
CREATE INDEX IF NOT EXISTS idx_focus_sessions_activity ON focus_sessions (activity_session_id);
```

### browser_sessions

**Product feature:** [MCP Browser § What you see](../product/product-features.md#what-you-see) — Browser activity stream (default: on during agent sessions).

The audit trail for MCP Browser actions. Every navigate, click, type, and screenshot the AI performs in a browser session is logged here, powering the execution log timeline shown in the MCP Browser view. Serves the "you are always in control" principle — the user can review exactly what the AI did. Linked to `agent_operations` to trace which agent task triggered each browser action. Write path: the agents process emits browser action events via IPC to the capture process, which persists them — maintaining the "no cross-process writes" rule.

> **Phase 4.** Table created in Phase 1B (capture process owns the migration), populated when MCP Browser ships.

```sql
CREATE TABLE IF NOT EXISTS browser_sessions (
  id                    TEXT PRIMARY KEY,
  agent_operation_id    TEXT,            -- agent_operations.id (cross-process, not enforced as FK)
  url                   TEXT NOT NULL,
  page_title            TEXT,
  action_type           TEXT NOT NULL,   -- 'navigate', 'click', 'type', 'screenshot'
  action_detail         TEXT,            -- JSON with action-specific data
  captured_at           TEXT NOT NULL,
  created_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_browser_sessions_captured_at ON browser_sessions (captured_at);
CREATE INDEX IF NOT EXISTS idx_browser_sessions_operation ON browser_sessions (agent_operation_id);
```

---

## Agent Tables

Owned by the Agents & RAG Utility process. Written via async `@libsql/client` API.

### agent_operations

**Product feature:** [Chat § On-demand conversation](../product/product-features.md#on-demand-conversation) + [MCP Browser § How execution works](../product/product-features.md#how-execution-works).

Every chat interaction and every MCP Browser execution task is an agent operation. This is the execution backbone — it tracks what the agent is doing (`status`), how much it costs (`accumulated_cost_usd`), and whether it's waiting for the user (`pending_tool_call` for approval gates). Operations survive crashes so the UI can show "this task was interrupted" rather than silently dropping work. See [system-architecture.md](./system-architecture.md) § Agent Runtime.

```sql
CREATE TABLE IF NOT EXISTS agent_operations (
  id                    TEXT PRIMARY KEY,
  thread_id             TEXT,            -- Mastra thread reference
  status                TEXT NOT NULL DEFAULT 'idle',
                                         -- idle | running | done | error | waiting_for_human
  step_count            INTEGER NOT NULL DEFAULT 0,
  max_steps             INTEGER NOT NULL DEFAULT 50,
  accumulated_cost_usd  REAL NOT NULL DEFAULT 0.0,
  model_id              TEXT,            -- e.g. 'openrouter/anthropic/claude-sonnet-4.5'
  pending_tool_call     TEXT,            -- JSON: tool call awaiting human approval
  error_message         TEXT,
  started_at            TEXT,
  completed_at          TEXT,
  created_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_agent_operations_status ON agent_operations (status);
CREATE INDEX IF NOT EXISTS idx_agent_operations_thread ON agent_operations (thread_id);
```

### embedding_queue

**Product feature:** [Context § What the AI remembers](../product/product-features.md#what-the-ai-remembers).

The bridge between raw capture data and searchable memory. When new activity sessions or keystroke chunks are written, the embedding queue tracks them as "pending." The embedding pipeline processes pending items through Qwen3-Embedding-0.6B, stores vectors via LibSQLVector, and marks them "completed." This is what makes zero-context queries work — "What was that refund ticket I was looking at yesterday?" searches embedded activity data via RAG.

> **Phase 2.** Table created in Phase 1B, populated when embedding pipeline ships.

```sql
CREATE TABLE IF NOT EXISTS embedding_queue (
  id              TEXT PRIMARY KEY,
  source_table    TEXT NOT NULL,          -- 'activity_sessions', 'keystroke_chunks', etc.
  source_id       TEXT NOT NULL,          -- PK of the source row
  index_name      TEXT NOT NULL,          -- target vector index: embed_activity, embed_conversation, embed_observation
  status          TEXT NOT NULL DEFAULT 'pending',
                                          -- pending | processing | completed | failed
  error_message   TEXT,
  attempts        INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  processed_at    TEXT
);

CREATE INDEX IF NOT EXISTS idx_embedding_queue_status ON embedding_queue (status);
CREATE UNIQUE INDEX IF NOT EXISTS idx_embedding_queue_source ON embedding_queue (source_table, source_id, index_name);
```

### mcp_servers

**Product feature:** [MCP Integrations](../product/product-features.md#2-mcp-integrations).

The foundation layer that both Apps and Chat depend on. Each row is a connected service — Zendesk, Gmail, Slack, etc. The user adds connections through the Integrations view; the platform agent and installed apps both use the tools these servers expose. Transport config (stdio command or SSE URL) plus encrypted credentials enables the MCPManager to start/stop servers on boot and reconnect after crashes. Credentials (OAuth tokens, API keys) encrypted via Electron `safeStorage` API and stored as BLOB in this table per [system-architecture.md](./system-architecture.md) § MCP Connections — the OS keychain holds the encryption key, the encrypted payload lives in `cowork.db` keyed by `credential_key`.

```sql
CREATE TABLE IF NOT EXISTS mcp_servers (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,          -- display name
  transport       TEXT NOT NULL DEFAULT 'stdio',  -- stdio | sse
  command         TEXT,                   -- stdio: executable path
  args            TEXT,                   -- stdio: JSON array of args
  url             TEXT,                   -- sse: server URL
  env             TEXT,                   -- JSON: non-sensitive env vars
  credential_key  TEXT,                   -- logical key: 'cowork-mcp-{serverUrlHash}'
  credential_encrypted BLOB,              -- safeStorage.encryptString() output (OS-encrypted OAuth tokens / API keys)
  enabled         INTEGER NOT NULL DEFAULT 1,
  created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  updated_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),

  CHECK (
    (transport = 'stdio' AND length(command) > 0) OR
    (transport = 'sse' AND length(url) > 0)
  )
);
```

### mcp_connection_state

**Product feature:** [MCP Integrations](../product/product-features.md#2-mcp-integrations) — connection health monitoring.

Drives the status dots (green/red/grey) in the Integrations SideSheet card and the "X needs attention" health summary. Separated from config (`mcp_servers`) because connection state is process-lifetime — persisted to disk during runtime but reset on process restart, while config persists across restarts. Also tracks `tools_count` so the UI can show available tools per connection.

```sql
CREATE TABLE IF NOT EXISTS mcp_connection_state (
  server_id             TEXT PRIMARY KEY, -- mcp_servers.id (same-process FK)
  status                TEXT NOT NULL DEFAULT 'disconnected',
                                          -- disconnected | starting | running | error
  last_health_check_at  TEXT,
  last_error            TEXT,
  consecutive_failures  INTEGER NOT NULL DEFAULT 0,
  tools_count           INTEGER NOT NULL DEFAULT 0,
  updated_at            TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),

  FOREIGN KEY (server_id) REFERENCES mcp_servers (id) ON DELETE CASCADE
);
```

### tool_policies

**Product feature:** [MCP Browser § Approval gates](../product/product-features.md#how-execution-works).

Implements "configurable per app, per category, or per action — the user controls how much autonomy the AI gets." Each MCP tool can be set to `allow` (auto-execute), `ask` (require approval), or `deny` (block). Default is `ask` — the cautious default that lets users gradually increase autonomy as trust builds. Checked by the agents process when executing any MCP tool call, regardless of whether the caller is the platform agent or an app. Distinct from `app_permissions` which gates whether an app can reach tools at all — `tool_policies` gates what happens when the tool actually fires.

```sql
CREATE TABLE IF NOT EXISTS tool_policies (
  id          TEXT PRIMARY KEY,
  server_id   TEXT NOT NULL,              -- mcp_servers.id
  tool_name   TEXT NOT NULL,
  policy      TEXT NOT NULL DEFAULT 'ask', -- allow | ask | deny
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  updated_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),

  FOREIGN KEY (server_id) REFERENCES mcp_servers (id) ON DELETE CASCADE,
  UNIQUE (server_id, tool_name)
);
```

### automations

**Product feature:** [Automations](../product/product-features.md#5-automations).

Stores workflow definitions with three DDL trigger types: `schedule` maps to product-features.md's time-based triggers ("Every morning at 9am, summarize my unread emails"), `event` covers both event-based ("When a high-priority Zendesk ticket arrives, draft a response") and activity-pattern triggers ("When I start a Zoom call, mute Slack notifications") — the distinction lives in `trigger_config` JSON, and `manual` enables user-triggered one-off runs. `trigger_config` and `action_config` are JSON to support the different shapes per trigger type. `run_count` + `last_run_status` power the "Active Automations" SideSheet card.

> **Phase 4.** Table created in Phase 1B, populated when automations engine ships.

```sql
CREATE TABLE IF NOT EXISTS automations (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  description     TEXT,
  trigger_type    TEXT NOT NULL,           -- schedule | event | manual
  trigger_config  TEXT NOT NULL,           -- JSON: trigger-specific config
  action_config   TEXT NOT NULL,           -- JSON: action steps
  enabled         INTEGER NOT NULL DEFAULT 1,
  last_run_at     TEXT,
  last_run_status TEXT,                    -- success | error | skipped
  run_count       INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  updated_at      TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);
```

### notification_log

**Product feature:** [Chat § Proactive notifications](../product/product-features.md#proactive-notifications).

Enables notification throttling: dismissal learning (consistently dismissed triggers get deprioritized), hourly caps per tier, and cooldown periods. Each notification's lifecycle (`pending` → `shown` → `dismissed`/`acted`) is tracked here. `dismissed_at` timestamps feed the learning algorithm that adjusts future notification frequency. Also serves as an audit trail — users can review what the AI surfaced and when.

> **Phase 3.** Table created in Phase 1B, populated when proactive notifications ship.

```sql
CREATE TABLE IF NOT EXISTS notification_log (
  id            TEXT PRIMARY KEY,
  type          TEXT NOT NULL,             -- notification category
  title         TEXT NOT NULL,
  body          TEXT,
  source        TEXT,                      -- agent/automation that triggered it
  status        TEXT NOT NULL DEFAULT 'pending',
                                           -- pending | shown | dismissed | acted
  dismissed_at  TEXT,
  created_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX IF NOT EXISTS idx_notification_log_created_at ON notification_log (created_at);
CREATE INDEX IF NOT EXISTS idx_notification_log_status ON notification_log (status);
```

### installed_apps

**Product feature:** [Apps](../product/product-features.md#1-apps) + [Apps § App Gallery](../product/product-features.md#app-gallery).

Registry for apps uploaded to Cowork.ai. Each installed app gets a sandboxed WebContentsView and a compact status card in the SideSheet. The `manifest` column stores the full `cowork.manifest.json` which declares what MCP tools the app needs. `status` tracks whether the app is running (`active`), user-disabled (`disabled`), or failed to load (`error`). Drives the App Gallery's "installed" state and the Apps view listing.

```sql
CREATE TABLE IF NOT EXISTS installed_apps (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  version       TEXT,
  description   TEXT,
  author        TEXT,
  icon_path     TEXT,                      -- relative to app install directory
  entry_point   TEXT NOT NULL DEFAULT 'index.html',
  manifest      TEXT,                      -- full cowork.manifest.json as JSON
  install_path  TEXT NOT NULL,             -- relative path within apps directory
  status        TEXT NOT NULL DEFAULT 'active',  -- active | disabled | error
  installed_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  updated_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);
```

### app_permissions

**Product feature:** [Apps § App permissions](../product/product-features.md#1-apps).

Implements "Each app declares what MCP tools it needs (like app permissions on a phone). The platform grants scoped access per app." Populated from the app's manifest on install, then editable by the user. Two permission types: `tool` (specific MCP tool access, including platform-provided tools like `platform_chat` for agent-as-tool use cases) and `capture_read` (can read activity context via MCP resources). The app preload SDK checks these before forwarding any IPC call.

```sql
CREATE TABLE IF NOT EXISTS app_permissions (
  id                TEXT PRIMARY KEY,
  app_id            TEXT NOT NULL,         -- installed_apps.id
  permission_type   TEXT NOT NULL,         -- tool | capture_read
  permission_target TEXT CHECK (permission_target IS NULL OR length(permission_target) > 0),  -- tool name, or NULL for broad type-level grant
  granted           INTEGER NOT NULL DEFAULT 1,
  source            TEXT NOT NULL DEFAULT 'manifest',  -- manifest | user
  created_at        TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),

  FOREIGN KEY (app_id) REFERENCES installed_apps (id) ON DELETE CASCADE
);

-- COALESCE guards against SQLite NULL != NULL in UNIQUE — prevents duplicate broad grants
CREATE UNIQUE INDEX IF NOT EXISTS idx_app_permissions_unique
  ON app_permissions (app_id, permission_type, COALESCE(permission_target, ''));
CREATE INDEX IF NOT EXISTS idx_app_permissions_app ON app_permissions (app_id);
```

---

## Mastra-Managed Tables

Mastra creates and migrates its own tables via `LibSQLStore` and `LibSQLVector`. We configure — not define — these.

**What we control:**

| Setting | Value |
|---|---|
| DB file path | `cowork.db` (same file as app tables) |
| LibSQLStore | `new LibSQLStore({ url: 'file:cowork.db' })` |
| LibSQLVector | `new LibSQLVector({ connectionUrl: 'file:cowork.db' })` |

**Vector indexes we create:**

| Index name | Data source | Created in |
|---|---|---|
| `embed_activity` | Window titles, URLs, app sessions from `activity_sessions` | Phase 2 |
| `embed_conversation` | Past conversations from Mastra conversation history | Phase 2 |
| `embed_observation` | Observational memory summaries from Mastra | Phase 2 |

Indexes created via `LibSQLVector.createIndex()` with Qwen3-Embedding-0.6B dimensions. Exact dimension count TBD at implementation time based on model config.

**Mastra internal tables** (do not modify, do not migrate, do not query directly):
- Conversation threads and messages — [Chat § On-demand conversation](../product/product-features.md#on-demand-conversation): stores the full chat history that gives the AI conversational continuity
- Working memory (user profile, preferences) — [Context § What the AI remembers](../product/product-features.md#what-the-ai-remembers): how the user communicates, their preferences, learned context that makes the AI "get smarter the longer you use it"
- Observational memory (compressed summaries) — [Context § What the AI remembers](../product/product-features.md#what-the-ai-remembers): long-term compressed observations that survive retention cleanup of raw capture data
- Agent traces and telemetry — internal diagnostics, not user-facing
- Embedding storage tables — [Context § What the AI remembers](../product/product-features.md#what-the-ai-remembers): vector storage for semantic recall via RAG, enabling zero-context queries across activity and conversations

---

## Retention Policies

| Table | Retention | Enforcement |
|---|---|---|
| `activity_sessions` | Rolling — configurable days (default TBD) | `DELETE WHERE ended_at < strftime('%Y-%m-%dT%H:%M:%fZ', 'now', '-N days')` |
| `keystroke_chunks` | Rolling — configurable days (default TBD) | Same pattern |
| `clipboard_events` | Rolling — configurable days (default TBD) | Same pattern |
| `focus_sessions` | Rolling — configurable days (default TBD) | Same pattern |
| `browser_sessions` | Rolling — configurable days (default TBD) | Same pattern |
| `embedding_queue` | Prune completed — 7 days after processing | `DELETE WHERE status = 'completed' AND processed_at < strftime('%Y-%m-%dT%H:%M:%fZ', 'now', '-7 days')` |
| `agent_operations` | Permanent | None |
| `automations` | Permanent | None |
| `mcp_servers` | Permanent | None |
| `mcp_connection_state` | Process-lifetime — reset on restart | `DELETE FROM mcp_connection_state` on process startup |
| `tool_policies` | Permanent | None |
| `notification_log` | Rolling — 30 days | `DELETE WHERE created_at < strftime('%Y-%m-%dT%H:%M:%fZ', 'now', '-30 days')` |
| `installed_apps` | Permanent | None |
| `app_permissions` | Permanent (cascade on app delete) | FK CASCADE |

Retention enforcement runs as a periodic task in each owning process. Capture process cleans capture tables; agents process cleans agent tables. Not implemented in Phase 1B — retention defaults are "keep everything" until enforcement is built.

---

## Process Ownership Map

```
cowork.db (WAL mode, busy_timeout = 5000ms)
│
├── Capture Utility Process (sync libsql client)
│   ├── CREATE/ALTER: activity_sessions
│   ├── CREATE/ALTER: keystroke_chunks
│   ├── CREATE/ALTER: clipboard_events
│   ├── CREATE/ALTER: focus_sessions
│   ├── CREATE/ALTER: browser_sessions
│   ├── WRITE: all above
│   └── READ: own tables only
│
├── Agents & RAG Utility Process (async @libsql/client)
│   ├── CREATE/ALTER: agent_operations
│   ├── CREATE/ALTER: embedding_queue
│   ├── CREATE/ALTER: mcp_servers
│   ├── CREATE/ALTER: mcp_connection_state
│   ├── CREATE/ALTER: tool_policies
│   ├── CREATE/ALTER: automations
│   ├── CREATE/ALTER: notification_log
│   ├── CREATE/ALTER: installed_apps
│   ├── CREATE/ALTER: app_permissions
│   ├── WRITE: all above
│   ├── READ: own tables + capture tables (for context injection)
│   └── DELEGATE: Mastra LibSQLStore + LibSQLVector (same client)
│
└── Mastra (via @mastra/libsql)
    ├── CREATE/ALTER: internal tables (conversations, memory, traces, embeddings)
    ├── WRITE: internal tables
    └── READ: internal tables
```

**Key rules:**
1. Each process creates its tables on startup (`CREATE TABLE IF NOT EXISTS`)
2. No process ALTERs tables owned by another process
3. Agents process can READ capture tables (for `context:getRecentActivity` and pre-chat context injection)
4. Cross-process write contention handled by WAL busy timeout (5000ms)
5. Mastra tables are a black box — query through Mastra APIs, never raw SQL

---

## Migration Strategy

v0.1 uses additive migrations only. No version-tracked migration framework.

**On process startup:**
1. Run all `CREATE TABLE IF NOT EXISTS` statements for owned tables
2. Run all `CREATE INDEX IF NOT EXISTS` statements for owned tables
3. For schema changes in future versions: `ALTER TABLE ADD COLUMN` wrapped in try/catch (ignore "column already exists")

**Persistent pragma** (set once — stored in the database file, survives restarts):
```sql
PRAGMA journal_mode = WAL;
```

**Per-connection pragmas** (every process must set these on each new connection):
```sql
PRAGMA busy_timeout = 5000;
PRAGMA foreign_keys = ON;
```

---

## TypeScript Row Types

Mapped domain types for each table — what application code works with after the data access layer converts raw SQLite values (e.g., `0|1` → `boolean`). Implementation copies these into `src/shared/types/database.ts`.

```typescript
// ── Capture Tables ──────────────────────────────────────

interface ActivitySession {
  id: string;
  app_name: string;
  bundle_id: string | null;
  window_title: string | null;
  browser_url: string | null;
  started_at: string;
  ended_at: string | null;
  last_observed_at: string | null;
  duration_ms: number | null;
  created_at: string;
}

interface KeystrokeChunk {
  id: string;
  activity_session_id: string | null;  // optional correlation hint
  app_name: string | null;             // denormalized: active app when typed
  bundle_id: string | null;            // denormalized: active app bundle ID
  chunk_text: string;
  char_count: number;
  special_keys: string | null;       // JSON: string[]
  started_at: string;
  ended_at: string;
  created_at: string;
}

interface ClipboardEvent {
  id: string;
  activity_session_id: string | null;
  direction: 'copy' | 'paste';
  content_type: 'text' | 'image' | 'file';
  content_text: string | null;
  content_hash: string | null;
  source_app: string | null;
  captured_at: string;
  created_at: string;
}

interface FocusSession {
  id: string;
  activity_session_id: string | null;
  app_name: string;
  bundle_id: string | null;
  started_at: string;
  ended_at: string | null;
  duration_ms: number | null;
  threshold_ms: number;
  created_at: string;
}

interface BrowserSession {
  id: string;
  agent_operation_id: string | null;
  url: string;
  page_title: string | null;
  action_type: 'navigate' | 'click' | 'type' | 'screenshot';
  action_detail: string | null;      // JSON
  captured_at: string;
  created_at: string;
}

// ── Agent Tables ────────────────────────────────────────

interface AgentOperation {
  id: string;
  thread_id: string | null;
  status: 'idle' | 'running' | 'done' | 'error' | 'waiting_for_human';
  step_count: number;
  max_steps: number;
  accumulated_cost_usd: number;
  model_id: string | null;
  pending_tool_call: string | null;  // JSON
  error_message: string | null;
  started_at: string | null;
  completed_at: string | null;
  created_at: string;
}

interface EmbeddingQueueItem {
  id: string;
  source_table: string;
  source_id: string;
  index_name: 'embed_activity' | 'embed_conversation' | 'embed_observation';
  status: 'pending' | 'processing' | 'completed' | 'failed';
  error_message: string | null;
  attempts: number;
  created_at: string;
  processed_at: string | null;
}

interface McpServer {
  id: string;
  name: string;
  transport: 'stdio' | 'sse';
  command: string | null;
  args: string | null;               // JSON: string[]
  url: string | null;
  env: string | null;                // JSON: Record<string, string>
  credential_key: string | null;
  credential_encrypted: Buffer | null;  // safeStorage.encryptString() output
  enabled: boolean;                  // stored as INTEGER 0/1
  created_at: string;
  updated_at: string;
}

interface McpConnectionState {
  server_id: string;
  status: 'disconnected' | 'starting' | 'running' | 'error';
  last_health_check_at: string | null;
  last_error: string | null;
  consecutive_failures: number;
  tools_count: number;
  updated_at: string;
}

interface ToolPolicy {
  id: string;
  server_id: string;
  tool_name: string;
  policy: 'allow' | 'ask' | 'deny';
  created_at: string;
  updated_at: string;
}

interface Automation {
  id: string;
  name: string;
  description: string | null;
  trigger_type: 'schedule' | 'event' | 'manual';
  trigger_config: string;            // JSON
  action_config: string;             // JSON
  enabled: boolean;                  // stored as INTEGER 0/1
  last_run_at: string | null;
  last_run_status: 'success' | 'error' | 'skipped' | null;
  run_count: number;
  created_at: string;
  updated_at: string;
}

interface NotificationLogEntry {
  id: string;
  type: string;
  title: string;
  body: string | null;
  source: string | null;
  status: 'pending' | 'shown' | 'dismissed' | 'acted';
  dismissed_at: string | null;
  created_at: string;
}

interface InstalledApp {
  id: string;
  name: string;
  version: string | null;
  description: string | null;
  author: string | null;
  icon_path: string | null;
  entry_point: string;
  manifest: string | null;           // JSON: full cowork.manifest.json
  install_path: string;
  status: 'active' | 'disabled' | 'error';
  installed_at: string;
  updated_at: string;
}

interface AppPermission {
  id: string;
  app_id: string;
  permission_type: 'tool' | 'capture_read';
  permission_target: string | null;
  granted: boolean;                  // stored as INTEGER 0/1
  source: 'manifest' | 'user';
  created_at: string;
}
```

---

## Changelog

**v10 (Feb 17, 2026):** Independent capture streams. Added `app_name` and `bundle_id` to `keystroke_chunks` (denormalized from active activity, same pattern as `clipboard_events.source_app`). Updated all `activity_session_id` DDL comments across capture tables from FK-style references to "optional correlation hint (not enforced as FK, nullable)." Added `idx_keystroke_chunks_app_name` index. Added design note "Why capture streams are independent peers" explaining the product-driven rationale and departure from the old parent-child model. Updated KeystrokeChunk TypeScript type.

**v9 (Feb 17, 2026):** Capture-session durability update. Added `activity_sessions.last_observed_at` heartbeat column for passive/read-only session recovery accuracy. Updated `activity_sessions` write semantics text (insert on start, finalize on end) and added crash-recovery close note for stale open sessions. Updated keystroke chunk flush semantics to include parent activity end, and added `last_observed_at` to TypeScript row type.

**v8 (Feb 17, 2026):** Final review pass. No schema/DDL changes required. Updated document status metadata from `Draft v6` to `Draft v7` to match the latest applied review version.

**v7 (Feb 17, 2026):** Follow-up consistency pass. Tightened app permission model to match product-features.md's "Apps get tools, not agents" rule: removed `chat` as a first-class `permission_type` and documented agent access as a platform-provided MCP tool (`platform_chat`) under `tool` permissions. Updated DDL comments, rationale text, and TypeScript type union.

**v6 (Feb 17, 2026):** Fourth Codex review — cross-doc consistency fixes. (1) Renamed `app_permissions.permission_type` value from `agent` to `chat` — "Apps get tools, not agents" (product-features.md). The `chat` type gates `window.cowork.chat()`, which is the agent-as-tool pattern: the app sees a request/response interface, the platform runs the agent with full control. (2) Added `credential_encrypted BLOB` column to `mcp_servers` — system-architecture.md and decision log specify encrypted credentials persisted to `cowork.db` via safeStorage, but schema only had the lookup key. Now both the key (`credential_key`) and the encrypted payload (`credential_encrypted`) are in the DDL. (3) Documented `browser_sessions` write path — agents process emits IPC events, capture process persists, maintaining no-cross-process-write rule. (4) Normalized `mcp_connection_state` lifecycle from contradictory "ephemeral"/"Permanent" to "process-lifetime — persisted to disk during runtime, reset on restart."

**v5 (Feb 17, 2026):** Added product feature rationale to every table — each now traces back to a specific section in [product-features.md](../product/product-features.md) with explanation of what product capability the table enables. Also added feature references to Mastra-managed internal tables. Self-review fixes: (1) `tool_policies` — removed incorrect "Apps § App permissions" cross-reference and fixed enforcement layer (agents process checks these during execution, not the app preload SDK; distinct from `app_permissions`). (2) `automations` — fixed trigger type mapping (`manual` ≠ activity-pattern; activity-pattern maps to `event` type with pattern config in JSON). (3) `clipboard_events` — replaced "same sensitivity reasons as keystroke capture" with specific clipboard sensitivity rationale.

**v4 (Feb 17, 2026):** Third Codex review pass. Tightened `mcp_servers` CHECK — `length(command) > 0` / `length(url) > 0` instead of `IS NOT NULL` (empty strings are unusable configs). Added CHECK on `app_permissions.permission_target` blocking empty strings — prevents collision with COALESCE sentinel in unique index. Fixed header version to match changelog.

**v3 (Feb 17, 2026):** Second Codex review pass. Clarified TypeScript types are mapped domain objects (post `0|1 → boolean` conversion), not raw SQLite rows. Added `updated_at` write-path convention (application code responsibility, no triggers). No schema changes — findings on v1→v2 migration (no deployed DBs exist) and enum CHECK constraints (intentionally app-level) dismissed.

**v2 (Feb 17, 2026):** Review fixes from Claude + Codex cross-review. (1) Replaced all `datetime('now')` DEFAULTs with `strftime('%Y-%m-%dT%H:%M:%fZ', 'now')` to match ISO 8601 convention — fixes timestamp format mismatch that would break retention queries and time comparisons. Updated retention DELETE examples to use `strftime` consistently. Added timestamp function convention note. (2) Replaced `UNIQUE (app_id, permission_type, permission_target)` on `app_permissions` with COALESCE-based unique index — SQLite treats NULL as distinct in UNIQUE, allowing duplicate broad grants. Added design note. (3) Split WAL pragmas into persistent (`journal_mode`) and per-connection (`busy_timeout`, `foreign_keys`) — the latter must be set by every process on each connection, not just "whichever opens first." (4) Added `CHECK` constraint on `mcp_servers` enforcing `stdio` requires `command` and `sse` requires `url`. (5) Added `activity_session_id` to `focus_sessions` — explicit back-reference to the triggering activity session, avoids time-range joins. Added index and updated TypeScript type. (6) Added user preferences storage note (Electron config store, not libsql). (7) Fixed "5 v0.1 capture streams" wording — 4 populated in v0.1, `browser_sessions` in Phase 4.

**v1 (Feb 17, 2026):** Initial schema design. 14 tables across two ownership zones (5 capture, 9 agent) plus Mastra-managed internals. Derived from product-features.md (5 v0.1 capture streams), system-architecture.md (database hardening, agent runtime, MCP, apps), and phase-1b-sprint-plan.md (table inventory). Tables for Phase 2+ features (embedding_queue, browser_sessions, automations, notification_log) defined now, populated later. Retention defaults and exact rolling-window durations TBD.
