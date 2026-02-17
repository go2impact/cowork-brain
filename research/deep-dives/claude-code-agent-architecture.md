# Claude Code Agent Architecture

> How Claude Code achieves "computer use" without actual Computer Use — structured local tool execution via a single-threaded agent loop.

**Date:** 2026-02-17
**Relevance to Cowork.ai:** Agent loop patterns, tool execution model, context management, sub-agent orchestration — all directly applicable to how our Mastra agents should operate.

---

## TL;DR

Claude Code does not use Anthropic's Computer Use API (screenshot + pixel coordinates). For local machine interaction, it uses a **single-threaded while-loop** that calls the Claude API, receives structured tool calls (read file, run bash, edit file), executes them directly on the local OS, feeds results back, and repeats. No pixel coordinates, no virtual display. Just text in, tool call out, execute, repeat. For browser interaction, it uses either a Chrome extension or Playwright MCP — both use structured tools rather than raw screen captures, though the Chrome extension's element identification mechanism is undocumented.

---

## 1. The Agent Loop

The core of Claude Code is a classic agentic while-loop:

```
User prompt
  → Claude API call (message + tool definitions)
    → Claude responds with one or more tool_use blocks
      → Execute tool(s) locally (parallel if independent)
        → Send tool_result(s) back
          → Claude decides: more tool calls, or final text response
            → If tool calls: repeat
            → If text: loop exits, return to user
```

Claude can request **multiple tool calls in a single response**. Independent calls execute in parallel (e.g., reading three files at once). Dependent calls execute sequentially across loop iterations.

The loop has three blended phases per iteration:

| Phase | What happens |
|-------|-------------|
| **Gather** | Read files, search code, pull in context |
| **Act** | Edit files, run commands, create files |
| **Verify** | Run tests, check output, inspect diffs |

These aren't rigid stages. A simple question might only gather. A bug fix cycles through all three repeatedly. Claude decides what each step requires based on what it learned from the previous step, chaining dozens of actions together.

### Why single-threaded?

Anthropic deliberately chose a single loop over multi-agent swarms. The thesis: **a simple loop with disciplined tools and planning delivers controllable autonomy**. One thread means flat message history, easy debugging, full audit trail, and predictable behavior. Complexity comes from tool composition, not orchestration topology.

---

## 2. Tool Execution Model

Claude the model can only output text. Tools are what make it agentic. Claude Code provides **~20 built-in tools** that execute directly on the user's machine, in their shell, with their user's permissions.

### Complete Tool Inventory

> **Note:** Tool names evolve across Claude Code versions. The names below are from a Feb 2026 build. Earlier versions may use different names (e.g., `BashOutput` → `TaskOutput`, `TodoWrite` → `TaskCreate`). The capabilities remain the same.

#### File Operations (5 tools)

| Tool | What it does |
|------|-------------|
| **Read** | Reads file contents from the local filesystem (~2000 lines default). Supports line offsets for large files. Also reads images (multimodal vision), PDFs (with page ranges), and Jupyter notebooks (all cells + outputs). |
| **Edit** | Exact string replacement in a file — requires matching the `old_string` verbatim including whitespace. Fails if the match isn't unique (forces you to provide more context). Has `replace_all` mode for renaming across a file. |
| **Write** | Creates a new file or overwrites an existing file entirely. Requires reading the file first if it already exists. |
| **Glob** | Fast file pattern matching (`**/*.ts`, `src/**/*.tsx`). Returns paths sorted by modification time. Replaces `find` and `ls`. |
| **Grep** | Regex content search powered by ripgrep. Supports file type filters, glob filters, context lines, multiple output modes (`content`, `files_with_matches`, `count`). Replaces `grep`/`rg`. |

#### Shell Execution (1 tool)

| Tool | What it does |
|------|-------------|
| **Bash** | Runs commands in the user's actual terminal environment. Has access to anything on the command line: git, npm, docker, python, build tools, system utilities. Optional timeout (up to 10 min). Can run in background via `run_in_background`. |

Key Bash constraints:
- Working directory **persists** between calls
- Shell state (env vars, aliases, background processes) **does not persist** — each call gets a fresh shell
- Dangerous commands (`rm -rf`, `git push --force`, `reset --hard`) require explicit user confirmation

#### Notebook Operations (1 tool)

| Tool | What it does |
|------|-------------|
| **NotebookEdit** | Replaces, inserts, or deletes a specific cell in a Jupyter notebook. Operates by cell index (0-based). Supports `code` and `markdown` cell types. (Reading notebooks is handled by the Read tool.) |

#### Web Access (2 tools)

| Tool | What it does |
|------|-------------|
| **WebSearch** | Web search via search API. Returns result snippets with links. Supports domain allow/block lists. |
| **WebFetch** | Fetches a URL, converts HTML to markdown, processes the content with a smaller/faster model using a user-provided prompt. Has a 15-minute self-cleaning cache. |

#### Planning & Modes (2 tools)

| Tool | What it does |
|------|-------------|
| **EnterPlanMode** | Transitions Claude into plan mode — read-only tools only. Used before non-trivial implementation tasks so Claude can explore the codebase and design an approach for user approval. |
| **ExitPlanMode** | Signals that Claude has finished writing a plan and is ready for user approval. Transitions from read-only plan mode to implementation mode. |

#### Task Management (4 tools)

| Tool | What it does |
|------|-------------|
| **TaskCreate** | Creates a structured task with subject, description, and active-form text (shown in a spinner while in progress). Tasks render as interactive checklists in the UI. |
| **TaskGet** | Retrieves a task by ID — returns full details including description, status, and dependency graph (blocks/blockedBy). |
| **TaskUpdate** | Updates a task's status (`pending` → `in_progress` → `completed`), description, ownership, or dependencies. Also used to delete tasks. |
| **TaskList** | Lists all tasks in summary form — IDs, subjects, statuses, owners, blockers. Used to find the next available task. |

#### Background Task Management (2 tools)

| Tool | What it does |
|------|-------------|
| **TaskOutput** | Retrieves output from a running or completed background task (background Bash shell, background sub-agent, or remote session). Supports blocking and non-blocking modes. |
| **TaskStop** | Terminates a running background task by ID. Used to stop long-running processes (dev servers, watch modes, background agents). |

#### User Interaction (2 tools)

| Tool | What it does |
|------|-------------|
| **AskUserQuestion** | Pauses the loop, presents a multiple-choice question (2-4 options per question, up to 4 questions) to the user, resumes with their selection. Supports multi-select. Users can always type a custom "Other" response. |
| **Skill** | Executes user-invocable slash commands (`/commit`, `/review-pr`, etc.) — skills that expand into full prompts with specialized behavior. |

#### Sub-Agent Orchestration (1 tool)

| Tool | What it does |
|------|-------------|
| **Task** | Spawns a child Claude instance with its own fresh context window. Multiple specialized agent types available (Bash, Explore, Plan, code-reviewer, etc.). Most agent types explicitly exclude the Task tool, preventing recursive sub-agent spawning. Results return as a summary to the parent. Can run in background. |

### Tool Count Summary

| Category | Count | Tools |
|----------|-------|-------|
| File operations | 5 | Read, Edit, Write, Glob, Grep |
| Shell execution | 1 | Bash |
| Notebook operations | 1 | NotebookEdit |
| Web access | 2 | WebSearch, WebFetch |
| Planning & modes | 2 | EnterPlanMode, ExitPlanMode |
| Task management | 4 | TaskCreate, TaskGet, TaskUpdate, TaskList |
| Background task mgmt | 2 | TaskOutput, TaskStop |
| User interaction | 2 | AskUserQuestion, Skill |
| Sub-agent orchestration | 1 | Task |
| **Total** | **20** | |

### Under the Hood

Every tool call follows the standard Claude API `tool_use` contract:

1. Claude Code sends the user message + all tool definitions to the Claude API
2. Claude responds with `stop_reason: "tool_use"` and structured input:
   ```json
   {
     "tool": "Bash",
     "input": { "command": "npm test" }
   }
   ```
3. Claude Code executes that tool **locally on the OS**
4. The result (stdout, file contents, error output) goes back as a `tool_result` message
5. Claude sees the result → decides next action

This is the same API pattern as the Computer Use tool, but the tools are **structured local operations** rather than screenshot + pixel coordinates.

---

## 3. Real-Time Steering

Users can type corrections mid-execution without restarting. An asynchronous queue system allows:

- Injecting new instructions while Claude is actively working
- Pause/resume functionality
- Mid-task course correction without losing progress

This makes Claude Code interactive/streaming rather than batch-oriented. The user is part of the loop — they can interrupt at any point to steer Claude in a different direction.

> **Source note:** Third-party reverse-engineering (PromptLayer) identified internal codenames "h2A" for the steering queue and "nO" for the main loop by analyzing minified code. These are not official Anthropic terminology.

---

## 4. Sub-Agent Spawning

The **Task** tool spawns child Claude instances for parallel or isolated work:

- Each sub-agent gets its **own fresh context window** (separate from the parent's)
- Sub-agents **can use tools** but most agent types **explicitly exclude the Task tool**, preventing recursive sub-agent spawning
- Results return as a summary to the parent loop
- The parent's context window stays clean — sub-agent work doesn't bloat it

Use cases: parallel research queries, long file analysis that would consume too much parent context, specialized agent types (code review, exploration, planning).

---

## 5. Context Management

### The Context Window

The context window holds: system instructions, CLAUDE.md contents, conversation history, tool results (file contents, terminal output), loaded skill definitions, and MCP tool definitions. Window size depends on the model (e.g., 200K tokens for Claude Sonnet 4.5).

### Auto-Compaction

As the context window fills, auto-compaction triggers:

1. **First pass:** Clears older tool outputs (file reads, command results)
2. **Second pass:** Summarizes conversation history if still over budget
3. **Preserved:** Recent user requests, key code snippets, system instructions

Persistent instructions belong in `CLAUDE.md` — they survive compaction. In-conversation instructions from early in a session may be lost.

> **Source note:** Third-party reverse-engineering (PromptLayer) claims compaction triggers at ~92% utilization and identified the internal codename "wU2" for the compressor. Official docs simply say "Claude Code manages context automatically as you approach the limit."

### Long-Term Memory

- **CLAUDE.md** — Project-specific instructions loaded every session
- **Auto memory** — Claude can write notes to `~/.claude/projects/` that persist across sessions
- **Skills** — Loaded on-demand to avoid context waste

---

## 6. Safety & Permissions

### Permission Modes (Shift+Tab cycles)

| Mode | Behavior |
|------|----------|
| **Default** | Asks before file edits and shell commands |
| **Auto-accept edits** | Edits files freely, still asks for commands |
| **Plan mode** | Read-only tools only — produces a plan for approval |
| **Delegate mode** | Coordinates work through agent teammates only, no direct implementation. Only available when an agent team is active. |

### Checkpoints

Before every file edit, Claude Code snapshots the file. Users can rewind with `Esc Esc`. This is local to the session, separate from git. Only covers file changes — actions affecting remote systems (databases, APIs, deployments) can't be checkpointed.

### Command Allow-Lists

Trusted commands (e.g., `npm test`, `git status`) can be pre-approved in `.claude/settings.json` so Claude doesn't ask each time. Settings scope from org-wide policies down to personal preferences.

---

## 7. Browser Use (Not Computer Use)

When Claude Code needs a browser, it has two options — neither uses the raw Computer Use API:

### Option A: Claude in Chrome Extension

- Communicates via Chrome **Native Messaging** protocol (bidirectional pipe between terminal process and Chrome extension)
- Opens tabs in the user's **real browser** with their login state and cookies
- Uses structured browser tools exposed via MCP (click, type, navigate, read console, take screenshots, record GIFs)
- Exact element identification mechanism is undocumented — may combine vision and DOM access
- Pauses for login pages and CAPTCHAs

### Option B: Playwright MCP Server

- Launches a Playwright-controlled Chrome window
- Uses **accessibility tree snapshots** (~2-5KB structured text) instead of screenshots (~500KB-2MB images)
- Claude targets elements by semantic `ref` attributes, not pixel coordinates
- ~20-25 tools (varies by version): `browser_click`, `browser_type`, `browser_navigate`, `browser_snapshot`, etc.
- Far more token-efficient than vision-based approaches

### Comparison with Raw Computer Use API

| Aspect | Computer Use API | Playwright MCP | Chrome Extension |
|--------|-----------------|----------------|-----------------|
| **Input** | Screenshots (base64 images) | Accessibility tree (structured text) | Undocumented (likely vision + DOM) |
| **Targeting** | Pixel coordinates `[x, y]` | Semantic element refs | Structured tool parameters |
| **Environment** | Sandboxed VM / Docker | Playwright-controlled Chrome | User's real Chrome with login state |
| **Agent loop** | You build it | Built into Claude Code | Built into Claude Code |
| **Token cost** | High (vision tokens) | Low (~2-5KB text per snapshot) | Unknown |
| **Setup** | Docker + Xvfb + implementation | `claude mcp add playwright ...` | Install Chrome extension + `--chrome` |

---

## 8. Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Bun |
| Language | TypeScript |
| UI framework | React + Ink (terminal React renderer) |
| Layout engine | Yoga (Flexbox for terminal) |
| API | Claude Messages API with tool_use |
| File search | ripgrep (via Grep tool) |
| File matching | glob patterns (via Glob tool) |

90% of Claude Code's own code was written by Claude Code itself.

---

## 9. Relevance to Cowork.ai

Patterns we should study and potentially adopt:

| Pattern | Application |
|---------|------------|
| **Single-threaded agent loop** | Our Mastra agents could follow the same think-act-observe-repeat pattern rather than complex multi-agent orchestration |
| **Structured tool execution** | MCP tools for Zendesk/Gmail/Slack follow the same contract — tool definitions sent with each request, tool_use responses, local execution, results fed back |
| **Sub-agent isolation** | Fresh context windows for sub-tasks prevent context bloat in long sessions — relevant for our Electron utility process agent architecture |
| **Real-time steering** | Users should be able to interrupt and redirect agents mid-task, not just at turn boundaries |
| **Context compaction** | Auto-summarize as context fills — critical for long-running sidecar sessions |
| **Checkpoint/rewind** | Snapshot before destructive actions, allow undo — applies to our capture/flush and agent action patterns |
| **Permission modes** | Graduated autonomy (ask everything → auto-accept reads → auto-accept writes → full auto) maps directly to our trust model |

---

## Sources

- [How Claude Code works — Official Docs](https://code.claude.com/docs/en/how-claude-code-works)
- [Claude Code: Behind-the-scenes of the master agent loop — PromptLayer](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/)
- [How does Claude Code actually work? — Rubric Labs](https://rubriclabs.com/blog/how-does-claude-code-actually-work)
- [Claude Code Agent Architecture — ZenML](https://www.zenml.io/llmops-database/claude-code-agent-architecture-single-threaded-master-loop-for-autonomous-coding)
- [Computer Use Tool — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- [Use Claude Code with Chrome — Official Docs](https://code.claude.com/docs/en/chrome)
- [Playwright MCP with Claude Code — Simon Willison](https://til.simonwillison.net/claude-code/playwright-mcp-claude-code)
- [Anthropic's Computer Use vs OpenAI's CUA — WorkOS](https://workos.com/blog/anthropics-computer-use-versus-openais-computer-using-agent-cua)

---

## Changelog

| Date | Change | Author |
|------|--------|--------|
| 2026-02-17 | Initial deep-dive | Rustan |
| 2026-02-17 | Fact-check pass 1: corrected tool names to Feb 2026 build, removed phantom tools, added 7 missing tools, attributed reverse-engineered codenames as unofficial, softened unverified claims, added Delegate permission mode | Rustan |
| 2026-02-17 | Fact-check pass 2: scoped TL;DR "no vision" to local machine only, added parallel tool calls to agent loop, softened sub-agent recursion claim, fixed 92% figure inconsistency in relevance section, split browser comparison table into 3 columns (Computer Use API vs Playwright MCP vs Chrome Extension) | Rustan |
