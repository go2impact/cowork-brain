# 0. Foundational Layer: Vercel AI SDK

[< Back to index](README.md)

**What it is:** [Vercel AI SDK](https://ai-sdk.dev/) (package: `ai`) is an open-source TypeScript toolkit for building AI-powered applications. It provides the unified primitives — streaming, tool calling, embeddings, provider switching — that everything else in AIME Chat builds on.

**Why this matters:** AI SDK is invisible infrastructure. It doesn't have a visible feature or UI — it's the plumbing underneath Mastra, the provider system, embeddings, and streaming. But it's the single most important dependency in the stack. Mastra wraps it. Providers implement its interfaces. The React frontend consumes its hooks. Every LLM interaction flows through AI SDK primitives.

**Maps to Cowork.ai:** Every feature. AI SDK is the interface contract between our agents and LLM providers, and between our main process and React frontend.

## Packages Used by AIME Chat

| Package | Version | Role |
|---------|---------|------|
| `ai` | ^5.0.93 | Core — `streamText`, `generateText`, `embedMany`, `createUIMessageStream`, tool calling, message types |
| `@ai-sdk/react` | ^2.0.87 | Frontend — `useChat` hook with custom IPC transport |
| `@ai-sdk/openai` | ^2.0.63 | Provider adapter (OpenAI API — GPT, embeddings, TTS) |
| `@ai-sdk/google` | ^2.0.29 | Provider adapter (Gemini — with thinkingConfig support) |
| `@ai-sdk/deepseek` | ^1.0.31 | Provider adapter (DeepSeek — with extended thinking) |
| `@ai-sdk/openai-compatible` | ^1.0.28 | Generic adapter for any OpenAI-API-compatible endpoint (Ollama, LMStudio, ModelScope, ZhipuAI) |
| `@ai-sdk/anthropic` | ^3.0.0 | Provider adapter (Claude — listed, minimal direct usage) |
| `@ai-sdk/xai` | ^3.0.0 | Provider adapter (Grok — listed, minimal direct usage) |
| `@ai-sdk/replicate` | ^2.0.1 | Provider adapter (Replicate — listed, minimal direct usage) |
| `@mastra/ai-sdk` | ^1.0.0-beta.12 | Mastra's bridge — `toAISdkFormat()` converts Mastra agent streams to AI SDK chunk types |

## Where AI SDK Sits in the Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                         │
│  React UI ← useChat() hook ← @ai-sdk/react                     │
│  Custom IpcChatTransport bridges Electron IPC boundary          │
└─────────────────────────┬───────────────────────────────────────┘
                          │ IPC (SSE-formatted JSON chunks)
┌─────────────────────────▼───────────────────────────────────────┐
│                     ORCHESTRATION LAYER                           │
│  Mastra.ai (agents, memory, workflows, tools)                    │
│  └── @mastra/ai-sdk: toAISdkFormat() converts agent streams     │
└─────────────────────────┬───────────────────────────────────────┘
                          │ calls
┌─────────────────────────▼───────────────────────────────────────┐
│                   AI SDK CORE (package: "ai")                    │
│                                                                   │
│  Text:       generateText() / streamText()                       │
│  Structured: generateObject() / streamObject()  (available,      │
│              not used by AIME — they use Zod tool schemas instead)│
│  Embeddings: embed() / embedMany()                               │
│  Tools:      tool() + Zod schemas                                │
│  Streaming:  createUIMessageStream() + typed chunk protocol      │
│  Types:      UIMessage, ModelMessage, LanguageModelUsage, etc.   │
│                                                                   │
│  Interfaces: LanguageModelV2, EmbeddingModelV2, ImageModelV2     │
│              TranscriptionModelV2, SpeechModelV2                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │ implements
┌─────────────────────────▼───────────────────────────────────────┐
│                      PROVIDER ADAPTERS                            │
│                                                                   │
│  @ai-sdk/openai ──────── OpenAI API (GPT, embeddings, TTS)      │
│  @ai-sdk/google ──────── Gemini API (+ thinkingConfig)           │
│  @ai-sdk/deepseek ────── DeepSeek API (+ extended thinking)      │
│  @ai-sdk/openai-compatible ── Ollama, LMStudio, ModelScope,      │
│                               ZhipuAI (any OpenAI-compat endpoint)│
│  @ai-sdk/anthropic ───── Claude API (minimal direct use)         │
│  @ai-sdk/xai ────────── Grok API (minimal direct use)            │
└──────────────────────────────────────────────────────────────────┘
```

## How AIME Chat Actually Uses AI SDK

### 1. Streaming: Main Process → Renderer via IPC

This is the most critical integration. AI SDK defines the streaming chunk protocol; AIME bridges it across Electron's process boundary.

**Main process** (`src/main/mastra/index.ts`):
```
agent.stream(inputMessage, options)        ← Mastra agent call
       │
       ▼
toAISdkFormat(stream, { from: 'agent' })   ← @mastra/ai-sdk converts to AI SDK chunks
       │
       ▼
for each chunk:
  appManager.sendEvent(                    ← IPC emit
    `chat:event:${chatId}`,
    { type: ChatChunk, data: JSON.stringify(chunk) }
  )
```

**Renderer** (`src/renderer/pages/chat/ipc-chat-transport.ts`):
```
class IpcChatTransport implements ChatTransport<UIMessage>
       │
       ├── sendMessages(): sends via IPC, returns ReadableStream
       │     TextEncoder encodes chunks as SSE: "data: ${JSON.stringify(chunk)}\n\n"
       │
       └── processResponseStream(): validates with Zod schema (uiMessageChunkSchema)
```

**`useChat` hook** (`src/renderer/hooks/use-chat.tsx`):
```
useChat({
  id: threadId,
  transport: transportRef.current,    ← IpcChatTransport instance
  onFinish: (event) => { ... },
  onData: (dataPart) => { ... },
  onError: (err) => { ... },
})
```

### Chunk Types Handled (Zod-validated)

| Category | Chunk Types | Purpose |
|----------|-------------|---------|
| **Text** | `text-start`, `text-delta`, `text-end` | Streaming text generation |
| **Reasoning** | `reasoning-start`, `reasoning-delta`, `reasoning-end` | Extended thinking (DeepSeek, o1). **Note:** disabled in main chat stream (`sendReasoning: false` in `mastra/index.ts:1235`). |
| **Tool lifecycle** | `tool-input-start`, `tool-input-delta`, `tool-input-available`, `tool-input-error` | Tool call construction |
| **Tool approval** | `tool-approval-requested`, `tool-call-approval`, `tool-output-available`, `tool-output-denied`, `tool-output-error` | Human-in-the-loop tool execution |
| **Sources** | `source-url`, `source-document` | RAG source references |
| **Error/File** | `error`, `file` | Error propagation, file attachments |
| **Meta** | `start-step`, `finish-step`, `start`, `finish`, `abort`, `message-metadata` | Stream lifecycle |
| **Custom data** | `data-usage`, `data-compress-start`, `data-compress-end` | Token tracking, context compression |

### 2. Embeddings: `embedMany()` for Knowledge Base

**File:** `src/main/knowledge-base/index.ts`
```
embedMany({
  model: provider.textEmbeddingModel(modelId),   ← AI SDK EmbeddingModelV2
  values: chunks                                   ← string[]
})
       │
       ▼
float32[][] → INSERT INTO [kb_{id}_{dim}] (embedding F32_BLOB)
```

`src/main/providers/local-provider.ts` implements the `EmbeddingModelV2` interface directly (using `@huggingface/transformers` for local embedding), but does not call `embedMany()` itself — it provides the model that `embedMany()` consumes.

### 3. Provider Resolution: Unified Model Access

**File:** `src/main/providers/index.ts`
```
// ProvidersManager resolves "providerId/modelId" → LanguageModelV2
getLanguageModel("ollama/deepseek-r1:8b")
       │
       ├── Split: provider = "ollama", model = "deepseek-r1:8b"
       ├── getProvider("ollama") → OllamaProvider instance
       └── provider.languageModel("deepseek-r1:8b") → LanguageModelV2
```

> **Note:** `createGateway`/`gateway` are imported from `ai` but not called at runtime. The provider resolution is a custom `ProvidersManager` implementation, not AI SDK's gateway API.

### 4. OpenAI-Compatible Adapter: The Universal Provider

**File:** `src/main/providers/ollama-provider.ts` (same pattern in 6 providers)
```
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

languageModel(modelId: string): LanguageModelV2 {
  return createOpenAICompatible({
    baseURL: this.provider.apiBase || 'http://localhost:11434',
    apiKey: this.provider.apiKey,
    name: 'ollama',
  }).languageModel(modelId);
}
```

Actively called at runtime by: Ollama, LMStudio, ZhipuAI, Google, and a fallback path in `ProvidersManager`. DeepSeek has dead code (early return before the call), ModelScope's call is commented out. This is how they connect to local models without custom HTTP clients.

### 5. Tool Definitions: Zod Schemas → AI SDK `tool()`

**File:** `src/main/tools/base-tool.ts`
```
abstract class BaseTool<T> implements Tool {
  abstract id: string;
  abstract inputSchema: ZodSchema;          ← AI SDK tool contract
  outputSchema?: ZodSchema;
  requireApproval?: boolean;                ← triggers suspended → approve flow
  execute?: ToolAction['execute'];          ← Mastra format
  toModelOutput?: (output) => LanguageModelV2ToolResultPart['output'];  ← AI SDK format
  format?: 'mastra' | 'ai-sdk';
}
```

Tools support both Mastra's `createTool()` and AI SDK's `tool()` format. The `toModelOutput()` method bridges the gap — converts Mastra tool output to AI SDK's `LanguageModelV2ToolResultPart`.

### 6. `generateText()` — Imported but Not Called

`generateText` is imported in ~24 tool files (`web-search.ts`, `tool.ts`, `bash.ts`, etc.) but has zero call sites. These are dead imports — likely from a shared tool template or IDE auto-import. The tools execute through Mastra's agent loop, not direct `generateText()` calls.

## What AI SDK Gives Us (That We'd Otherwise Build)

| Capability | AI SDK Primitive | Without AI SDK |
|------------|-----------------|----------------|
| **Provider switching** | Change `model` param, same API | Rewrite HTTP client per provider |
| **Streaming protocol** | Typed chunks (text-delta, tool-call, reasoning, etc.) + `createUIMessageStream()` | Custom SSE/chunking protocol per provider |
| **React integration** | `useChat()` hook with streaming, abort, retry | Custom WebSocket/SSE + state management |
| **Embeddings** | `embedMany({ model, values })` | Direct API calls + batching logic per provider |
| **Tool calling** | `tool({ inputSchema: zodSchema, execute })` — automatic function calling protocol | Manual tool-use JSON parsing per provider |
| **Provider adapters** | `@ai-sdk/openai-compatible` wraps any OpenAI-compat endpoint | Custom HTTP client per endpoint |
| **Message types** | `UIMessage`, `ModelMessage`, conversion utilities | Custom message format + serialization |
| **Abort/cancel** | Built-in `AbortController` threading | Manual signal propagation |

## Implications for Cowork.ai

1. **AI SDK is our interface contract.** Every LLM interaction — complexity router picking a model, Mastra agents streaming responses, React frontend rendering chunks — flows through AI SDK primitives. Understanding AI SDK means understanding what Mastra does under the hood, which is critical for debugging.

2. **Provider flexibility is free.** Our OpenRouter gateway is the primary path, but `@ai-sdk/openai-compatible` means direct Ollama access requires zero custom code. Just swap the adapter.

3. **The IPC streaming pattern is the hardest part.** AIME's `IpcChatTransport` + Zod-validated chunk protocol is the key bridge between AI SDK's React hooks and Electron's process model. This is the pattern we need most.

4. **Version: we use AI SDK v6.** AIME Chat uses v5 (`ai` ^5.0.93). AI SDK v6 shipped Dec 22, 2025 (currently at 6.0.86). **Decision: start on v6.** Mastra 1.0+ supports both v5 and v6 natively via npm aliasing (`@ai-sdk/provider-v5` + `@ai-sdk/provider-v6` bundled internally). The breaking changes are mechanical renames — `CoreMessage` → `ModelMessage`, `textEmbeddingModel()` → `embeddingModel()`, `@ai-sdk/react` v2 → v3 — with an automated codemod (`npx @ai-sdk/codemod v6`). The critical patterns (`useChat()` hook, custom `ChatTransport`, streaming chunk protocol, `tool()` + Zod) are stable across both versions. Mastra hides the version on the backend (agent orchestration, tools, embeddings), but `@ai-sdk/react` is a direct frontend dependency — our `IpcChatTransport` implements AI SDK's `ChatTransport<UIMessage>` interface directly. See [decision log](../../decisions/decision-log.md#2026-02-16--ai-sdk-version-v6-not-v5).

5. **`generateObject()` / `streamObject()` are unused.** AIME does structured output via Zod tool schemas instead. We should evaluate whether `generateObject()` is useful for our complexity router's structured analysis of user intent. Note: v6 merges these into `generateText()` / `streamText()` with an `output` setting.

## What We Should Do

| Action | Detail |
|--------|--------|
| **Decided: AI SDK v6** | AIME uses v5, we start on v6. Mastra 1.0+ supports both. Breaking changes are mechanical renames with automated codemod. See [decision log](../../decisions/decision-log.md#2026-02-16--ai-sdk-version-v6-not-v5). |
| **Copy pattern** | `IpcChatTransport` — custom `ChatTransport<UIMessage>` that bridges `useChat()` across Electron IPC. This is the most valuable AI SDK pattern in the codebase. |
| **Copy pattern** | `@ai-sdk/openai-compatible` for Ollama. One-liner to connect local brain: `createOpenAICompatible({ baseURL: 'http://localhost:11434' })`. |
| **Copy pattern** | Zod-validated streaming chunk protocol (`uiMessageChunkSchema`). Type-safe contract between main and renderer. |
| **Study** | `toAISdkFormat()` from `@mastra/ai-sdk` — how Mastra agent streams convert to AI SDK chunks. This is the glue between orchestration and UI. |
| **Study** | AI SDK's `gateway` API for provider abstraction. Imported but unused by AIME — evaluate if useful for routing between OpenRouter and direct providers. |
| **Evaluate** | `generateObject()` with Zod schemas for our complexity router. Could provide structured intent analysis without manual JSON parsing. |
| **Skip** | Their provider management UI (API key entry, model selection). We abstract behind tiers + complexity router. |
