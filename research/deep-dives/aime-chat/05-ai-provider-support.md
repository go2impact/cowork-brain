# 5. Multiple AI Provider Support

[< Back to index](README.md)

**Their claim:** "Integrated with mainstream AI providers including OpenAI, DeepSeek, Google, Zhipu AI, Ollama, LMStudio, ModelScope, and more"

**Maps to Cowork.ai:** Our complexity router + OpenRouter gateway. Fundamentally different approach — they expose every provider, we abstract behind a router.

## How It Works

`ProvidersManager` is a factory that creates provider instances from a `BaseProvider` abstract class. Each provider implements the Vercel AI SDK interfaces (`LanguageModelV2`, `EmbeddingModelV2`, etc.). Model metadata lives in a 65K-line `models.json` file.

**Key files:**
- `src/main/providers/index.ts` — ProvidersManager
- `src/main/providers/base-provider.ts` — BaseProvider abstract class
- `src/main/providers/{provider}-provider.ts` — 14 provider implementations
- `assets/models.json` — Static model metadata (65,654 lines)

## Provider Architecture

```
┌────────────────────── PROVIDER SYSTEM ──────────────────────┐
│                                                               │
│  ProvidersManager                                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  getLanguageModel(modelId)                          │    │
│  │       │                                              │    │
│  │       ▼                                              │    │
│  │  Split: "providerId/modelId"                        │    │
│  │       │                                              │    │
│  │       ▼                                              │    │
│  │  getProvider(providerId) ──▶ switch(type):           │    │
│  │       │                                              │    │
│  │       ├─ 'openai'    → new OpenAIProvider()         │    │
│  │       ├─ 'deepseek'  → new DeepSeekProvider()       │    │
│  │       ├─ 'google'    → new GoogleProvider()         │    │
│  │       ├─ 'ollama'    → new OllamaProvider()         │    │
│  │       ├─ 'lmstudio'  → new LmstudioProvider()      │    │
│  │       ├─ 'zhipuai'   → new ZhipuAIProvider()       │    │
│  │       ├─ 'modelscope'→ new ModelScopeProvider()     │    │
│  │       ├─ 'jina_ai'   → new JinaAIProvider()        │    │
│  │       ├─ 'brave'     → new BraveSearchProvider()    │    │
│  │       ├─ 'tavily'    → new TavilyProvider()         │    │
│  │       ├─ 'serpapi'   → new SerpapiProvider()        │    │
│  │       ├─ 'local'     → new LocalProvider()          │    │
│  │       └─ *           → createOpenAICompatible()     │    │
│  │       │                                              │    │
│  │       ▼                                              │    │
│  │  provider.languageModel(modelId)                    │    │
│  │       │                                              │    │
│  │       ▼                                              │    │
│  │  Returns LanguageModelV2 (Vercel AI SDK interface)  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  BaseProvider                                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  abstract languageModel(id): LanguageModelV2        │    │
│  │  abstract textEmbeddingModel(id): EmbeddingModelV2  │    │
│  │  imageModel?(id): ImageModelV2                      │    │
│  │  transcriptionModel?(id): TranscriptionModelV2      │    │
│  │  speechModel?(id): SpeechModelV2                    │    │
│  │  rerankModel?(id): RerankModel                      │    │
│  │                                                      │    │
│  │  abstract getLanguageModelList()                    │    │
│  │  abstract getEmbeddingModelList()                   │    │
│  │  abstract getCredits()                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  Provider Adapters:                                          │
│  ┌──────────────────────┐  ┌──────────────────────────┐    │
│  │  NATIVE SDK           │  │  OPENAI-COMPATIBLE       │    │
│  │  @ai-sdk/openai      │  │  @ai-sdk/openai-compat   │    │
│  │  @ai-sdk/google      │  │  Used by: Ollama,        │    │
│  │  @ai-sdk/deepseek    │  │  LMStudio, Google,       │    │
│  │                       │  │  ZhipuAI, ModelScope,    │    │
│  │  NOTE: @ai-sdk/      │  │  MiniMax, OpenAI, and    │    │
│  │  anthropic + xai in  │  │  as universal fallback   │    │
│  │  package.json but    │  │  for unmatched providers  │    │
│  │  never imported      │  │                           │    │
│  └──────────────────────┘  └──────────────────────────┘    │
│                                                               │
│  Provider Tags (capability flags):                           │
│  EMBEDDING | RERANKER | WEB_SEARCH | WEB_READER |            │
│  IMAGE_GENERATION | OCR                                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

## Model Capability Metadata (models.json)

```json
{
  "openai": {
    "models": {
      "gpt-4o": {
        "reasoning": true,
        "tool_call": true,
        "structured_output": true,
        "modalities": { "input": ["text","image"], "output": ["text"] },
        "cost": { "input": 0.005, "output": 0.015 },
        "limit": { "context": 128000, "output": 4096 }
      }
    }
  }
}
```

## What We Should Do

| Action | Detail |
|--------|--------|
| **Study** | Their `BaseProvider` abstraction — clean interface for adding new providers. If we ever need to bypass OpenRouter for specific providers, this is the pattern. |
| **Study** | `models.json` static metadata (cost, context limits, capabilities). We should maintain a similar registry for our complexity router's model selection. |
| **Skip** | Exposing 15+ providers to users. We deliberately abstract this behind OpenRouter + the complexity router. User picks a tier, not a model. |
| **Skip** | Their UI for provider management (API key entry, model toggling). Our UX is simpler by design. |
| **Note** | Their `@ai-sdk/openai-compatible` adapter is useful — it wraps any OpenAI-API-compatible service (Ollama, LMStudio, DeepSeek). If we need direct Ollama access outside OpenRouter, this is how. |
