# IPC Contract

| | |
|---|---|
| **Status** | Draft v6 |
| **Last Updated** | 2026-02-17 |
| **Owner** | Rustan |
| **Sprint** | Phase 1B, Sprint 2 |
| **Parent** | [system-architecture.md](./system-architecture.md) § IPC Contract |
| **Implements** | `src/shared/ipc-channels.ts`, `src/shared/ipc-schemas.ts` |

---

## Overview

This document is the full IPC contract for Cowork.ai's Electron multi-process architecture. [system-architecture.md](./system-architecture.md) defines the three IPC patterns and handler infrastructure. This document enumerates every channel with Zod schemas for request and response payloads.

**Three IPC patterns** (defined in system-architecture.md — not repeated here):

| Pattern | Used for | Key detail |
|---|---|---|
| **invoke/handle** | All request-response operations | Standard `ipcRenderer.invoke()` / `ipcMain.handle()` |
| **MessagePort streaming** | Chat response streaming | SSE-formatted JSON chunks, ACK gate, proven in Sprint 0 spike |
| **Event push** | Health monitoring | `webContents.send()` — no polling, event-driven |

**Process routing rule:** The renderer never talks directly to capture or agents. Everything routes through main. Capture and agents never communicate via IPC — they share data through `cowork.db`.

---

## Design Notes

How this contract was derived and the key decisions behind it.

### Sources

The channel inventory wasn't invented — it was extracted from four existing docs:

1. **[phase-1b-sprint-plan.md](./phase-1b-sprint-plan.md) § Sprint 2** — Already had the 9-namespace channel table with Phase 1B flags. That table is the skeleton. This doc adds the Zod schemas.
2. **[system-architecture.md](./system-architecture.md) § IPC Contract** — Defines the three patterns (invoke/handle, MessagePort streaming, event push), the `@channel` decorator pattern, `tracedInvoke` observability, and the preload namespace table. This doc doesn't redefine patterns — it populates them.
3. **[product-features.md](../product/product-features.md)** — The 6 features (Apps, MCP Integrations, Chat, MCP Browser, Automations, Context) define what data needs to flow. Each namespace maps to a feature: `chat:*` ↔ Chat, `mcp:*` ↔ MCP Integrations, `apps:*` ↔ Apps, `context:*` ↔ Context, `browser:*` ↔ MCP Browser, `automation:*` ↔ Automations.
4. **[design-system.md](../design/design-system.md)** — The three-state UI model (Closed → SideSheet → Detail Canvas) tells us which views trigger which channels. Context Card needs `context:getContextCard`, Chat FAB needs `chat:sendMessage`, Settings needs `capture:toggleStream` + `system:health`, etc.
5. **[database-schema.md](./database-schema.md)** — Defines the persistence model this IPC layer reads/writes (`mcp_servers`, `mcp_connection_state`, `installed_apps`, `app_permissions`, capture tables used by context/chat). IPC payloads map to these ownership boundaries.

### Why these schema shapes

**`chat:sendMessage` returns immediately, streams separately.** The invoke returns `{ threadId }` so the renderer knows which thread to attach the stream to. The actual response streams over a separate MessagePort. Alternative considered: returning a ReadableStream from invoke — rejected because Electron's invoke/handle serializes return values (no streaming). The MessagePort transfer pattern was proven in the Sprint 0 spike.

**`threadId` is optional on `sendMessage`.** Omitting it creates a new thread. The agents process generates the ID and returns it. This avoids the renderer needing to generate IDs (which would require coordination on format/uniqueness with the agents process).

**`activityRecord` is a flat object with optional fields, not a discriminated union per activity type.** Simpler for the Context view to render — it just maps over a list. The `type` field tells the UI which icon/layout to use. The alternative (separate `WindowActivity`, `FocusActivity`, etc. types) would force the renderer to handle each variant, which isn't needed for a list view.

**`apps:*` channels are main-renderer-only (management).** App lifecycle management (`apps:install`, `apps:list`, `apps:remove`) is used by the main renderer's Apps view. The previous design had `apps:callTool` / `apps:listTools` / `apps:getAppConfig` shared between the main renderer and app preloads — that dual-caller pattern is superseded. Sandboxed apps now use the read lane (`cowork:read`) instead of `apps:*` channels. See [system-architecture.md — Context Pipeline](./system-architecture.md#context-pipeline) for the full rationale on why apps get a typed read-only SDK instead of direct tool access.

**MCP `apiKey` is in the connect request, not a separate channel.** It's encrypted via Electron `safeStorage` at the main process layer before persisting to libsql. The renderer sends the plaintext key once; main encrypts and stores it. No separate "store credential" flow needed for Phase 1B (API key auth only).

**`system:health` is push, not poll.** The renderer doesn't ask "are you healthy?" — main pushes state transitions as they happen. This is simpler and eliminates the question of poll interval. The renderer just listens and updates the store. Matches the event-driven pattern in system-architecture.md.

### Why Zod at IPC boundaries

Electron IPC is untyped at runtime — `ipcRenderer.invoke()` accepts `any` and returns `any`. Zod schemas at the boundary give us:

- **Runtime validation** — catch shape mismatches before they cause cryptic errors deep in handler logic
- **TypeScript types via `z.infer`** — single source of truth, no manual type/schema sync
- **Documentation** — the schemas ARE the API spec, readable in this doc and enforced in code

Validation happens at the handler entry point (parse request) and at the preload return (parse response). The `@channel` decorator from system-architecture.md can auto-wrap handlers with schema validation.

### What's deferred

- **`browser:*` and `automation:*` schemas** — Phase 4. Channel names are listed for completeness but no Zod schemas. These features aren't designed in enough detail yet to define payloads.
- **`app:*` extensibility** — `app:getSettings` returns a minimal shape. More fields (LLM provider selection, capture defaults, notification preferences) will be added as those features land.
- **App tool execution** — `apps:callTool` and `apps:listTools` schemas are retained but deferred from Phase 1B app preload. Apps use the read lane (`cowork:read`) for data access. Tool execution from apps is out of scope for Phase 1B — will be designed if concrete app use cases require it.
- **Pagination** — Only `chat:getThreads` has cursor-based pagination. Other list endpoints (`apps:list`, `mcp:listConnections`) are expected to have small result sets in v0.1. Pagination can be added if lists grow.

### Decisions locked for Phase 1B

1. **Keep `context:query` separate from `context:getRecentActivity`.** They share direct SQL behavior in Phase 1B, but diverge in Phase 2 (`query` moves to vector retrieval while `getRecentActivity` stays relational).
2. **Streaming chunks omit `threadId` for v0.1.** The renderer binds stream events to the `threadId` returned by `chat:sendMessage`. Add chunk-level `threadId` only when parallel generations become a supported UX.
3. **`apps:install` accepts `zipPath` (not raw buffer).** This avoids large IPC payload serialization and matches Electron file picker flow.

---

## Channel Inventory

All channels across all phases. Phase 1B implements everything except `browser:*`, `automation:*`, and the three deferred `apps:*` SDK channels (`callTool`, `listTools`, `getAppConfig`).

| Channel | Direction | Pattern | Handler Process | Phase |
|---|---|---|---|---|
| `capture:getStatus` | Renderer → Main → Capture | invoke/handle | Capture | 1B |
| `capture:toggleStream` | Renderer → Main → Capture | invoke/handle | Capture | 1B |
| `capture:getStreamConfig` | Renderer → Main → Capture | invoke/handle | Capture | 1B |
| `chat:sendMessage` | Renderer → Main → Agents | invoke/handle + MessagePort | Agents | 1B |
| `chat:streamPort` | Main → Renderer | port transfer | Main (relay) | 1B |
| `chat:getThreads` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `chat:getThread` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `context:query` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `context:getRecentActivity` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `context:getContextCard` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `mcp:connect` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `mcp:disconnect` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `mcp:listConnections` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `mcp:listTools` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `mcp:getStatus` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `apps:install` | Renderer → Main → Agents | invoke/handle | Main + Agents | 1B |
| `apps:list` | Renderer → Main → Agents | invoke/handle | Agents | 1B |
| `apps:remove` | Renderer → Main → Agents | invoke/handle | Main + Agents | 1B |
| `apps:callTool` | Renderer → Main → Agents | invoke/handle | Agents | Deferred |
| `apps:listTools` | Renderer → Main → Agents | invoke/handle | Agents | Deferred |
| `apps:getAppConfig` | Renderer → Main → Agents | invoke/handle | Agents | Deferred |
| `cowork:read` | App → Main → Agents | invoke/handle | Agents | 1B |
| `app:getSettings` | Renderer → Main | invoke/handle | Main | 1B |
| `app:setTheme` | Renderer → Main | invoke/handle | Main | 1B |
| `system:health` | Main → Renderer | event push | Main | 1B |
| `system:retry` | Renderer → Main | invoke/handle | Main | 1B |
| `browser:execute` | Renderer → Main → Agents → Playwright | invoke/handle | Agents | 4 |
| `browser:getState` | Renderer → Main → Agents → Playwright | invoke/handle | Agents | 4 |
| `browser:stop` | Renderer → Main → Agents → Playwright | invoke/handle | Agents | 4 |
| `browser:getExecutionLog` | Renderer → Main → Agents | invoke/handle | Agents | 4 |
| `automation:create` | Renderer → Main → Agents | invoke/handle | Agents | 4 |
| `automation:update` | Renderer → Main → Agents | invoke/handle | Agents | 4 |
| `automation:delete` | Renderer → Main → Agents | invoke/handle | Agents | 4 |
| `automation:trigger` | Renderer → Main → Agents | invoke/handle | Agents | 4 |
| `automation:getRunLog` | Renderer → Main → Agents | invoke/handle | Agents | 4 |

**Note on `apps:*` scope change:** `apps:callTool`, `apps:listTools`, and `apps:getAppConfig` were previously designed as dual-caller channels (main renderer + app preload). These are now **deferred** — app preloads use the read lane (`cowork:read`) instead. The schemas are retained for potential future use but are not exposed to sandboxed apps in Phase 1B. `apps:install`, `apps:list`, and `apps:remove` remain main-renderer-only channels for app lifecycle management.

**Note on `cowork:read`:** Single channel for all app read lane requests. The app preload maps typed SDK methods (`window.cowork.context.*`, etc.) to `cowork:read` invocations with namespace + method dispatch. See [App Preload SDK](#app-preload-sdk) and [system-architecture.md — Context Pipeline](./system-architecture.md#context-pipeline) for the full protocol.

---

## Shared Types

Common Zod schemas used across multiple channel namespaces. These go in `src/shared/ipc-schemas.ts`.

```ts
import { z } from 'zod';

// ── Enums ──────────────────────────────────────────────

export const captureStreamSchema = z.enum([
  'window_tracking',
  'keystroke',
  'focus',
  'browser',
  'clipboard',
]);
export type CaptureStream = z.infer<typeof captureStreamSchema>;

export const serviceNameSchema = z.enum([
  'capture',
  'agents',
  'ollama',
  'mcp',
]);
export type ServiceName = z.infer<typeof serviceNameSchema>;

export const serviceStateSchema = z.enum([
  'starting',
  'running',
  'degraded',
  'restarting',
  'failed',
]);
export type ServiceState = z.infer<typeof serviceStateSchema>;

export const themeSchema = z.enum(['dark', 'light', 'system']);
export type Theme = z.infer<typeof themeSchema>;

export const mcpConnectionStatusSchema = z.enum([
  'starting',
  'running',
  'stopped',
  'error',
]);
export type MCPConnectionStatus = z.infer<typeof mcpConnectionStatusSchema>;

export const appStatusSchema = z.enum([
  'installed',
  'running',
  'error',
]);
export type AppStatus = z.infer<typeof appStatusSchema>;

// ── Shared Objects ─────────────────────────────────────

export const streamStatusSchema = z.object({
  stream: captureStreamSchema,
  enabled: z.boolean(),
  active: z.boolean(),
  error: z.string().optional(),
});
export type StreamStatus = z.infer<typeof streamStatusSchema>;

export const chatMessageSchema = z.object({
  id: z.string(),
  role: z.enum(['user', 'assistant', 'system']),
  content: z.string(),
  createdAt: z.string().datetime(),
});
export type ChatMessage = z.infer<typeof chatMessageSchema>;

export const threadSummarySchema = z.object({
  threadId: z.string(),
  title: z.string().optional(),
  lastMessage: z.string(),
  messageCount: z.number().int().nonnegative(),
  updatedAt: z.string().datetime(),
});
export type ThreadSummary = z.infer<typeof threadSummarySchema>;

export const threadSchema = z.object({
  threadId: z.string(),
  title: z.string().optional(),
  messages: z.array(chatMessageSchema),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});
export type Thread = z.infer<typeof threadSchema>;

export const activityRecordSchema = z.object({
  id: z.string(),
  type: z.enum(['window', 'focus', 'keystroke', 'clipboard', 'browser']),
  timestamp: z.string().datetime(),
  appName: z.string().optional(),
  windowTitle: z.string().optional(),
  url: z.string().optional(),
  duration: z.number().nonnegative().optional(),
  summary: z.string().optional(),
});
export type ActivityRecord = z.infer<typeof activityRecordSchema>;

export const mcpToolSchema = z.object({
  name: z.string(),
  description: z.string(),
  inputSchema: z.record(z.unknown()),
  serverId: z.string(),
});
export type MCPTool = z.infer<typeof mcpToolSchema>;

export const mcpConnectionSchema = z.object({
  serverId: z.string(),
  name: z.string(),
  status: mcpConnectionStatusSchema,
  toolCount: z.number().int().nonnegative(),
  error: z.string().optional(),
});
export type MCPConnection = z.infer<typeof mcpConnectionSchema>;

export const appPermissionsSchema = z.object({
  tools: z.array(z.string()).default([]),
  captureRead: z.boolean().default(false),
});
export type AppPermissions = z.infer<typeof appPermissionsSchema>;

export const installedAppSchema = z.object({
  appId: z.string(),
  name: z.string(),
  version: z.string(),
  permissions: appPermissionsSchema,
  status: appStatusSchema,
  installedAt: z.string().datetime(),
});
export type InstalledApp = z.infer<typeof installedAppSchema>;
```

---

## Channel Reference

### `capture:*`

Main relays to capture utility process. All three channels use invoke/handle.

#### `capture:getStatus`

Returns the current status of all capture streams.

```ts
// Request: void (no payload)

export const captureGetStatusResponseSchema = z.object({
  streams: z.array(streamStatusSchema),
});
export type CaptureGetStatusResponse = z.infer<typeof captureGetStatusResponseSchema>;
```

#### `capture:toggleStream`

Enable or disable a specific capture stream.

```ts
export const captureToggleStreamRequestSchema = z.object({
  stream: captureStreamSchema,
  enabled: z.boolean(),
});
export type CaptureToggleStreamRequest = z.infer<typeof captureToggleStreamRequestSchema>;

export const captureToggleStreamResponseSchema = z.object({
  stream: captureStreamSchema,
  enabled: z.boolean(),
});
export type CaptureToggleStreamResponse = z.infer<typeof captureToggleStreamResponseSchema>;
```

#### `capture:getStreamConfig`

Returns per-stream configuration including defaults.

```ts
// Request: void (no payload)

export const captureGetStreamConfigResponseSchema = z.object({
  streams: z.array(z.object({
    stream: captureStreamSchema,
    enabled: z.boolean(),
    defaultEnabled: z.boolean(),
  })),
});
export type CaptureGetStreamConfigResponse = z.infer<typeof captureGetStreamConfigResponseSchema>;
```

**Stream defaults** (from product-features.md):

| Stream | Default |
|---|---|
| `window_tracking` | ON |
| `focus` | ON |
| `browser` | ON (during sessions) |
| `keystroke` | OFF |
| `clipboard` | OFF |

---

### `chat:*`

Chat messages route through main to the agents utility process. `sendMessage` triggers both an invoke/handle response (thread metadata) and a MessagePort stream (agent response chunks). See [Streaming Protocol](#streaming-protocol) for the full flow.

#### `chat:sendMessage`

Start a chat generation. Returns thread metadata immediately. Agent response streams via MessagePort (see `chat:streamPort`).

```ts
export const chatSendMessageRequestSchema = z.object({
  threadId: z.string().optional(),  // omit to create new thread
  content: z.string().min(1),
});
export type ChatSendMessageRequest = z.infer<typeof chatSendMessageRequestSchema>;

export const chatSendMessageResponseSchema = z.object({
  threadId: z.string(),
});
export type ChatSendMessageResponse = z.infer<typeof chatSendMessageResponseSchema>;
```

**What happens in the agents process** (Phase 1B — direct SQL, no embeddings):

1. Query recent capture rows from `activity_sessions`, `focus_sessions`, etc.
2. Inject activity context into system prompt
3. Call `agent.stream()` via Mastra
4. Stream response chunks over MessagePort

#### `chat:streamPort`

Not an invoke/handle channel. Main uses `webContents.postMessage('chat:streamPort', null, [port])` to transfer a MessagePort to the renderer. The renderer listens for this to receive streaming chunks. See [Streaming Protocol](#streaming-protocol).

#### `chat:getThreads`

List conversation threads with pagination.

```ts
export const chatGetThreadsRequestSchema = z.object({
  limit: z.number().int().positive().default(50),
  cursor: z.string().optional(),
});
export type ChatGetThreadsRequest = z.infer<typeof chatGetThreadsRequestSchema>;

export const chatGetThreadsResponseSchema = z.object({
  threads: z.array(threadSummarySchema),
  nextCursor: z.string().optional(),
});
export type ChatGetThreadsResponse = z.infer<typeof chatGetThreadsResponseSchema>;
```

#### `chat:getThread`

Fetch a single thread with all messages.

```ts
export const chatGetThreadRequestSchema = z.object({
  threadId: z.string(),
});
export type ChatGetThreadRequest = z.infer<typeof chatGetThreadRequestSchema>;

export const chatGetThreadResponseSchema = threadSchema;
export type ChatGetThreadResponse = z.infer<typeof chatGetThreadResponseSchema>;
```

---

### `context:*`

Context queries route through main to agents. In Phase 1B, all queries use direct SQL against capture tables. Phase 2 adds vector search via LibSQLVector.

#### `context:query`

Semantic search across activity data. Phase 1B falls back to keyword matching against recent rows.

```ts
export const contextQueryRequestSchema = z.object({
  query: z.string().min(1),
  limit: z.number().int().positive().default(10),
});
export type ContextQueryRequest = z.infer<typeof contextQueryRequestSchema>;

export const contextQueryResponseSchema = z.object({
  results: z.array(activityRecordSchema),
});
export type ContextQueryResponse = z.infer<typeof contextQueryResponseSchema>;
```

#### `context:getRecentActivity`

Direct SQL query against capture tables. Returns recent activity records.
Backed by capture tables in [database-schema.md](./database-schema.md): `activity_sessions` (primary), with optional merge/projection from `focus_sessions`, `keystroke_chunks`, `clipboard_events`, and `browser_sessions` as needed for unified timeline output.

```ts
export const contextGetRecentActivityRequestSchema = z.object({
  limit: z.number().int().positive().default(20),
  since: z.string().datetime().optional(),
});
export type ContextGetRecentActivityRequest = z.infer<typeof contextGetRecentActivityRequestSchema>;

export const contextGetRecentActivityResponseSchema = z.object({
  activities: z.array(activityRecordSchema),
});
export type ContextGetRecentActivityResponse = z.infer<typeof contextGetRecentActivityResponseSchema>;
```

#### `context:getContextCard`

Returns the current working context for the Context Card (always visible at top of SideSheet per design-system.md).

```ts
// Request: void (no payload)

export const contextGetContextCardResponseSchema = z.object({
  activeApp: z.string().optional(),
  activeDocument: z.string().optional(),
  activeUrl: z.string().url().optional(),
  sessionStart: z.string().datetime().optional(),
  recentApps: z.array(z.string()),
  suggestions: z.array(z.string()),
});
export type ContextGetContextCardResponse = z.infer<typeof contextGetContextCardResponseSchema>;
```

---

### `mcp:*`

MCP server management. All channels route through main to agents. Phase 1B supports stdio transport with API key auth only (OAuth deferred to Phase 3).

#### `mcp:connect`

Start an MCP server and persist its configuration. API keys encrypted via Electron `safeStorage` before storage.
Persistence mapping: config row in `mcp_servers` plus runtime health/state row in `mcp_connection_state` (see [database-schema.md](./database-schema.md)).

```ts
export const mcpConnectRequestSchema = z.object({
  serverId: z.string(),
  name: z.string(),
  transport: z.literal('stdio'),
  command: z.string(),
  args: z.array(z.string()).default([]),
  env: z.record(z.string()).default({}),
  apiKey: z.string().optional(),
});
export type MCPConnectRequest = z.infer<typeof mcpConnectRequestSchema>;

export const mcpConnectResponseSchema = z.object({
  serverId: z.string(),
  status: mcpConnectionStatusSchema,
  toolCount: z.number().int().nonnegative(),
});
export type MCPConnectResponse = z.infer<typeof mcpConnectResponseSchema>;
```

#### `mcp:disconnect`

Stop an MCP server and update its persisted status.

```ts
export const mcpDisconnectRequestSchema = z.object({
  serverId: z.string(),
});
export type MCPDisconnectRequest = z.infer<typeof mcpDisconnectRequestSchema>;

export const mcpDisconnectResponseSchema = z.object({
  success: z.boolean(),
});
export type MCPDisconnectResponse = z.infer<typeof mcpDisconnectResponseSchema>;
```

#### `mcp:listConnections`

List all configured MCP servers with their current status.

```ts
// Request: void (no payload)

export const mcpListConnectionsResponseSchema = z.object({
  connections: z.array(mcpConnectionSchema),
});
export type MCPListConnectionsResponse = z.infer<typeof mcpListConnectionsResponseSchema>;
```

#### `mcp:listTools`

List tools from connected MCP servers. Omit `serverId` to list tools from all servers.

```ts
export const mcpListToolsRequestSchema = z.object({
  serverId: z.string().optional(),
});
export type MCPListToolsRequest = z.infer<typeof mcpListToolsRequestSchema>;

export const mcpListToolsResponseSchema = z.object({
  tools: z.array(mcpToolSchema),
});
export type MCPListToolsResponse = z.infer<typeof mcpListToolsResponseSchema>;
```

#### `mcp:getStatus`

Health status for MCP servers. Omit `serverId` for all servers.

```ts
export const mcpGetStatusRequestSchema = z.object({
  serverId: z.string().optional(),
});
export type MCPGetStatusRequest = z.infer<typeof mcpGetStatusRequestSchema>;

export const mcpGetStatusResponseSchema = z.object({
  servers: z.array(z.object({
    serverId: z.string(),
    status: mcpConnectionStatusSchema,
    error: z.string().optional(),
    lastHealthCheck: z.string().datetime().optional(),
  })),
});
export type MCPGetStatusResponse = z.infer<typeof mcpGetStatusResponseSchema>;
```

---

### `apps:*`

App lifecycle management. Called by the main renderer (Apps view) only. Sandboxed apps use the read lane (`cowork:read`) — see [App Preload SDK](#app-preload-sdk).

#### `apps:install`

Install an app from a zip file. Main handles archive extraction and hardening (zip-slip, symlink, zip-bomb checks). Agents handles manifest parsing and metadata persistence.

```ts
export const appsInstallRequestSchema = z.object({
  zipPath: z.string(),
});
export type AppsInstallRequest = z.infer<typeof appsInstallRequestSchema>;

export const appsInstallResponseSchema = z.object({
  appId: z.string(),
  name: z.string(),
  version: z.string(),
  permissions: appPermissionsSchema,
});
export type AppsInstallResponse = z.infer<typeof appsInstallResponseSchema>;
```

#### `apps:list`

List all installed apps.

```ts
// Request: void (no payload)

export const appsListResponseSchema = z.object({
  apps: z.array(installedAppSchema),
});
export type AppsListResponse = z.infer<typeof appsListResponseSchema>;
```

#### `apps:remove`

Uninstall an app. Removes files, metadata, and permissions.

```ts
export const appsRemoveRequestSchema = z.object({
  appId: z.string(),
});
export type AppsRemoveRequest = z.infer<typeof appsRemoveRequestSchema>;

export const appsRemoveResponseSchema = z.object({
  success: z.boolean(),
});
export type AppsRemoveResponse = z.infer<typeof appsRemoveResponseSchema>;
```

#### `apps:callTool` (Deferred)

Execute an MCP tool on behalf of an app. **Deferred from Phase 1B app SDK.** Schema retained for potential future use. Apps use the read lane (`cowork:read`) for data access in Phase 1B. See [system-architecture.md — Context Pipeline](./system-architecture.md#context-pipeline).

```ts
export const appsCallToolRequestSchema = z.object({
  appId: z.string(),
  toolName: z.string(),
  args: z.record(z.unknown()).default({}),
});
export type AppsCallToolRequest = z.infer<typeof appsCallToolRequestSchema>;

export const appsCallToolResponseSchema = z.object({
  result: z.unknown(),
});
export type AppsCallToolResponse = z.infer<typeof appsCallToolResponseSchema>;
```

#### `apps:listTools` (Deferred)

List MCP tools available to a specific app. **Deferred from Phase 1B app SDK.** Apps use the read lane for data access — they don't need tool discovery.

```ts
export const appsListToolsRequestSchema = z.object({
  appId: z.string(),
});
export type AppsListToolsRequest = z.infer<typeof appsListToolsRequestSchema>;

export const appsListToolsResponseSchema = z.object({
  tools: z.array(mcpToolSchema),
});
export type AppsListToolsResponse = z.infer<typeof appsListToolsResponseSchema>;
```

#### `apps:getAppConfig` (Deferred)

Return app metadata and granted permissions. **Deferred from Phase 1B app SDK.** App identity is injected by the preload — apps don't need to query their own config via IPC in the read lane model.

```ts
export const appsGetAppConfigRequestSchema = z.object({
  appId: z.string(),
});
export type AppsGetAppConfigRequest = z.infer<typeof appsGetAppConfigRequestSchema>;

export const appsGetAppConfigResponseSchema = z.object({
  appId: z.string(),
  name: z.string(),
  version: z.string(),
  permissions: appPermissionsSchema,
  connectedServices: z.array(z.string()),
});
export type AppsGetAppConfigResponse = z.infer<typeof appsGetAppConfigResponseSchema>;
```

---

### `app:*`

Application-level settings. Handled directly by main — no relay to utility processes.

#### `app:getSettings`

Return current application settings.

```ts
// Request: void (no payload)

export const appGetSettingsResponseSchema = z.object({
  theme: themeSchema,
  captureStreams: z.record(captureStreamSchema, z.boolean()),
});
export type AppGetSettingsResponse = z.infer<typeof appGetSettingsResponseSchema>;
```

#### `app:setTheme`

Change the application theme.

```ts
export const appSetThemeRequestSchema = z.object({
  theme: themeSchema,
});
export type AppSetThemeRequest = z.infer<typeof appSetThemeRequestSchema>;

export const appSetThemeResponseSchema = z.object({
  theme: themeSchema,
});
export type AppSetThemeResponse = z.infer<typeof appSetThemeResponseSchema>;
```

---

### `system:*`

Health monitoring and service recovery.

#### `system:health`

**Event push** (not invoke/handle). Main sends health events to the renderer on any service state transition. No polling.

```ts
export const systemHealthEventSchema = z.object({
  service: serviceNameSchema,
  state: serviceStateSchema,
  message: z.string().optional(),
});
export type SystemHealthEvent = z.infer<typeof systemHealthEventSchema>;
```

Renderer listens via:
```ts
window.cowork.system.onHealth((event: SystemHealthEvent) => { ... });
```

Preload wires this as:
```ts
// preload.ts
onHealth: (callback: (event: SystemHealthEvent) => void) => {
  ipcRenderer.on('system:health', (_event, payload) => {
    callback(systemHealthEventSchema.parse(payload));
  });
}
```

#### `system:retry`

Manually retry a failed or degraded service.

```ts
export const systemRetryRequestSchema = z.object({
  service: serviceNameSchema,
});
export type SystemRetryRequest = z.infer<typeof systemRetryRequestSchema>;

export const systemRetryResponseSchema = z.object({
  success: z.boolean(),
  newState: serviceStateSchema,
});
export type SystemRetryResponse = z.infer<typeof systemRetryResponseSchema>;
```

---

### `cowork:*`

App read lane. Single channel used by sandboxed app preloads to access platform data. All requests are read-only and default-allowed (no permission grants required). Routes through main to agents utility.

For the full protocol design, rationale, and SDK surface definition, see [system-architecture.md — Context Pipeline](./system-architecture.md#context-pipeline) and [Apps Runtime](./system-architecture.md#apps-runtime).

#### `cowork:read`

Generic read dispatch channel. The app preload maps typed SDK methods to this channel with namespace + method routing.

```ts
export const coworkReadNamespaceSchema = z.enum(['context', 'user', 'data']);
export type CoworkReadNamespace = z.infer<typeof coworkReadNamespaceSchema>;

export const coworkReadRequestSchema = z.object({
  appId: z.string(),
  ns: coworkReadNamespaceSchema,
  method: z.string(),
  args: z.record(z.unknown()).default({}),
});
export type CoworkReadRequest = z.infer<typeof coworkReadRequestSchema>;

export const coworkReadResponseSchema = z.object({
  data: z.unknown(),
});
export type CoworkReadResponse = z.infer<typeof coworkReadResponseSchema>;
```

**Why generic `z.unknown()` response:** Each method returns a different shape. The dispatch layer in agents utility validates per-method responses internally. The IPC envelope is intentionally loose — type safety for individual methods comes from the TypeScript SDK types that the app preload exposes, not from the IPC channel schema.

**Supported methods (Phase 1B):**

| Namespace | Method | Returns | Backing data |
|---|---|---|---|
| `context` | `activeWindow` | Current focused window/app info | Latest `activity_sessions` row |
| `context` | `recentActivity` | Recent activity summary | `activity_sessions` + merge tables |
| `context` | `currentTime` | Local time + timezone | System clock |
| `user` | `preferences` | User settings | `app:getSettings` equivalent |
| `user` | `profile` | User name, role | User profile store |
| `data` | `conversations` | This app's conversation history | Agent thread storage, scoped by `appId` |
| `data` | `search` | Semantic search over activity | `context:query` equivalent |

**args for methods that accept options:**

```ts
// context.recentActivity
{ limit?: number; since?: string }  // defaults: limit=20, since=undefined

// data.conversations
{ limit?: number; cursor?: string }  // defaults: limit=50, cursor=undefined

// data.search
{ query: string; limit?: number }    // defaults: limit=10
```

---

### `browser:*` (Phase 4)

Browser automation via Playwright. Channels defined but not implemented in Phase 1B.

| Channel | Purpose |
|---|---|
| `browser:execute` | Run a browser automation script |
| `browser:getState` | Get current page URL, title, screenshot |
| `browser:stop` | Stop an active browser session |
| `browser:getExecutionLog` | Retrieve execution step log for a session |

Schemas deferred to Phase 4 sprint plan.

---

### `automation:*` (Phase 4)

Automation engine. Channels defined but not implemented in Phase 1B.

| Channel | Purpose |
|---|---|
| `automation:create` | Create an automation rule (trigger + actions) |
| `automation:update` | Update an existing automation |
| `automation:delete` | Delete an automation |
| `automation:trigger` | Manually trigger an automation |
| `automation:getRunLog` | Get execution log for an automation |

Schemas deferred to Phase 4 sprint plan.

---

## Streaming Protocol

Chat response streaming uses MessagePort transfer with an ACK gate. This pattern was proven in the Sprint 0 Mastra spike.

### Flow

```
Renderer                       Main                         Agents Utility
   |                            |                               |
   |  invoke('chat:sendMessage') |                               |
   |--------------------------->|                               |
   |                            |  postMessage({stream, threadId, content}, [port1])
   |                            |------------------------------->|
   |                            |                               |
   |  postMessage('chat:streamPort', null, [port2])             |
   |<---------------------------|                               |
   |                            |                               |
   |  port2.postMessage('ACK')  |                               |
   |---------(via port)---------|---------(via port)----------->|
   |                            |                               |
   |                            |         agent.stream() begins |
   |                            |                               |
   |  port2.onmessage = chunk   |   port1.postMessage(chunk)    |
   |<--------(via port)---------|----------(via port)-----------|
   |  ... more chunks ...       |                               |
   |                            |                               |
   |  port2.onmessage = finish  |   port1.postMessage(finish)   |
   |<--------(via port)---------|----------(via port)-----------|
   |                            |                               |
   |  invoke returns { threadId }|                               |
   |<---------------------------|                               |
```

### Steps

1. **Renderer** calls `window.cowork.chat.sendMessage({ threadId?, content })` → `ipcRenderer.invoke('chat:sendMessage', payload)`
2. **Main** creates a `MessageChannelMain` pair `[port1, port2]`
3. **Main** sends `port1` to agents utility: `agentsUtility.postMessage({ channel: 'chat:stream', threadId, content }, [port1])`
4. **Main** sends `port2` to renderer: `mainWindow.webContents.postMessage('chat:streamPort', null, [port2])`
5. **Main** returns `{ threadId }` to the invoke caller
6. **Renderer** receives `port2` via `ipcRenderer.on('chat:streamPort')`, sends ACK: `port2.postMessage('ACK')`
7. **Agents** receives ACK on `port1`, begins `agent.stream()` execution
8. **Agents** sends chunks: `port1.postMessage(chunk)` — each chunk is a string in SSE format
9. **Renderer** receives chunks on `port2.onmessage`, feeds to `IpcChatTransport` → `useChat()`
10. **Agents** sends finish chunk, closes port

### Chunk format

Chunks over MessagePort are SSE-formatted strings, matching the AI SDK Data Stream Protocol:

```
data: {"type":"text-delta","textDelta":"Hello"}\n\n
data: {"type":"text-delta","textDelta":" world"}\n\n
data: {"type":"tool-call","toolCallId":"tc_1","toolName":"get_weather","args":{"city":"SF"}}\n\n
data: {"type":"tool-result","toolCallId":"tc_1","result":{"temp":65}}\n\n
data: {"type":"finish","usage":{"promptTokens":150,"completionTokens":42}}\n\n
```

Validation schema for parsed chunk objects (after stripping `data: ` prefix and parsing JSON):

```ts
export const streamChunkSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('text-delta'),
    textDelta: z.string(),
  }),
  z.object({
    type: z.literal('tool-call'),
    toolCallId: z.string(),
    toolName: z.string(),
    args: z.record(z.unknown()),
  }),
  z.object({
    type: z.literal('tool-result'),
    toolCallId: z.string(),
    result: z.unknown(),
  }),
  z.object({
    type: z.literal('error'),
    error: z.string(),
  }),
  z.object({
    type: z.literal('finish'),
    usage: z.object({
      promptTokens: z.number().int().nonnegative(),
      completionTokens: z.number().int().nonnegative(),
    }).optional(),
  }),
]);
export type StreamChunk = z.infer<typeof streamChunkSchema>;
```

### Error recovery

If the agents utility crashes mid-stream, main detects the broken MessagePort and synthesizes an error chunk to the renderer:

```ts
port2.postMessage('data: {"type":"error","error":"Agents process crashed. Retrying..."}\n\n');
```

The renderer shows the error in the chat UI. If auto-restart succeeds, the user can resend.

### ACK gate

The ACK gate prevents the agents process from streaming before the renderer has set up its port listener. Without it, early chunks would be lost.

- Agents **must not** call `agent.stream()` until it receives `'ACK'` on `port1`
- If ACK doesn't arrive within 5s, agents closes the port and logs an error
- Renderer sends ACK immediately after attaching `port2.onmessage`

---

## App Preload SDK

Sandboxed apps get a typed, read-only `window.cowork.*` API via `app-preload.ts`. This preload maps SDK methods to the `cowork:read` IPC channel with automatic `appId` injection.

For the full protocol rationale (why typed SDK, why not `callTool`, why not MCP), see [system-architecture.md — Why Two Separate Lanes](./system-architecture.md#why-two-separate-lanes).

```ts
// app-preload.ts (conceptual — not full implementation)
const APP_ID = getAppIdFromPartition(); // derived from session partition

// Helper: all read lane calls go through one IPC channel
const read = (ns: string, method: string, args: Record<string, unknown> = {}) =>
  ipcRenderer.invoke('cowork:read', { appId: APP_ID, ns, method, args });

contextBridge.exposeInMainWorld('cowork', {
  context: {
    activeWindow: () => read('context', 'activeWindow'),
    recentActivity: (opts?: { limit?: number; since?: string }) =>
      read('context', 'recentActivity', opts ?? {}),
    currentTime: () => read('context', 'currentTime'),
  },
  user: {
    preferences: () => read('user', 'preferences'),
    profile: () => read('user', 'profile'),
  },
  data: {
    conversations: (opts?: { limit?: number; cursor?: string }) =>
      read('data', 'conversations', opts ?? {}),
    search: (query: string, opts?: { limit?: number }) =>
      read('data', 'search', { query, ...opts }),
  },
});
```

Key differences from previous design:

- **No permission checks in preload.** All read lane methods are default-allowed. No grants table, no denial logic, no refresh signaling.
- **No `callTool` / `listTools` / `getAppConfig`.** Apps don't call tools or discover tools. They read data through typed methods.
- **One IPC channel.** Every SDK method maps to `cowork:read` with `{ ns, method, args }`. Main relay is a single generic handler.
- **`appId` injected by preload.** Apps cannot spoof their identity — the preload derives `appId` from the session partition, same as before.

---

## Implementation Design

### `src/shared/ipc-channels.ts`

Single source of truth for all channel name strings. Imported by all four processes.

```ts
export const IpcChannels = {
  capture: {
    getStatus: 'capture:getStatus',
    toggleStream: 'capture:toggleStream',
    getStreamConfig: 'capture:getStreamConfig',
  },
  chat: {
    sendMessage: 'chat:sendMessage',
    streamPort: 'chat:streamPort',
    getThreads: 'chat:getThreads',
    getThread: 'chat:getThread',
  },
  context: {
    query: 'context:query',
    getRecentActivity: 'context:getRecentActivity',
    getContextCard: 'context:getContextCard',
  },
  mcp: {
    connect: 'mcp:connect',
    disconnect: 'mcp:disconnect',
    listConnections: 'mcp:listConnections',
    listTools: 'mcp:listTools',
    getStatus: 'mcp:getStatus',
  },
  apps: {
    install: 'apps:install',
    list: 'apps:list',
    remove: 'apps:remove',
    callTool: 'apps:callTool',       // deferred from Phase 1B app SDK
    listTools: 'apps:listTools',     // deferred from Phase 1B app SDK
    getAppConfig: 'apps:getAppConfig', // deferred from Phase 1B app SDK
  },
  cowork: {
    read: 'cowork:read',  // app read lane — all app SDK calls route here
  },
  app: {
    getSettings: 'app:getSettings',
    setTheme: 'app:setTheme',
  },
  system: {
    health: 'system:health',
    retry: 'system:retry',
  },
  browser: {
    execute: 'browser:execute',
    getState: 'browser:getState',
    stop: 'browser:stop',
    getExecutionLog: 'browser:getExecutionLog',
  },
  automation: {
    create: 'automation:create',
    update: 'automation:update',
    delete: 'automation:delete',
    trigger: 'automation:trigger',
    getRunLog: 'automation:getRunLog',
  },
} as const;

export type IpcChannel = typeof IpcChannels;
```

### `src/shared/ipc-schemas.ts`

Contains all Zod schemas and inferred TypeScript types from the [Shared Types](#shared-types) and [Channel Reference](#channel-reference) sections above. One file, flat exports. Split into sub-modules only if it exceeds ~500 lines.

### Type-safe IPC helpers

The channel-to-schema mapping enables type-safe invoke/handle wrappers:

```ts
// Type map: channel string → { request, response }
type IpcHandlerMap = {
  'capture:getStatus': { request: void; response: CaptureGetStatusResponse };
  'capture:toggleStream': { request: CaptureToggleStreamRequest; response: CaptureToggleStreamResponse };
  'capture:getStreamConfig': { request: void; response: CaptureGetStreamConfigResponse };
  'chat:sendMessage': { request: ChatSendMessageRequest; response: ChatSendMessageResponse };
  'chat:getThreads': { request: ChatGetThreadsRequest; response: ChatGetThreadsResponse };
  'chat:getThread': { request: ChatGetThreadRequest; response: ChatGetThreadResponse };
  'context:query': { request: ContextQueryRequest; response: ContextQueryResponse };
  'context:getRecentActivity': { request: ContextGetRecentActivityRequest; response: ContextGetRecentActivityResponse };
  'context:getContextCard': { request: void; response: ContextGetContextCardResponse };
  'mcp:connect': { request: MCPConnectRequest; response: MCPConnectResponse };
  'mcp:disconnect': { request: MCPDisconnectRequest; response: MCPDisconnectResponse };
  'mcp:listConnections': { request: void; response: MCPListConnectionsResponse };
  'mcp:listTools': { request: MCPListToolsRequest; response: MCPListToolsResponse };
  'mcp:getStatus': { request: MCPGetStatusRequest; response: MCPGetStatusResponse };
  'apps:install': { request: AppsInstallRequest; response: AppsInstallResponse };
  'apps:list': { request: void; response: AppsListResponse };
  'apps:remove': { request: AppsRemoveRequest; response: AppsRemoveResponse };
  // apps:callTool, apps:listTools, apps:getAppConfig — deferred from Phase 1B app SDK
  'cowork:read': { request: CoworkReadRequest; response: CoworkReadResponse };
  'app:getSettings': { request: void; response: AppGetSettingsResponse };
  'app:setTheme': { request: AppSetThemeRequest; response: AppSetThemeResponse };
  'system:retry': { request: SystemRetryRequest; response: SystemRetryResponse };
};

// Type-safe invoke (renderer side)
async function invoke<C extends keyof IpcHandlerMap>(
  channel: C,
  ...args: IpcHandlerMap[C]['request'] extends void ? [] : [IpcHandlerMap[C]['request']]
): Promise<IpcHandlerMap[C]['response']> {
  return ipcRenderer.invoke(channel, ...args);
}

// Type-safe handle (main/utility side)
function handle<C extends keyof IpcHandlerMap>(
  channel: C,
  handler: (
    event: Electron.IpcMainInvokeEvent,
    ...args: IpcHandlerMap[C]['request'] extends void ? [] : [IpcHandlerMap[C]['request']]
  ) => Promise<IpcHandlerMap[C]['response']>
): void {
  ipcMain.handle(channel, handler as any);
}
```

These helpers are implemented in Sprint 3 (folder restructure) or Sprint 4/5 (when first handlers are wired). The type map is the contract — handlers and callers must conform.

---

## Changelog

**v6 (Feb 17, 2026):** App read lane integration. Added `cowork:read` channel with namespace dispatch schema and per-method reference table. Deferred `apps:callTool`, `apps:listTools`, `apps:getAppConfig` from Phase 1B app SDK — schemas retained for future use but not exposed to sandboxed apps. Rewrote App Preload SDK section: replaced `callTool`/permission-gate model with typed read-only SDK (`window.cowork.{context,user,data}.*`) mapping to single `cowork:read` IPC channel. Updated channel inventory, design notes (dual-caller → main-renderer-only), `IpcChannels` constant, and `IpcHandlerMap`. Aligns with agent-context-pipeline.md v7 (now consolidated into system-architecture.md).

**v5 (Feb 17, 2026):** App preload permission wording alignment. Clarified that preload grant checks use projected in-memory grants hydrated from `app_permissions`, and added explicit runtime requirement to refresh grants on permission updates before subsequent tool calls (reload/recycle or equivalent signaling).

**v4 (Feb 17, 2026):** Finalized Sprint 2 draft contract for implementation sync. Added explicit alignment with [database-schema.md](./database-schema.md) for context reads, MCP persistence (`mcp_servers` + `mcp_connection_state`), and app permission enforcement (`app_permissions`). Replaced "Open questions in this draft" with locked Phase 1B decisions.

**v3 (Feb 17, 2026):** Apps permission-model alignment. Removed `apps:chat` channel and all direct app `chat()` SDK references to match "Apps get tools, not agents." App-side reasoning now flows through `apps:callTool` with a platform tool (e.g., `platform_chat`). Updated channel inventory, apps dual-caller rationale, app preload example, `IpcChannels.apps` map, handler type map, and `appPermissionsSchema` (`agent` → `captureRead`).

**v2 (Feb 17, 2026):** Added Design Notes section — documents sources (sprint plan, system-architecture, product-features, design-system), rationale for schema shapes (streaming split, optional threadId, flat activityRecord, dual-caller apps channels, inline apiKey, push health), why Zod at IPC boundaries, what's deferred, and 3 open questions.

**v1 (Feb 17, 2026):** Initial IPC contract. 35 channels across 9 namespaces. Full Zod schemas for all Phase 1B channels (27 channels). Phase 4 channels (browser:*, automation:*) listed as stubs. Streaming protocol documented with MessagePort flow, ACK gate, chunk format, and error recovery. App preload SDK documented. Type-safe IPC helper design included.
