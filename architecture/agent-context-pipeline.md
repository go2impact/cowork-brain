# Context Pipeline: Communication Architecture for Agent and Apps

| | |
|---|---|
| **Status** | Draft |
| **Last Updated** | 2026-02-17 |
| **Owner** | Rustan |
| **Related** | [system-architecture.md](./system-architecture.md), [ipc-contract.md](./ipc-contract.md), [database-schema.md](./database-schema.md) |

---

## Why This Doc Exists

This doc defines **how context and data move across processes at runtime** for:

1. User chat with the Cowork agent (agent path)
2. Third-party app data access through `window.cowork.*` (read lane)

This is a communication and boundary spec, not a schema spec.

---

## Scope

**In scope:**

- Process boundaries and trust boundaries
- IPC channels and call paths
- Layer 1 vs Layer 2 runtime behavior (agent path)
- App SDK surface and read lane protocol
- Phase evolution of communication behavior

**Out of scope (covered elsewhere):**

- Table/column definitions and SQL details: [database-schema.md](./database-schema.md)
- Channel payload schemas and handler typings: [ipc-contract.md](./ipc-contract.md)
- App runtime option analysis: [apps-runtime-architecture.md](./apps-runtime-architecture.md)

---

## Architecture Overview (TL;DR)

```
┌──────────────────────────────────────────────────────────────────────┐
│ RENDERERS                                                            │
│                                                                      │
│ Main UI (React)                     Third-party app (WebContentsView)│
│ chat:sendMessage                    window.cowork.context.*          │
│ context:*, mcp:*, apps:*            window.cowork.user.*             │
│        │                            window.cowork.data.*             │
│        │                                      │                      │
│        │          IPC                         │                      │
│        │      (chat:*, context:*,             │                      │
│        │       mcp:*, apps:*)            (cowork:read)               │
│        └────────────────────┬────────────────┘                      │
└─────────────────────────────┼───────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ MAIN PROCESS                                                         │
│ Relay only. No LLM execution. No DB querying. No business logic.     │
└─────────────────────────────┬───────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ AGENTS & RAG UTILITY PROCESS                                         │
│ - Handles `chat:sendMessage` (user chat path)                        │
│ - Handles `cowork:read` (app read lane)                              │
│ - Runs Layer 1 (automatic injection) + Layer 2 (tool execution)      │
│ - Accesses cowork.db through agents-owned data access layer          │
└─────────────────────────────┬───────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ CAPTURE UTILITY PROCESS                                               │
│ Writes capture streams to cowork.db.                                  │
│ Agents process reads capture context; apps never access DB directly.  │
└──────────────────────────────────────────────────────────────────────┘
```

**Core pattern:** two distinct lanes. User chat goes through the full agent runtime (Layer 1 + Layer 2). App data access goes through a lightweight read lane — no agent reasoning, no tool execution, just data retrieval.

---

## Trust and Access Boundaries

| Actor | Can access | Cannot access |
|---|---|---|
| Main UI renderer | `chat:*`, `context:*`, `mcp:*`, `apps:*` (management), UI-facing namespaces | Direct DB, direct LLM provider APIs |
| App renderer (`WebContentsView`) | `window.cowork.{context,user,data}.*` (read lane only) | Node/Electron APIs, raw MCP, direct DB, agent internals, write operations, tool execution |
| Main process | IPC relay + process coordination | Business logic, agent decisions, DB queries |
| Agents utility | Agent runtime, tool execution, MCP mediation, read lane dispatch | Capture stream writes |
| Capture utility | Capture stream ingestion/writes | Agent reasoning/orchestration |

---

## The Two-Layer Pattern (Agent Runtime)

The two-layer pattern applies to the **agent path only** (user chat via `chat:sendMessage`). Apps do not participate in Layer 1 or Layer 2 — they have their own read lane (see [App Read Lane](#app-read-lane-protocol)).

### Layer 1: Automatic Context Injection

- Trigger: every `chat:sendMessage` request
- Location: agents utility request path before model invocation
- Responsibility: build a lightweight runtime context snapshot and attach it to agent instructions
- Property: deterministic application behavior (not model-decided)

### Layer 2: Tool Execution

- Trigger: model emits a tool call during agent reasoning
- Location: agents utility tool execution layer
- Responsibility: execute platform and MCP tools and return structured results
- Property: on-demand, driven by agent reasoning only

**Important:** Layer 1 and Layer 2 are complementary, not alternatives. Layer 1 gives the agent awareness of what's happening. Layer 2 gives it the ability to take action.

---

## Communication Flows

### Flow A: User Chat with Cowork Agent

```
Renderer (Main UI)
  → ipcRenderer.invoke('chat:sendMessage', payload)
  → Main process relay
  → Agents utility:
       1) Apply Layer 1 context injection
       2) Run agent invocation
       3) Agent may invoke Layer 2 tools
       4) Stream response chunks back
  → Renderer receives streamed output
```

Transport notes:

- `chat:sendMessage` returns immediately with metadata; response content streams separately via `chat:streamPort` MessagePort.
- Chat response transport uses MessagePort streaming for token/chunk output.
- ACK-gate behavior and stream protocol are defined in [ipc-contract.md](./ipc-contract.md).

### Flow B: Third-Party App Reads Platform Data (Read Lane)

```
App JS
  → window.cowork.context.activeWindow()
  → App preload maps to ipcRenderer.invoke('cowork:read', {
      appId: APP_ID_FROM_PARTITION,
      ns: 'context',
      method: 'activeWindow',
      args: {}
    })
  → Main process relay
  → Agents utility dispatches to read handler
  → Agents utility queries cowork.db
  → Structured result returns through same IPC path
  → App receives resolved promise
```

Key properties:

- **No per-method permission gate.** Read lane methods are default-allowed in Phase 1B. No runtime grant checks or preload denials for read calls.
- **No agent reasoning.** Read lane requests go straight to data access — no LLM invocation, no context injection, no tool execution.
- **One IPC channel.** All read lane calls use `cowork:read` with namespace + method dispatch. Main process relay is trivial.
- **Typed SDK.** App developers interact with a structured `window.cowork.*` object with TypeScript types, not raw tool names.
- **Trusted app identity.** `appId` is injected by preload from the app's session partition. App JS cannot set or spoof it.

---

## App Read Lane Protocol

### Why a read lane (not tools, not agent-mediated)

We evaluated three alternatives for how third-party apps access platform data. The read lane was chosen for specific reasons:

**Alternative 1: App → Agent → Tools (agent-mediated)**
Apps describe intent in natural language, agent decides which tools to call. Rejected because third-party apps are highly varied and unpredictable — agent-mediated responses would be non-deterministic, slow (LLM round-trip per call), and expensive (tokens for every data fetch). Apps built through AI Studio templates need predictable, fast data access — not another layer of AI reasoning.

**Alternative 2: App → callTool(name, args) (direct tool calls)**
Apps call platform/MCP tools by name. Rejected because it creates tight coupling between apps and the internal tool surface. Tool names/schemas change → apps break. Apps would need documentation for every tool. It also conflates "reading data" with "taking actions" — most app needs are read-only, and gating every read behind a permission check adds friction without security benefit.

**Alternative 3: App → MCP Server (standard MCP protocol)**
Apps connect to a Cowork-hosted MCP server. Rejected because MCP is designed for LLM-to-service communication, not app-to-platform data access. It adds protocol overhead (JSON-RPC framing, capability negotiation) for what are simple data lookups. App developers would need to learn MCP to build a Cowork app.

**Chosen: App → Typed SDK → Read Lane**
Apps call typed methods on `window.cowork.*`. The preload maps these to a single `cowork:read` IPC channel. The agents utility dispatches to data access handlers that query `cowork.db` and return structured results. This gives apps a normal JavaScript API — discoverable via TypeScript types, fast (no LLM), predictable (deterministic queries), and frictionless (no permission grants for reads).

### SDK Surface

The app preload exposes a typed `window.cowork` object with namespaced methods. All methods return promises. All are read-only.

```
window.cowork
  │
  ├── .context
  │     .activeWindow()         → current focused window/app info
  │     .recentActivity(opts?)  → recent activity summary
  │     .currentTime()          → local time + timezone
  │
  ├── .user
  │     .preferences()          → user settings/preferences
  │     .profile()              → user name, role, etc.
  │
  └── .data
        .conversations(opts?)   → this app's own conversation history
        .search(query)          → semantic search over activity
```

This is the Phase 1B surface. New namespaces and methods can be added without changing the protocol — just add handlers in agents utility and types in the SDK package.

### IPC Plumbing

```
App JS                              Preload                         Main          Agents Utility
  │                                    │                              │                │
  │  window.cowork.context             │                              │                │
  │    .activeWindow()                 │                              │                │
  │                                    │                              │                │
  │  ─── JS call ──────────────────▶   │                              │                │
  │                                    │  ipcRenderer.invoke(         │                │
  │                                    │    'cowork:read',            │                │
  │                                    │    { appId: APP_ID,          │                │
  │                                    │      ns: 'context',          │                │
  │                                    │      method:                 │                │
  │                                    │        'activeWindow',       │                │
  │                                    │      args: {} })             │                │
  │                                    │                              │                │
  │                                    │  ─── IPC invoke ──────────▶  │                │
  │                                    │                              │  relay ──────▶ │
  │                                    │                              │                │
  │                                    │                              │                │ dispatch:
  │                                    │                              │                │   ns='context'
  │                                    │                              │                │   method=
  │                                    │                              │                │   'activeWindow'
  │                                    │                              │                │
  │                                    │                              │                │ query cowork.db
  │                                    │                              │                │
  │                                    │                              │  ◀── result ── │
  │                                    │  ◀── result ─────────────── │                │
  │  ◀── resolved promise ─────────   │                              │                │
```

Implementation notes:

- **One IPC channel:** `cowork:read` handles all app read requests. The `ns` and `method` fields route to the correct handler inside agents utility.
- **Preload is thin:** Each SDK method is a one-liner that maps to `ipcRenderer.invoke('cowork:read', { appId, ns, method, args })`.
- **No permission check in preload:** Read lane is default-allowed. The preload does not inspect grants or reject calls.
- **Main process relay is generic:** `ipcMain.handle('cowork:read', (e, req) => agentsUtility.send(req))` — no per-method logic.
- **Identity is preload-owned:** App JS never passes `appId`; preload derives it from `partition: persist:app-{appId}` and injects it into every request.

### What apps cannot do (and why)

| Capability | Available? | Rationale |
|---|---|---|
| Read platform context/data | Yes (read lane) | Core app use case. Read-only and bounded to the disclosed data envelope in Phase 1B. |
| Execute MCP tools (Zendesk, Slack, Gmail) | No | Write/mutation actions are out of scope for apps in Phase 1B |
| Invoke agent reasoning | No | Non-deterministic, expensive, hard to predict across varied apps |
| Write to database | No | Apps are sandboxed readers — all writes go through platform processes |
| Access other apps' data | No | Each app's `conversations()` is scoped to its own `appId` |

---

## App Access Model

### Read Lane Only (Default Access)

All read lane methods are available to every installed app with no permission grants required.

**Rationale:** The read lane exposes the same context data that the agent's Layer 1 injection uses — activity summaries, current window, user preferences. While this data is sensitive, the access path is read-only and bounded to a fixed SDK envelope in Phase 1B. Gating every method call behind prompts adds friction and permission fatigue. Risk is controlled with explicit install-time disclosure, sandbox/process isolation, trusted preload `appId` injection, and strict per-app scoping for app-owned data.

### Risk Model and Install Disclosure

Default-allowed read access is **not** "zero risk." It is a bounded-risk model with explicit controls.

- **Data classes exposed in Phase 1B:** current window/activity summary, user preferences/profile, app-scoped conversation history, semantic search results.
- **Primary risk:** sensitive context leakage to a malicious or careless app.
- **Primary mitigations:** read-only surface (no mutation path), no Node/Electron API access, no direct DB access, trusted `appId` injection in preload, and app-scoped reads for app-owned data.
- **Required UX control:** every app install must show a clear disclosure of the read-lane data envelope before the app is enabled.

### Why not gate-everything (previous design)

The v6 design gated every `callTool()` call through a preload permission check against `app_permissions` in the database. This was rejected because:

1. **Most app needs are read-only.** Permission prompts for reading the current window or recent activity create friction with no security payoff.
2. **Permission fatigue.** Users clicking "allow" on every data read trains them to approve without thinking — undermining the security model for the grants that actually matter.
3. **Developer friction.** App developers building via AI Studio templates need predictable behavior. A permission-gated read that might be denied makes app behavior non-deterministic.
4. **Simpler implementation.** No grants table hydration, no preload denial logic, no permission refresh signaling for read-only access.

---

## Tool Exposure Model

The agent and apps have **separate access paths** with different capabilities:

| Path | Entry point | Can do | Cannot do |
|---|---|---|---|
| **Agent (Layer 2)** | Model emits tool call during `chat:sendMessage` | Execute platform tools, execute MCP tools, query DB, call external services | N/A (full access within agents utility) |
| **App (read lane)** | `window.cowork.{context,user,data}.*` | Read platform context, read user preferences, search activity | Execute tools, invoke agent, write data, call MCP services |

**Policy: apps get data, not tools.**

Previous design ("apps get tools, not agents") gave apps direct tool execution via `callTool()`. The updated policy is stricter: apps get **read-only data access** through a typed SDK. Tool execution is reserved for the agent. App write/action capabilities are out of scope for Phase 1B — if needed later, the design will be driven by concrete app use cases, not speculative architecture.

---

## Data Ownership (Communication View)

- Capture utility owns writes to capture streams.
- Agents utility reads capture context, manages agent-side persistence, and serves app read lane requests.
- App renderers read data through `window.cowork.*` (read lane) — never read/write database directly.
- Main process coordinates transport only.

For table ownership and retention details, use [database-schema.md](./database-schema.md).

---

## Phase Evolution (Communication Surface)

| Phase | Chat path (agent) | App path (read lane) | Key behavior |
|---|---|---|---|
| **Phase 1B** | `chat:sendMessage` with streamed response | `cowork:read` request/response (typed SDK) | Core relay topology, two-layer agent runtime, read lane established |
| **Phase 2** | Same channels; richer retrieval inside agents utility | Same read lane; more data available (embeddings, richer search) | Read lane contract stays stable while retrieval internals deepen |
| **Phase 3+** | Same baseline | Read lane stable; app write/action capabilities designed if concrete use cases emerge | SDK surface expands driven by demand, not speculation |

---

## Cross-Doc Map

| Doc | Use it for |
|---|---|
| [ipc-contract.md](./ipc-contract.md) | Exact channel names, request/response schemas, streaming protocol. Updated to v6 with `cowork:read` channel and read lane App Preload SDK. |
| [database-schema.md](./database-schema.md) | Table ownership, schema definitions, retention conventions |
| [system-architecture.md](./system-architecture.md) | End-to-end architecture and resolved decisions |

---

## Key Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Two-layer runtime (automatic injection + tools) — agent path only | Keeps ambient context always available for the agent while preserving on-demand depth. Apps don't participate in layers — they have their own read lane. |
| 2 | Read lane for apps, not direct tool calls | Apps need fast, predictable, read-only data access. Direct tool calls create tight coupling to internal tool surface and conflate reads with writes. Typed SDK methods are discoverable, deterministic, and don't require permission grants. |
| 3 | Default access for read lane (no per-method permission gate) | Read path is bounded and read-only in Phase 1B. Gating every read creates permission fatigue and developer friction. Risk is managed with install disclosure, sandboxing, and app-scoped data access. |
| 4 | Not agent-mediated app access | Third-party apps are highly varied — agent-mediated responses are non-deterministic, slow (LLM round-trip), and expensive (tokens per call). AI Studio templates make app behavior predictable; adding agent reasoning in the middle undermines that predictability. |
| 5 | Not MCP protocol for app communication | MCP is for LLM-to-service communication. Using it for app-to-platform data access adds unnecessary protocol overhead (JSON-RPC, capability negotiation) for simple data lookups. App developers shouldn't need to learn MCP. |
| 6 | Single `cowork:read` IPC channel with namespace dispatch | One channel for all app reads simplifies main process relay (one handler) and keeps the preload thin. Namespace + method dispatch in agents utility is extensible — add new data sources without new IPC channels. |
| 7 | Main process as relay only | Clear separation of orchestration vs transport concerns. Main doesn't inspect app requests or agent responses. |
| 8 | Stable app SDK surface across phases | Read lane contract doesn't change as retrieval internals evolve. New methods/namespaces are additive — existing apps don't break. |
| 9 | Preload-owned app identity (`appId`) | Prevents app JS from spoofing identity. Every read request is scoped server-side using a trusted app identity source (session partition). |

---

## Changelog

**v8 (Feb 17, 2026):** Security and boundary hardening pass for the read-lane model. Added explicit trusted `appId` injection in flow diagrams and implementation notes (preload derives identity from session partition; app JS cannot spoof identity). Reframed read-lane risk language from "no risk" to bounded-risk and documented mandatory install-time disclosure of the read data envelope. Clarified that default-allow means "no per-method runtime gate" (not "no security controls"), with safeguards: sandboxing, read-only lane, and app-scoped data access.

**v7 (Feb 17, 2026):** Replaced app tool-call model (Flows B+C, `apps:callTool`, `platform_chat`, preload permission gate) with read lane architecture. Apps now access platform data through typed SDK (`window.cowork.{context,user,data}.*`) via single `cowork:read` IPC channel — all reads default-allowed, no permission gate. Removed agent lane / `platform_chat` concept entirely — app write/action capabilities are out of scope for Phase 1B, will be designed from concrete use cases if needed. Added full rationale for why not agent-mediated, why not direct tool calls, why not MCP. Updated trust boundaries, tool exposure model ("apps get data, not tools"), access model, phase evolution, and key decisions. ipc-contract.md updated to v6 in sync.

**v6 (Feb 17, 2026):** Refocused this document to communication architecture only. Removed schema/field-level SQL and tool parameter details. Consolidated around process boundaries, IPC call paths, two-layer runtime behavior, app permission gating flow, and phase-level transport behavior. Added explicit scope section with pointers to `database-schema.md` (data model) and `ipc-contract.md` (payload/channel details).

Earlier revision details remain in git history.
