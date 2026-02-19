# AnythingLLM → Cowork.ai Adaptation Guide

**Purpose:** Maps the key patterns from AnythingLLM to Cowork.ai — what we copy, what we adapt, and what we skip. Each section explains *why* a pattern changes for our architecture.

**Audience:** Engineering (Rustan + team)
**Target architecture:** [`architecture/system-architecture.md`](../../architecture/system-architecture.md)

**Why AnythingLLM?** AnythingLLM has the most mature RAG pipeline in our competitive set. Its `fillSourceWindow` backfill, query-mode refusal guard, and multi-source context assembly are battle-tested patterns for retrieval-grounded conversations. Its no-code agent flow executor maps to our Automations feature. The MCP hypervisor shows production lifecycle management for stdio/http/sse servers. These patterns fill gaps that aime-chat, Cherry Studio, and Chatbox don't cover.

**Key difference from other guides:** AnythingLLM is a server-first architecture (Express + collector service) with a separate downstream Electron wrapper not in the core repo. We cannot audit their Electron IPC — all patterns here are server-side logic that we'll port into our Agents & RAG Utility Process.

---

## Sections

| # | Section | Status |
|---|---|---|
| 0 | [RAG Context Assembly & fillSourceWindow](#0-rag-context-assembly--fillsourcewindow) | Done |
| 1 | [Query-Mode Refusal Guard](#1-query-mode-refusal-guard) | Done |
| 2 | [MCP Hypervisor Lifecycle](#2-mcp-hypervisor-lifecycle) | Done |
| 3 | [No-Code Agent Flow Executor](#3-no-code-agent-flow-executor) | Done |
| 4 | [Summary](#4-summary) | Done |

---

## 0. RAG Context Assembly & fillSourceWindow

**Source:** AnythingLLM's `server/utils/helpers/chat/index.js:348-442` + `server/utils/chats/stream.js:115-181`

Cowork.ai's Context feature (product-features.md §6) needs to assemble grounded context for every agent response. The hard problem is follow-up queries: the user asks "tell me more about that ticket" and the vector search returns nothing because the follow-up doesn't contain the original keywords. AnythingLLM solved this with `fillSourceWindow` — a history-based backfill that re-injects previously cited chunks when fresh retrieval comes up short.

### Copy

#### Multi-source context assembly order

AnythingLLM assembles context in a strict priority order that ensures the most relevant material appears first:

```
1. Pinned documents        → always included first (but can be displaced by compression in step 5)
2. Parsed-file context     → temporary per-thread uploads (not yet embedded)
3. Vector similarity search → standard RAG retrieval
4. History backfill         → fillSourceWindow (prior cited chunks)
5. Compress to fit          → cannonball middle-truncation (may truncate any context, including pinned docs)
```

**Why copy:** This priority order prevents context pollution. Pinned docs (equivalent to our "observation anchors") get first placement, though cannonball compression may truncate them if the total context exceeds the token limit. Transient context (equivalent to our screen-capture data before it's embedded) gets second priority. Vector search fills the middle. History backfill is last-resort gap-filling.

**Our mapping:**

| AnythingLLM source | Cowork.ai equivalent | Process |
|---|---|---|
| Pinned docs (`DocumentManager.pinnedDocs()`) | Observation anchors (user-pinned captures) | Agents Utility |
| Parsed files (`workspace_parsed_files`) | Recent capture data (pre-embedding buffer) | Agents Utility |
| Vector similarity search | libsql vector search across `activity_embeddings` | Agents Utility |
| `fillSourceWindow` backfill | Prior-citation re-injection (see below) | Agents Utility |
| `messageArrayCompressor` | Vercel AI SDK token management | Agents Utility |

#### `fillSourceWindow` — Prior-citation backfill

The core algorithm:

```
function fillSourceWindow({ nDocs = 4, searchResults, history, filterIdentifiers }):
  1. Start with searchResults.sources (vector matches)
  2. If sources.length >= nDocs → return early (enough context)
  3. If history.length === 0 → return early (nothing to backfill from)
  4. Walk history in REVERSE (most recent first):
     a. Parse response JSON → extract .sources array
     b. Filter out: pinned doc IDs, sources without score/text, already-seen chunk IDs
     c. Add valid sources until sources.length >= nDocs
  5. Return { sources, contextTexts: sources.map(s => s.text) }
```

**Key design decisions to copy:**

- **Deduplication by chunk ID** — uses a `Set` to track `seenChunks`, prevents the same chunk appearing twice in context
- **Pinned-doc exclusion** — `filterIdentifiers` prevents backfilling chunks that are already injected as pinned docs
- **Recency-first** — iterates `history.reverse()` so the most recent conversation's citations get priority
- **Window cap** — `nDocs` default of 4 prevents backfill from overwhelming fresh search results
- **Pre-limited history** — caller passes a limited number of messages (default 20, configurable via `workspace.openAiHistory`) to `fillSourceWindow`, not the entire conversation

**How citations are stored (data structure):**

```
workspace_chats row:
  response: JSON string → {
    text: "LLM response...",
    sources: [
      { id: "chunk-uuid", text: "chunk content", score: 0.87, title: "doc.pdf", published: timestamp }
    ],
    type: "chat" | "query"
  }
```

The sources array is embedded in the response JSON. `fillSourceWindow` parses this with `safeJsonParse` to extract prior citations.

**Our adaptation:** In Cowork.ai, agent responses are stored in `conversation_messages` (system-architecture.md §4). We'd store cited chunk IDs in a `citations` JSON column on each message row. The backfill logic lives in the Agents Utility Process and queries `conversation_messages` + `activity_embeddings` via libsql.

### Adapt

#### Context source → activity capture data

AnythingLLM's context comes from user-uploaded documents (PDFs, web links, text files). Cowork.ai's context comes from screen captures, clipboard events, and tool activity. The assembly order is the same, but the input pipeline is different:

| AnythingLLM | Cowork.ai |
|---|---|
| File upload → collector service → chunking → embedding | Screen capture → Capture Utility → OCR/extraction → chunking → embedding |
| `workspace_parsed_files` (transient pre-embed) | Recent capture buffer (last N captures before embedding completes) |
| `workspace_documents` (permanent) | `activities` table (permanent) |

The backfill algorithm doesn't care about the source — it only needs: chunk ID, chunk text, relevance score, and deduplication. Copy the algorithm as-is, change the storage layer.

#### Compression strategy

AnythingLLM uses "cannonball" middle-truncation: keep the first N and last M tokens, drop the middle. This is crude but fast.

**Our approach:** Use Vercel AI SDK v6's built-in token management and `maxTokens` constraints. For cases where we need manual compression, implement a recency-weighted strategy instead of middle-truncation — recent captures should survive over old ones, which aligns better with a sidecar that processes a continuous stream of activity data.

### Skip

#### File-based document JSON persistence

AnythingLLM stores parsed documents as JSON files on disk (`storage/documents/`), with a separate SQLite row linking to the file path. This made sense for their server architecture but creates sync/migration problems.

**Why skip:** We use libsql for everything. Chunk text lives in the vector table alongside embeddings. No file-JSON layer needed.

#### Monolithic chat handler orchestration

AnythingLLM's `stream.js` (~316 lines) handles mode detection, pinned doc loading, parsed file loading, vector search, backfill, compression, LLM call, response streaming, and history saving in a single flow.

**Why skip:** Mastra's agent system already separates these concerns into tools, context providers, and memory. The RAG assembly should be a composable pipeline, not a monolith.

---

## 1. Query-Mode Refusal Guard

**Source:** AnythingLLM's `server/utils/chats/stream.js:65-92` (refusal point 1) and `:198-227` (refusal point 2)

Cowork.ai's Chat feature (product-features.md §3) needs to distinguish between freeform conversation and grounded query. When a user asks "what did the customer say in that ticket?" and we have no captured context about that ticket, the correct behavior is to say so — not hallucinate an answer.

### Copy

#### Two-checkpoint refusal pattern

AnythingLLM checks twice, at different stages:

**Checkpoint 1 — No embeddings exist at all:**
```
if (chatMode === "query" AND (no vectorized namespace OR embeddingsCount === 0)):
  → return custom refusal message
  → save to history with include: false
  → exit before any LLM call
```

**Checkpoint 2 — Embeddings exist but full context assembly returned nothing:**
```
if (chatMode === "query" AND contextTexts.length === 0 after pinned + parsed + search + backfill):
  → return custom refusal message
  → save to history with include: false
  → exit before any LLM call
```

**Why two checkpoints:** Checkpoint 1 is a fast-path (skip vector search entirely if workspace is empty). Checkpoint 2 catches the case where embeddings exist but the full context assembly (pinned docs + parsed files + vector search + backfill) still returned nothing relevant.

**Key design decisions to copy:**

- **`include: false` on refusal messages** — the refusal is saved to chat history for the user to see, but flagged so it won't be loaded into future context windows. This prevents "I don't know" answers from polluting the context for later queries.
- **Custom refusal message** — `workspace.queryRefusalResponse` allows per-workspace customization. Default: `"There is no relevant information in this workspace to answer your query."`
- **No LLM call on refusal** — saves tokens and latency. The refusal is deterministic, not generated.
- **Chat mode bypasses both checkpoints** — in chat mode, the LLM can respond using only conversation history + system prompt, even with zero RAG context.

**Our mapping:**

| AnythingLLM | Cowork.ai |
|---|---|
| `chatMode === "query"` | Agent configured with `groundedOnly: true` (e.g., context-lookup tool) |
| `workspace.queryRefusalResponse` | Per-agent refusal message in agent config |
| `include: false` on history | `excluded_from_context: true` flag on `conversation_messages` row |
| Checkpoint 1 (no embeddings) | Check `activity_embeddings` count for relevant time window |
| Checkpoint 2 (no search results) | Check `contextTexts.length === 0` after vector search + backfill |

### Adapt

#### Refusal granularity

AnythingLLM has binary modes: "chat" (can hallucinate) vs "query" (must have evidence). Cowork.ai needs finer-grained control because our agents serve different functions:

- **Capture summary agent:** Should refuse if no captures exist for the time range asked about
- **Tool action agent:** Should refuse if the referenced MCP tool/integration isn't connected
- **General chat:** Should never refuse (freeform conversation)

**Implementation:** Instead of a global `chatMode`, each Mastra agent gets a `refusalPolicy` in its config: `"never"` | `"no-context"` | `"no-tool"`. The two-checkpoint pattern from AnythingLLM applies to `"no-context"` agents.

### Skip

#### Workspace-level mode toggle

AnythingLLM lets users toggle chat/query mode per workspace via a UI dropdown. This makes sense for their document-Q&A product.

**Why skip:** Cowork.ai's agents have inherent modes based on their function. The capture summary agent is always grounded; the chat agent is always freeform. The mode is architectural, not user-toggled.

---

## 2. MCP Hypervisor Lifecycle

**Source:** AnythingLLM's `server/utils/MCP/hypervisor/index.js` + `server/utils/MCP/index.js`

Cowork.ai's MCP Integrations feature (product-features.md §2) needs to manage the lifecycle of multiple MCP servers — stdio processes, HTTP endpoints, SSE connections. AnythingLLM's MCP Hypervisor is the most complete lifecycle manager in our competitive set, handling boot/stop/timeout/transport-abstraction in a single class.

### Copy

#### Singleton boot with autoStart gating

```
bootMCPServers():
  1. If any servers already running → return cached results (skip re-boot)
  2. For each server in config:
     a. If server.anythingllm.autoStart === false → skip, log reason
     b. Try #startMCPServer(name, server)
     c. Store result in mcpLoadingResults[name] = { status, message }
     d. On failure: cleanup (close client, delete from map)
  3. Return mcpLoadingResults
```

**Why copy:** The singleton check + autoStart gate prevents duplicate connections and lets users disable servers without removing config. The `mcpLoadingResults` map gives the UI a single source of truth for server status.

**Our mapping:** In Cowork.ai, MCP servers are managed by the Agents & RAG Utility Process (system-architecture.md §1.3). The hypervisor singleton lives there. The renderer gets status via IPC.

#### 30-second connection timeout

```
#startMCPServer():
  1. Parse server type (stdio | http | sse)
  2. Validate config by type
  3. Create Client + Transport
  4. Setup event handlers (onclose, onerror, onmessage)
  5. Race: mcp.connect(transport) vs setTimeout(30_000)
  6. On timeout → throw Error("Connection timeout") → caught by caller → cleanup
```

**Why copy:** Without a timeout, a misconfigured server (e.g., wrong command path, unreachable URL) blocks startup indefinitely. 30 seconds is generous enough for slow npx installs but catches hard failures.

#### SIGTERM cleanup on prune

```
pruneMCPServer(name):
  1. Access mcp.transport._process (ChildProcess for stdio servers)
  2. childProcess.kill("SIGTERM")
  3. mcp.transport.close()
  4. Delete from mcps map
  5. Set mcpLoadingResults[name] = { status: "failed", message: "stopped manually" }
```

**Why copy:** SIGTERM gives stdio MCP servers a chance to clean up (flush logs, close connections). The explicit `_process` access pattern is important because the MCP SDK's `close()` doesn't always kill the child process.

### Adapt

#### Transport setup location

AnythingLLM runs in a Node.js Express server — all transports (stdio, HTTP, SSE) are created directly. In Cowork.ai, transport creation depends on process:

| Transport | AnythingLLM | Cowork.ai |
|---|---|---|
| stdio | Express server spawns child process | Agents Utility Process spawns child process |
| HTTP/SSE | Express server creates HTTP client | Agents Utility Process creates HTTP client |
| In-memory | Not used | Built-in servers (like Cherry Studio's factory pattern) |

The key difference: our Agents Utility Process is a Node.js utility process inside Electron, not a standalone server. Spawning child processes from a utility process works fine (Node.js `child_process` is available), but we need to handle Electron's ASAR unpack for any native modules the MCP servers depend on.

#### Environment path resolution

AnythingLLM has special Docker handling:

```javascript
if (process.env.ANYTHING_LLM_RUNTIME === "docker") {
  baseEnv = { PATH: "/usr/local/bin:/usr/bin:/bin", ... }
}
```

**Our adaptation:** We don't run in Docker. But we have the same shell PATH problem — Electron apps don't inherit the user's shell PATH. Cherry Studio solves this with `shell-env` (covered in that guide). Copy Cherry Studio's approach for PATH resolution, not AnythingLLM's Docker-conditional logic.

#### Tool-to-agent plugin conversion

AnythingLLM converts MCP tools to "Aibitat plugins" via `convertServerToolsToPlugins()`:

```
For each tool from mcp.listTools():
  Create plugin = {
    name: `${serverName}-${toolName}`,
    handler: async (args) => mcpClient.callTool({ name, arguments: args }),
    parameters: tool.inputSchema,
    isMCPTool: true
  }
```

**Our adaptation:** We use Mastra, not Aibitat. Mastra has its own tool registration API. The conversion pattern is the same (MCP tool schema → agent-callable function), but the target format differs:

| AnythingLLM | Cowork.ai |
|---|---|
| Aibitat `function()` registration | Mastra `createTool()` |
| Plugin naming: `serverName-toolName` | Tool naming: `mcp__serverName__toolName` (Cherry Studio convention) |
| Error: return error string | Error: return as tool result (not throw) |

### Skip

#### JSON-file server config

AnythingLLM stores MCP server definitions in `storage/plugins/anythingllm_mcp_servers.json`. This is fine for a server app but fragile for a desktop app (file permissions, migration, backup).

**Why skip:** We store MCP server configs in libsql (`mcp_servers` table). Config changes are atomic transactions, queryable, and included in the single-file backup story.

#### Singleton re-instantiation pattern

AnythingLLM's `MCPCompatibilityLayer` creates a new instance on every tool call to get the current `mcps` map:

```javascript
handler: async function (args) {
  const mcpLayer = new MCPCompatibilityLayer();
  const currentMcp = mcpLayer.mcps[name];
  // ...
}
```

**Why skip:** This works because their singleton stores state in a module-scoped variable. In our utility process, the hypervisor instance is the single source of truth — tool handlers get a direct reference, no re-instantiation needed.

---

## 3. No-Code Agent Flow Executor

**Source:** AnythingLLM's `server/utils/agentFlows/executor.js` + `server/utils/agentFlows/flowTypes.js`

Cowork.ai's Automations feature (product-features.md §5) needs a flow execution engine: trigger → sequence of steps → output. AnythingLLM's flow executor is the only implementation in our competitive set that solves variable substitution, step chaining, direct output, and error halting in a clean sequential model.

### Copy

#### Variable substitution with dot notation

AnythingLLM's `replaceVariables()` pattern:

```
1. Walk every string in the step config (recursive deep replace)
2. Match pattern: ${varName.path.to.value}
3. Resolve via getValueFromPath():
   - Handles dot notation: "response.data.name"
   - Handles array indices: "items[0].title"
   - Handles nested brackets: "users[2].address.city"
   - Returns undefined if path doesn't resolve → keeps ${...} literal
4. Replace match with resolved value (or leave as-is if unresolved)
```

**Why copy:** This is exactly what Automations needs. A trigger fires, passes data into step 1, step 1 stores output in a variable, step 2 references `${step1_output.data.ticketId}` in its config. The dot-notation + bracket support handles real API response shapes.

**Our mapping:**

| AnythingLLM | Cowork.ai |
|---|---|
| `${api_response.data.id}` | Same syntax |
| `getValueFromPath(variables, path)` | Same function, port directly |
| `replaceVariables(config)` | Same function, port directly |
| Unresolved → keep `${...}` literal | Same behavior (allows partial resolution) |

#### Sequential step execution with error halting

```
executeFlow(flow, initialVariables, aibitat):
  1. Initialize variables from START block + caller overrides
  2. For each step in flow.config.steps:
     a. replaceVariables(step.config)
     b. executeStep(step) → dispatch by type
     c. If step returns directOutput: true → break, return immediately
     d. If step throws → break, mark success: false
     e. Store result in variables[step.responseVariable]
  3. Return { success, results[], variables, directOutput }
```

**Key design decisions to copy:**

- **Fail-fast** — first error stops the flow. No partial execution with dangling state.
- **Direct output** — a step can set `directOutput: true` to bypass remaining steps and return its result immediately. Useful for "early return" patterns (e.g., API returns cached result, skip LLM processing).
- **Variable accumulation** — each step's output is stored back into the variables map, available to all subsequent steps. Simple and predictable.
- **Caller overrides** — `initialVariables` from the trigger can override START block defaults. The trigger knows runtime values (e.g., which ticket ID fired the automation).

### Adapt

#### Block types

AnythingLLM supports 4 block types: `start`, `apiCall`, `llmInstruction`, `webScraping`.

Cowork.ai's Automations need different block types aligned with our features:

| AnythingLLM block | Cowork.ai equivalent | Notes |
|---|---|---|
| `start` | `trigger` | Same concept: initialize variables. But ours is event-driven (capture event, schedule, MCP notification) |
| `apiCall` | `mcpToolCall` | We call MCP tools, not raw HTTP endpoints. The MCP layer handles auth/transport. |
| `llmInstruction` | `agentStep` | Route to a specific Mastra agent (with model selection via complexity router) |
| `webScraping` | Skip | We capture screen data via the Capture Utility, not web scraping |

**Additional block types we need:**

- `condition` — if/else branching based on variable values (AnythingLLM doesn't have this)
- `delay` — wait N seconds (for rate-limited tool calls)
- `notification` — send result to user via system notification

#### Flow storage

AnythingLLM stores flows as JSON files: `storage/plugins/agent-flows/{uuid}.json`.

**Our approach:** Store flow definitions in libsql (`automations` table). The flow config is a JSON column. This gives us: atomic updates, queryable metadata (name, trigger type, enabled/disabled), and inclusion in the single-DB backup.

#### Flow-as-tool registration

AnythingLLM registers flows as Aibitat agent tools, so the LLM can invoke a flow mid-conversation:

```javascript
aibitat.function({
  name: `flow_${uuid}`,
  handler: (args) => executeFlow(uuid, args, aibitat)
})
```

**Our adaptation:** Register automations as Mastra tools. The agent can invoke an automation as a tool call, passing variables as arguments. The flow executor runs in the Agents Utility Process.

### Skip

#### Web scraping block

AnythingLLM's `webScraping` block uses a collector service to fetch + parse web pages, with optional LLM summarization for long content.

**Why skip:** Cowork.ai captures screen content via the Capture Utility Process (screenshots, OCR). We don't need a separate web scraping pipeline. If a user wants web content, they browse to it and we capture it.

#### Aibitat integration specifics

The tight coupling between flows and Aibitat's `introspect()` / `handlerProps.log` / `getProviderForConfig()` APIs doesn't translate to Mastra. Copy the executor logic, not the Aibitat-specific wiring.

---

## 4. Summary

### Copy directly

| Pattern | Source | Cowork.ai target | Rationale |
|---|---|---|---|
| Multi-source context assembly order | `stream.js:115-196` | Agents Utility — RAG pipeline | Proven priority order: pins → transient → search → backfill (pins can be displaced by compression) |
| `fillSourceWindow` backfill algorithm | `helpers/chat/index.js:348-442` | Agents Utility — context builder | Solves follow-up query problem without re-embedding |
| Two-checkpoint query refusal | `stream.js:65-92, 198-227` | Per-agent refusal policy | Prevents hallucination in grounded-query scenarios |
| `include: false` on refusal history | `stream.js:80` | `excluded_from_context` flag | Prevents "I don't know" from polluting future context |
| Singleton MCP boot + autoStart gate | `hypervisor/index.js:434-491` | Agents Utility — MCP manager | Prevents duplicate connections, respects user config |
| 30-second connection timeout | `hypervisor/index.js:389-427` | Agents Utility — MCP manager | Catches misconfigured servers without blocking startup |
| SIGTERM cleanup on server prune | `hypervisor/index.js:192-232` | Agents Utility — MCP manager | Graceful stdio process termination |
| Variable substitution (dot + bracket) | `executor.js:30-128` | Automations engine | Handles real API response shapes with ${var.path[0].key} |
| Sequential execution with fail-fast | `executor.js:188-229` | Automations engine | Simple, predictable, debuggable flow execution |
| Direct output early-return | `executor.js:217-220` | Automations engine | Skip remaining steps when result is already available |

### Study (learn from, adapt approach)

| Pattern | What to learn | How it changes for us |
|---|---|---|
| Cannonball middle-truncation | Fast compression when token limit exceeded | Use Vercel AI SDK token management + recency-weighted strategy |
| LiteLLM context-window map caching | Remote model metadata with local fallback | Useful for OpenRouter model selection; cache in libsql |
| Plugin naming (`#`, `@@flow_`, `@@mcp_`) | Late-binding function registry with namespace prefixes | Adapt to Mastra tool registration format |
| Ephemeral vs stateful agent runtimes | Different lifetime models for API vs interactive agents | Maps to our tool-call agents (ephemeral) vs conversation agents (stateful) |
| Parsed-files transient context | Pre-embedding temporary documents in thread scope | Maps to our capture buffer (recent data before embedding completes) |
| Flow executor logging via `introspect()` | Real-time step-by-step visibility into flow execution | Implement via IPC events from Agents Utility → Renderer |

### Skip

| Pattern | Why skip |
|---|---|
| Giant switch-based provider factory | Use Mastra's provider registry + Vercel AI SDK provider abstraction |
| File-based document JSON persistence | Use libsql for everything (single-DB story) |
| JSON-file MCP server config | Store in libsql `mcp_servers` table |
| JSON-file flow storage | Store in libsql `automations` table |
| Monolithic ~316-line chat handler | Mastra separates tools, context, memory into composable pieces |
| Web scraping block type | We capture screen content via Capture Utility, not web scraping |
| `@agent` prefix invocation | Our agents are invoked by the system (routing), not by user prefix |
| Workspace-level chat/query mode toggle | Agent function determines grounding mode, not user toggle |
| ENV-driven provider switching | Typed config in libsql, not environment variables |
| Docker runtime conditionals | Desktop-only app, no Docker path |

---

## Changelog

**v1 (Feb 17, 2026):** Initial draft. All 4 sections complete.

**v1.1 (Feb 17, 2026):** Fact-check corrections: pinned docs can be displaced by cannonball compression, history limit is configurable via `workspace.openAiHistory` (not always 20), checkpoint 2 runs after full context assembly (not just search+backfill), autoStart check path is `server.anythingllm.autoStart`, stream.js is ~316 lines.
