# 4. Open CoWork — Tool System

[< Back to index](README.md)

**Their claim:** "AI is not just for chatting, it can perform actual operations like file editing, code execution, web searching, and more"

**Maps to Cowork.ai:** [MCP Browser](../../../product/product-features.md#4-mcp-browser) — our execution paths (Agent → Tools → MCP and Agent → Tools → Browser). They have both paths too.

## How It Works

Tools extend `BaseTool` (single function) or `BaseToolkit` (collection). Each tool defines a Zod input/output schema, an async `execute()` method, and optional config. The `ToolsManager` registers 25+ built-in tools, wraps them via Mastra's `createTool()`, and exposes them to agents.

**Key files:**
- `src/main/tools/base-tool.ts` — BaseTool abstract class
- `src/main/tools/base-toolkit.ts` — BaseToolkit container
- `src/main/tools/index.ts` — ToolsManager (registration, building, execution)
- `src/main/instances/browser-instance.ts` — Playwright + Stagehand

## Tool Architecture

```
┌──────────────────────────── TOOL SYSTEM ────────────────────────────┐
│                                                                      │
│  BaseTool                              BaseToolkit                   │
│  ┌──────────────────────┐              ┌──────────────────────┐     │
│  │ id: string           │              │ id: string           │     │
│  │ description: string  │              │ tools: BaseTool[]    │     │
│  │ inputSchema: Zod     │              │ isToolkit: true      │     │
│  │ outputSchema?: Zod   │              │ configSchema?: Zod   │     │
│  │ configSchema?: Zod   │              └──────────────────────┘     │
│  │ requireApproval?: bool│                                           │
│  │ execute(input, ctx)  │                                           │
│  └──────────────────────┘                                           │
│                                                                      │
│  ToolsManager.registerBuiltInTools():                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                                                                │ │
│  │  SHELL               FILES            WEB                      │ │
│  │  ├─ BashToolkit       ├─ FileSystem   ├─ WebSearch             │ │
│  │  │  ├─ Bash           │  ├─ Read      │  (Brave/Google/        │ │
│  │  │  ├─ KillBash       │  ├─ ReadBin   │   Tavily/SerpAPI/      │ │
│  │  │  ├─ BashOutput     │  ├─ Write     │   Jina)                │ │
│  │  │  └─ ListBash       │  ├─ Edit      ├─ WebFetch              │ │
│  │  │                    │  ├─ Glob      │                        │ │
│  │  ├─ CodeExec          │  └─ Grep      │                        │ │
│  │  │  (Python/uv)       │               │                        │ │
│  │                        │               │                        │ │
│  │  IMAGE/VISION         AUDIO           DATABASE                  │ │
│  │  ├─ GenerateImage     ├─ AudioToolkit ├─ LibSQLToolkit         │ │
│  │  ├─ EditImage         │  ├─ STT       │  (query, format)       │ │
│  │  ├─ RemoveBackground  │  └─ TTS       │                        │ │
│  │  └─ Vision (OCR)      │               │                        │ │
│  │                        │               │                        │ │
│  │  AGENT/META           INTERACTION     WORK                      │ │
│  │  ├─ Task              ├─ AskUser      ├─ Extract               │ │
│  │  ├─ MemoryToolkit     ├─ SendEvent    ├─ Translation           │ │
│  │  ├─ ToolToolkit       │               │                        │ │
│  │  └─ TodoToolkit       │               │                        │ │
│  │                        │               │                        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## Tool Execution Context

```
execute(input, context) where context = {
    requestContext: {
        threadId,          ◀── conversation thread
        workspace,         ◀── project directory
        model,             ◀── current LLM
        file_last_read_time, ◀── file modification tracking
        usage,             ◀── token tracking
    },
    abortSignal,           ◀── cancellation support
}
```

## Browser Automation: Three Modes

```
┌──────────────────── BROWSER AUTOMATION ────────────────────┐
│                                                              │
│  Mode 1: STAGEHAND (LLM-driven)                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Stagehand({ env: 'LOCAL', model, llmClient })     │    │
│  │       │                                             │    │
│  │       ▼                                             │    │
│  │  LLM sees page → decides what to click/type        │    │
│  │  No explicit selectors needed                      │    │
│  │  Uses: @browserbasehq/stagehand                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  Mode 2: PERSISTENT CONTEXT (Playwright direct)             │
│  ┌────────────────────────────────────────────────────┐    │
│  │  chromium.launchPersistentContext(userDataPath)     │    │
│  │       │                                             │    │
│  │       ▼                                             │    │
│  │  Explicit Playwright API calls                     │    │
│  │  Session persists across restarts                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  Mode 3: CDP REMOTE (attach to user's Chrome)               │
│  ┌────────────────────────────────────────────────────┐    │
│  │  chromium.connectOverCDP(ws://localhost:9222)       │    │
│  │       │                                             │    │
│  │       ▼                                             │    │
│  │  Attaches to user's existing browser               │    │
│  │  Shares cookies, sessions, login state             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  InstancesManager                                            │
│  ├─ Session pool (Map<id, InstanceInfo>)                    │
│  ├─ Auto-detect Chrome/Edge profiles                        │
│  └─ Persistent DB storage (Instances entity)                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Code Execution Sandboxing

```
PYTHON (via uv runtime):                    BASH:
┌─────────────────────────┐                 ┌──────────────────────┐
│ 1. Create temp directory│                 │ child_process.spawn() │
│ 2. uv init + venv      │                 │                       │
│ 3. uv add {packages}   │                 │ Timeout: 120s (max    │
│ 4. Inject sitecustomize │                 │ 600s configurable)    │
│    .py (MCP bridge)    │                 │                       │
│ 5. uv run script.py    │                 │ SIGTERM → 200ms →     │
│ 6. Delete temp dir     │                 │ SIGKILL escalation    │
└─────────────────────────┘                 │                       │
                                            │ Process group kill:   │
MCP bridge in Python:                       │ kill(-pid, signal)    │
  http://localhost:41100/mcp                └──────────────────────┘
  → tools callable from Python
```

## What We Should Do

| Action | Detail |
|--------|--------|
| **Study deeply** | Stagehand integration — LLM-driven browser automation without explicit selectors. This is the next evolution of our MCP Browser's "Agent → Browser" path. |
| **Study with constraints** | CDP remote connection (Mode 3) — attaching to user's existing Chrome. Powerful for operating inside logged-in sessions, but requires privacy guardrails: explicit user consent before attaching, clear indicator when CDP is active, and no capturing of session cookies/credentials beyond the active task. Must align with our anti-surveillance positioning. |
| **Copy pattern** | `BaseTool` / `BaseToolkit` abstraction with Zod schemas. Clean, testable tool contracts. |
| **Copy pattern** | AbortController per tool execution with SIGTERM→SIGKILL escalation for bash. Robust cancellation. |
| **Study** | Python execution via `uv` with MCP bridge injection. If we ever need code execution, this is the pattern. |
| **Skip** | Most of their built-in tools (OCR, image gen, translation, audio). Not relevant for our target user. |
| **Note** | Their tool count (25+) is impressive but bloated. We should expose tools via MCP, not bake them in. |
