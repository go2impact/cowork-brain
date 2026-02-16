# 1. Intelligent Conversations — Mastra Agent System

[< Back to index](README.md)

**Their claim:** "Powerful AI Agent system based on Mastra framework, supporting streaming responses and tool calling"

**Maps to Cowork.ai:** [Chat](../../../product/product-features.md#3-chat) + [MCP Browser](../../../product/product-features.md#4-mcp-browser) — our agent orchestration layer powers both on-demand conversation and execution.

## How It Works

Mastra runs entirely in Electron's main process. A single `Mastra` instance is created at boot with a `LibSQLStore` backend. Agents are built dynamically per chat request — the `AgentManager` resolves config from the DB, loads tools via `ToolsManager`, and creates a `MastraAgent` with memory.

**Key files:**
- `src/main/mastra/index.ts` — MastraManager (1,807 lines)
- `src/main/mastra/agents/index.ts` — AgentManager (375 lines)
- `src/main/mastra/agents/base-agent.ts` — BaseAgent class
- `src/main/mastra/storage/index.ts` — LibSQL storage config

## Architecture Diagram

```
┌─────────────────────────── ELECTRON MAIN PROCESS ───────────────────────────┐
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    MastraManager (singleton)                         │   │
│  │                                                                      │   │
│  │  Mastra instance ─── LibSQLStore (file:data/main.db via getDbPath()) │  │
│  │       │                                                              │   │
│  │       ├── Workflows: claudeCodeWorkflow, chatWorkflow               │   │
│  │       └── Agents: {} (registered dynamically)                       │   │
│  │                                                                      │   │
│  │  Express HTTP server (port 41100) ── POST /mcp (MCP bridge)        │   │
│  │                                                                      │   │
│  │  IPC channel: MastraChannel.Chat (mode: 'on' = streaming)          │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│       │ uses                           │ uses                               │
│       ▼                                ▼                                    │
│  ┌──────────────┐              ┌──────────────────┐                        │
│  │ AgentManager  │              │  ToolsManager    │                        │
│  │              │              │                  │                        │
│  │ buildAgent() │──── calls ──▶│ buildTools()     │                        │
│  │  ├ resolve DB│              │  ├ built-in      │                        │
│  │  ├ load tools│              │  ├ MCP clients   │                        │
│  │  ├ sub-agents│              │  └ skills        │                        │
│  │  └ Memory()  │              └──────────────────┘                        │
│  └──────────────┘                                                          │
│       │ creates                                                             │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  MastraAgent { id, name, instructions, model, memory, tools }       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow: User Message → Agent Response

```
RENDERER                          MAIN PROCESS
────────                          ────────────
User types message
       │
       ▼
IPC: MastraChannel.Chat ────────▶ MastraManager.chat()
  {                                      │
    agentId,                             ├─ Resolve thread from LibSQLStore
    chatId,                              ├─ AgentManager.buildAgent(agentId)
    messages,                            │    ├─ Load agent config from DB
    model,                               │    ├─ ToolsManager.buildTools(toolIds)
    tools[]                              │    └─ new MastraAgent({ model, tools, memory })
  }                                      │
                                         ├─ WHILE LOOP (multi-step execution):
                                         │    │
                                         │    ├─ IF tokens > 70% of max context:
                                         │    │    └─ compressMessages() via configurable fast model
                                         │    │       (appInfo.defaultModel.fastModel — user-selected)
                                         │    │
                                         │    ├─ agent.stream(input, options)
                                         │    │    ├─ LLM inference (Claude/Gemini/etc)
                                         │    │    ├─ Tool calls ──▶ execute ──▶ result
                                         │    │    └─ Save messages to LibSQL progressively
                                         │    │
                                         │    ├─ Convert stream: toAISdkFormat()
                                         │    │
                                         │    ├─ Read chunks, emit via IPC:
                                         │    │    appManager.sendEvent(
                                         │    │      `chat:event:${chatId}`,
                                         │    │      { type: ChatChunk, data: chunk }
                                         │    │    )
                                         │    │
                                         │    └─ Check status:
                                         │         'suspended' → wait for tool approval
                                         │         'success' + role=assistant → done
                                         │         'success' + role=tool → loop again
                                         │
                                         └─ Save usage (tokenlens cost calculation)

       ◀──────── IPC events ────────────

Renderer receives:
  chat:event:${chatId}
       │
       ├─ text-delta → append text
       ├─ tool-invocation → show tool UI
       ├─ tool-result → show result
       ├─ reasoning → collapsible section
       └─ finish-reason → mark complete
       │
       ▼
  Zustand store update → React re-render
```

## Tool Approval Flow

```
Agent wants to call tool
       │
       ▼
stream.status == 'suspended'
       │
       ▼
UI shows approval dialog ──────▶ User approves/rejects
       │                                │
       ▼                                ▼
agent.approveToolCall()          agent.declineToolCall()
       │                                │
       └────── resume stream ───────────┘
```

## Agent Summary

| Agent | Visible | Registered | Tools | Sub-agents | Invocation |
|-------|---------|------------|-------|------------|------------|
| DefaultAgent | No (`isHidden=true`) | Yes | 0 | None | Auto-fallback when no agent selected |
| CodeAgent | **Yes** (`isHidden=false`) | Yes | 25 | Explore, Plan | User selects in UI |
| Explore | No (`isHidden=true`) | Yes | 14 | None | Spawned via Task tool from CodeAgent |
| Plan | No (`isHidden=true`) | Yes | 11 | None | Spawned via Task tool / EnterPlanMode |
| CompressAgent | N/A | **No** — standalone | 0 | None | Auto-triggered at 70% context threshold |
| SkillCreator | No (`isHidden=true`) | **No** — dead code | 11 | None | Never invoked |
| Translation | No (`isHidden=true`) | **No** — not registered | 0 | None | Translation tool creates fresh instance |

> **Note:** Four agents are registered via `AgentManager.registerAgents()` (DefaultAgent, CodeAgent, Explore, Plan). Only CodeAgent has `isHidden=false` — it's the only agent visible in `getAvailableAgents()`. DefaultAgent serves as the fallback when no agent ID is specified. CompressAgent is instantiated directly in `MastraManager` as a raw `Agent` from `@mastra/core/agent`, outside AgentManager's lifecycle. Translation is imported only by the Translation tool, which creates a fresh instance per invocation. SkillCreator exists as a file but is never imported or registered anywhere.

## Agent Definitions (Per-Agent Detail)

### DefaultAgent — Fallback Assistant

**File:** `src/main/mastra/agents/default-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | `"You are a helpful assistant."` |
| **Tools** | None |
| **Sub-agents** | None |
| **Model** | Runtime — `agentEntity.defaultModelId` or `appInfo.defaultModel.model` |
| **Memory** | Enabled (via `buildAgent()`) |

**When invoked:** `MastraManager.chat()` uses DefaultAgent as the fallback when no `agentId` is provided in the chat request. It resolves the model from the agent's DB entity or the app's global default model setting.

### CodeAgent — Primary User-Facing Agent

**File:** `src/main/mastra/agents/code-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | Dynamic — loaded from `prompts/code-agent-prompt.ts`. Includes environment info (workspace, git status, platform, date), tool-specific policy sections, and database tool instructions when LibSQL tools are available. Core identity: *"You are a code agent that helps users with software engineering tasks."* |
| **Tools (25)** | AskUserQuestion, Bash, BashOutput, KillBash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, CodeExecution, Skill, Task, SendEvent, TaskCreate, TaskGet, TaskList, TaskUpdate, LibSQLListTable, LibSQLDescribeTable, LibSQLDatabaseInfo, LibSQLRun |
| **Sub-agents** | Explore, Plan (auto-injects Task tool) |
| **Model** | Runtime — user-selected per chat |
| **Memory** | Enabled |

**When invoked:** User selects CodeAgent from the agent dropdown in the UI. It's the only user-visible agent.

**Key instruction behaviors:**
- Delegates codebase exploration to Explore sub-agent via Task tool
- Delegates implementation planning to Plan sub-agent via EnterPlanMode
- Uses TaskCreate/TaskUpdate tools to track multi-step work
- Instructions dynamically inject database tool docs and task management policy based on available tools

### Explore — Codebase Search Specialist

**File:** `src/main/mastra/agents/explore-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | Dynamic — *"You are a file search specialist. You excel at thoroughly navigating and exploring codebases."* Includes environment info. Instructions prohibit file creation/modification despite having Write/Edit tools. |
| **Tools (14)** | TaskCreate, TaskGet, TaskList, TaskUpdate, Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, CodeExecution |
| **Sub-agents** | None |
| **Model** | Inherits from parent agent's model |
| **Memory** | Enabled |

**When invoked:** Spawned by CodeAgent via the Task tool with `subagent_type: "Explore"`. Also spawned during EnterPlanMode Phase 1 (up to 3 in parallel). Runs with `maxSteps: 100`.

**Key constraint:** Instructions say read-only, but Write/Edit tools are technically available. The prohibition is prompt-level, not enforced by tool availability.

### Plan — Software Architect

**File:** `src/main/mastra/agents/plan-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | Dynamic — *"You are a software architect and planning specialist."* Strictly read-only: cannot create, modify, or delete files. Must end response with "Critical Files for Implementation" section listing 3-5 key files. |
| **Tools (11)** | Bash, Glob, Grep, Read, WebFetch, TaskCreate, TaskGet, TaskList, TaskUpdate, WebSearch, Skill |
| **Sub-agents** | None |
| **Model** | Inherits from parent agent's model |
| **Memory** | Enabled |

**When invoked:** Spawned by CodeAgent via Task tool with `subagent_type: "Plan"`, or during EnterPlanMode Phase 2. Runs with `maxSteps: 100`.

**Key constraint:** Truly read-only — no Write or Edit tools available (unlike Explore which has them but is told not to use them).

> **Note:** Plan's `description` field is a copy-paste of Explore's description (codebase search specialist text). This is a bug in the source — the description doesn't match Plan's actual purpose as an architect.

### CompressAgent — Context Summarizer

**File:** `src/main/mastra/agents/compress-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | `"You are a helpful AI assistant tasked with summarizing conversations."` |
| **Tools** | None |
| **Model** | Hardcoded `'openai/gpt-5-mini'` in definition, but **overridden at runtime** with `appInfo.defaultModel.fastModel` (user-configurable fast model) |
| **Memory** | Not configured (standalone `Agent`, not `BaseAgent`) |

**When invoked:** `MastraManager.compressMessages()` calls `compressAgent.generate()` (not `.stream()`) when `tokenCount >= maxContextSize * 0.7`. The hardcoded model ID is dead code — line 1493 of `mastra/index.ts` overwrites it: `compressAgent.model = options.model`.

**Summarization prompt requests 9 structured sections:**
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and Fixes
5. Problem Solving
6. All User Messages
7. Pending Tasks
8. Current Work
9. Optional Next Step

After compression, original messages are soft-deleted (moved to `resourceId + '.history'`), and the compressed summary replaces them as a single user message.

### SkillCreator — Dead Code

**File:** `src/main/mastra/agents/skill-creator-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | *"You are a file search specialist."* (copy-paste from Explore — doesn't match the skill creation purpose) |
| **Tools (11)** | TodoWrite, Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, CodeExecution, Skill |
| **Working directory** | `app.getPath('userData')/skills` |

**Status:** Not registered in `AgentManager.registerAgents()`, not imported anywhere else. Dead code.

### Translation — On-Demand Translation

**File:** `src/main/mastra/agents/translation-agent.ts`

| Property | Value |
|----------|-------|
| **System prompt** | `"You are a translation expert."` |
| **Tools** | None |
| **Model** | Runtime — `appInfo.defaultModel.fastModel` with fallback |
| **Memory** | Enabled (BaseAgent), but effectively unused (fresh instance per call) |

**When invoked:** The Translation tool (`src/main/tools/work/translation.ts`) creates a fresh `TranslationAgent` instance per call: `new TranslationAgent({}).buildAgent({ model })`. Calls `.generate()` (not `.stream()`) with a hardened prompt that resists prompt injection attempts. Returns `<<SAME_LANGUAGE>>` sentinel if source and target languages match.

## Why These Agents Exist: Design Rationale

AIME Chat's agent system solves three interconnected problems: **context management**, **capability specialization**, and **workflow safety**. Understanding why each agent exists explains the entire architecture.

### The Core Problem: Context Windows Are Finite

A coding assistant that reads 50 files to find a pattern fills its context window with raw file contents. The user's conversation gets pushed out. Subsequent responses degrade because the model can't see earlier context. This is the fundamental tension: agents need information to be useful, but gathering information consumes the resource they need to be useful.

**AIME's solution: separation of concerns via disposable sub-agents.** CodeAgent is the user's persistent conversation partner. When it needs to explore a codebase or design a plan, it spawns a separate agent in a clean context window. The sub-agent does the heavy lifting (reading files, searching patterns, reasoning about architecture), then returns a **summary** to CodeAgent. The raw file contents never enter the user's conversation context.

This is why Explore and Plan exist as separate agents rather than being behaviors of CodeAgent. They're not different personalities — they're **context isolation boundaries**.

### Why Each Agent Exists

| Agent | Problem It Solves | End-User Feature |
|-------|------------------|-----------------|
| **CodeAgent** | Users need a single, persistent conversational partner that can write code, run commands, and coordinate complex tasks | The chat interface — everything the user interacts with. User selects it from the agent selector (`ChatAgentSelector`), which shows a searchable dialog of available agents (only CodeAgent has `isHidden=false`). |
| **DefaultAgent** | The app needs a working state before the user configures anything. If someone starts a chat without selecting an agent, something needs to respond. | Invisible to the user. They just see "a chat that works" — DefaultAgent provides the minimal fallback (no tools, basic prompt). |
| **Explore** | CodeAgent can't afford to read dozens of files in its own context. Searching a codebase requires many tool calls (Glob, Grep, Read) that produce large outputs. | The user sees a collapsible Task tool invocation card in the chat UI showing `@Explore` badge, a description, and the prompt. Results stream back as `data-task-{toolCallId}` events. The user sees the search happening but the raw content stays outside their conversation. |
| **Plan** | Before writing code, the agent needs to explore patterns, consider trade-offs, and produce a structured implementation plan — all without accidentally modifying files. A read-only constraint is needed. | Same UI as Explore (Task tool card with `@Plan` badge). The user sees the architectural analysis streaming in. Plan's required "Critical Files for Implementation" output gives the user a concrete artifact to review. |
| **CompressAgent** | Long conversations hit context limits. Without compression, the agent would either refuse to continue or lose earlier context. | Invisible to the user. They see `data-compress-start` / `data-compress-end` events (likely a loading indicator), then the conversation continues normally. The original messages move to a `.history` resource (soft delete), replaced by a structured summary. |
| **Translation** | The Translation tool needs an LLM to translate text, but it's a utility function, not a conversation. It needs a one-shot generate call with no tool access. | User triggers Translation via a tool in the chat. The agent is invisible — they just see the translated output. |

## Agent Selection: What the User Sees

**File:** `src/renderer/components/chat-ui/chat-agent-selector.tsx`

The user's experience is simple: a badge in the chat input area that shows `@Code Agent` (or `Select Agent` if none chosen). Clicking it opens a searchable `Command` dialog listing available agents. The agent list comes from `getAvailableAgents()`, which filters the DB for `isActive: true, isHidden: false` — in practice, only CodeAgent appears.

```
┌──────────────────────────────────────────────────┐
│ Chat Input Area                                    │
│                                                    │
│  [@Code Agent ×]  [model selector]                │
│                                                    │
│  ┌──────────────────────────────────────────────┐│
│  │ Type your message...                          ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘

User clicks [@Code Agent]:

┌──────────────────────────────┐
│  Search agents...              │
│  ┌──────────────────────────┐│
│  │ Code Agent            ✓  ││
│  │ A code agent that can    ││
│  │ help with code related   ││
│  │ tasks.                   ││
│  └──────────────────────────┘│
└──────────────────────────────┘
```

There is **no automatic routing**, no complexity analysis, no intent classification. The user picks the agent (CodeAgent is the only option) and the model. This is a fundamentally different approach from Cowork.ai's complexity router — they expose the choice, we abstract it.

**Technical flow:**

```
Renderer: ChatAgentSelector                    Main Process
  │                                                │
  ├─ getAvailableAgents() ─── IPC ──────────────▶ AgentManager
  │                                                │
  │                                                ├─ DB query: isActive=true, isHidden=false
  │                                                └─ Returns: [{ id: "CodeAgent", name: "Code Agent", ... }]
  │
  ├─ User selects "CodeAgent"
  ├─ User types message, hits send
  │
  └─ IPC: MastraChannel.Chat ──────────────────▶ MastraManager.chat({
       { agentId: "CodeAgent",                       agentId,
         model: "openai/gpt-4o",                     model,
         messages: [...],                             ...
         tools: [...] }                           })
                                                     │
                                                     ├─ agentId provided → buildAgent("CodeAgent")
                                                     └─ (if missing) → fallback to DefaultAgent
```

## Sub-Agent Delegation: How CodeAgent Spawns Workers

**File:** `src/main/tools/common/task.ts`

**Why delegation exists:** When CodeAgent needs to search a codebase, it faces a choice: read files directly (consuming its own context) or delegate to a specialist (keeping its context clean). AIME chose delegation. This mirrors how a senior developer works — they don't read every file themselves; they ask a colleague to investigate and report back.

**How the Task tool self-describes:** The Task tool's description is **dynamically generated** at agent build time based on which sub-agents are available. The `getDescription()` method builds a prompt that tells CodeAgent what sub-agents exist, what tools each has, and when to use them:

```typescript
// At buildAgent() time, if subAgents configured:
this.description = this.getDescription(subAgents);

// Generates:
// "Available agent types and the tools they have access to:
//  - general-purpose: ... (Tools: *)
//  - Explore: Fast agent specialized for exploring codebases... (Tools: TaskCreate, Bash, Read, ...)
//  - Plan: ... (Tools: Bash, Glob, Grep, Read, ...)"
```

This means CodeAgent's instructions don't hardcode knowledge of Explore or Plan — it learns about available sub-agents from the Task tool's description. Adding a new sub-agent type only requires registering it; the Task tool description auto-updates.

**What the user sees in the UI:**

When CodeAgent spawns a sub-agent, the chat shows a tool invocation card (rendered by `ChatToolResultPreview`):

```
┌──────────────────────────────────────┐
│  Task                                  │
│  ┌──────────────────────────────────┐│
│  │ @Explore                          ││
│  │ Find authentication patterns      ││
│  │ ┌──────────────────────────────┐ ││
│  │ │ Search the codebase for all  │ ││
│  │ │ files related to auth...     │ ││
│  │ └──────────────────────────────┘ ││
│  └──────────────────────────────────┘│
│                                        │
│  Output (when done):                   │
│  ┌──────────────────────────────────┐│
│  │ Found 5 relevant files:          ││
│  │ /src/auth/handler.ts - main...   ││
│  │ /src/middleware/auth.ts - ...     ││
│  └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

While the sub-agent runs, step results stream back via `data-task-{toolCallId}` custom data events. The user can see the sub-agent's progress (text and tool calls) in real-time.

**Full delegation flow:**

```
User: "Where is authentication handled in this codebase?"
       │
       ▼
CodeAgent (in its context window):
  "I need to search the codebase. I'll use the Task tool with Explore."
       │
       ▼
Task tool execute():
  1. agentManager.buildAgent("Explore", { modelId: rootAgentModel })
     └── Creates fresh Explore agent with same model as CodeAgent
  2. agent.stream(prompt, { maxSteps: 100 })
     └── Explore runs in its OWN context window:
         - Calls Glob("**/auth*")
         - Calls Grep("authentication|authorize")
         - Calls Read("/src/auth/handler.ts")
         - Calls Read("/src/middleware/auth.ts")
         - ... (potentially dozens of tool calls)
         - Compiles findings into a summary
  3. for each chunk in stream.fullStream:
     └── Streams step-finish events to CodeAgent via writer:
         { type: "data-task-{toolCallId}", data: { text, tool-calls } }
  4. Returns: concatenated text summary
       │
       ▼
CodeAgent receives: "Found 5 relevant files: ..."
  └── Synthesizes a response for the user
  └── Explore's raw file contents never entered CodeAgent's context
```

**Key design properties:**
- **Stateless:** Each sub-agent invocation is independent — no follow-up messages, no memory of previous calls
- **Model inheritance:** Sub-agents use the parent agent's model (not hardcoded)
- **Parallel execution:** CodeAgent can spawn multiple sub-agents in one turn (e.g., 3 Explore agents searching different parts of the codebase)
- **`maxSteps: 100`:** Sub-agents have an upper bound on tool call iterations (CodeAgent doesn't)
- **Context isolation:** This is the entire point — sub-agent tool outputs stay in their context, only the summary returns

## Plan Mode: Structured Design Before Execution

**Files:** `src/main/tools/common/enter-plan-mode.ts`, `exit-plan-mode.ts`

**Why plan mode exists:** Complex coding tasks fail when the agent jumps straight to writing code. It modifies the wrong files, misunderstands the architecture, or implements something the user didn't want. Plan mode is a **forced exploration phase** — the agent must understand the codebase and get user buy-in before touching any files.

The insight is that planning and execution have different risk profiles. Reading files is safe and reversible. Writing files is dangerous and hard to undo. Plan mode enforces this boundary.

**What the user experiences:**

The user asks CodeAgent for a non-trivial task (e.g., "Add user authentication"). CodeAgent's instructions tell it to call `EnterPlanMode` for any task involving multiple files, architectural decisions, or unclear requirements. The flow from the user's perspective:

```
User: "Add OAuth login to the app"
       │
       ▼
CodeAgent: "I'll enter plan mode to explore the codebase first."
       │ (calls EnterPlanMode tool)
       │
       ▼
[PLAN MODE — read-only constraint active]
       │
       ├── Phase 1: CodeAgent spawns Explore agents
       │   User sees: Task tool cards with @Explore searching for auth patterns,
       │   existing login flow, OAuth libraries in package.json
       │
       ├── CodeAgent may ask: "Do you want OAuth via Google, GitHub, or both?"
       │   User sees: AskUserQuestion dialog
       │
       ├── Phase 2: CodeAgent spawns Plan agent
       │   User sees: Task tool card with @Plan designing the implementation
       │   Plan output ends with "Critical Files for Implementation" listing
       │
       ├── Phase 3: CodeAgent reads the critical files, verifies understanding
       │
       ├── Phase 4: CodeAgent writes plan to {workspace}/plans/peppy-jumping-wadler.md
       │   (the ONLY file it can edit in plan mode)
       │
       └── Phase 5: CodeAgent calls ExitPlanMode with plan summary
           User sees: the plan in the chat
           │
           ▼
[PLAN MODE EXITS — execution allowed]
           │
           ▼
CodeAgent: begins implementing the plan (can now edit files)
```

**The read-only constraint is prompt-enforced, not system-enforced.** EnterPlanMode returns a `<system-reminder>` block that instructs the agent: *"you MUST NOT make any edits... This supersedes any other instructions."* The only exception is the plan file itself. Tools like Write and Edit are still technically available — the agent is told not to use them, but nothing prevents it mechanically.

**ExitPlanMode auto-approves.** Despite the tool name suggesting user approval, `ExitPlanMode.execute()` simply returns `"User has approved your plan. You can now start coding."` The "approval" is implicit — the user sees the plan in the chat stream and can interrupt or redirect before the agent starts writing code.

**Plan file path is hardcoded:** `{workspace}/plans/peppy-jumping-wadler.md`. This appears to be a placeholder name (a Haikunator-style random name) that was never randomized at runtime.

> **Note for Cowork.ai:** Plan mode's value is the forced exploration phase, not the specific implementation. Our complexity router already decides execution strategy — but the concept of "explore before executing" applies to our MCP Browser's Agent → Browser path, where the agent should understand the target application before taking actions.

## Agent Lifecycle & Memory

**Every chat turn gets a fresh agent instance.** Agents are ephemeral — `agentManager.buildAgent()` creates a new `MastraAgent` with tools, memory, and model resolved from the DB entity and runtime parameters. There are no long-lived agent objects.

**Memory configuration** (in `buildAgent()`, `src/main/mastra/agents/index.ts:179-198`):

```
Memory({
  storage: storage,
  options: {
    generateTitle: false,       ← title generated separately by MastraManager
    semanticRecall: false,      ← disabled
    workingMemory: { enabled: false },
    lastMessages: false,
  },
  vector: getVectorStore(),     ← configured but not actively used
})

OutputProcessors: MessageHistory({ storage })  ← persists messages to DB
```

**At chat time** (`MastraManager.chat()`, `src/main/mastra/index.ts:581-593`):

```
memory: {
  thread: { id: chatId },      ← conversation thread
  resource: resourceId,         ← "project:{projectId}" or default
  options: { readOnly: false, lastMessages: false },
}
```

**Message flow:**
1. Load history: `memoryStore.listMessages({ threadId, resourceId })`
2. Convert format: `toAISdkV5Messages()` (Mastra v2 → AI SDK v5 UIMessage)
3. Agent streams response
4. `MessageHistory` output processor auto-persists new messages
5. On abort: manual save of interrupted messages (lines 1118-1139)

## What We Should Do

| Action | Detail |
|--------|--------|
| **Study** | Their `while(true)` multi-step execution loop with compression. We need this for long agent sessions. |
| **Study** | The `toAISdkFormat()` stream conversion — how Mastra streams convert to UI-consumable chunks via IPC. |
| **Study** | Tool approval flow (`suspended` → approve/decline → resume). Maps directly to our MCP Browser approval gates. |
| **Study** | Task tool sub-agent delegation pattern — dynamic description generation, stateless spawning, step-result streaming back to parent. |
| **Study** | Plan mode workflow — structured 5-phase approach with read-only constraints and parallel sub-agent exploration. |
| **Copy pattern** | Context compression via fast model when tokens > 70% threshold. Essential for long conversations. |
| **Copy pattern** | Dynamic system prompt generation — instructions as functions that access `requestContext` (workspace, git status, available tools). |
| **Skip** | Their agent types (Code, Explore, Plan, etc.) — we have a single platform agent with complexity routing instead. |
| **Skip** | Their agent selection model (user picks agent from UI). We abstract behind the complexity router. |
