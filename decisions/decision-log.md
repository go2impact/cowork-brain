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

---

## 2026-02-12 — Design system: Material Design 3

**Changed:** Adopted M3 (Material Design 3) as the mandatory design system for all Cowork.ai UI surfaces.

**From → To:** No formal design system (ad hoc styling per prototype) → M3 dark theme token system, documented in `design/design-system.md`.

**Why:**
1. Cowork.ai is a platform where users install and build MCP apps. Those apps generate UI. AI generates UI. Without a rigid, well-documented token system, the product becomes visually incoherent within weeks.
2. M3 has the most complete publicly documented token system (colors, elevation, shape, motion all have named specs). When an LLM generates a component, it can follow M3 specs and produce something that looks native.
3. Google maintains M3 — we don't need to maintain a custom design system with a small team.
4. M3 tokens map cleanly to Tailwind, which is our entire frontend stack.
5. Dark theme is first-class in M3, not bolted on. Desktop sidecar = dark by default.

**Alternatives considered:**
- Custom design system: More flexible but requires dedicated design resource to maintain. Not viable at our team size.
- Shadcn/Radix: Good component libraries but no comprehensive token system for AI-generated UIs.
- Apple HIG: Strong but not well-documented for web, and would conflict with cross-platform goals (51 countries, many on Windows).

**Approved by:** Scott

---

## 2026-02-12 — Interaction model: Three-state sidecar

**Changed:** Locked the interaction model to a three-state pattern: Closed (menu bar only) → SideSheet (right panel) → SideSheet + Detail Canvas (full workspace).

**From → To:** Various prototype approaches (some with nav rail always visible, some with tab switching) → Standardized three-state model documented in `design/design-system.md`.

**Why:**
1. Workers need zero visual intrusion when they're not using us. State 1 (closed) means Cowork.ai doesn't exist on screen. This is critical for the trust model — we're not surveillance software.
2. State 2 (sidesheet) is the quick-glance dashboard. Check status, scan context, 5 seconds, back to work. The sidesheet is a scrollable card list, not a tabbed interface — tabs force navigation decisions, cards let you scan.
3. State 3 (detail canvas) opens to the LEFT of the sidesheet, keeping the sidesheet visible as persistent navigation. This means you can jump between chat, apps, and MCP browser without closing and reopening.
4. The fav prototype nailed this model. The best-looking prototype had better visual polish but used a nav rail with tabs instead of the three-state pattern. We take the interaction from fav, the visual style from best-looking.

**Approved by:** Scott

---

## 2026-02-12 — Added product overview, rewrote README

**Changed:** Created `product/product-overview.md` and rewrote `README.md` to lead with what Cowork.ai is before explaining the repo structure.

**From → To:** README explained the repo but not the product. No standalone product overview existed. → README opens with product explanation and links to full product overview. New `/product` directory added.

**Why:** Issue #3. The repo had strong technical architecture (v4), business strategy (v4), design system (v1.0), and GTM (v1.1) docs, but no single document explaining what Cowork.ai is, what it does, and who it's for. A new engineer, investor, or partner had to reverse-engineer the product from docs that assumed prior context. This isn't a cost/model/routing change, but it's structural — every other doc in this repo is more useful when the reader already understands the product.

**What the product overview covers:** Plain-language product explanation, two-brain architecture summary, desktop sidecar interaction model, MCP integration, feature set (chat, apps, MCP browser, automations), target users, Go2 relationship, roadmap, and links to all detailed docs.

**What we intentionally excluded (per issue discussion):** Detailed user personas, acceptance criteria, sprint roadmaps, competitive analysis matrices. Those are product management process artifacts — they belong in Linear/GitHub Issues, not in the decision layer. Cowork-brain captures reasoning, not execution tracking.

**Approved by:** Scott

---

## 2026-02-15 — README rewrite: ecosystem and rake framing

**Changed:** Rewrote README to frame Cowork.ai around the ecosystem Go2 already owns and the rake on token pass-through, not around building an AI agent.

**From → To:** README opened with "desktop AI agent for remote workers" and led with the technical product (local model, MCP). → README opens with the market opportunity (workers nobody else is building for), adds an explicit Business Model section (5.5% rake on cloud token pass-through), and positions the AI capabilities as the mechanism, not the differentiator.

**Why:**
1. The first version put Cowork.ai in the "AI agent" lane — same positioning as OpenAI, Anthropic, and every other company with 100x more resources building agent frameworks. That's a fight we lose.
2. The actual advantage is the ecosystem: 2M+ verified worker profiles, thousands of small business clients, hardware specs on file, zero CAC. Nobody else has this distribution in this market.
3. The business model is a rake on inference, not building or hosting models. That needed to be stated explicitly — the first README never mentioned how we make money.
4. "Why Now" section now ends with "these are enablers, not differentiators" — MCP, local models, and cost collapse are available to everyone. Owning the ecosystem where they get used is not.

**What stayed:** Trust Paradox section (unchanged — it's the strongest insight in the repo), The One Rule, repo guide structure, Current State table with clickable links.

**What was added:** The Opportunity section (market framing), The Business Model section (tier table + 5.5% rake), small business clients as a distribution asset.

**What was removed:** "What Does NOT Live Here" section (defensive, unnecessary — The One Rule already covers scope).

**Approved by:** Scott
