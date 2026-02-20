# LobeChat → Cowork.ai Adaptation Guide

**Purpose:** Maps the key patterns from LobeChat to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our architecture.

**Audience:** Engineering (Rustan + team)
**Target architecture:** [`architecture/system-architecture.md`](../../architecture/system-architecture.md)

**Why LobeChat?** LobeChat has the most sophisticated agent runtime and MCP integration in our competitive set. Its operation-based execution model with persisted state, human intervention policy system, and multi-step MCP install flow are patterns no other app in our set implements at this level. These fill gaps in our agent orchestration and tool-approval UX that aime-chat, Cherry Studio, and Chatbox don't address.

**Key difference from other guides:** LobeChat is a Next.js web app with an Electron desktop wrapper — server-first architecture. We're Electron-first with no web server. Their patterns need process-boundary translation: their TRPC server logic → our Agents Utility Process, their Next.js API routes → our IPC handlers.

---

## Sections

| # | Section | Status |
|---|---|---|
| 0 | [Operation-Based Agent Runtime](#0-operation-based-agent-runtime) | Done |
| 1 | [Human Intervention Policy System](#1-human-intervention-policy-system) | Done |
| 2 | [MCP Install State Machine](#2-mcp-install-state-machine) | Done |
| 3 | [Stream Protocol Normalization](#3-stream-protocol-normalization) | Done |
| 4 | [Summary](#4-summary) | Done |

---

## 0. Operation-Based Agent Runtime

**Source:** LobeChat's `packages/agent-runtime/src/core/runtime.ts` + `src/server/services/agentRuntime/AgentRuntimeService.ts`

Cowork.ai's agent system (system-architecture.md §5) needs to handle multi-step agent executions that may span tool calls, user approvals, and model switches. The core problem: what happens when the agent is mid-execution and the user closes the app? LobeChat solved this with operations — persistent, resumable execution units that survive interruptions.

### Copy

#### Operation as persistent execution unit

An operation captures the full state of an agent execution:

```
Operation:
  operationId: unique ID linking to business context (session, topic, agent, user)
  state: AgentState
    messages: conversation history
    toolManifestMap: available tool definitions
    userInterventionConfig: approval settings
    status: 'idle' | 'running' | 'waiting_for_human' | 'interrupted' | 'done' | 'error'
    stepCount: number of execution steps completed
    cost: accumulated token cost
    usage: accumulated token usage
    pendingToolsCalling: tools awaiting human approval
  metadata: agentConfig, modelRuntimeConfig, userId, appContext
```

**Why copy:** Without persistent operations, a crash mid-tool-call loses context. The agent doesn't know what tools it already called, what results it got, or where it stopped. Operations make execution resumable.

**Our mapping:**

| LobeChat | Cowork.ai |
|---|---|
| Operation stored via coordinator (server-side) | Operation stored in libsql `agent_operations` table |
| TRPC endpoint triggers steps | IPC message triggers steps in Agents Utility |
| Queue service schedules next step | Event loop in Agents Utility schedules next step |
| `operationId` links to session/topic | `operationId` links to conversation/agent |

**Important:** LobeChat stores operation state in-memory (via `AgentStateManager`) or Redis — NOT in Postgres/Neon. Their database (Postgres) stores messages and conversations, but the runtime state for active operations lives in memory or Redis. This means operations do not survive server restarts in LobeChat's architecture.

#### Instruction executor pattern

The agent loop separates decision-making from execution:

```
Agent step:
  1. Load persisted state for operation
  2. Call agent.runner(context, state) → returns instruction(s)
  3. For each instruction:
     a. Look up executor by instruction.type
     b. Execute: executor(instruction, state, context) → { events, newState }
     c. If state becomes 'waiting_for_human' → save and pause
     d. If state becomes 'done' or 'error' → save and stop
  4. Persist updated state
  5. If should continue → schedule next step
```

**Instruction types to copy:**

| Instruction | Purpose | Cowork.ai equivalent |
|---|---|---|
| `call_llm` | Send messages to LLM, get response | Mastra agent step |
| `call_tool` | Execute single MCP tool | Mastra tool execution |
| `call_tools_batch` | Execute multiple tools in parallel | Mastra parallel tool calls |
| `request_human_approve` | Pause for user approval of tool call | Approval notification via IPC |
| `finish` | End execution normally | Agent done |
| `compress_context` | Compress message history when too long | Vercel AI SDK token management |

**Skipped instructions (not needed for v0.1):**
- `exec_task` / `exec_tasks` / `exec_client_task(s)` — server/client-side task execution
- `request_human_prompt` / `request_human_select` — complex human-in-loop UX beyond approval

**Why copy the separation:** The agent's `runner()` function is pure decision logic — given this state, what should I do next? The executors handle side effects (LLM calls, tool calls, state persistence). This makes testing easy: mock the executors, verify the decisions.

#### Max-step safety limit

```
On each step:
  state.stepCount += 1
  if (stepCount > maxSteps):
    → directly return a 'done' event/result (not a finish instruction)
```

**Why copy:** Without a step limit, a misbehaving agent loops forever (LLM calls tool → tool output triggers another tool → infinite loop). LobeChat defaults to a reasonable max. Our complexity router already has token budgets; the step limit adds a second safety net.

### Adapt

#### Operation persistence location

LobeChat stores operation state via `AgentStateManager` in-memory or Redis — NOT in their server database. This means active operations don't survive server restarts in LobeChat. In Cowork.ai, we improve on this by persisting to libsql:

```
LobeChat:  AgentRuntimeService → AgentStateManager → InMemory / Redis
Cowork.ai: Agents Utility Process → libsql → agent_operations table (survives crashes)
```

The schema adapts to our single-DB story:

```sql
CREATE TABLE agent_operations (
  operation_id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  agent_id TEXT NOT NULL,
  state JSON NOT NULL,        -- full AgentState
  status TEXT NOT NULL,        -- idle/running/waiting_for_human/done/error
  step_count INTEGER DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### Step scheduling

LobeChat uses a queue service with HTTP endpoints to schedule the next step:

```typescript
await this.queueService.scheduleMessage({
  endpoint: `${this.baseURL}/run`,
  operationId,
  delay: 50,
  priority: 'high'
});
```

**Our adaptation:** No HTTP queue needed. The Agents Utility Process uses an in-process event loop:

```typescript
// After step completes, schedule next step
if (shouldContinue(stepResult)) {
  setImmediate(() => this.executeStep(operationId, stepIndex + 1));
}
```

For operations paused by `waiting_for_human`, the renderer sends an IPC message when the user approves, which resumes the operation.

### Skip

#### Supervisor group orchestration

LobeChat has a sophisticated multi-agent system with a supervisor state machine that can `speak`, `broadcast`, `delegate`, `execute_task`, `execute_tasks`, and `finish` across agent groups.

**Why skip:** Cowork.ai v0.1 has single-agent interactions. The complexity router picks the right model, not the right agent. Multi-agent orchestration is a v0.3+ feature. Study the pattern (it's well-designed), but don't build it yet.

#### Operation deferred callbacks

LobeChat's `registerAfterCompletion` pattern schedules orchestration actions after message persistence completes, avoiding race conditions with the database.

**Why skip for now:** This pattern matters for multi-agent coordination where agent A's output triggers agent B. For single-agent operations, the step-by-step loop handles ordering naturally.

---

## 1. Human Intervention Policy System

**Source:** LobeChat's `packages/agent-runtime/src/agents/GeneralChatAgent.ts:52-245`

Cowork.ai's MCP tools execute real actions — sending emails, updating tickets, creating calendar events. The user needs to approve dangerous tool calls before execution. LobeChat has a 6-phase policy hierarchy that decides, per tool call, whether to execute automatically or pause for approval. This is the most complete intervention system in our competitive set.

### Copy

#### Multi-phase policy hierarchy

When the LLM requests a tool call, LobeChat evaluates it through a multi-phase hierarchy (effectively 7 decision phases, with a separate manual phase after allow-list):

```
Phase 1: Global security resolvers (hardcoded blacklist)
  → If tool matches security blacklist → route to intervention (in non-headless)
  → In headless mode with 'always' policy → skip execution entirely

Phase 2: Headless mode check
  → If running in async/background mode → auto-execute (unless Phase 1 blocked)

Phase 3: Per-tool dynamic resolver
  → If tool has custom resolver function → run it → BLOCK or ALLOW

Phase 3.5: Overridable global block
  → If Phase 1 blocked with non-'always' policy → BLOCK (can be overridden by user)

Phase 4: 'Always' policy matching
  → If tool config says "always approve" → BLOCK (regardless of user settings)

Phase 5: User 'auto-run' mode
  → If user enables auto-run → ALLOW all remaining tools

Phase 6: User 'allow-list' or 'manual' mode
  → allow-list: tool in list → ALLOW, else BLOCK
  → manual: use tool's own config (default: BLOCK)
```

**Why copy this hierarchy:** It separates security (phases 1, 3.5, 4) from user preference (phases 5, 6) from automation context (phase 2). Security phases route to intervention regardless of user settings. User preferences only apply to tools that aren't security-critical.

**Caveat:** LobeChat's server-side intervention handlers (`AgentRuntimeService`) are currently TODO stubs — the approve/reject flow is fully implemented on the client store side but not wired through the server runtime service yet. This means the pattern is well-designed but not fully production-tested on the server path.

**Our mapping:**

| LobeChat phase | Cowork.ai equivalent |
|---|---|
| Phase 1: Global security blacklist | Hardcoded dangerous-action list (e.g., `delete_*`, `send_email`, `execute_command`) |
| Phase 2: Headless mode | Automations engine — auto-execute within automation flows |
| Phase 3: Dynamic resolver | Per-MCP-server policy from `mcp_servers` table |
| Phase 4: 'Always' approve | Tool-level config: `requiresApproval: 'always'` |
| Phase 5: Auto-run mode | User setting: "Trust this integration" |
| Phase 6: Allow-list / manual | Default mode: show approval dialog for unknown tools |

#### Mixed execution — safe tools first, then request approval

When the LLM requests multiple tool calls, LobeChat splits them:

```
toolsCalling = [emailSend, calendarRead, fileDelete]

After policy evaluation:
  toolsToExecute = [calendarRead]              ← safe, execute immediately
  toolsNeedingIntervention = [emailSend, fileDelete]  ← dangerous, request approval

Instructions generated:
  1. { type: 'call_tool', toolCalling: calendarRead }     ← execute now
  2. { type: 'request_human_approve', pendingTools: [emailSend, fileDelete] }  ← pause
```

**Why copy:** The agent doesn't have to wait for approval of ALL tools before doing anything. Safe tools execute immediately while the user reviews dangerous ones. This makes the agent feel responsive — the user sees partial results while deciding on the approval.

#### Approval state and resumption

When approval is requested:

```
1. State transitions to 'waiting_for_human'
2. pendingToolsCalling stores the tools awaiting approval
3. Operation is persisted in this paused state
4. UI shows approval dialog via 'human_approve_required' event

When user approves:
5. Operation resumes with context.phase = 'human_approved_tool'
6. Approved tool is executed directly (skips runner(), goes straight to call_tool executor)
7. After execution, agent.runner() is called again for next decision

When user rejects:
8. Tool is marked as rejected
9. Agent gets rejection info and decides next action (may try alternative)
```

**Our mapping:** The approval dialog is a Renderer component. The operation state (`waiting_for_human` + `pendingToolsCalling`) is stored in libsql. The approval/rejection comes via IPC from Renderer → Main → Agents Utility.

### Adapt

#### Approval UI integration

LobeChat shows approval inline in the chat UI with tool call details. Cowork.ai needs the same, but also system notifications for when the user isn't looking at the chat:

```
If chat window is focused:
  → Show inline approval card in conversation
If chat window is not focused:
  → Show system notification: "Cowork needs approval to send email via Gmail"
  → Clicking notification opens chat with approval card focused
```

#### Policy configuration storage

LobeChat stores intervention config in the agent state (runtime config). We store it in libsql:

```sql
-- Per-MCP-server policy
ALTER TABLE mcp_servers ADD COLUMN approval_policy TEXT DEFAULT 'manual';
-- Values: 'auto-run' | 'allow-list' | 'manual'
ALTER TABLE mcp_servers ADD COLUMN approval_allow_list JSON DEFAULT '[]';

-- Per-tool override (for dangerous tools that always need approval)
CREATE TABLE tool_policies (
  server_id TEXT NOT NULL,
  tool_name TEXT NOT NULL,
  policy TEXT NOT NULL DEFAULT 'inherit',  -- 'always' | 'never' | 'inherit'
  PRIMARY KEY (server_id, tool_name)
);
```

### Skip

#### Dynamic resolver functions

LobeChat supports custom JavaScript resolver functions per tool that evaluate tool arguments at runtime to decide approval. This is powerful but complex.

**Why skip for v0.1:** Static policies (always/never/inherit) + user mode (auto-run/allow-list/manual) cover 95% of cases. Dynamic resolvers are a v0.2 feature for power users.

#### Headless mode for async tasks

LobeChat's headless mode auto-approves tools when running in background task context. This is for their server-side async task execution.

**Why skip for v0.1:** Our Automations feature (product-features.md §5) will need this, but it's not in v0.1 scope. When we build Automations, copy this pattern.

---

## 2. MCP Install State Machine

**Source:** LobeChat's `src/store/tool/slices/mcpStore/action.ts:196-705` + `src/server/services/mcp/deps/MCPSystemDepsCheckService.ts`

Cowork.ai's MCP Integrations feature (product-features.md §2) needs to guide users through installing MCP servers. This is the hardest UX problem in our MCP story: the user clicks "Install Gmail MCP" and we need to check if they have Node.js, if the npm package exists, if they need API credentials, and if the server actually starts. LobeChat has a 7-step state machine with pause/resume for dependency and configuration gates.

### Copy

#### 7-step install state machine

```
Step 1: FETCHING_MANIFEST (15%)
  → Fetch MCP server manifest from registry
  → Get deployment options (stdio/http/cloud)

Step 2: CHECKING_INSTALLATION (30%)
  → Run dependency checks (Node.js version, npm/python available, package exists)
  → Determine connection type

Step 3: DEPENDENCIES_REQUIRED (40%) — PAUSE
  → If dependencies missing → show what's needed + install instructions
  → User installs dependencies externally
  → User clicks "Retry" → resume from step 2

Step 4: CONFIGURATION_REQUIRED (50%) — PAUSE
  → If server needs config (API keys, URLs) → show config form from schema
  → User fills in config
  → User clicks "Continue" → resume from step 5

Step 5: GETTING_SERVER_MANIFEST (70%)
  → Start the MCP server (stdio/http)
  → Fetch its tool manifest (listTools)
  → Validate server is working

Step 6: INSTALLING_PLUGIN (90%)
  → Save connection config
  → Register tools
  → Enable server

Step 7: COMPLETED (100%)
  → Show success
  → Clean up install state

ERROR state:
  → Capture error type + metadata + stderr
  → Show actionable error message
  → Allow retry
```

**Note:** HTTP and cloud connection types skip the dependency check phases (steps 2-3) and jump directly to configuration/manifest steps. The cloud path also builds the tool manifest from marketplace data instead of starting the server locally.

**Why copy:** Without this state machine, MCP installation is a black box — the user clicks "Install" and either it works or it doesn't, with no way to fix intermediate problems. The pause/resume pattern for dependencies and config is critical: the user needs to leave the app, install Node.js, come back, and continue where they left off.

**Our mapping:**

| LobeChat step | Cowork.ai implementation |
|---|---|
| Fetch manifest | Main process fetches from MCP registry |
| Check installation | Agents Utility runs dependency checks |
| Dependencies required | Renderer shows dependency dialog with install instructions |
| Configuration required | Renderer shows config form (generated from JSON schema) |
| Get server manifest | Agents Utility starts server, lists tools |
| Install plugin | Save to libsql `mcp_servers` + `mcp_tools` tables |
| Completed | Renderer updates MCP Integrations UI |

#### Platform-specific dependency checking

LobeChat's `MCPSystemDepsCheckService` checks system dependencies with version requirements:

```
For each dependency:
  1. Run checkCommand (e.g., "node --version")
  2. Parse version from stdout (handle v prefix, multi-format)
  3. Compare against requiredVersion (supports >=, >, <=, <, =)
  4. If missing: return platform-specific install instructions from manifest
     (manifest provides installInstructions per platform, not hardcoded)
```

**Package-level checks:**

```
NPM: npm list -g {package} → if not global, try npx -y {package} --version
Python: python -m pip list | grep {package} → if not found, try import {package}
```

**Why copy:** Users don't know what's on their system. Telling them "node >= 18.0.0 required — install with `brew install node`" is infinitely better than a cryptic "ENOENT: spawn npx" error.

#### Abort/cancel with cleanup

```
On cancel:
  1. abortController.abort()  → cancels any in-flight HTTP requests
  2. Delete abort controller from store
  3. Clear install progress state
  4. Clear loading indicator

On abort signal during any step:
  → Check abortController.signal.aborted before each async operation
  → If aborted → return early, no state mutation
```

**Why copy:** A user who changes their mind mid-install needs a clean escape. Without abort, they're stuck watching a progress bar they can't cancel.

### Adapt

#### Install state persistence

LobeChat stores install state in Zustand (in-memory store). If the app closes during install, state is lost.

**Our adaptation:** Store install state in libsql so it survives app restarts:

```sql
CREATE TABLE mcp_install_progress (
  identifier TEXT PRIMARY KEY,
  step TEXT NOT NULL,
  progress INTEGER NOT NULL,
  manifest JSON,
  connection JSON,
  config_schema JSON,
  error_info JSON,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

If the app crashes during "DEPENDENCIES_REQUIRED" pause, on restart we can show "You were installing Gmail MCP — continue?"

#### MCP registry source

LobeChat fetches manifests from their marketplace. We'll need our own MCP registry or use a community standard. For v0.1, support manual configuration (user provides command + args + env) without a registry.

### Skip

#### Cloud MCP connection type

LobeChat supports "cloud" MCP connections (market-hosted endpoints). We don't have a cloud marketplace.

**Why skip:** Desktop-only app. All MCP servers run locally (stdio) or connect to user-owned endpoints (HTTP/SSE).

#### Install telemetry/reporting

LobeChat reports install success/failure metrics to their marketplace for analytics. We don't have a marketplace to report to.

**Why skip:** No telemetry infrastructure. If we add analytics later, the install state machine already captures the data needed (step, duration, error type).

---

## 3. Stream Protocol Normalization

**Source:** LobeChat's `packages/model-runtime/src/core/streams/protocol.ts` + `openai/openai.ts`

Cowork.ai's chat UI needs to handle streaming responses from both local (Ollama/DeepSeek) and cloud (OpenRouter → various models) LLMs. Different providers emit different chunk formats. LobeChat normalizes all provider streams into a single internal protocol before the UI sees them.

### Study

#### Unified stream protocol

LobeChat defines internal event types that all providers normalize to:

```
Core protocol events:
  text          — generated text chunk
  reasoning     — thinking/reasoning content (for o1-style models)
  tool_calls    — tool call request from model
  usage         — token usage report
  grounding     — search grounding citations
  error         — stream error

Additional events (not exhaustive):
  base64_image  — inline image data
  content_part  — structured content parts
  reasoning_part / reasoning_signature — reasoning metadata
  stop          — stream termination signal
  speed         — generation speed metrics
  data          — generic data payload
```

Each provider's stream handler transforms native chunks into these events, then `createSSEProtocolTransformer` converts them to SSE format for the client.

**What to learn:** The normalization layer means the UI only handles one format. Adding a new provider doesn't touch UI code — just add a new normalizer.

**Our application:** Vercel AI SDK v6 already handles most of this — its `streamText()` returns a unified stream format. But if we add custom stream events (e.g., "agent switched from local to cloud mid-response"), we'll need a protocol extension layer similar to LobeChat's approach.

#### DB-backed provider auth resolution

```
On each chat request:
  1. Load provider settings from DB (not env vars)
  2. Decrypt API keys from key vault
  3. Build provider-specific auth payload
  4. Initialize model runtime with resolved credentials
```

**What to learn:** Provider credentials in the database (not environment variables) enables per-user, per-workspace credential management. This is important for Cowork.ai's multi-tier model — Free tier uses shared OpenRouter keys, Pro tier uses user's own keys.

**Our application:** Store OpenRouter API keys (and any direct provider keys) in libsql with encryption-at-rest via Electron's `safeStorage`. The Agents Utility Process decrypts on use.

### Skip

#### 60+ provider matrix

LobeChat ships an in-house `model-runtime` package supporting 60+ providers. This is their competitive moat but enormous maintenance overhead.

**Why skip:** We use Vercel AI SDK v6 + OpenRouter. OpenRouter gives us access to dozens of providers through one API key. We need exactly 2 provider adapters: Ollama (local) and OpenRouter (cloud).

#### Router fallback runtime

LobeChat's `createRouterRuntime` supports multi-channel failover — if one provider endpoint fails, try another. Complex and useful for their server product.

**Why skip:** Our complexity router already handles model selection. If OpenRouter is down, we fall back to local. No need for multi-channel failover within a single provider.

---

## 4. Summary

### Copy directly

| Pattern | Source | Cowork.ai target | Rationale |
|---|---|---|---|
| Operation as execution unit (in-memory/Redis in LobeChat) | `runtime.ts`, `AgentRuntimeService.ts` | libsql `agent_operations` table (we persist, unlike LobeChat) | Resumable agent execution that survives crashes |
| Instruction executor pattern | `runtime.ts:26-49` | Agents Utility — agent loop | Clean separation of decisions from side effects |
| Max-step safety limit | `runtime.ts:84-105` | Agent config | Prevents infinite tool-call loops |
| Multi-phase intervention policy hierarchy | `GeneralChatAgent.ts:52-245` | Agents Utility — policy engine | Security phases route to intervention regardless of user preferences |
| Mixed execution (safe first, approval second) | `GeneralChatAgent.ts:386-432` | Agents Utility — tool dispatch | Agent stays responsive while user reviews dangerous tools |
| Approval state + resumption flow | `runtime.ts:566-588` | IPC-based approval dialog | User approves → operation resumes exactly where it paused |
| 7-step MCP install state machine | `mcpStore/action.ts:196-705` | Main process — MCP installer | Guided install with dependency and config gates |
| Platform-specific dependency checking | `MCPSystemDepsCheckService.ts:61-159` | Agents Utility — dep checker | Actionable error messages instead of cryptic failures |
| NPM/Python package detection | `NpmInstallationChecker.ts`, `PythonInstallationChecker.ts` | Agents Utility — package checker | Multiple detection strategies for reliability |
| Install abort/cancel with cleanup | `mcpStore/action.ts:157-176` | MCP install controller | Clean escape from in-progress installation |

### Study (learn from, adapt approach)

| Pattern | What to learn | How it changes for us |
|---|---|---|
| Unified stream protocol normalization | One internal format for all providers | Vercel AI SDK covers most of this; extend for custom events |
| DB-backed provider auth resolution | Credentials in DB not env vars | Store in libsql with safeStorage encryption |
| Supervisor group orchestration state machine | Multi-agent coordination patterns | v0.3+ feature; study the `speak`/`broadcast`/`delegate`/`execute_task` decisions |
| Deferred orchestration callbacks (`registerAfterCompletion`) | Avoid race conditions in multi-agent flows | Needed when we build Automations |
| Connection-type routing (cloud/http/stdio) | Single tool invocation API across connection types | Already doing this (see Cherry Studio guide §1) |
| Custom `app://` protocol static serving | Production asset serving with range request support | Good pattern for our Electron production build |

### Skip

| Pattern | Why skip |
|---|---|
| 60+ provider `model-runtime` package | Use Vercel AI SDK + OpenRouter (2 adapters) |
| Router fallback runtime (multi-channel failover) | Complexity router handles model selection |
| Cloud MCP marketplace connection type | Desktop-only, no cloud marketplace |
| Install telemetry reporting | No marketplace infrastructure |
| Full group orchestration surface (speak/broadcast/delegate/execute_task) | v0.1 is single-agent |
| Dynamic JavaScript resolver functions for approval | Static policies + user modes cover 95% of cases |
| Headless auto-approval mode | Needed for Automations (v0.2+) |
| Next.js-specific patterns (TRPC, API routes, SSR) | We use Electron IPC, not HTTP |
| Drizzle ORM patterns | We use libsql directly |
| PGlite/Neon driver switching | Single DB engine (libsql) |

---

## Changelog

**v1 (Feb 17, 2026):** Initial draft. All 4 sections complete.

**v1.1 (Feb 17, 2026):** Fact-check corrections: operations stored in-memory/Redis not Postgres (we persist to libsql as an improvement), AgentState includes `interrupted` status, max-step returns `done` event not `finish` instruction, server intervention handlers are TODO stubs (client-side is implemented), HTTP/cloud install paths skip dependency checks, stream protocol has many more event types, install instructions come from manifest not hardcoded, supervisor decisions are speak/broadcast/delegate/execute_task(s)/finish.
