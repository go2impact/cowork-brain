# 8. Architecture Comparison: AIME Chat vs Cowork.ai

[< Back to index](README.md)

Side-by-side comparison of architectural decisions. AIME Chat is the existing reference implementation; Cowork.ai is our target architecture per [system-architecture.md](../../../architecture/system-architecture.md).

## Process Model

```
AIME CHAT (monolithic)                    COWORK.AI (multi-process)
──────────────────────                    ────────────────────────

┌─── MAIN PROCESS ──────┐                ┌─── MAIN PROCESS ──────┐
│                        │                │                        │
│  12 Manager singletons │                │  Coordinator only      │
│  Express API server    │                │  IPC router            │
│  Mastra + agents       │                │  System tray           │
│  RAG + embeddings      │                │  Thermal monitor       │
│  MCP clients           │                │  No heavy work         │
│  Playwright pool       │                │                        │
│  TypeORM + libsql      │                └────────┬───────────────┘
│  ALL heavy work here   │                         │
│                        │                ┌────────┼────────────┐
└────────────────────────┘                │        │            │
         │                          ┌─────▼──┐ ┌──▼────────┐ ┌▼──────────┐
    Context-Isolated IPC            │Capture │ │Agents &   │ │Renderer   │
         │                          │Utility │ │RAG Utility│ │(sandboxed)│
┌────────▼───────────────┐          │        │ │           │ │           │
│  RENDERER              │          │Native  │ │Mastra.ai  │ │React+Vite │
│  React 19 + Webpack    │          │addons  │ │Ollama     │ │M3 Design  │
│  shadcn/ui + Tailwind  │          │libsql  │ │OpenRouter │ │Tailwind   │
│  Zustand + Router 7    │          │(sync)  │ │MCP SDK    │ │           │
└────────────────────────┘          └────────┘ │libsql     │ └───────────┘
                                               │(async)    │
                                               └─────┬─────┘
                                                     │
                                               ┌─────▼─────┐
                                               │Playwright  │
                                               │Child Proc  │
                                               └────────────┘
```

**Key difference:** AIME runs everything in the main process. If Mastra crashes, the whole app dies. Our multi-process model isolates crashes — capture, agents, and UI are independent. This matters because native C++ addons can segfault, embedding generation is CPU-heavy, and Playwright can hang.

## Database

| Aspect | AIME Chat | Cowork.ai |
|--------|-----------|-----------|
| **Files** | Single `main.db` | Single `cowork.db` |
| **Access layers** | TypeORM+better-sqlite3 (app data) + libsql (Mastra/vectors) — two ORMs on one file | libsql only — one SDK for everything |
| **ORM** | TypeORM with `synchronize: true` (auto-migrate) | Raw libsql (manual migrations) |
| **Vector storage** | libsql `F32_BLOB` + `vector_distance_cos()` | Same — libsql built-in vector search |
| **Concurrent access** | Single process, no contention | WAL mode — Capture writes (sync), Agents reads/writes (async) |

## AI Foundation

| Aspect | AIME Chat | Cowork.ai |
|--------|-----------|-----------|
| **AI SDK** | Vercel AI SDK v5 (`ai` ^5.0.93) — 10 packages | AI SDK v6 (`ai` ^6.0.x, `@ai-sdk/react` ^3.0.x). Mastra 1.0+ supports both v5/v6. |
| **Orchestration** | Mastra.ai wraps AI SDK via `@mastra/ai-sdk` | Same — Mastra wraps AI SDK |
| **Streaming** | AI SDK chunk protocol → IPC → `useChat()` via custom `IpcChatTransport` | Same pattern needed — IPC bridge is the key problem |
| **Provider adapters** | `@ai-sdk/openai`, `@ai-sdk/google`, `@ai-sdk/deepseek` + `@ai-sdk/openai-compatible` for Ollama | `@ai-sdk/openai-compatible` for Ollama (local brain); OpenRouter for cloud (single adapter) |
| **Embeddings** | `embedMany()` from `ai` + `EmbeddingModelV2` interface | Same `embedMany()` — v6 renames `textEmbeddingModel()` → `embeddingModel()`, `EmbeddingModelV2` → `EmbeddingModelV3` |
| **Tool contract** | Zod schemas + `BaseTool` → AI SDK `tool()` / Mastra `createTool()` | Same via Mastra |

## Bundler & UI

| Aspect | AIME Chat | Cowork.ai |
|--------|-----------|-----------|
| **Bundler** | Webpack (HMR dev, code-split prod) | Vite |
| **Component library** | shadcn/ui (Radix primitives) | M3 (Material Design 3) |
| **State management** | Zustand v5 | TBD |
| **Routing** | React Router v7 | TBD |

## Feature Gaps

| Capability | AIME Chat | Cowork.ai |
|------------|-----------|-----------|
| **Ambient capture** | Manual screenshot only | 6 continuous input streams (window, focus, browser, keystroke, screen, clipboard) |
| **Native addons** | None — pure JS | Custom C++ N-API (`coworkai-keystroke-capture`, `coworkai-activity-capture`) |
| **Proactive notifications** | None — purely reactive | Agent-triggered macOS native notifications |
| **Complexity router** | User manually selects model | Rule-based auto-routing (<10ms, local vs cloud) |
| **Thermal management** | None | macOS `ProcessInfo.thermalState` polling, auto-degradation |
| **App ecosystem** | Monolithic, skills require code | App Gallery with scoped MCP permissions |
| **User-facing automations** | None (internal task queue only) | Time/event/activity triggers |

## What This Means for Implementation

1. **IPC patterns translate, process boundaries don't.** Their `BaseManager` + `@channel` pattern is clean, but every manager lives in main. We need to adapt: some managers in Capture Utility, some in Agents & RAG Utility, communication via Electron `MessagePort`.

2. **Their Mastra integration validates ours.** They prove Mastra + libsql + Electron works. Our utility process approach was validated by the PoC spike (Feb 17, 2026).

3. **Their DB approach is messier, ours is cleaner.** Two ORMs on one file is fragile. Our single-SDK approach (libsql for everything) avoids TypeORM's `synchronize: true` risks and double-locking concerns.

4. **Their tool system is a reference, not a template.** 25+ built-in tools is bloat. We expose capabilities via MCP servers, keeping the core lean. But their `BaseTool` abstraction and Zod schemas are clean patterns to adopt.

5. **AI SDK is the invisible foundation we share.** Both stacks build on Vercel AI SDK → Mastra → Electron. Their `IpcChatTransport` solves the hardest integration problem (streaming across Electron IPC) and is directly reusable. We start on AI SDK v6 (AIME uses v5) — Mastra 1.0+ supports both, breaking changes are mechanical renames, and the core patterns (`useChat()`, `ChatTransport`, chunk protocol) are stable across versions.
