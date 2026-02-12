# Cowork.ai — LLM Architecture Spec

**v4 — February 12, 2026**
**Audience:** Engineering (Rustan + team)
**Purpose:** Build spec. v0.1 is Mac-only. Attack in whatever order makes sense.

---

## Design Principle

Two brains: a local one (free, instant, uses your machine) and a cloud one (smarter, metered, costs tokens). User picks at onboarding, can switch anytime. The system makes smart defaults but never hides the tradeoff.

---

## Key Terms

| Term | Meaning |
|------|---------|
| ANE | Apple Neural Engine — dedicated ML chip on Apple Silicon, separate from GPU/CPU |
| GGUF | File format for quantized LLM weights (used by llama.cpp / Ollama) |
| Core ML | Apple's framework for running ML models on-device (can target ANE, GPU, or CPU) |
| BYOK | Bring Your Own Key — user provides their own API key instead of using Cowork credits |
| PKCE | Proof Key for Code Exchange — secure OAuth flow for public clients (no client secret) |
| OpenRouter | API gateway that proxies to multiple LLM providers (Anthropic, Google, etc.) |
| RAG | Retrieval-Augmented Generation — searching local docs/data to add context to LLM prompts |

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        User Request                          │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
                  ┌──────────────────┐
                  │ Complexity Router │ (<10ms, rule-based)
                  └────────┬─────────┘
                           │
              ┌────────────┼─────────────┐
              ▼            │             ▼
       ┌─────────────┐    │     ┌───────────────┐
       │  Local Brain │    │     │  Cloud Brain   │
       │ DeepSeek-R1  │    │     │  OpenRouter    │
       │ (Ollama/GPU) │    │     │  ┌───────────┐ │
       └──────────────┘    │     │  │Free: Gem2.5│ │
                           │     │  │Boost:Gem3  │ │
  ┌──────────────┐         │     │  │Pro: Sonnet │ │
  │   The Ears   │         │     │  │Max: Opus   │ │
  │ Whisper (ANE)│         │     │  └───────────┘ │
  │ Always local │         │     └───────────────┘
  └──────┬───────┘         │
         │            ┌────┴─────┐
         │            │  Budget  │
         │            │  Check   │
         │            └────┬─────┘
         ▼                 ▼
┌─────────────────────────────────────────────────────────────┐
│              Activity Store (SQLite + SQLite-vec)             │
│        Transcripts · Embeddings · Action Rules · Logs        │
└─────────────────────────────────────────────────────────────┘
```

---

## v0.1 Scope: Mac Only

Windows comes in v0.2. Everything below targets Apple Silicon.

### Hardware Profiles

| Machine | What's Possible | Default Mode |
|---------|----------------|--------------|
| 8GB Mac | Whisper on ANE only. Can't fit local LLM. | Cloud-only (no choice to make) |
| 16GB+ Mac | Whisper on ANE + DeepSeek-R1 on GPU | User chooses at onboarding |

### The Onboarding Choice (16GB+ only)

First run, after hardware detection:

```
"How do you want your AI to work?"

Option A: "Run on my machine"
  → Free. No token costs for most tasks.
  → Uses ~7GB of your RAM. Your laptop may run warm.
  → Downloads a ~5GB AI model (one-time).
  → Complex tasks still go to cloud when needed.

Option B: "Run in the cloud"
  → Faster, no impact on your machine.
  → Uses your free token allocation (~1 week/month).
  → After that: pay as you go or wait for next month.

[You can switch anytime in settings]
```

8GB machines skip this screen — they get cloud-only with Whisper local, no choice needed.

**Implementation:** This sets a single flag: `inference_mode: local | cloud`. The complexity router respects this flag but can override for tasks that require cloud (e.g., complex multi-tool chains, large context windows).

---

## The Local Brain

### DeepSeek-R1 8B (4-bit quantized)

**One model. One job. Be the free, instant brain on the user's machine.**

| Spec | Value |
|------|-------|
| Model | DeepSeek-R1-Distill-Qwen-8B (4-bit GGUF) |
| Runtime | Ollama |
| Model size on disk | ~5 GB |
| RAM required | ~5.5 GB |
| Why this model | Reasoning capability (thinks before answering) + Qwen base (strong multilingual/Tagalog) |
| What it handles | Ticket summaries, draft responses, classify emails, explain SOPs, simple Q&A |
| What it can't do | Large context (>8K tokens), vision/images, complex multi-step planning |

**Why DeepSeek-R1 specifically:**
- It's a reasoning model. For CX agents, this matters — "summarize this ticket and suggest a response" requires thinking, not just pattern matching. The difference between a useful local model and a toy is whether it can reason through a ticket with 3 customer replies and contradictory information.
- Built on Qwen architecture → Tagalog + English coverage without a separate multilingual model.
- At 4-bit quantization, fits comfortably on a 16GB Mac alongside Whisper (~5.5GB model + ~1.5GB Whisper + ~6-8GB OS/apps = ~15GB total).

**Why Ollama (not MLX):**
- MLX is ~10-15% faster on Mac
- Ollama works on Mac AND Windows
- When v0.2 ships Windows, we don't rewrite the inference layer
- Speed difference is imperceptible for paragraph-length generation

### The Dual-Brain Hardware Split

```
Apple Neural Engine (ANE): Whisper transcription — always running
GPU (Metal):               DeepSeek-R1 — runs on demand
CPU/RAM:                   macOS + Chrome + Zendesk + everything else
```

Whisper on ANE and DeepSeek on GPU are separate silicon. They don't compete. The user can be on a call (Whisper transcribing on ANE) while the agent drafts a response (DeepSeek generating on GPU) without either stuttering. This is why Apple Silicon matters and why v0.1 is Mac-only.

### When Local Isn't Enough

Local DeepSeek handles ~70% of daily CX work. For the rest:

```
Auto-routes to cloud when:
  • Input exceeds ~8K tokens (long ticket threads, big documents)
  • Task requires vision (screenshots, images)
  • Multi-tool chains (search + draft + file + send)
  • User taps "try harder" on a previous response

What user sees:
  • "This one needed cloud AI (used X tokens)"
  • They learn the pattern naturally — simple = free, complex = costs
```

---

## The Cloud Brain

### Gateway: OpenRouter API

All cloud inference goes through OpenRouter. Single integration, multiple model providers, automatic failover.

### Model Routing Table

| Tier | Model | OpenRouter ID | Fallback |
|------|-------|---------------|----------|
| Free | Gemini 2.5 Flash | `google/gemini-2.5-flash` | None — quota exhausted = wait |
| Boost | Gemini 3 Flash | `google/gemini-3-flash-preview` | Gemini 2.5 Flash |
| Pro | Claude Sonnet 4.5 | `anthropic/claude-sonnet-4-5` | Gemini 3 Flash |
| Max | Claude Opus 4.6 | `anthropic/claude-opus-4-6` | Claude Sonnet 4.5 |

> ⚠️ Gemini 2.0 Flash deprecated March 31, 2026. Do not build on it.
> Model IDs change as preview → stable. Pull from OpenRouter `/api/v1/models`. Routing table is a config file, not hardcoded.

### Pricing (Feb 2026 — pull from OpenRouter at runtime, never hardcode)

| Model | Input/1M tokens | Output/1M tokens |
|-------|----------------|------------------|
| Gemini 2.5 Flash | $0.30 | $2.50 |
| Gemini 3 Flash | $0.50 | $3.00 |
| Claude Sonnet 4.5 | $3.00 | $15.00 |
| Claude Opus 4.6 | $5.00 | $25.00 |

```
cost = (input_tokens / 1_000_000 × input_price) + (output_tokens / 1_000_000 × output_price)
```

### Auth

**Cowork Credits (default):** Cowork backend holds master OpenRouter key. User never sees OpenRouter. We track spend per user against their credit balance.

**BYOK (power users):** User provides their own OpenRouter API key via OAuth PKCE. Cowork earns nothing on inference — earns on platform value. Not the path for mass market.

### API Pattern

```
POST to OpenRouter /api/v1/chat/completions
Headers:
  Authorization: Bearer {api_key}
  HTTP-Referer: https://cowork.ai
  X-Title: Cowork.ai
Body:
  model: {selected_model_id}
  messages: [...]
  max_tokens: {cap}
  temperature: 0.7
  tools: [...]              // MCP tool definitions when needed
  route: "fallback"         // auto-failover if primary provider is down
```

### Billing Flow (Cowork Credits)

```
1. User buys credits through Cowork
   - Cowork charges: amount × 1.055 (5.5% service fee)
   - Cowork sends that to OpenRouter (covers OR's 5.5% top-up fee)
   - Result: full amount in usable credits

2. Each cloud request:
   - Log: user_id, model, input_tokens, output_tokens, cost
   - Deduct from user's credit balance

3. Low balance:
   - Warn at 20% remaining
   - Offer manual "Add credits" prompt (auto-top-up is v0.2+)
   - At $0: downgrade to free tier until recharged
```

---

## The Ears: Whisper Integration

### Architecture

```
Audio input (mic / system audio / meeting)
    │
    ▼
Whisper.cpp (Core ML → ANE)
    │
    ▼
Real-time transcript stream
    │
    ▼
Activity store (SQLite) + Agent context window
```

### Model

| Model | Size | RAM | Use Case |
|-------|------|-----|----------|
| Whisper Turbo (Core ML) | 809 MB | ~1.5 GB | Default for 16GB+ machines |
| Whisper Small (Core ML) | 244 MB | ~0.5 GB | Fallback for 8GB machines |

Whisper runs on ANE via Core ML backend. Dedicated hardware — doesn't compete with GPU (DeepSeek) or CPU (everything else).

### Keep Warm Strategy (v0.1)

Whisper model load on ANE takes ~2 seconds. Two approaches:

**v0.1 (ship this):** On 16GB+ machines, keep Whisper warm in memory at all times. Costs 1.5GB RAM but eliminates cold-start entirely. On 8GB machines, load on demand (user accepts the 2s delay).

**v0.2 (upgrade later):** Smart meeting detection — pre-load when calendar event starts in <2 min or when Zoom/Meet/Teams process launches. Unload after 5 min of no audio. Saves RAM but needs calendar API + process monitoring.

### Audio Never Leaves the Device

Non-negotiable. Whisper is always local. If hardware can't run Whisper at all (edge case on Apple Silicon):

1. Explicit consent dialog: "Audio will be sent to a cloud service. Your recordings will leave this device."
2. User must actively enable — never auto-enabled
3. Flag cloud-transcribed content in activity store

---

## Complexity Router

Rule-based. No ML classifier. <10ms decision time.

### Flow

```
Request arrives
    │
    ▼
Step 0: DETECT (instant, no model call)
    • Count/estimate input tokens
    • Check: image/file attached?
    • Check: requires tool use?
    • Check: user's inference_mode setting (local / cloud)
    │
    ▼
Step 1: ROUTE
    │
    ├── User set "prefer local" + task is simple?
    │   → DeepSeek-R1 local (free, instant)
    │
    ├── User set "prefer local" + task is complex?
    │   → Cloud (auto-upgrade, notify: "This one needed cloud")
    │
    ├── User set "prefer cloud"?
    │   → Cloud always (uses token allocation)
    │
    └── User tapped "try harder" on previous response?
        → Next tier up (show cost estimate first)

Step 2: BUDGET CHECK
    • Free user: remaining monthly quota?
    • Paid user: within spend caps?
    • Yes → proceed
    • No → friendly message + upgrade prompt
    │
    ▼
Step 3: RESULT EVALUATION (post-response)
    • "Retry with smarter model? (~$X)" button on every response
    • Free users: button shows upgrade CTA instead of cost
```

### What's "Simple" vs "Complex"

```
SIMPLE (→ local if available):
  • <500 input tokens
  • No attachments
  • No tool calls needed
  • Single-turn Q&A, summaries, short drafts

COMPLEX (→ cloud):
  • >8K input tokens (exceeds local context window)
  • Any attachment (image, file, screenshot)
  • Multi-tool chains
  • Keywords: ["review","analyze","compare","plan","explain in detail"]
  • User explicitly requests cloud
```

Keyword list is a config array. Tune with real usage data.

> **v0.1 Rule: When in doubt, route UP to cloud.**
> A bad local answer costs more than a cloud token. Optimize down in v0.2 when you have real usage data to tune thresholds.

---

## Token Budget System

### Free Tier

**Monthly allowance: ~625K tokens (500K input + 125K output)**

On Gemini 2.5 Flash:
- ~250 meaningful interactions
- Roughly 1 solid work week (50 interactions/day × 5 days)
- After quota: local features continue, cloud pauses until next month
- Reset: midnight UTC, 1st of each month

### Cost Guardrail: Waitlist, Not Degradation

```
Never degrade the free experience. Control volume instead.

• Track: total_monthly_free_tier_cost (real-time)
• Threshold: configurable monthly cap
• Approaching threshold: new free signups → waitlist
• Existing users: keep full allocation
• NEVER downgrade model quality for existing free users
```

### Paid Tier Budgets

Pay-as-you-go against Cowork Credits with configurable caps:

| Cap | Default | Enforcement |
|-----|---------|-------------|
| Per interaction | $0.50 | Hard stop, notify |
| Per background task chain | $1.00 | Hard pause, tap to continue |
| Daily | $10.00 | Warn at 80%, stop at limit |
| Weekly | $50.00 | Warn at 80%, stop at limit |
| Monthly | $200.00 | Soft cap (user can override) |

Real-time spend ticker on every interaction: model used, tokens, cost, daily total.

---

## Embeddings & Local RAG

- **Model:** Qwen3-Embedding-0.6B via Ollama (~500MB RAM, strong Tagalog+EN)
- **Vector store:** SQLite-vec (same .db as activity store — single file, zero-ops)
- **Fallback:** Keyword search against SQLite if embeddings can't run locally

---

## Thermal Awareness

Users won't know what "thermal throttling" means. Handle it silently.

### Auto-Detection

macOS: `ProcessInfo.thermalState` (poll every 30 seconds)

```
nominal / fair  → normal operation
serious         → kill local LLM (DeepSeek), route to cloud
                  keep Whisper on ANE (separate hardware)
                  notify: "Switched to cloud — machine was getting hot"
critical        → kill ALL local inference including Whisper
                  cloud-only until recovery
                  notify: "All AI moved to cloud"

On recovery (back to nominal/fair):
  → Do NOT auto-restore (cold restart is expensive)
  → notify: "Cooled down. Tap to restore local AI."
  → User taps → models reload
```

Manual Low-Power Mode toggle also available in settings.

---

## Agent Action Rules

### How It Works

```
FIRST ENCOUNTER:
  "Should I always file emails from [client] under [folder]?"
  User says yes/no with conditions.
  Rule stored: {trigger, action, conditions, approved}

SUBSEQUENT:
  Match + approved → execute, log it
  Match + denied → skip silently
  No match → ask user, create new rule
```

### Safety Rails

- All rules viewable/editable/deletable in settings
- All actions logged with undo window (default 5 min)
- Destructive actions: always confirm first time
- Money actions: always require explicit approval, no auto-rule

### Loop Breaker

| Condition | Action |
|-----------|--------|
| Same tool called 3× with same/similar input | Pause: "I seem to be stuck" |
| Task chain exceeds $1.00 in 10 min | Hard pause, tap to continue |
| Confidence drops 2+ consecutive steps | Pause, offer human review |
| Task chain > 5 minutes | Check-in: "Still working on X?" |
| Error rate > 50% in chain | Abort, show error summary |

Implement "same tool 3×" first. Most common loop pattern.

---

## Transparency Layer

Every cloud interaction shows: model name, token count, cost, daily running total.

Lightweight overlay/widget in UI. API endpoint for usage dashboard (Phase 2).

---

## Implementation Priority

### Phase 1: The Ears
1. Whisper.cpp with Core ML backend (ANE)
2. Keep-warm daemon (16GB+ machines)
3. Streaming transcription → SQLite activity store
4. Trigger engine: keyword watcher on live transcript

### Phase 2: The Brain
5. Ollama + DeepSeek-R1 8B model pull
6. OpenRouter integration (API, auth, routing)
7. Onboarding choice UI ("your machine" vs "the cloud")
8. Complexity router (local vs cloud logic)

### Phase 3: The Memory & Cost
9. Token tracking + cost calculation
10. Free tier quota (monthly budget, enforcement, reset)
11. Spend caps (per-interaction, daily, weekly)
12. Qwen3-Embedding-0.6B + SQLite-vec RAG

### Phase 4: Autonomy & Safety
13. Action rule learning
14. Loop breaker
15. Thermal auto-detection + failover
16. BYOK auth
17. Cowork Credits billing
18. Free tier waitlist system

---

## Not in v0.1

Explicitly out of scope — do not build these yet:

- Windows support (v0.2)
- Smart meeting detection for Whisper pre-load (v0.2 — needs calendar API + process monitoring)
- Usage dashboard API endpoint (Phase 2)
- Auto-top-up billing
- Referral-based waitlist priority

---

## Business & Unit Economics

> This section is for product/business context. Engineers can skip it — nothing here affects implementation.

**Cost per free user: ~$0.46/month** (based on 625K tokens on Gemini 2.5 Flash)

| Scale | Monthly cost | Notes |
|-------|-------------|-------|
| 10K free users | $4,600 | Seed stage |
| 50K free users | $23,000 | Growth phase |
| 100K free users | $46,000 | Waitlist kicks in before this hurts |

**"Prefer local" users on 16GB+ Macs** burn cloud tokens only on complex tasks and "try harder" requests. They might stretch free allocation to 2-3 weeks. Still hit the wall eventually — the wall is what converts.

**"Retry with smarter model"** button is a top-revenue UI element. Make it prominent. This is the natural upsell — user sees the value, taps for more.

**Waitlist priority:** Referrals from paid users activate faster. Controls cost while rewarding organic growth.

**Billing margin:** Cowork charges `amount × 1.055` (5.5% service fee) which covers OpenRouter's 5.5% top-up fee. Result: full amount in usable credits, Cowork earns on platform value, not inference margin.

---

## Changelog

**v4 (Feb 12, 2026):**
- Local LLM: single model — DeepSeek-R1-Distill-Qwen-8B via Ollama (was multiple Qwen models)
- Added onboarding choice: "run on your machine" vs "run in the cloud"
- Free tier cloud: Gemini 2.5 Flash (was 2.0 Flash — deprecated March 31, 2026)
- Thermal auto-detection with auto-failover to cloud
- Whisper keep-warm (v0.1 simple: always warm on 16GB+)
- Vector store: SQLite-vec (was LanceDB)
- Cost guardrail: waitlist system (never degrade quality)
- Phases restructured: Ears → Brain → Memory → Autonomy
- Scoped explicitly to Mac-only for v0.1

**v3 (Feb 12, 2026):** Initial engineering spec

---

*v4 — February 12, 2026*

**See also:** [Product Overview](../product/product-overview.md) · [LLM Strategy & Economics](../strategy/llm-strategy.md) · [Design System](../design/design-system.md)
