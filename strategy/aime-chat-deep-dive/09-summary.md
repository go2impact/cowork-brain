# 9. Summary: Copy / Study / Skip

[< Back to index](README.md)

## Copy These Patterns

These are proven, MIT-licensed patterns we should adopt:

| Pattern | Source File | Why |
|---------|------------|-----|
| `IpcChatTransport` (AI SDK ↔ Electron IPC bridge) | `src/renderer/pages/chat/ipc-chat-transport.ts` | Custom `ChatTransport<UIMessage>` bridging `useChat()` across Electron process boundary. The most valuable AI SDK pattern — streaming, abort, Zod-validated chunks. |
| `@ai-sdk/openai-compatible` for Ollama | `src/main/providers/ollama-provider.ts` | One-liner local brain connection: `createOpenAICompatible({ baseURL: 'http://localhost:11434' })`. No custom HTTP client needed. |
| Zod-validated streaming chunk protocol | `src/renderer/pages/chat/ipc-chat-transport.ts` | `uiMessageChunkSchema` — type-safe contract between main and renderer for all chunk types (text, reasoning, tool lifecycle, sources, meta). |
| `BaseManager` + `@channel` IPC decorator | `src/main/BaseManager.ts`, `src/main/ipc/IpcController.ts` | Clean Electron IPC with zero boilerplate. Auto-registers handlers. |
| libsql vector schema (`F32_BLOB`, `vector_distance_cos`) | `src/main/knowledge-base/index.ts` | Exact SQL patterns for our activity embeddings. |
| Context compression via configurable fast model | `src/main/mastra/index.ts` (compressMessages) | Keeps long conversations within context limits. Their implementation uses `appInfo.defaultModel.fastModel` (user-configurable). We should wire this to our complexity router's local brain. |
| Tool approval flow (suspended → approve → resume) | `src/main/mastra/index.ts` | Direct implementation for our MCP Browser approval gates. |
| Background task queue with concurrency groups | `src/main/task-queue/index.ts` | Need this for automations, embedding jobs, MCP operations. |
| MCPClient status management | `src/main/tools/index.ts` | Status tracking, reconnect, health monitoring for our MCP Integrations. |
| CDP remote browser connection | `src/main/instances/browser-instance.ts` | Attach to user's logged-in Chrome for MCP Browser execution. **Study with constraints** — requires explicit consent, active indicator, and privacy guardrails per our anti-surveillance positioning. |
| Screen capture (macOS `screencapture` + desktopCapturer fallback) | `src/main/app/index.ts` | Native-quality capture for our Context feature. |

## Study These (Reference, Don't Copy Directly)

| Topic | Source | What to Learn |
|-------|--------|---------------|
| Stagehand (LLM-driven browser automation) | `src/main/instances/browser-instance.ts` | LLM decides what to click — next evolution of our browser automation. Evaluate for v0.2+. |
| `@mastra/rag` v2 chunking | `src/main/knowledge-base/index.ts` | Chunking strategies, parameters. Benchmark for our activity data. |
| Reranker integration | `src/main/knowledge-base/index.ts` | Second-pass ranking for RAG quality. Worth adding to our retrieval pipeline. |
| `toAISdkFormat()` (Mastra → AI SDK bridge) | `src/main/mastra/index.ts`, `@mastra/ai-sdk` | How Mastra agent streams convert to AI SDK chunk types. The glue between orchestration and UI. |
| AI SDK `gateway` API | `src/main/providers/index.ts` | Imported but unused by AIME. Evaluate for our provider routing between OpenRouter and direct Ollama. |
| `generateObject()` for structured output | Not used by AIME — evaluate for our complexity router | Zod-schema-validated structured LLM output without manual JSON parsing. |
| `models.json` capability metadata | `assets/models.json` | Model costs, context limits, capabilities. Feed our complexity router. |
| TypeORM entity schema | `src/entities/*.ts` | Their 12-table schema is a reference for what app state needs persisting. |
| Preload script organization | `src/main/preload.ts` | 383 lines of typed IPC bindings. Reference for our API surface. |
| Python code execution via `uv` + MCP | `src/main/tools/code/code-execution.ts` | If we ever need code execution, this sandboxing approach is solid. |
| OAuth for MCP servers | `src/main/tools/mcp/oauth-client-provider.ts` | We need OAuth for Zendesk/Gmail/Slack MCP connections. |

## Skip These

| Feature | Why |
|---------|-----|
| 15+ exposed AI providers | We abstract behind OpenRouter + complexity router. User picks a tier, not a model. |
| Manual document upload KB | Our context comes from ambient capture, not user uploads. |
| Built-in code execution (Python/Node) | Not relevant for our target user (remote workers, not developers). |
| Built-in OCR, image gen, audio tools | Feature bloat. We expose capabilities via MCP, not built-in tools. |
| Excalidraw, tldraw, drawing tools | Not relevant for our UX. |
| Project-based workspace organization | We're a sidecar (always-on), not a project tool. |
| `synchronize: true` (auto-migrate) | Fine for solo dev, risky for production. Use proper migrations. |
| i18n (for now) | English-only for v0.1. |

## What They Don't Have (Our Moat)

These are features in our `product-features.md` that AIME Chat has no equivalent for:

| Cowork.ai Feature | Their Gap |
|-------------------|-----------|
| **Ambient context capture** (6 input streams) | They only capture via manual screenshot triggers |
| **Proactive notifications** (push model) | Purely reactive — user must initiate everything |
| **App ecosystem** (AI Studio → Cowork, scoped MCP) | They have a skills system (user-created automation scripts via SkillCreatorAgent), but skills require code and have no marketplace/distribution model. No scoped MCP access for third parties. |
| **Automations** (time/event/activity triggers) | They have a `TaskQueueManager` for background jobs, but it's internal-only infrastructure — no user-facing trigger system (time/event/activity-based). |
| **Complexity router** (auto model selection) | User manually picks model from 15+ options |
| **Anti-surveillance positioning** ("worker owns data") | "Local-first" (generic), no anti-surveillance narrative |
| **Context Card** (ambient awareness UI) | No equivalent — no ambient data to display |

---

## Changelog

| Date | Changes |
|------|---------|
| 2026-02-16 | Initial deep dive from codebase (v0.3.17). All 8 feature areas reverse-engineered with ASCII diagrams. |
| 2026-02-16 | Applied Codex review fixes: corrected single-DB architecture (not two DBs), fixed OAuth paths, fixed compression model description, added registered vs utility agent distinction, removed MCP "client-only" contradiction, softened moat claims, added privacy guardrails to CDP recommendation, fixed cross-doc links, added utility-process caveats. Added Section 9: Architecture Comparison vs system-architecture.md. |
| 2026-02-16 | Fact-check against actual codebase (`~/code/aime-chat/`). Fixed 9 errors: agent registration table (4 registered not 2, all isHidden=true, SkillCreator/Translation not registered, CompressAgent standalone); `@napi-rs/ocr` → `@napi-rs/system-ocr`; BashToolkit/FileSystem tool grouping corrected; provider switch corrected (removed phantom Anthropic/XAI cases, added BraveSearch/Serpapi/Tavily/JinaAI, added OpenAI-compatible fallback); provider count 12 → 14; adapter list corrected (`@ai-sdk/deepseek` active, `@ai-sdk/anthropic`+`@ai-sdk/xai` unused); preload.ts line count 1,384 → 383; "Framer Motion + Motion v12" → "Motion v12 (formerly Framer Motion)"; added Tabler Icons alongside Lucide React. |
| 2026-02-16 | Added Section 0: Foundational Layer — Vercel AI SDK. Full reverse-engineering of how AIME Chat uses AI SDK: 10 packages mapped, IPC streaming protocol documented (IpcChatTransport + Zod-validated chunk types), provider gateway pattern, embedMany() usage, OpenAI-compatible adapter for Ollama, toAISdkFormat() bridge, tool schema patterns. Updated Summary with 3 new Copy patterns and 3 new Study items. |
| 2026-02-16 | Resolved AI SDK version decision: v6 (not v5). Research confirmed Mastra 1.0+ supports both via npm aliasing, breaking changes are mechanical renames with automated codemod, core patterns (useChat, ChatTransport, chunk protocol) stable across versions. Updated: Implications §4, What We Should Do table, Architecture Comparison AI Foundation table, What This Means §5. Added decision log entry. |
| 2026-02-16 | Codex review fixes for Section 0: (1) `createGateway`/`gateway` imported but never called — rewrote provider gateway subsection and Study items to note dead import; (2) `embedMany()` not called in `local-provider.ts` — clarified it implements `EmbeddingModelV2` interface, doesn't call `embedMany()`; (3) OpenAI-compatible adapter usage overstated — noted DeepSeek has dead code, ModelScope commented out; (4) `generateText()` imported in 24 tool files but zero call sites — rewrote subsection 6 as dead imports; (5) `parameters` → `inputSchema` in comparison table; (6) Added missing `error` and `file` chunk types, noted `sendReasoning: false` disables reasoning in main stream; (7) Removed `/v1` suffix from Ollama URL (3 locations). |
| 2026-02-16 | Expanded Section 1 agent documentation. Replaced simple agent table with comprehensive per-agent deep dive: full system prompts, tool lists, model configuration, and invocation patterns for all 7 agents. Added: agent selection logic (no auto-routing — user-driven + fallback), sub-agent delegation via Task tool (dynamic description, stateless spawning, maxSteps: 100, step-result streaming), Plan mode workflow (5-phase structured approach with EnterPlanMode/ExitPlanMode), agent lifecycle & memory configuration (ephemeral instances, Memory options, MessageHistory processor). Fixed CodeAgent visibility bug (was listed as `isHidden=true`, actually `isHidden=false` — only visible agent). Noted Plan agent description copy-paste bug (uses Explore's description). Noted CompressAgent hardcoded model is dead code (overridden at runtime). |
| 2026-02-16 | Rewrote agent selection, sub-agent delegation, and plan mode sections with design rationale. Added "Why These Agents Exist" section explaining context isolation as the core architectural motivation. Documented UI-side agent experience: ChatAgentSelector dialog, Task tool invocation cards (`@Explore`/`@Plan` badges), data-task streaming, plan mode user flow. Added ASCII diagrams for agent selector UI and Task tool output rendering. Connected each agent to its end-user feature. Explained why Plan mode's read-only constraint is prompt-enforced not system-enforced, why ExitPlanMode auto-approves, and why the plan file path is hardcoded. |
| 2026-02-16 | Split into subdirectory `strategy/aime-chat-deep-dive/` with one file per section for navigability. |
