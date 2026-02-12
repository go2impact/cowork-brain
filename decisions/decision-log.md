# Cowork.ai — Decision Log

Every significant architecture or strategy change gets an entry here. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the format and rules.

---

## 2026-02-12 — Repository initialized

**Changed:** Created this repo with v4 of both architecture and strategy docs.

**Why:** Architecture and strategy decisions were living in chat conversations and loose files. As we move to implementation, Rustan and team need a single source of truth that tracks not just what we decided but why. Changes to model selection, routing logic, or cost structure ripple through the entire business — we need the reasoning preserved so we can audit decisions when reality diverges from projections.

**Approved by:** Scott

---

## 2026-02-12 — Free tier model: Gemini 2.0 Flash → Gemini 2.5 Flash

**Changed:** Free tier cloud model upgraded from Gemini 2.0 Flash to Gemini 2.5 Flash.

**From → To:** $0.10/user/month → $0.46/user/month. At 100K users: $10K/mo → $46K/mo.

**Why:**
1. Gemini 2.0 Flash is deprecated March 31, 2026. Not an option regardless.
2. The free week must prove enough value that losing it feels like a real loss. Gemini 2.5 Flash has thinking capabilities and 1M context — it feels genuinely useful. A mediocre free experience converts at 2-3% (industry standard freemium). We need 10%+ to make the economics work.
3. At 10% conversion with 100K free users: 10K paid × $10/mo = $100K GMV. The $46K subsidy is justified as customer acquisition.
4. If costs get too high, we waitlist new free signups rather than degrade the model. 100 users who say "holy shit" beats 10,000 who say "meh."

**Cost impact:** ~4.6x increase per free user. Mitigated by local-first users costing ~$0.15-0.25/month (DeepSeek handles most work), waitlist system capping total spend, and model price trends (-50% every 6-12 months).

**Alternatives considered:**
- Gemini 2.5 Flash Lite ($0.10/user/month): Same price as 2.0 Flash, decent quality bump, but not a "wow." Rejected — marginal improvement doesn't justify rebuilding on it.
- Blended routing (Lite for simple, Flash for complex on free tier): Saves ~40% but adds routing complexity for marginal gain. Rejected — simplicity wins at this stage.

**Approved by:** Scott

---

## 2026-02-12 — Local LLM: consolidated to DeepSeek-R1 8B only

**Changed:** Removed Qwen3-4B and Qwen3-8B from local model options. DeepSeek-R1-Distill-Qwen-8B is the single local brain.

**From → To:** Three local LLM options (Qwen3-4B, Qwen3-8B, DeepSeek-R1-8B "alt") → One option (DeepSeek-R1-8B primary).

**Why:**
1. v0.1 is Mac-only. The only hardware profile that runs a local LLM is 16GB+ Apple Silicon. Qwen3-4B was for 8GB machines that can't fit the 8B — but 8GB Macs get cloud-only anyway.
2. DeepSeek-R1-Distill-Qwen-8B is built on Qwen architecture (good Tagalog/multilingual) with DeepSeek's reasoning training. You get both capabilities in one model.
3. For CX agents, reasoning capability is the difference between useful and toy. "Summarize this ticket with contradictory customer replies and suggest a response" requires thinking, not pattern matching.
4. One model = simpler onboarding, simpler Ollama management, simpler debugging. Complexity adds when Windows arrives in v0.2.

**Cost impact:** None (local inference is free). Slightly higher RAM usage than Qwen3-4B but irrelevant since 8GB machines don't run local LLM.

**Alternatives considered:**
- Dual local model (fast small + big reasoning): Gemini review suggested this. Rejected — doubles RAM pressure, doubles complexity, and the cloud tier already handles "think harder."
- Qwen3-8B as primary (better multilingual, no reasoning training): DeepSeek-R1-Distill-Qwen is literally Qwen + reasoning. Strictly better.
- MLX runtime instead of Ollama (10-15% faster on Mac): Rejected — Ollama works on Mac AND Windows. Saves a full inference layer rewrite when v0.2 ships.

**Approved by:** Scott

---

## 2026-02-12 — User chooses local vs cloud at onboarding

**Changed:** Added explicit onboarding choice for 16GB+ Mac users: "Run on my machine" vs "Run in the cloud."

**From → To:** System auto-detected hardware and defaulted to local → User makes an informed choice with clear tradeoffs explained.

**Why:**
1. Downloading a 5GB model and giving up 7GB of RAM is a meaningful tradeoff. Users should understand it.
2. "Your laptop may run warm" is an honest disclosure, not a fear tactic. Filipino remote workers on company machines may prefer cloud to avoid hardware complaints.
3. Users who choose local buy into the tradeoff and blame the app less when their laptop gets warm.
4. The choice creates a natural segmentation: local-first users cost us ~$0.15-0.25/month in cloud tokens, cloud-first users cost ~$0.46/month. Knowing the split helps forecast costs.

**Cost impact:** Reduces blended free tier cost. If 40% of users choose local, blended cost drops from $0.46 to ~$0.33/user/month.

**Approved by:** Scott

---

## 2026-02-12 — Thermal auto-detection and failover

**Changed:** Added automatic thermal state monitoring with auto-failover from local to cloud inference.

**From → To:** Manual Low-Power Mode toggle only → Auto-detection via macOS `ProcessInfo.thermalState` with automatic failover at "serious" state.

**Why:**
1. Target users (Filipino remote workers, 8-16GB machines) won't know what thermal throttling means. They'll just know their laptop is hot and slow and blame the app.
2. Auto-failover to cloud when thermals hit "serious" prevents the worst user experience (stuttering local inference on a throttled machine) without requiring the user to understand the problem.
3. Manual restore on recovery (not auto-restore) because cold-starting a 5GB model is expensive and the user may have adapted to cloud mode during the thermal event.

**Cost impact:** Minimal — thermal events are temporary. May add a few cents/month in cloud tokens during hot periods.

**Source:** Gemini technical review (Feb 12, 2026)

**Approved by:** Scott

---

## 2026-02-12 — Whisper warm-up: keep-warm daemon (v0.1), smart detection (v0.2)

**Changed:** Added Whisper pre-loading strategy to eliminate 2-second cold-start gap.

**From → To:** Load Whisper on demand (2s delay at start of transcription) → Keep Whisper warm in memory at all times on 16GB+ machines.

**Why:**
1. First impressions. If transcription starts with 2 seconds of missed audio, the product feels broken. Instant transcription feels magical.
2. v0.1 simple approach: just keep Whisper warm. Costs 1.5GB RAM on 16GB+ machines but eliminates cold-start entirely. On these machines, 1.5GB is affordable.
3. v0.2 smart approach (deferred): detect calendar events and meeting app launches, pre-load 2 min before. More RAM-efficient but requires calendar API + process monitoring. Not worth the engineering lift for v0.1.

**Cost impact:** ~1.5GB RAM permanently allocated on 16GB+ machines. Acceptable tradeoff — these machines have headroom.

**Source:** Gemini technical review (Feb 12, 2026)

**Approved by:** Scott

---

## 2026-02-12 — Vector store: SQLite-vec (not LanceDB)

**Changed:** Locked local vector store to SQLite-vec instead of "LanceDB or SQLite-vec."

**From → To:** Two options under consideration → SQLite-vec only.

**Why:**
1. Already using SQLite for activity store. Embeddings live in the same .db file. One database, one file, zero-ops.
2. LanceDB is a separate dependency with its own process. On 8GB Windows machines (v0.2), every extra dependency is a failure point.
3. At our scale (thousands to tens of thousands of embeddings per user), SQLite-vec performance is more than sufficient.
4. Simpler backup/restore: one file.

**Cost impact:** None — both are free/local. SQLite-vec is slightly less engineering effort.

**Source:** Gemini technical review (Feb 12, 2026)

**Approved by:** Scott

---

## 2026-02-12 — Cost guardrail: waitlist system for free tier

**Changed:** Added waitlist mechanism when free tier costs exceed configurable threshold.

**From → To:** No cost ceiling on free tier → Hard ceiling (default $50K/month) triggers waitlist for new signups.

**Why:**
1. The free experience must prove real, measurable value. Degrading model quality or reducing quotas to save money defeats the purpose.
2. At $46K/month for 100K free users, we need conversion revenue to justify the subsidy. If conversion is below target, we can't afford unlimited free signups.
3. Waitlist preserves quality for existing users while controlling volume. Priority activation for referrals from paid users creates viral incentive.
4. Better to have fewer users getting the real experience than more users getting a watered-down one.

**Cost impact:** Caps free tier at configurable threshold. Prevents runaway subsidy costs during growth.

**Approved by:** Scott
