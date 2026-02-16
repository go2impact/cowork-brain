# AIME Chat → Cowork.ai Adaptation Guide

**Purpose:** Maps each section of the [AIME Chat deep-dive](../deep-dives/aime-chat/README.md) to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our multi-process architecture.

**Audience:** Engineering (Rustan + team)

**Source material:** `research/deep-dives/aime-chat/` (reverse-engineering of AIME Chat's codebase)
**Target architecture:** [`architecture/system-architecture.md`](../../architecture/system-architecture.md)

---

## Sections

| # | AIME Section | Status |
|---|---|---|
| 0 | [Vercel AI SDK](#0-vercel-ai-sdk) | Done |
| 1 | [Mastra Agent System](#1-mastra-agent-system) | Done |
| 2 | [RAG & Knowledge Base](#2-rag--knowledge-base) | Done |
| 3 | [MCP Integration](#3-mcp-integration) | Done |
| 4 | [Tool System](#4-tool-system) | Done |
| 5 | [AI Provider Support](#5-ai-provider-support) | Done |
| 6 | [Electron Architecture](#6-electron-architecture) | Done |
| 7 | [UI & i18n](#7-ui--i18n) | Done |
| 8 | [Architecture Comparison](#8-architecture-comparison) | Done |
| 9 | [Summary](#9-summary) | Done |

---

## 0. Vercel AI SDK

**Source:** [`research/deep-dives/aime-chat/00-vercel-ai-sdk.md`](../deep-dives/aime-chat/00-vercel-ai-sdk.md)

AI SDK is the interface contract for everything — agents, providers, streaming, frontend hooks. AIME and Cowork.ai both build on it. The differences come from *where* things run (their process model vs. ours) and *which version* we target.

### Copy

#### IPC Chat Transport (`IpcChatTransport`)

AIME's most valuable AI SDK pattern. A custom `ChatTransport<UIMessage>` implementation that bridges `useChat()` across the Electron IPC boundary.

**AIME's flow:**
```
Renderer (useChat + IpcChatTransport)
    ↕ IPC
Main process (Mastra agent → toAISdkFormat → IPC emit)
```

**Our flow — extra hop because agents run in a utility process:**
```
Renderer (useChat + IpcChatTransport)
    ↕ IPC
Main process (IPC router — no agent logic)
    ↕ IPC
Agents & RAG utility process (Mastra agent → toAISdkFormat)
```

What changes:
- **Main process becomes a passthrough.** AIME's main process calls `agent.stream()` directly. Ours routes the message to the utility process and relays chunks back. Main adds zero logic — just forwards.
- **Stream origin shifts.** In AIME, `appManager.sendEvent()` fires from the same process running the agent. In our architecture, the utility process generates chunks and sends them to main via `MessagePort`, then main relays to the renderer. The chunk format is identical — only the transport hops change.
- **Error surface grows.** Two IPC boundaries instead of one. If the utility process crashes mid-stream, main needs to detect the broken pipe and send an `error` chunk to the renderer. AIME doesn't have this problem because the agent and IPC sender are in the same process.

What stays the same:
- `IpcChatTransport` implements `ChatTransport<UIMessage>` — same interface, same hook integration.
- Renderer-side code is identical. `useChat()` doesn't know or care that chunks travel through an extra process.
- SSE-formatted JSON chunks (`"data: ${JSON.stringify(chunk)}\n\n"`) — same encoding.

**Reference:** [system-architecture.md — IPC Contract](../../architecture/system-architecture.md#ipc-contract)

#### `@ai-sdk/openai-compatible` for Ollama

One-liner to connect to local brain. Directly reusable.

```
createOpenAICompatible({
  baseURL: 'http://localhost:11434',
  name: 'ollama',
}).languageModel('deepseek-r1:8b')
```

AIME uses this same pattern for Ollama, LMStudio, and other OpenAI-compatible endpoints. We only need Ollama for v0.1 (DeepSeek-R1-8B local brain). This call lives in the Agents & RAG utility process, not main.

#### Zod-validated chunk protocol

AIME validates every streaming chunk against a Zod schema (`uiMessageChunkSchema`). This gives a typed contract between main and renderer — malformed chunks fail loudly instead of corrupting UI state.

We adopt this directly. The chunk types we use are a subset of AIME's:

| Category | AIME chunk types | We use? | Notes |
|---|---|---|---|
| **Text** | `text-start`, `text-delta`, `text-end` | Yes | Core streaming |
| **Reasoning** | `reasoning-start`, `reasoning-delta`, `reasoning-end` | Yes | DeepSeek extended thinking — AIME disables this (`sendReasoning: false`), we may want it for transparency |
| **Tool lifecycle** | `tool-input-start/delta/available/error` | Yes | Agent tool execution |
| **Tool approval** | `tool-approval-requested`, `tool-call-approval`, `tool-output-*` | Yes | Human-in-the-loop safety rails |
| **Sources** | `source-url`, `source-document` | Yes | RAG source attribution |
| **Error/File** | `error`, `file` | Yes | Error propagation, attachments |
| **Meta** | `start-step`, `finish-step`, `start`, `finish`, `abort`, `message-metadata` | Yes | Stream lifecycle |
| **Custom data** | `data-usage` | Yes | Token tracking for budget enforcement |
| **Custom data** | `data-compress-start`, `data-compress-end` | No | AIME's context compression — we handle this differently via observational memory |

#### AI SDK v6 (not v5)

AIME uses `ai` ^5.0.93. We start on v6 (shipped Dec 2025, currently 6.0.86). Breaking changes are mechanical renames with an automated codemod (`npx @ai-sdk/codemod v6`):

| v5 | v6 |
|---|---|
| `CoreMessage` | `ModelMessage` |
| `textEmbeddingModel()` | `embeddingModel()` |
| `@ai-sdk/react` v2 | `@ai-sdk/react` v3 |
| `generateObject()` separate | Merged into `generateText()` with `output` setting |

Mastra 1.0+ supports both v5 and v6 natively (npm aliasing: `@ai-sdk/provider-v5` + `@ai-sdk/provider-v6` bundled internally). The backend is transparent. Our direct dependency is `@ai-sdk/react` v3 in the renderer.

See [decision log](../../decisions/decision-log.md#2026-02-16--ai-sdk-version-v6-not-v5).

#### `embedMany()` for embeddings

AIME's pattern: `embedMany({ model, values })` → float arrays → store in DB.

We use the same AI SDK primitive, but with different plumbing:
- **Model:** Qwen3-Embedding-0.6B via Ollama (not OpenAI's embedding model)
- **Storage:** libsql built-in vector search (`F32_BLOB` + `vector_distance_cos`), managed by `@mastra/libsql`'s `LibSQLVector`
- **Process:** Runs in Agents & RAG utility process, not main

The `embedMany()` call itself is identical. What feeds it (capture data + conversation history) and where vectors land (libsql, not a separate vector DB) are our differences.

#### Tool definitions with Zod schemas

AIME's `BaseTool` pattern: `tool()` with Zod `inputSchema`, optional `outputSchema`, `requireApproval` flag for human-in-the-loop.

We adopt this directly. Our tools connect to MCP servers (Zendesk, Gmail, Slack) and Playwright — different tools, same contract. The `requireApproval` pattern maps to our safety rails (destructive actions always confirm, money actions always approve).

### Study

#### `toAISdkFormat()` from `@mastra/ai-sdk`

The bridge between Mastra agent streams and AI SDK chunk types. This is internal to Mastra — we don't write it, but we need to understand it for debugging.

When a Mastra agent calls `agent.stream()`, the raw output is Mastra's internal format. `toAISdkFormat({ from: 'agent' })` converts it to the typed chunks that `useChat()` consumes. If streaming breaks, this is the first place to look.

Key question: does `toAISdkFormat()` work the same in a utility process? It should — it's a pure stream transform with no process-model assumptions. But this needs verification during the Mastra-in-utility-process spike.

#### AI SDK `gateway` API

Imported but unused by AIME. Provides a unified interface for multiple providers behind a single gateway.

**Evaluate for:** Our complexity router could use `gateway` to abstract the local/cloud decision. Instead of `getLanguageModel("ollama/deepseek-r1:8b")` vs. `getLanguageModel("openrouter/google/gemini-2.5-flash")`, the router could resolve to a provider behind a gateway, keeping the calling code clean.

**Likely skip:** Our complexity router is rule-based and explicit (<10ms, no ML). A gateway abstraction might hide the routing decision when we want it visible for logging and debugging. Evaluate during implementation, but default to explicit provider resolution.

#### `generateObject()` / `streamObject()` for structured output

AIME doesn't use these — they do structured output via Zod tool schemas instead. In v6, these merge into `generateText()` / `streamText()` with an `output` setting.

**Evaluate for:** Complexity router's structured analysis of user intent. When a message arrives, the router needs to classify: simple vs. complex, tools needed, estimated tokens. `generateText({ output: zodSchema })` could provide this classification with type safety — but only if the classification step itself uses an LLM (which would blow our <10ms budget).

**Likely skip for v0.1:** Our complexity router is rule-based, not LLM-based. Structured output is useful if we later add an LLM-based intent classifier, but that's not in scope.

### Skip

#### Provider management UI

AIME has UI for entering API keys, selecting models per provider, toggling providers on/off. Users directly choose which model to use.

**Why skip:** Cowork.ai abstracts this entirely. Users pick a tier (Free / Boost / Pro / Max), and the complexity router picks the model. Local users don't configure Ollama through UI — the app manages the local brain automatically. The only keys users enter are: Gemini API key (free tier, entered at onboarding) and OpenRouter API key (paid tiers). No model selection, no provider toggling.

#### `ProvidersManager` pattern

AIME's custom provider registry that resolves `"providerId/modelId"` → `LanguageModelV2`. It manages multiple providers with different API keys, health states, model lists.

**Why skip:** We have three providers — Ollama (local, no key), Gemini direct (free tier, one key), and OpenRouter (paid tiers, one key) — managed via AI SDK's `createProviderRegistry()` (see [Section 5](#5-ai-provider-support)). That's a three-line registry, not a custom base class. We don't need a generic provider manager with health checking, model list fetching, and credit balance tracking for three known providers.

#### Dead `generateText()` imports

AIME has `generateText` imported in ~24 tool files with zero call sites. Dead code from a shared template. Obviously skip.

---

## 1. Mastra Agent System

**Source:** [`research/deep-dives/aime-chat/01-mastra-agent-system.md`](../deep-dives/aime-chat/01-mastra-agent-system.md)

AIME's agent system is a coding assistant — one user-facing agent (CodeAgent) with sub-agents for codebase exploration and planning. Cowork.ai is a work sidecar — a single platform agent that handles heterogeneous tasks (email, tickets, scheduling) across many MCP services. The agent patterns differ fundamentally, but the execution infrastructure is directly reusable.

### Copy

#### Context compression — use Mastra's Observational Memory, not AIME's custom approach

AIME built a custom `CompressAgent` (~100+ lines) that triggers at 70% context, runs a synchronous LLM call to summarize into 9 sections, and replaces original messages. This was necessary because AIME uses Mastra with memory features mostly disabled.

**We don't need to copy this.** Mastra ships built-in **Observational Memory (OM)** in `@mastra/memory@1.1.0` that does this better:

| | AIME (custom) | Mastra OM (built-in) |
|---|---|---|
| Setup | Custom agent + manager code | `observationalMemory: true` in Memory config |
| Trigger | 70% threshold, synchronous (blocks conversation) | Configurable token threshold (default 30k), **async buffering** by default |
| Compression | Single LLM pass → one summary | Two-stage: Observer → observations, Reflector → reflections (5-40x compression) |
| Benchmark | N/A | 94.87% on LongMemEval (SOTA) |
| Storage | Custom soft-delete to `.history` resource | Handled internally by Memory storage adapter |

Minimal config:
```typescript
new Memory({
  options: {
    observationalMemory: true, // defaults to gemini-2.5-flash
  },
})
```

What we configure:
- **Observer/Reflector model:** Default is `gemini-2.5-flash` (1M context helps). Docs warn Claude 4.5 models perform poorly here. **Open question:** can DeepSeek-R1-8B serve as Observer locally, or must this go through OpenRouter? Needs testing.
- **`scope: "resource"`** (experimental): Shares observations across conversation threads for the same user. Directly aligned with our sidecar model — the agent's understanding of a user persists across separate conversations. Tradeoff: disables async buffering.
- **Token thresholds:** Default 30k for observation trigger, 40k for reflection trigger. May need tuning for our 8K local context (DeepSeek-R1-8B) vs. larger cloud contexts.

This maps to our memory system's third layer (observational memory). The two-stage Observer→Reflector pipeline is exactly the "background agents compress raw history → dense logs" flow described in [system-architecture.md — Memory System](../../architecture/system-architecture.md#memory-system). We get it for free.

#### Dynamic system prompt generation

AIME builds agent instructions as functions, not static strings. CodeAgent's prompt includes runtime context: workspace path, git status, platform, available tools, and database tool documentation — all injected at `buildAgent()` time.

We adopt this pattern. Our system prompt includes:
- Active MCP connections and their available tools
- User's working memory (profile, preferences, communication style)
- Current capture context (active app, recent activity summary)
- Tier/budget status (remaining tokens, active spend caps)
- Available execution paths (local brain available? Playwright running?)

The difference: AIME's context is developer-oriented (workspace, git). Ours is work-oriented (services, activity, budget). Same mechanism, different payload.

#### Tool approval flow (suspended → approve/decline → resume)

When an agent calls a tool with `requireApproval: true`, the stream enters `suspended` status. The UI shows an approval dialog. The user approves or declines. The stream resumes.

Maps directly to our safety rails:
- **Destructive MCP actions** (delete ticket, send email, post to Slack) → approval required
- **Money actions** (anything with cost implications) → always explicit approval
- **MCP Browser actions** (clicking Send, submitting forms) → approval gates

The chunk types (`tool-approval-requested`, `tool-call-approval`, `tool-output-available`, `tool-output-denied`) are part of the AI SDK protocol we already adopt in Section 0.

#### Multi-step execution loop

AIME's `while(true)` loop: stream agent response → check status → if tool result, loop again → if assistant text, done → if suspended, wait for approval. This handles multi-tool chains where the agent calls Tool A, gets a result, calls Tool B based on that result, etc.

We need this for any non-trivial agent task. A Zendesk ticket reply involves: read ticket (tool call) → read customer history (tool call) → draft reply (generation) → user reviews → paste in browser (tool call). That's multiple loop iterations.

What changes: the loop runs in our Agents & RAG utility process, not main. Each iteration's chunks are forwarded through main to the renderer. The loop logic itself is identical — it's Mastra's built-in execution model.

### Adapt

#### Sub-agent delegation (Task tool pattern)

AIME's most architecturally interesting pattern. CodeAgent delegates to Explore/Plan sub-agents to protect its context window. Sub-agents run in their own context, do heavy lifting (dozens of file reads), and return a summary. Raw file contents never pollute the parent's context.

**The context isolation principle is valuable. The specific agent types are not.**

What we keep:
- **Context isolation via disposable sub-agents.** When our platform agent needs to analyze 50 Zendesk tickets to find a pattern, it should delegate to a sub-agent rather than loading all 50 into its own context. The sub-agent reads the tickets, identifies the pattern, returns a summary.
- **Dynamic description generation.** The Task tool auto-describes available sub-agents to the parent. Adding a new specialist only requires registration — the prompt updates automatically.
- **Stateless spawning.** Each sub-agent invocation is independent. No follow-up messages, no memory of previous calls. Clean and predictable.
- **Step-result streaming.** Sub-agent progress streams back to the UI in real-time via `data-task-{toolCallId}` events. Users see work happening, not a black box.

What changes:
- **Different specialists.** AIME has Explore (code search) and Plan (architecture). We'd have specialists aligned to our features — e.g., a ticket analysis sub-agent, an email batch processor, a context summarizer. These emerge from actual usage patterns, not upfront design.
- **Process boundary.** AIME's sub-agents run in the same main process as CodeAgent. Ours run in the Agents & RAG utility process — all sub-agent spawning is internal to that process, no extra IPC hop.

#### Plan mode → "Explore before acting" for MCP Browser

AIME's plan mode forces a structured exploration phase before code modification: explore → ask user → plan → write plan file → get approval → execute.

We don't need a formal "plan mode" for chat — our complexity router handles execution strategy. But the concept of **forced exploration before action** maps to MCP Browser:

Before the agent takes visible browser actions (clicking buttons, filling forms, submitting), it should:
1. Read the page state via Playwright (explore)
2. Identify the target elements and current form state (understand)
3. Propose the action sequence to the user (plan)
4. Execute with approval gates (act)

This isn't a mode toggle like AIME's — it's built into the MCP Browser execution flow. Every browser action sequence starts with exploration. The safety rails (Section "Agent Orchestration" in system-architecture.md) already require approval for destructive visible actions.

#### Memory configuration — we use what AIME disables

AIME configures Mastra Memory but disables most features:
- `semanticRecall: false` — no vector-based memory retrieval
- `workingMemory: { enabled: false }` — no persistent user profile
- `lastMessages: false` — custom message loading instead
- `generateTitle: false` — handled separately

We enable all of these. Our four-layer memory system (conversation history, working memory, semantic recall, observational memory) maps to Mastra's Memory primitives:

| Mastra primitive | AIME setting | Our setting | Our use |
|---|---|---|---|
| `semanticRecall` | Disabled | **Enabled** | RAG retrieval from embedded conversations + capture data |
| `workingMemory` | Disabled | **Enabled** | Persistent user profile, preferences, communication style |
| `lastMessages` | Disabled (custom) | **Enabled** | Recent N messages for short-term continuity |
| `vector` (LibSQLVector) | Configured but unused | **Active** | Qwen3-0.6B embeddings stored in libsql |
| `MessageHistory` output processor | Enabled | **Enabled** | Persist messages to cowork.db |

This is the biggest behavioral difference between AIME's agents and ours. AIME's agents are stateless between sessions — memory is just message history in a thread. Our agent builds a persistent understanding of the user over time.

### Skip

#### Agent types (CodeAgent, Explore, Plan, DefaultAgent, etc.)

AIME's agents are specialized for coding: CodeAgent writes code, Explore searches codebases, Plan designs implementations. These are the wrong abstractions for a work sidecar.

We have a single platform agent that handles all user-facing interaction. Specialization happens at the tool level (MCP services, browser automation) and routing level (complexity router picks the model), not the agent level. If we need context isolation, we use the sub-agent delegation pattern (adapted above) with work-oriented specialists, not code-oriented ones.

#### Agent selection UI

AIME lets users pick an agent from a dropdown (though only CodeAgent is visible). We don't expose agent selection at all. The user types a message; the complexity router decides what happens. The concept of "picking an agent" doesn't exist in our UX.

#### Express HTTP server for MCP bridge

AIME runs an Express server on port 41100 inside the main process for MCP communication. We don't need this — our MCP connections are managed directly by the Agents & RAG utility process via `@mastra/mcp`. No HTTP server inside Electron.

#### Dead code (SkillCreator agent, dead `generateText` imports)

Not relevant.

## 2. RAG & Knowledge Base

**Source:** [`research/deep-dives/aime-chat/02-rag-knowledge-base.md`](../deep-dives/aime-chat/02-rag-knowledge-base.md)

AIME's RAG is a manual document upload system — users add PDFs, DOCX files, URLs, and images to a knowledge base. Cowork.ai's RAG is ambient — context arrives automatically from capture (window activity, keystrokes, clipboard) and conversation history. Same vector infrastructure, fundamentally different input source.

### Copy

#### Background embedding task queue

AIME uses `TaskQueueManager` with `groupMaxConcurrency: 1` per knowledge base to serialize embedding jobs. This prevents parallel embedding calls from overwhelming the GPU/API.

We need the same pattern for a different reason: our capture process generates a continuous stream of embedable content (window titles, URLs, clipboard snippets, keystroke chunks). Without a queue, burst activity (rapid window switching, heavy typing) could flood the embedding pipeline. Single-concurrency queue per input stream keeps GPU utilization steady.

This runs in the Agents & RAG utility process. The capture process writes raw data to cowork.db; the embedding queue picks it up asynchronously.

### Adapt

#### Vector storage — use Mastra's `LibSQLVector`, not raw SQL

AIME manages vector tables manually with custom SQL — creating `[kb_{id}_{dim}]` tables, inserting with `vector32()`, querying with `vector_distance_cos()`. Their `KnowledgeBaseManager` is 531 lines of hand-rolled vector operations.

We don't need this. `@mastra/libsql`'s `LibSQLVector` abstracts vector table creation, insertion, and similarity search. It uses the same libsql primitives under the hood (`F32_BLOB`, `vector_distance_cos`) but we never write raw vector SQL.

What we configure on `LibSQLVector`:
- Embedding dimensions (512–1024, matching Qwen3-Embedding-0.6B output)
- Distance metric (cosine similarity — same as AIME)
- Top-K and similarity threshold for retrieval

What AIME does manually that LibSQLVector handles:
- Table creation per embedding dimension
- `F32_BLOB` column management
- Cosine similarity query construction
- Soft-delete filtering (`is_enable` flag)

#### Ingestion pipeline — ambient capture replaces document upload

AIME's pipeline: User uploads file → parse format → chunk text → embed → store vectors.

Our pipeline: Capture process writes raw activity → embedding queue picks up → chunk → embed → store vectors.

| Pipeline stage | AIME | Cowork.ai |
|---|---|---|
| **Input** | Manual file upload (PDF, DOCX, XLSX, images, URLs) | Ambient capture (window titles, URLs, keystroke chunks, clipboard, conversation history) |
| **Parsing** | Format-specific parsers (pdf-parse, mammoth, xlsx, officeparser, OCR) | No parsing needed — capture data is already text |
| **Chunking** | `@mastra/rag` v2: `MDocument.fromText().chunk()` (512 chars, 50 overlap) | Same library, different chunk sizes — activity data is shorter and more structured than documents |
| **Embedding** | `embedMany()` via AI SDK (provider-configurable) | `embedMany()` via Qwen3-Embedding-0.6B (Ollama, local) |
| **Storage** | Custom SQL against libsql | `LibSQLVector` from `@mastra/libsql` |
| **Process** | Main process | Agents & RAG utility process |

#### Chunking parameters need benchmarking

AIME uses 512-char chunks with 50-char overlap — conservative defaults for document text. Our activity data has different structure:

- **Window titles:** Short (10-80 chars). Likely one chunk per title, no splitting needed.
- **URLs:** Short. One chunk each.
- **Keystroke chunks:** Variable (up to 1000 chars per flush). May chunk well at 512.
- **Clipboard snippets:** Variable. Could be anything from a URL to a full email body.
- **Conversation messages:** Variable. AI SDK message boundaries are natural chunk points.

The 512/50 defaults are a starting point, but we should benchmark retrieval quality with activity data specifically. The `@mastra/rag` v2 chunking library supports multiple strategies (markdown, recursive) — recursive with smaller chunks may work better for our heterogeneous input.

### Study

#### Reranker pass

AIME has an optional reranker after cosine similarity retrieval. A rerank model rescores the top-K results for relevance, improving precision.

Worth evaluating for our RAG. When the agent assembles context from mixed sources (window titles + conversation history + clipboard), a reranker could prioritize the most relevant fragments. But this adds latency and requires a rerank model (either local or API). Evaluate during implementation — not needed for v0.1.

#### Query construction

AIME's RAG query is straightforward: embed the query, cosine similarity search, threshold at 0.5, top-K = 10. Our retrieval is more complex because we query across multiple data types:

- Embedded activity data (window titles, URLs)
- Embedded conversation history
- Embedded observational summaries (from Mastra OM)

Whether we run one unified vector search or separate searches per data type (with result merging) depends on how `LibSQLVector` organizes tables. Study Mastra's vector namespace/index architecture during implementation.

### Skip

#### Document parsers

AIME includes parsers for PDF (`pdf-parse`), Word (`mammoth`, `word-extractor`), Excel (`xlsx`), PowerPoint (`officeparser`), and images (`@napi-rs/system-ocr`). These support their manual upload feature.

We don't ingest documents. Our input is text from capture and conversations — no format parsing needed. If we later add document understanding (e.g., user shares a PDF in chat), we can add parsers then. Not in v0.1 scope.

#### Knowledge base management UI

AIME has UI for creating knowledge bases, uploading files, toggling items on/off, viewing chunk counts. We don't need any of this — our context is captured automatically, and the Context Card in the renderer shows what the AI observes (not what the user uploaded).

#### Per-KB vector tables

AIME creates separate vector tables per knowledge base (`kb_{kbId}_{dim}`). This makes sense for user-managed document collections. We don't have user-managed KBs — our vectors are organized by data type and time, managed by Mastra's `LibSQLVector`.

## 3. MCP Integration

**Source:** [`research/deep-dives/aime-chat/03-mcp-integration.md`](../deep-dives/aime-chat/03-mcp-integration.md)

AIME is an MCP client that connects to external servers, plus it exposes its own tools as an MCP server via Express. Cowork.ai does both of these, but our MCP server role is fundamentally different — we provide platform capabilities to third-party apps (Google AI Studio apps) with scoped permissions, not just internal tool access.

### Copy

#### MCPClient connection management

AIME tracks MCPClient instances in an array with status lifecycle: `starting` → `running` → `stopped` / `error`. Includes reconnect logic and toggle on/off per server.

We adopt this directly for our MCP Integrations feature. Each service connection (Zendesk, Gmail, Slack) is an MCPClient with health tracking. The status maps to our connection health UI:

| AIME status | Our UX |
|---|---|
| `starting` | "Connecting to Zendesk..." |
| `running` | Green dot, tools available |
| `stopped` | Grey dot, user toggled off |
| `error` | Red dot + error message + reconnect action |

What changes: MCPClient instances live in the Agents & RAG utility process, not main. The renderer gets status updates via IPC through main. Connection health polling and reconnect logic are internal to the utility process.

**Reference:** [system-architecture.md — MCP Connections](../../architecture/system-architecture.md#mcp-connections)

#### OAuth credential storage

AIME stores OAuth credentials per MCP server in `{userData}/.mcp/`:
- `{serverUrlHash}_client_info.json` — OAuth client metadata
- `{serverUrlHash}_tokens.json` — access/refresh tokens
- `{serverUrlHash}_code_verifier.txt` — PKCE verifier

Implements `OAuthClientProvider` from `@modelcontextprotocol/sdk/client/auth.js`. Opens browser for login, stores tokens locally.

We need this for Zendesk, Gmail, and Slack — all require OAuth. The file-based storage pattern and PKCE flow transfer directly. What changes:
- **OAuth flow initiation:** AIME opens the system browser. We do the same — the renderer can't handle OAuth popups (sandboxed), so main opens the browser and captures the callback.
- **Token refresh:** AIME's pattern handles access/refresh rotation. We need the same, plus detection of expired tokens that triggers a "reconnect" prompt in the UI.
- **Credential location:** Same `{userData}/.mcp/` directory. Credentials stay on-device.

### Adapt

#### MCP server role — platform capabilities for third-party apps

AIME exposes its built-in tools (bash, read, write, grep, web search, etc.) as an MCP server via an Express endpoint (`POST /mcp`). This is for internal access — external clients can call AIME's tools over HTTP.

We have a fundamentally different use case. Our MCP server exposes **platform capabilities** to third-party apps (Google AI Studio apps running in the Apps feature):

| | AIME's MCP server | Our MCP server |
|---|---|---|
| **Purpose** | Internal tool access via HTTP | Platform API for third-party apps |
| **Consumer** | Any MCP client (dev tool) | Google AI Studio apps running inside Cowork.ai |
| **Tools exposed** | All built-in tools (bash, file ops, web) | Scoped per-app: capture context, agent memory, MCP service access |
| **Auth** | None (localhost only) | Per-app permission grants (app declares what it needs, platform grants) |
| **Transport** | Express HTTP (`StreamableHTTPServerTransport`) | TBD — likely IPC-based since apps run in the renderer |

The Express HTTP server pattern doesn't transfer. Our apps run inside the Electron renderer — they communicate via IPC, not HTTP. The MCP server interface is the same (tool definitions, `listTools()`, `callTool()`), but the transport is different.

**Key design question:** How do third-party apps access the MCP server? Options:
1. IPC bridge in preload script (app calls `window.cowork.mcp.callTool()`)
2. Local HTTP server like AIME (app uses standard MCP HTTP client)

Option 1 is more secure (no network exposure) but requires custom transport. Option 2 reuses AIME's pattern but opens a port. This is open question #7 in [system-architecture.md — Open Architecture Questions](../../architecture/system-architecture.md#open-architecture-questions).

#### Tool discovery and build flow

AIME's flow: user adds MCP server config → `MCPClient` connects → `listTools()` discovers available tools → `buildTools()` makes them available to agents.

Our flow is similar but the trigger is different:
- **AIME:** User manually adds MCP server JSON config in Settings.
- **Cowork.ai:** User connects a service (e.g., "Connect Zendesk") → OAuth flow → MCPClient auto-configures from the service's MCP server definition → tools discovered automatically.

The user never sees MCP server JSON. They see "Zendesk" with a Connect button. Under the hood, it's the same `MCPClient` + `listTools()` flow, but wrapped in a service-specific onboarding UX.

Tool namespacing works the same: `{serverName}_{toolName}` (e.g., `zendesk_list_tickets`, `gmail_send_email`). This is `@mastra/mcp`'s built-in behavior.

### Study

#### Transport normalization (`configToMastraMCPServerDefinition`)

AIME's utility that normalizes stdio vs. HTTP transport config into `@mastra/mcp`'s server definition format. Clean abstraction — handles `command` + `args` + `env` for stdio, `url` + `headers` for HTTP.

We'll need this for supporting different MCP server types. Some services may offer stdio servers (bundled with the app), others may be remote HTTP. Study the normalization pattern, but our service configs are likely simpler (most will be HTTP/SSE to remote servers).

### Skip

#### Express HTTP server inside Electron

AIME runs Express on port 41100 for its MCP server endpoint. This is unnecessary complexity for us — our MCP server consumers (apps) are inside the same Electron process tree. IPC is more secure and more efficient than localhost HTTP.

If we do need HTTP for external MCP clients (e.g., a user's custom script calling Cowork.ai tools), that's a future feature, not v0.1.

#### User-editable MCP server JSON config

AIME lets users paste raw MCP server config JSON in Settings. This is developer-friendly but wrong for our audience. Our users connect services via OAuth buttons, not JSON blobs. The MCP config is generated internally from the service definition.

## 4. Tool System

**Source:** [`research/deep-dives/aime-chat/04-tool-system.md`](../deep-dives/aime-chat/04-tool-system.md)

AIME has 25+ built-in tools baked into the app (bash, file ops, web search, image gen, audio, database). Cowork.ai's tool system is bidirectional:

- **Inbound (agent consumes):** Our agent gets tools from connected MCP servers (Zendesk, Gmail, Slack) plus a small set of platform-provided tools (browser automation, capture queries, user interaction).
- **Outbound (platform provides):** We expose platform capabilities as MCP tools/resources to third-party apps. Capture context is exposed as read-only MCP resources. Connected service data is exposed as scoped MCP tools. If an app needs agent reasoning, the platform exposes it via the agent-as-tool pattern — the app sees a tool call, the platform runs the agent with full safety rails.

The flow for third-party apps: `Capture data → cowork.db → platform-provided MCP tools/resources → third-party apps (Google AI Studio)`

This is fundamentally different from AIME's monolithic tool registry. Our tools are a two-way MCP surface, not a baked-in list.

### Copy

#### `BaseTool` / `BaseToolkit` abstraction

Clean tool contract: Zod input/output schemas, async `execute(input, context)`, optional `requireApproval`, optional `configSchema`. `BaseToolkit` groups related tools (e.g., `BashToolkit` contains `Bash`, `KillBash`, `BashOutput`, `ListBash`).

We adopt this for two categories of tools:

**Agent-facing tools** (what the platform agent uses):

| Tool | Purpose | Approval required |
|---|---|---|
| `AskUser` | Request clarification from user | No |
| `BrowserNavigate` | Navigate Playwright to URL | No |
| `BrowserClick` / `BrowserType` / `BrowserSubmit` | Visible browser actions | Yes (first time per action type) |
| `CaptureQuery` | Read capture context from cowork.db | No |
| `SendNotification` | Dispatch macOS notification | No |

MCP-sourced tools (e.g., `zendesk_list_tickets`, `gmail_send_email`) come through `@mastra/mcp`'s `MCPClient` — they already have schemas from the MCP server definition and don't need `BaseTool`.

**App-facing tools** (what the platform exposes to third-party apps via MCP):

| MCP tool/resource | Purpose | Access |
|---|---|---|
| Capture context resources | Activity data, window history, focus sessions | Read-only |
| Connected service tools | Scoped access to Zendesk, Gmail, Slack data | Per-app permission grants |
| Agent-as-tool | App requests reasoning → platform runs agent with full control | Scoped, budget-enforced |

These outbound MCP tools use the same Zod schema contract — the MCP server definition describes inputs/outputs, and the platform enforces scoping per app (like app permissions on a phone).

#### AbortController with SIGTERM → SIGKILL escalation

AIME's bash execution: `child_process.spawn()` with `AbortController`, 120s timeout (configurable to 600s), SIGTERM first, 200ms grace period, then SIGKILL. Process group kill (`kill(-pid, signal)`) ensures child processes die too.

We need this for Playwright child process management. If a browser automation hangs (page unresponsive, infinite loop), we need the same escalation: signal the Playwright child → grace period → force kill → respawn. Maps directly to our "Crash behavior: Kill and respawn" for the Playwright process.

Also applies to Ollama inference: if a local model generation hangs or runs too long, abort the request gracefully before force-killing.

#### Tool execution context

AIME passes `requestContext` to every tool: `threadId`, `workspace`, `model`, `usage` (token tracking), `abortSignal`. This gives tools access to conversation state and cancellation.

Our execution context is similar but work-oriented:

| AIME context field | Our equivalent |
|---|---|
| `threadId` | `threadId` (same — conversation thread) |
| `workspace` | Not applicable (we're not a code editor) |
| `model` | `model` (current LLM for the request) |
| `usage` | `usage` + `budget` (token tracking + spend cap enforcement) |
| `abortSignal` | `abortSignal` (same) |
| — | `captureContext` (recent activity summary from capture process) |
| — | `mcpConnections` (which services are connected and healthy) |

### Adapt

#### Browser automation — Playwright child process, not in-process

AIME runs Playwright in three modes, all in the main process:
1. **Stagehand** — LLM-driven, no explicit selectors
2. **Persistent context** — Playwright with saved session state
3. **CDP remote** — attach to user's existing Chrome

We use Playwright but in a separate child process (see [system-architecture.md — Process Model](../../architecture/system-architecture.md#process-model)). The agent sends commands from the Agents & RAG utility process; the Playwright child executes them in an isolated Chromium instance.

| AIME mode | Our approach | Status |
|---|---|---|
| **Stagehand** (LLM-driven) | Study for v0.2+. LLM-driven automation without selectors aligns with our "coachable" MCP Browser vision — user says "click the reply button" and the agent figures out the selector. Requires vision model or DOM analysis. | Not v0.1 |
| **Persistent context** | Yes — our Playwright child launches with persistent user data so login state survives across sessions. Essential for operating inside Zendesk, Gmail, etc. | v0.1 |
| **CDP remote** | Study with caution. Attaching to the user's actual Chrome gives access to logged-in sessions without re-authentication. Powerful but privacy-sensitive — requires explicit consent, visible indicator ("AI is connected to your browser"), and strict scope (only the active task's tab). Must align with anti-surveillance positioning. | Evaluate for v0.2 |

Key architectural difference: AIME's browser tools call Playwright directly in the same process. Our agent sends browser commands via IPC to the Playwright child, receives results back. The tool interface is the same (`navigate`, `click`, `type`, `screenshot`), but every call crosses a process boundary. This adds latency (~1-2ms per IPC hop) but gains crash isolation — a hung browser page can't freeze the agent.

### Study

#### Stagehand for LLM-driven browser automation

`@browserbasehq/stagehand` wraps Playwright with an LLM layer: the model sees the page (via screenshot or DOM) and decides what to interact with. No CSS selectors, no XPath — just natural language intent.

This is the future of our MCP Browser's "coachable" interaction model. Instead of the agent needing to know Zendesk's DOM structure, it sees the page and acts. But it requires:
- A vision-capable model (or DOM serialization)
- Latency tolerance (LLM call per action)
- Cost tolerance (each browser action = one inference call)

For v0.1, we use explicit Playwright commands driven by MCP tool definitions (each service's MCP server knows how to navigate that service's UI). Stagehand is the v0.2 upgrade path when vision models are faster and cheaper.

#### Python code execution via `uv` with MCP bridge

AIME sandboxes Python execution: create temp dir → `uv init` + `venv` → `uv add` packages → inject `sitecustomize.py` (MCP bridge so Python can call back to AIME's tools) → `uv run` → cleanup.

Not needed for v0.1, but worth studying if we ever add code execution for automations (e.g., user writes a Python script that processes Zendesk ticket data). The MCP bridge injection is clever — it lets sandboxed code call platform tools without direct access to the host process.

### Skip

#### Most built-in tools

AIME bakes in 25+ tools: file operations (Read, Write, Edit, Glob, Grep), image manipulation (GenerateImage, EditImage, RemoveBackground), audio (STT, TTS), database queries (LibSQL toolkit), vision/OCR, web search, and more.

We don't need any of these as built-in tools:
- **File operations** — we're not a code editor. No filesystem access from the agent.
- **Image/audio/vision** — not in our feature set. Audio is Whisper for transcription only, handled by the capture layer, not as an agent tool.
- **Database queries** — the agent accesses cowork.db through tools (agent → tool → DB/vector), not raw SQL. Structured query tools for capture data (e.g., "how many tickets today?") and vector search tools for semantic recall (e.g., "find similar past conversations") — both run inside the Agents & RAG utility process that already has `@mastra/libsql` access. AIME's `LibSQLRun` (raw SQL) is too broad; our tools are purpose-built for the data types we store.
- **Web search** — if needed, this comes from an MCP server (e.g., a Brave Search MCP), not a built-in tool.

Our tool surface is intentionally small: browser automation, MCP service tools, user interaction, and capture context queries. Everything else is an MCP server that the user connects.

#### ToolsManager registration pattern

AIME's `ToolsManager` (1,404 lines) manages a registry of built-in tools, MCP tools, and skill tools with type-based routing (`build-in:*`, `mcp:*`, `skill:*`). This is over-engineered for our needs — we have a handful of platform tools plus MCP-sourced tools. Mastra's built-in tool registration handles our scale without a custom manager.

#### Code execution sandboxing (bash)

AIME wraps `child_process.spawn()` for bash execution with timeout, process group kill, and output streaming. We don't expose bash execution to the agent — it would be a security concern for a desktop sidecar with access to work services. The agent operates through structured tools (MCP, browser), not arbitrary shell commands.

## 5. AI Provider Support

**Source:** [`research/deep-dives/aime-chat/05-ai-provider-support.md`](../deep-dives/aime-chat/05-ai-provider-support.md)

AIME exposes 14+ providers to users — OpenAI, DeepSeek, Google, Ollama, LMStudio, ZhipuAI, ModelScope, and more. Users pick a provider, enter an API key, choose a model. Cowork.ai deliberately hides all of this behind the complexity router. But we do have three providers, not one monolithic gateway:

| Provider | Adapter | Use case |
|---|---|---|
| **Ollama** | `@ai-sdk/openai-compatible` | Local brain (DeepSeek-R1-8B). Free, instant, on-device. |
| **Gemini direct** | `@ai-sdk/google` | Free tier + development default. Native Google API, just a Gemini API key. No OpenRouter middleman. |
| **OpenRouter** | `@ai-sdk/openai-compatible` | Paid tiers (Boost → Gemini 3 Flash, Pro → Sonnet 4.5, Max → Opus 4.6). |

Gemini 2.5 Flash direct is the key addition — it's free via Google's API with generous rate limits, making it the natural default for the free cloud tier and for development. AIME already does this: their `GoogleProvider` wraps `@ai-sdk/google` natively.

### Copy

#### `@ai-sdk/google` for native Gemini access

AI SDK has first-party Gemini support. AIME uses it (their `GoogleProvider` wraps `@ai-sdk/google`). We use it directly:

```typescript
import { google } from '@ai-sdk/google';
// GOOGLE_GENERATIVE_AI_API_KEY set in env
const model = google('gemini-2.5-flash');
```

This avoids OpenRouter for free-tier traffic entirely. Simpler, lower latency, one fewer dependency in the critical path.

#### AI SDK `createProviderRegistry()` for clean three-provider management

With three providers, AI SDK's built-in `createProviderRegistry()` is the right abstraction — no custom `BaseProvider` needed:

```typescript
import { google } from '@ai-sdk/google';
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';
import { createProviderRegistry } from 'ai';

const registry = createProviderRegistry({
  ollama: createOpenAICompatible({
    baseURL: 'http://localhost:11434',
    name: 'ollama',
  }),
  gemini: google,
  openrouter: createOpenAICompatible({
    baseURL: 'https://openrouter.ai/api/v1',
    name: 'openrouter',
    apiKey: process.env.OPENROUTER_API_KEY,
  }),
});
```

The complexity router resolves to a registry string:

```
Complexity Router
    │
    ├─ simple (16GB+) ──→ registry.languageModel('ollama:deepseek-r1:8b')
    │
    ├─ complex (Free)  ──→ registry.languageModel('gemini:gemini-2.5-flash')
    │
    └─ complex (Paid)  ──→ registry.languageModel('openrouter:{model-by-tier}')
                              ├─ Boost → google/gemini-3-flash
                              ├─ Pro   → anthropic/claude-sonnet-4-5
                              └─ Max   → anthropic/claude-opus-4-6
```

This is AIME's `ProvidersManager` concept reduced to what we actually need — three providers, one registry, no custom base class. Adding a fourth provider later is one line in the registry, not a new class.

### Study

#### Model capability metadata

AIME maintains a 65K-line `models.json` with per-model metadata: reasoning capability, tool call support, structured output, modalities (text/image input/output), cost per token, context window, max output tokens.

Our complexity router needs similar metadata to make routing decisions — it needs to know which models support tool calls, what context windows are, and cost-per-token for budget enforcement. The difference:

- **AIME:** Static JSON file bundled with the app. Updated via app releases.
- **Us:** OpenRouter provides model metadata via API (`/api/v1/models`). Gemini capabilities are known (we control which model we use). Ollama model info comes from `ollama list` / `ollama show`.

**Open question:** Static metadata file (simpler, no network dependency) vs. dynamic metadata from OpenRouter API (always current, but adds a startup dependency)? Could do both — static defaults with dynamic refresh. Given we only use a handful of specific models, a small static config is probably sufficient.

### Skip

#### `ProvidersManager` and custom `BaseProvider`

AIME's 14-provider system with abstract `BaseProvider` class, per-provider implementations, health checking, model list fetching, and credit balance tracking.

**Why skip:** AI SDK's `createProviderRegistry()` gives us a clean registry without custom abstractions. Three providers, three registrations, done. AIME needed `BaseProvider` because they have 14 providers with different SDK adapters and capability profiles. We have three with known, fixed capabilities.

#### Provider management UI

AIME has screens for entering API keys per provider, toggling providers on/off, browsing available models, checking credit balances. Users directly choose which model to use for each conversation.

**Why skip:** Users never see providers or models. They see tiers with capabilities. The only keys users enter are: Gemini API key (free, entered at onboarding or auto-provisioned) and OpenRouter API key (entered when upgrading to a paid tier). No model selection, no provider toggling.

#### Provider capability tags

AIME tags providers with capability flags: `EMBEDDING`, `RERANKER`, `WEB_SEARCH`, `WEB_READER`, `IMAGE_GENERATION`, `OCR`. These drive which providers appear for which tasks.

**Why skip:** Our capability mapping is static and known at build time:
- **Language model:** Ollama (local) or Gemini/OpenRouter (cloud) — router decides
- **Embeddings:** Always Ollama (Qwen3-Embedding-0.6B) — local only
- **Transcription:** Always Whisper via Core ML — local only
- No image generation, no OCR, no web search via provider (web search is an MCP server if needed)

No need for dynamic capability discovery.

## 6. Electron Architecture

**Source:** [`research/deep-dives/aime-chat/06-electron-architecture.md`](../deep-dives/aime-chat/06-electron-architecture.md)

AIME is a standard two-process Electron app: main process runs everything heavy (12 manager singletons — DB, agents, MCP, providers, Playwright, background jobs), renderer communicates via context-isolated IPC. Cowork.ai is a five-process app: main is a thin coordinator, heavy work distributes across two utility processes (Capture, Agents & RAG) plus a Playwright child. The fundamental difference is process model — their patterns for IPC contracts and boot sequencing transfer, but where code lives changes completely.

### Copy

#### `BaseManager` + `@channel` decorator pattern

AIME's cleanest Electron pattern. Each manager extends `BaseManager` (35 lines). Methods decorated with `@channel('namespace:method')` auto-register as IPC handlers in the constructor. No manual `ipcMain.handle()` boilerplate.

```
@channel('agents:buildAgent')
async buildAgent(agentId, options) { ... }
// → auto-registers ipcMain.handle('agents:buildAgent', ...)
```

We adopt this for all three processes that expose IPC:

| Process | Managers | IPC direction |
|---|---|---|
| **Main** | AppManager (lifecycle, tray, settings), NotificationManager, ThermalManager | Renderer ↔ Main |
| **Capture Utility** | CaptureManager (activity + keystroke wrappers) | Capture → Main (events, status) |
| **Agents & RAG Utility** | AgentManager, MCPManager, MemoryManager, EmbeddingManager | Main ↔ Agents (bidirectional) |

What changes: AIME's `@channel` registers on `ipcMain` because everything lives in main. Our utility process managers register on the utility process's `parentPort` / `MessagePort` instead. The decorator pattern is the same, but the underlying transport differs per process. The `BaseManager` base class needs a process-aware variant that picks the right registration target.

#### Sequential boot with dependency ordering

AIME initializes 12 managers in strict order: DB → providers → settings → Mastra → KB → tools → models → agents → projects → updater → Playwright → background jobs. Each `init()` can depend on previous managers being ready.

We need the same per-process:

**Main process boot:**
1. AppManager (lifecycle, settings)
2. ThermalManager (hardware monitoring)
3. Spawn Capture Utility process
4. Spawn Agents & RAG Utility process
5. Wait for utility processes to signal ready
6. Create BrowserWindow (renderer)

**Capture Utility boot:**
1. NativeAddonManager (load `coworkai-keystroke-capture`, `coworkai-activity-capture`)
2. CaptureBufferManager (activity buffer, keystroke chunker)
3. DBManager (libsql sync connection to cowork.db)
4. Signal ready to main

**Agents & RAG Utility boot:**
1. DBManager (libsql async connection to cowork.db)
2. ProviderRegistry (Ollama + Gemini + OpenRouter via `createProviderRegistry()`)
3. MastraManager (agent orchestration, memory)
4. MCPManager (service connections, OAuth)
5. EmbeddingManager (queue, Qwen3-0.6B via Ollama)
6. Signal ready to main

The key difference from AIME: their boot is one flat sequence. Ours is three parallel sequences (main + two utilities), with main waiting for both utilities before showing UI.

#### Screen capture via macOS `screencapture` CLI

AIME uses `child_process.exec('screencapture -x -C ...')` for screenshots — native quality, no Electron API limitations. Falls back to `desktopCapturer` on Windows/Linux.

Directly useful for our Context feature's screen recording stream (opt-in). The macOS `screencapture` CLI is the right tool — higher quality than `desktopCapturer`, no permission dialogs beyond the initial Screen Recording grant.

### Adapt

#### IPC architecture — three-process routing instead of two-process

AIME's IPC is simple: renderer ↔ main. The preload script (1,384 lines) exposes `window.electron.*` namespaces, each mapping to `ipcRenderer.invoke()` calls that hit `@channel`-decorated handlers in main.

Our IPC has an extra hop for most operations:

```
AIME:     Renderer ←→ Main (handles everything)

Cowork:   Renderer ←→ Main (routes only) ←→ Agents & RAG Utility
                      Main ←─────────────── Capture Utility (push events)
```

What changes:
- **Preload is thinner.** Same `window.cowork.*` namespace pattern, but main process handlers are mostly pass-throughs that forward to the appropriate utility process.
- **Two-hop latency.** Renderer → Main → Agents adds ~1-2ms per IPC hop. Acceptable for chat and tool calls. For high-frequency updates (capture events), the capture process pushes to main, which batches before forwarding to renderer.
- **Error surface.** AIME's IPC errors are straightforward — main process crashed or handler threw. Ours can also fail on the second hop (utility process crashed). Main needs to detect broken utility process connections and surface meaningful errors to the renderer, not just timeouts.

The `@channel` decorator abstraction helps here — the renderer doesn't know or care that its IPC call traverses two processes. The routing logic lives in main's forwarding handlers.

**Reference:** [system-architecture.md — IPC Contract](../../architecture/system-architecture.md#ipc-contract)

#### Database — unified libsql instead of dual-access TypeORM + libsql

AIME uses two database access patterns against one file (`main.db`):
- **TypeORM + better-sqlite3** for app data (providers, settings, agents, projects, etc.)
- **libsql** for Mastra data (chat history, token usage, vectors)

This works but creates friction: two ORMs, two connection patterns, potential WAL conflicts between synchronous (better-sqlite3) and async (libsql) writers.

We chose to unify on libsql only (see [DATABASE_STACK_RESEARCH.md](../../decisions/DATABASE_STACK_RESEARCH.md)). Trade-offs:

| | AIME (TypeORM + libsql) | Us (libsql only) |
|---|---|---|
| **ORM** | TypeORM entities with decorators, auto-migration | No ORM — raw SQL or thin query builder |
| **Migration** | `synchronize: true` (auto-schema sync) | Manual migrations (safer for production) |
| **Relations** | TypeORM handles FK, cascades, lazy loading | Manual FK management, explicit joins |
| **Native modules** | Two: `better-sqlite3` + `libsql` | One: `libsql` — one ABI rebuild, one ASAR unpack |
| **Vectors** | libsql's `F32_BLOB` + `vector_distance_cos` | Same |

What we lose: TypeORM's auto-migration and entity decorators are convenient. What we gain: one native module, one connection pool, no dual-access WAL concerns. For our schema complexity (capture tables + Mastra tables), raw SQL with a migration tool is manageable.

#### Preload script — scoped to IPC routing, no logic

AIME's preload is 1,384 lines with namespace groupings (`app`, `providers`, `mastra`, `knowledgeBase`, `tools`, `agents`, etc.). Each namespace maps to `ipcRenderer.invoke()` calls.

Our preload follows the same pattern but is thinner — main process does less, so there are fewer direct handlers:

| AIME preload namespace | Our equivalent | Notes |
|---|---|---|
| `app` (settings, theme) | `cowork.app` | Same — main handles settings |
| `providers` (API keys, model lists) | Not needed | Provider registry is internal to Agents utility |
| `mastra` (chat, agent control) | `cowork.chat` | Routes through main to Agents utility |
| `knowledgeBase` (upload, manage) | Not needed | No manual KB management |
| `tools` (MCP config, skills) | `cowork.mcp` | Routes through main to Agents utility |
| `agents` (create, configure) | Not needed | Single platform agent, not user-configurable |
| `instances` (Playwright config) | `cowork.browser` | Routes through main to Agents → Playwright |
| — | `cowork.capture` | Capture status, stream toggles. Routes to Capture utility. |
| — | `cowork.context` | Context Card data. Queries capture data via Agents utility. |

### Study

#### `TaskQueueManager` for background jobs

AIME's background job system: named queues with configurable concurrency per group. Used for embedding jobs (`groupMaxConcurrency: 1` per knowledge base), model downloads, and long-running operations.

We need the same for:
- **Embedding pipeline:** Continuous capture data → embedding queue (single concurrency per stream)
- **Automations:** Scheduled or event-triggered agent tasks
- **MCP operations:** Batch operations across services (e.g., categorize 50 tickets)

Study their implementation for the queue interface. Our queues run in the Agents & RAG utility process. The queue itself might be Mastra's built-in workflow engine rather than a custom implementation — evaluate during implementation.

#### Preload typed IPC bindings

AIME's preload provides typed function signatures for every IPC call. The renderer gets autocomplete and type safety when calling `window.electron.agents.buildAgent(...)`.

Worth studying the pattern for our `window.cowork.*` API. The challenge: our preload types need to match handlers that live across two processes (main + agents utility), not just main. A shared type definition in `src/shared/` that both the preload and utility process handlers import would keep them in sync.

### Skip

#### Monolithic main process

AIME puts everything in main: 12 manager singletons, database, agents, MCP, Playwright, background jobs. If any manager crashes or hangs, the app is dead.

**Why skip:** This is the core architectural difference. Our main process is a thin coordinator — lifecycle, IPC routing, tray, thermal monitoring. All heavy work (capture, agents, Playwright) lives in isolated processes. We made this choice explicitly (see [DESKTOP_FRAMEWORK_DECISION.md](../../decisions/DESKTOP_FRAMEWORK_DECISION.md#multi-process-model)).

#### Express server inside Electron (port 41100)

AIME runs Express for MCP bridge access. Already covered in [Section 3 — Skip](#3-mcp-integration).

#### `synchronize: true` (TypeORM auto-migration)

AIME uses TypeORM's auto-schema synchronization — the ORM diffs the entity definitions against the DB schema and applies changes automatically on startup. Convenient for development, dangerous for production (can drop columns with data, no rollback).

**Why skip:** We use manual migrations. More work upfront, but safe for shipping desktop software where users have real data. Also moot — we don't use TypeORM.

#### Dual-database access pattern (TypeORM + libsql)

Already covered in Adapt above. We unified on libsql only.

## 7. UI & i18n

**Source:** [`research/deep-dives/aime-chat/07-ui-and-i18n.md`](../deep-dives/aime-chat/07-ui-and-i18n.md)

AIME's UI is a developer-oriented chat app: sidebar navigation (Chat, KB, Agents, Tools, Projects, Settings), chat conversation with tool invocations and reasoning panels, model selector, task manager. Built with shadcn/ui (Radix primitives) + Tailwind + Zustand + React Router. Cowork.ai's UI is a work sidecar with six feature views (Apps, MCP Integrations, Chat, MCP Browser, Automations, Context) built with M3 (Material Design 3) + Tailwind. Different design system, different navigation model, but several UI problems overlap.

### Study

#### Chat UI components

AIME has solved several hard rendering problems for AI chat:
- **Streaming text display** — rendering token-by-token output without layout thrash
- **Tool invocation panels** — showing tool calls in-progress (name, inputs, status, output)
- **Reasoning sections** — collapsible extended thinking (DeepSeek's `<think>` blocks)
- **Source citations** — inline source attribution from RAG results
- **Context usage** — token count display with budget indicator

Our Chat view needs all of these. The component structure and streaming rendering approach are worth studying even though we use M3 instead of shadcn/ui. The chunk types driving these UI elements are the same (AI SDK protocol — covered in [Section 0](#0-vercel-ai-sdk)).

#### Zustand for state management

AIME uses Zustand v5 — minimal boilerplate, works well with Electron IPC. Stores are simple: create a store, use it in components, update via actions. No Redux-style reducers or context provider trees.

Strong candidate for our state management. Zustand's `subscribe` API pairs well with IPC — a store can subscribe to IPC events from main and update automatically. This is a TBD decision for us.

#### React Router v7

AIME uses React Router v7 for navigation between views. Standard SPA routing within the Electron renderer.

We need routing for our six feature views (Apps, MCP Integrations, Chat, MCP Browser, Automations, Context) plus Settings. React Router v7 is the obvious choice — study their route structure as a reference.

#### Motion (Framer Motion) for animation

AIME uses Motion v12 (rebranded from Framer Motion). Used for panel transitions, message appear animations, sidebar collapse.

Worth evaluating for our M3 design system. M3 specifies motion tokens (duration, easing curves) — Motion can implement these. The alternative is CSS-only transitions, which are simpler but less capable for complex sequences (like the three-state interaction model in [design-system.md](../../design/design-system.md)).

### Skip

#### shadcn/ui (Radix primitives)

We're committed to M3 (Material Design 3). See [design-system.md](../../design/design-system.md). shadcn/ui is Radix-based — different component API, different design tokens, different visual language.

The Radix accessibility primitives underneath are solid (proper ARIA, keyboard navigation, focus management), but M3 component libraries handle this too. No reason to mix design systems.

#### UI layout (sidebar + content)

AIME's layout is a standard developer tool: sidebar with navigation categories (Chat, KB, Agents, Tools, Projects, Settings), main content area with breadcrumbs.

Our layout is different — the three-state interaction model from [design-system.md](../../design/design-system.md):
1. **Ambient** — minimized tray, proactive notifications
2. **Glanceable** — compact overlay, Context Card + quick actions
3. **Engaged** — full window, six feature views

AIME has one state (full window, always engaged). We have three. Their sidebar-content layout doesn't map to our interaction model.

#### Excalidraw, tldraw, Recharts

Drawing and charting libraries. Not in our feature set.

#### i18n (i18next)

AIME supports Chinese and English via `i18next` + `react-i18next` with DB-cached translations.

**Why skip:** English-only for v0.1. If we ever need i18n, `i18next` is the standard approach — nothing novel in their implementation. Note for later, not for now.

## 8. Architecture Comparison

**Source:** [`research/deep-dives/aime-chat/08-architecture-comparison.md`](../deep-dives/aime-chat/08-architecture-comparison.md)

This section is a side-by-side comparison in the deep-dive — not a feature to adapt. Rather than Copy/Adapt/Skip, this section synthesizes the comparison into implementation guidance: what the deep-dive validates about our architecture, what risks it surfaces, and what the comparison reveals about our actual differentiation.

### What the Comparison Validates

**Mastra + libsql + Electron works.** AIME proves this stack runs in production. Our additional bet — running Mastra in a utility process rather than main — is the only unproven piece. Everything else (AI SDK streaming, IPC transport, libsql vectors, MCP client management) has a working reference.

**AI SDK is the right abstraction layer.** Both stacks converge on Vercel AI SDK as the interface contract. The `useChat()` → `ChatTransport` → streaming chunks → tool lifecycle is stable across v5 and v6. AIME's implementation confirms our assumptions about chunk types, streaming protocol, and tool approval flow. We don't need to design these from scratch — we adapt a proven pattern.

**Single-DB architecture scales.** AIME puts everything in one `main.db` — structured data, agent memory, vectors. They haven't hit scaling issues. Our `cowork.db` follows the same pattern with a cleaner access layer (libsql only vs. their TypeORM + libsql dual access).

### What the Comparison Surfaces as Risks

**Utility process isolation is unproven.** This is the single biggest architectural risk the comparison highlights. AIME's monolithic main process is simpler to implement — no IPC forwarding, no cross-process error handling, no multi-process boot coordination. Our multi-process model is better for stability (crash isolation) but harder to build. The [MASTRA_ELECTRON_VIABILITY.md](../../decisions/MASTRA_ELECTRON_VIABILITY.md) spike must validate this before committing.

**Two-hop IPC adds complexity.** AIME's renderer ↔ main IPC is straightforward. Our renderer ↔ main ↔ utility adds error surface: utility process can crash mid-stream, MessagePort connections can break, boot timing requires coordination. Section 6 details the adaptation, but the engineering cost is real — expect IPC debugging to be a significant portion of early development.

**No reference for ambient capture + agents.** AIME is purely reactive — user types, agent responds. We add continuous ambient capture feeding into the same agent context. There's no reference implementation for this pattern (capture process → DB → embedding pipeline → agent context assembly). This is novel integration work.

### Where We Actually Differentiate

The comparison table in the deep-dive lists feature gaps, but the real differentiation is architectural:

| Dimension | AIME | Cowork.ai | Why it matters |
|---|---|---|---|
| **Input model** | User-initiated only | Ambient capture + user-initiated | Agent has context before the user asks |
| **Process model** | Monolithic main | Multi-process isolation | Native addons and GPU work can't crash the app |
| **Provider model** | 14+ user-managed providers | 3 providers behind complexity router | User never thinks about models |
| **Tool model** | 25+ built-in tools | Bidirectional MCP (consume + provide) | Extensible via MCP, not baked-in code |
| **App model** | Monolithic (skills = code) | App Gallery with scoped MCP permissions | Third-party apps extend the platform |
| **Memory model** | Stateless between sessions | Four-layer persistent memory | Agent learns the user over time |

These aren't incremental improvements — they're different products built on the same foundation (AI SDK + Mastra + Electron + libsql). The foundation transfers directly. The product layer is entirely ours.

### Updated Comparison Table

The deep-dive's provider comparison is now outdated. Updated version:

| Aspect | AIME Chat | Cowork.ai |
|---|---|---|
| **Provider adapters** | `@ai-sdk/openai`, `@ai-sdk/google`, `@ai-sdk/deepseek` + `@ai-sdk/openai-compatible` (14+ providers, user-managed) | `@ai-sdk/google` (Gemini direct, free tier) + `@ai-sdk/openai-compatible` (Ollama local + OpenRouter paid). Three providers via `createProviderRegistry()`. |

## 9. Summary

**Source:** [`research/deep-dives/aime-chat/09-summary.md`](../deep-dives/aime-chat/09-summary.md)

Consolidated quick-reference across all sections. The deep-dive's summary is organized as Copy/Study/Skip lists — this adaptation guide version updates those lists based on the analysis in Sections 0–8 and corrects items where our architecture diverges from the deep-dive's original recommendations.

### Copy (adopt directly)

Patterns proven in AIME's codebase that we use with minimal changes.

| Pattern | Section | What changes for us |
|---|---|---|
| `IpcChatTransport` (AI SDK ↔ Electron IPC) | [0](#0-vercel-ai-sdk) | Extra IPC hop — renderer → main → agents utility. Main is passthrough. |
| Zod-validated streaming chunk protocol | [0](#0-vercel-ai-sdk) | Same chunk types. We add reasoning chunks (AIME disables them). |
| `@ai-sdk/openai-compatible` for Ollama | [0](#0-vercel-ai-sdk) | Identical. Lives in Agents & RAG utility process. |
| `@ai-sdk/google` for native Gemini | [5](#5-ai-provider-support) | AIME uses this for their Google provider. We use it for free tier direct access. |
| `createProviderRegistry()` for three providers | [5](#5-ai-provider-support) | Ollama + Gemini direct + OpenRouter. Clean AI SDK built-in, no custom base class. |
| `embedMany()` for embeddings | [0](#0-vercel-ai-sdk) | Same call, different model (Qwen3-0.6B via Ollama) and storage (LibSQLVector). |
| Tool definitions with Zod schemas | [0](#0-vercel-ai-sdk) | Same contract. Our tools are MCP-sourced + small set of platform tools. |
| Tool approval flow (suspended → approve → resume) | [1](#1-mastra-agent-system) | Direct map to MCP Browser and destructive action gates. |
| Multi-step execution loop | [1](#1-mastra-agent-system) | Same loop, runs in utility process. |
| Dynamic system prompt generation | [1](#1-mastra-agent-system) | Same pattern, different payload (work context vs. dev context). |
| Mastra Observational Memory (not AIME's custom CompressAgent) | [1](#1-mastra-agent-system) | Built-in `observationalMemory: true`. Better than AIME's custom approach. |
| Background embedding task queue | [2](#2-rag--knowledge-base) | Same concurrency pattern, different input (ambient capture vs. document upload). |
| MCPClient connection health management | [3](#3-mcp-integration) | Status lifecycle (starting→running→stopped→error). Lives in utility process. |
| OAuth credential storage for MCP servers | [3](#3-mcp-integration) | Same file-based pattern. Needed for Zendesk, Gmail, Slack. |
| `BaseTool` / `BaseToolkit` abstraction | [4](#4-tool-system) | For platform tools + app-facing MCP tools. MCP-sourced tools get schemas from MCP server. |
| AbortController with SIGTERM → SIGKILL | [4](#4-tool-system) | For Playwright child + Ollama inference timeout. |
| `BaseManager` + `@channel` IPC decorator | [6](#6-electron-architecture) | Adapt for three processes — decorator registers on `parentPort`/`MessagePort`, not just `ipcMain`. |
| Sequential boot with dependency ordering | [6](#6-electron-architecture) | Three parallel boot sequences (main + two utilities) instead of one flat sequence. |
| Screen capture via macOS `screencapture` CLI | [6](#6-electron-architecture) | For Context feature's screen recording stream (opt-in). |

### Adapt (modify significantly)

Patterns we use but reshape for our architecture or product model.

| Pattern | Section | How it changes |
|---|---|---|
| Sub-agent delegation (Task tool) | [1](#1-mastra-agent-system) | Keep context isolation principle. Replace code-oriented agents (Explore, Plan) with work-oriented specialists. |
| "Explore before acting" (Plan mode) | [1](#1-mastra-agent-system) | Not a mode toggle — built into MCP Browser flow (read page → propose actions → approve → execute). |
| Memory configuration | [1](#1-mastra-agent-system) | AIME disables everything. We enable everything (semanticRecall, workingMemory, lastMessages, vector). |
| Vector storage via LibSQLVector | [2](#2-rag--knowledge-base) | Use Mastra's `LibSQLVector` abstraction instead of AIME's raw SQL. |
| Ingestion pipeline | [2](#2-rag--knowledge-base) | Ambient capture replaces document upload. No format parsers needed. |
| MCP server role | [3](#3-mcp-integration) | AIME exposes tools internally. We expose platform capabilities to third-party apps with scoped permissions. |
| Tool discovery flow | [3](#3-mcp-integration) | User connects service via OAuth button, not JSON config. MCPClient auto-configures. |
| Browser automation | [4](#4-tool-system) | Playwright in child process, not in-process. Persistent context for v0.1, CDP/Stagehand for later. |
| IPC architecture | [6](#6-electron-architecture) | Three-process routing (renderer ↔ main ↔ utility) instead of two-process. |
| Database access | [6](#6-electron-architecture) | Unified libsql instead of dual TypeORM + libsql. Manual migrations instead of auto-sync. |
| Preload script | [6](#6-electron-architecture) | Thinner — main is mostly pass-through. Different namespace mapping. |

### Study (evaluate, don't commit yet)

Worth understanding but not adopting for v0.1.

| Topic | Section | When relevant |
|---|---|---|
| `toAISdkFormat()` (Mastra → AI SDK bridge) | [0](#0-vercel-ai-sdk) | Debugging streaming issues. Internal to Mastra. |
| AI SDK `gateway` API | [0](#0-vercel-ai-sdk) | Likely skip — explicit provider resolution preferred for logging. |
| `generateObject()` / structured output | [0](#0-vercel-ai-sdk) | If we add LLM-based intent classification (not in v0.1 — router is rule-based). |
| Reranker pass for RAG | [2](#2-rag--knowledge-base) | Improves retrieval precision. Adds latency. Evaluate post-v0.1. |
| Multi-source query construction | [2](#2-rag--knowledge-base) | Unified vs. per-type vector search. Depends on LibSQLVector namespace design. |
| Transport normalization (`configToMastraMCPServerDefinition`) | [3](#3-mcp-integration) | When supporting mixed stdio/HTTP MCP servers. |
| Stagehand (LLM-driven browser automation) | [4](#4-tool-system) | v0.2+ — when vision models are faster/cheaper. |
| Python execution via `uv` + MCP bridge | [4](#4-tool-system) | If we ever add code execution for automations. |
| Model capability metadata | [5](#5-ai-provider-support) | Static file vs. dynamic API. Small static config probably sufficient for our 3 providers. |
| `TaskQueueManager` for background jobs | [6](#6-electron-architecture) | For embeddings, automations, batch MCP. May use Mastra workflows instead. |
| Zustand for state management | [7](#7-ui--i18n) | Strong candidate. TBD decision. |
| Chat UI components (streaming, tool panels, reasoning) | [7](#7-ui--i18n) | Hard UI problems they've solved. Study rendering approach. |
| Motion (Framer Motion) for animation | [7](#7-ui--i18n) | For M3 motion tokens. Evaluate vs. CSS-only. |

### Skip (don't need)

| What | Section | Why |
|---|---|---|
| Provider management UI / `ProvidersManager` | [0](#0-vercel-ai-sdk), [5](#5-ai-provider-support) | Users pick tiers, not providers. Three providers via registry. |
| Agent types (CodeAgent, Explore, Plan) | [1](#1-mastra-agent-system) | Wrong abstractions for a work sidecar. Single platform agent. |
| Agent selection UI | [1](#1-mastra-agent-system) | No agent selection — complexity router decides. |
| Document parsers (PDF, DOCX, XLSX, OCR) | [2](#2-rag--knowledge-base) | No document upload. Capture data is already text. |
| Knowledge base management UI | [2](#2-rag--knowledge-base) | No user-managed KBs. Context is ambient. |
| Express HTTP server inside Electron | [3](#3-mcp-integration) | Apps use IPC, not HTTP. |
| User-editable MCP server JSON config | [3](#3-mcp-integration) | OAuth buttons, not JSON blobs. |
| 25+ built-in tools | [4](#4-tool-system) | Capabilities via MCP, not baked-in. |
| `ToolsManager` (1,404-line registry) | [4](#4-tool-system) | Overkill for our tool count. |
| Bash execution sandboxing | [4](#4-tool-system) | Security concern for a work sidecar with service access. |
| Per-provider SDK adapters (14+) | [5](#5-ai-provider-support) | One adapter (`openai-compatible`) + `@ai-sdk/google`. |
| Provider capability tags | [5](#5-ai-provider-support) | Static and known at build time for 3 providers. |
| Monolithic main process | [6](#6-electron-architecture) | Core architectural difference — we isolate. |
| `synchronize: true` (TypeORM auto-migration) | [6](#6-electron-architecture) | Dangerous for production. Manual migrations. |
| Dual DB access (TypeORM + libsql) | [6](#6-electron-architecture) | Unified on libsql. |
| shadcn/ui | [7](#7-ui--i18n) | M3 design system. |
| Sidebar + content layout | [7](#7-ui--i18n) | Three-state interaction model, not single-state. |
| Excalidraw, tldraw, Recharts | [7](#7-ui--i18n) | Not in feature set. |
| i18n | [7](#7-ui--i18n) | English-only for v0.1. |

---

## Changelog

**v1 (Feb 16, 2026):** Created. Section 0 (Vercel AI SDK) complete.
**v2 (Feb 16, 2026):** Section 1 (Mastra Agent System) complete.
**v3 (Feb 16, 2026):** Section 2 (RAG & Knowledge Base) complete.
**v4 (Feb 16, 2026):** Section 3 (MCP Integration) complete.
**v5 (Feb 16, 2026):** Section 4 (Tool System) complete.
**v6 (Feb 17, 2026):** Section 5 (AI Provider Support) complete.
**v7 (Feb 17, 2026):** Section 6 (Electron Architecture) complete.
**v8 (Feb 17, 2026):** Section 7 (UI & i18n) complete.
**v9 (Feb 17, 2026):** Section 8 (Architecture Comparison) complete.
**v10 (Feb 17, 2026):** Section 9 (Summary) complete. All sections done.
