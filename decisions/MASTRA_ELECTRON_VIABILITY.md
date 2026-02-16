# Mastra in Electron Utility Process: Viability Research

| | |
|---|---|
| **Status** | Resolved — GO (all 7 PoC steps passed Feb 17, 2026) |
| **Last Updated** | 2026-02-17 |
| **Question** | Can Mastra.ai run embedded in an Electron utility process (not a separate server, not the main process)? |
| **Related** | [DESKTOP_SALVAGE_PLAN.md](./DESKTOP_SALVAGE_PLAN.md), [DATABASE_STACK_RESEARCH.md](./DATABASE_STACK_RESEARCH.md), [DESKTOP_FRAMEWORK_DECISION.md](./DESKTOP_FRAMEWORK_DECISION.md) |

---

## Why This Matters

Our architecture (DESKTOP_SALVAGE_PLAN.md) puts Mastra in an "Agents & RAG utility process" — crash-isolated from the main process, communicating via IPC. If this doesn't work, we either:

- Run Mastra in the main process (crash = app crash), or
- Run Mastra as a separate server process (operational complexity, two processes to manage)

Both are worse. The utility process pattern is the right answer *if it works*.

---

## Three Architecture Patterns

| Pattern | Crash isolation | Complexity | Who does this |
|---|---|---|---|
| **Main process** | None — Mastra crash kills the app | Lowest | aime-chat (only known Mastra+Electron app) |
| **Separate server** | Full — separate OS process | Highest — two processes, HTTP overhead, CORS | Official Mastra Electron guide |
| **Utility process** | Full — Chromium child process, auto-restartable | Medium — IPC wiring, no HTTP needed | **Nobody yet** |

---

## What IS Proven

### 1. Mastra runs embedded in Electron (main process)

**[aime-chat](https://github.com/DarkNoah/aime-chat)** — 21 stars, actively developed (last push Feb 15, 2026). Cross-platform AI desktop chat app.

Stack: Electron 35 + @mastra/core beta.19 + @mastra/libsql beta.10 + @libsql/client 0.15.15 + @mastra/fastembed + @mastra/mcp + @mastra/memory + @mastra/rag.

Architecture: Mastra runs directly in `src/main/main.ts` (Electron main process). Storage uses `LibSQLStore` and `LibSQLVector` from `@mastra/libsql` pointing to a local file DB. Agents are called programmatically — no HTTP server between main and renderer.

**What this proves:**
- @mastra/core initializes in Electron's Node.js context
- @mastra/libsql native bindings load and work in Electron
- `LibSQLStore` + `LibSQLVector` work with local `file:` URLs
- Agents run via programmatic API (no server required)
- IPC between main process and renderer works for agent responses

### 2. Mastra works without an HTTP server

`agent.generate()` and `agent.stream()` are direct method calls — no HTTP involved:

```typescript
import { Agent } from "@mastra/core/agent";

const agent = new Agent({
  name: "my-agent",
  instructions: "...",
  model: someModel,
});

// Direct call — no server, no HTTP, no port binding
const result = await agent.generate("What should I do next?");
```

Mastra also supports an in-memory server pattern (Hono adapter with `serverApp.fetch()`) that simulates HTTP without binding a port. But for our use case, direct `agent.generate()`/`agent.stream()` is simpler.

### 3. Electron utility processes support everything Mastra needs

Utility processes (introduced Electron 22, mature by Electron 37) provide:

| Capability | Status | Notes |
|---|---|---|
| Full Node.js APIs (fs, net, http, crypto) | Supported | Same as main process |
| N-API / node-addon-api native addons | Supported | Must be rebuilt for Electron ABI (same as main process) |
| IPC with main process | Supported | `process.parentPort.postMessage()` |
| MessagePort to renderer | Supported | Main process creates channel, transfers ports |
| Crash isolation | Supported | Utility crash does NOT kill main process |
| Auto-restart on crash | Supported | Main process can detect exit and re-fork |
| HTTP client requests | Supported | Both Node.js `https` and `electron.net` (Electron 27+) |
| V8 memory | ~4GB limit | Standard V8 cap on 64-bit |

**Real-world precedent:** VS Code uses utility processes for sandboxed Node.js workloads. @electron/llm (official Electron project) runs node-llama-cpp inference in a utility process with Mojo IPC for streaming.

### 4. @mastra/libsql native bindings work in Electron

Proven by aime-chat. The `@libsql/client` package uses N-API bindings with platform-specific Rust-compiled `.node` binaries distributed as `@libsql/[platform]` packages. These load correctly in Electron after ABI rebuild.

---

## What is NOT Proven

### 1. @mastra/core initialization in a utility process

Utility processes have the same Node.js APIs as the main process, but differ in module resolution context. Nobody has tested whether `@mastra/core`'s initialization (which scans for agents, tools, workflows) works in the utility process context.

**Risk: Low.** The utility process is a standard Node.js environment. Module resolution uses the same `require()`/`import` paths. There's no technical reason this should fail — but "no technical reason" is not the same as "tested."

### 2. @mastra/libsql native bindings in utility process

Native `.node` files load via `process.dlopen()` which works in utility processes. But the ASAR unpacking path and Electron's native module resolution may differ between main and utility processes.

**Risk: Medium.** This is the most likely point of failure. The existing `forge.config.ts` handles ASAR unpacking for the main process. We need to verify the unpacked native modules are accessible from the utility process's file path context.

### 3. Streaming agent responses via IPC

Our architecture needs `agent.stream()` results piped from the utility process to the renderer in real-time. This requires:

1. Utility process runs `agent.stream()` → gets async iterator
2. Each chunk is sent via `process.parentPort.postMessage()` to main
3. Main forwards to renderer via MessagePort or webContents

**Risk: Low.** MessagePort supports transferable objects and @electron/llm demonstrates this exact pattern (streaming LLM tokens from utility process to renderer). The IPC overhead per chunk is negligible.

### 4. Multiple Mastra agents + memory + tools in one utility process

aime-chat runs multiple agents in the main process. But the utility process has the same V8 memory limits. The question is whether Mastra's memory footprint plus libsql plus active agents stays reasonable.

**Risk: Low.** Mastra doesn't spawn child processes or worker threads internally. Memory usage is dominated by the LLM responses (which come from Ollama/OpenRouter, not Mastra itself). LibSQL is lightweight. The utility process's ~4GB V8 limit is more than sufficient.

### 5. Crash recovery and state preservation

If the utility process crashes mid-stream, we need to:
- Detect the crash in the main process (via `child.on('exit')`)
- Re-fork the utility process
- Re-initialize Mastra + libsql
- Resume or retry the interrupted operation

**Risk: Medium.** The crash detection and restart are straightforward Electron patterns. The harder part is preserving conversation state across restarts. Since libsql persists to disk, memory and thread state should survive — but this needs testing.

---

## Known Gotchas

### Event loop blocking bug (Electron 29+)

If the event loop is blocked for several seconds in a utility process, network requests made immediately after may fail with `ECONNRESET`. This is a [known Electron bug](https://github.com/electron/electron/issues/43186).

**Impact on us:** If a long-running synchronous operation blocks the event loop (e.g., large embedding computation), the next Ollama/OpenRouter API call might fail. Mitigation: add a `setTimeout(resolve, 10)` delay after heavy sync operations before making HTTP requests.

### ASAR unpacking for utility processes

Native `.node` files can't load from ASAR archives. The existing `AutoUnpackNativesPlugin` in `forge.config.ts` handles this, but we need to verify that utility processes resolve the unpacked paths correctly.

**Mitigation:** Test with a packaged (not dev) build. The utility process entry script must resolve native module paths relative to the app's resources directory, not the ASAR.

### ESM/CJS compatibility

`@libsql/client` has [known CommonJS/ESM module issues](https://github.com/tursodatabase/libsql-client-ts/issues/182). Electron's utility process may use either module system depending on the entry script configuration.

**Mitigation:** Use the `libsql` (sync) package for the capture utility process and `@libsql/client` (async, used by @mastra/libsql) for the agents utility process. Test both in the utility process context.

---

## What the Official Docs Say

### Mastra Electron Guide

The [official Mastra Electron guide](https://mastra.ai/guides/getting-started/electron) recommends a **client-server architecture**:

- Mastra runs as a separate dev server (`mastra dev`) on `localhost:4111`
- Electron renderer connects via HTTP with AI SDK's `useChat()` hook
- Requires two terminals, CORS configuration, CSP header changes

This is fine for development but wrong for production — users shouldn't run a separate server process. The guide doesn't mention utility processes or embedded patterns.

### Electron Process Model Docs

Electron's [process model documentation](https://www.electronjs.org/docs/latest/tutorial/process-model) explicitly recommends utility processes for:

- Untrusted services
- CPU-intensive tasks
- Crash-prone components that would previously run in the main process

Our Mastra + agents layer fits all three criteria.

---

## Assessment

| Component | Viability | Confidence | Evidence |
|---|---|---|---|
| @mastra/core in utility process | Viable | **High** (85%) | Same Node.js APIs as main process; aime-chat proves Electron compatibility |
| @mastra/libsql in utility process | Viable | **Medium** (70%) | N-API works in utility processes; ASAR unpacking path needs verification |
| agent.generate() via IPC | Viable | **High** (90%) | @electron/llm proves the pattern; MessagePort is well-documented |
| agent.stream() via IPC | Viable | **High** (85%) | Same pattern as LLM token streaming in @electron/llm |
| Crash isolation + restart | Viable | **High** (90%) | Core Electron feature; state persists in libsql on disk |
| Full architecture (all together) | Viable | **Medium** (65%) | No one has done exactly this; each piece works, integration is untested |

**Overall: Architecturally sound, practically untested.** Each individual component is proven or has strong precedent. The integration of all components in a utility process is the gap.

---

## Recommended Spike

Before committing this architecture to the codebase, run a proof-of-concept:

### Spike scope (1-2 days)

1. **Minimal Electron app** with utility process (`utilityProcess.fork()`)
2. **Import @mastra/core + @mastra/libsql** in the utility process entry script
3. **Initialize Mastra** with one agent, `LibSQLStore`, and `LibSQLVector` pointing to a local file DB
4. **Call `agent.generate()`** from main process via IPC, get response back
5. **Stream `agent.stream()`** from utility process to renderer via MessagePort
6. **Test crash isolation** — kill utility process, verify main process and renderer survive, re-fork and resume
7. **Package the app** with Electron Forge — verify ASAR unpacking and native module resolution work in the packaged build

### What success looks like

- Agent responds to prompts from the renderer
- Streaming tokens arrive in real-time in the renderer
- Killing the utility process doesn't crash the app
- Packaged build works on macOS (Apple Silicon)

### Fallback if it fails

If the utility process doesn't work (most likely failure: native module loading), fall back to:

1. **Main process** (aime-chat's pattern) — accept the crash risk, add defensive error handling
2. **Separate process** (official guide's pattern) — Mastra runs as a local HTTP server spawned by the app

---

## Sources

- [Electron UtilityProcess API](https://www.electronjs.org/docs/latest/api/utility-process) — official docs
- [Electron Process Model](https://www.electronjs.org/docs/latest/tutorial/process-model) — official tutorial
- [Electron MessagePorts](https://www.electronjs.org/docs/latest/tutorial/message-ports) — IPC patterns
- [@electron/llm](https://github.com/electron/llm) — official Electron project running LLM inference in utility process
- [aime-chat](https://github.com/DarkNoah/aime-chat) — only known Mastra+Electron production app (main process pattern)
- [Mastra Electron Guide](https://mastra.ai/guides/getting-started/electron) — official, recommends client-server
- [Mastra Agent API](https://mastra.ai/docs/agents/overview) — agent.generate() and agent.stream() docs
- [@libsql/client issues](https://github.com/tursodatabase/libsql-client-ts/issues/182) — ESM/CJS compatibility
- [Electron event loop bug #43186](https://github.com/electron/electron/issues/43186) — ECONNRESET after blocking
- [VS Code sandbox migration](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox) — utility process precedent

---

## Changelog

**v1 (Feb 16, 2026):** Initial research. Assessed viability of running Mastra.ai in Electron utility process. Found: architecturally sound, practically untested. One real-world Mastra+Electron app (aime-chat) runs in main process, not utility. Recommended proof-of-concept spike before committing architecture.
