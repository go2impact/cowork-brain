# Cowork.ai — LLM Strategy & Economics

**v4 — February 12, 2026**
**Purpose:** Living knowledge base. Current assumptions on pricing, cost structure, conversion theory, worker ROI. Internal reference — not investor-facing.

---

## The Premise

AI inference gets cheaper every quarter. The cost of making a worker 2x productive approaches zero. Whoever owns the distribution — the desktop agent workers use daily — captures the value. The moat is behavioral data, workflow rules, and the trust relationship, not the model.

Strategy: give away enough intelligence to prove undeniable value, charge for intelligence that creates leverage.

---

## Two Brains, One Agent

The worker sees one agent. Under the hood, there are two inference paths:

**Local brain (DeepSeek-R1 8B):** Free, instant, runs on the user's Mac. Handles ~70% of daily work — ticket summaries, draft responses, email classification, SOP lookups. No token cost. No latency. No data leaves the device.

**Cloud brain (Gemini 2.5 Flash → Claude Opus 4.6):** Smarter, metered, costs tokens. Handles complex tasks — long documents, multi-step workflows, vision, "think harder" requests. Quality scales with what the user pays.

**The user chooses at onboarding:** "Run on my machine" (free, uses RAM, laptop may run warm) or "Run in the cloud" (faster, costs tokens). Can switch anytime. 8GB machines get cloud-only by default — no choice needed.

**Why this matters for economics:** "Prefer local" users on 16GB+ Macs burn almost zero cloud tokens on daily work. Their free allocation stretches to 2-3 weeks instead of 1. They still hit the wall on complex tasks — and the wall is what converts. But their cost-to-serve is dramatically lower than cloud-only users.

---

## Tier Structure

### The Value-First Model

The free tier must be good enough to change how someone works for one week per month. Not a demo. A real work week where the agent handles real tasks, then cloud stops, and the worker feels the absence. Local features (Whisper, DeepSeek if enabled) keep running — but the agent gets noticeably dumber on anything complex.

| Tier | Experience | Monthly Cost to User | Conversion Trigger |
|------|-----------|---------------------|-------------------|
| **Free** | Local brain always works. Cloud brain works ~1 week/month (Gemini 2.5 Flash). | $0 | "I had to do all that manually for three weeks." |
| **Boost** | Local + cloud all month. Smarter cloud model (Gemini 3 Flash, 1M context). | ~$5-15 | "It handles my routine work faster than I can." |
| **Pro** | Local + Claude Sonnet 4.5. Complex drafts, analysis, semi-autonomous. | ~$30-60 | "It's doing 60% of my job." |
| **Max** | Local + Claude Opus 4.6. Near-autonomous. User handles judgment, agent handles volume. | ~$100-300 | "I'm making 2x and working half the hours." |

### Why This Works

The conversion lever is the **time gap**, not the quality gap. The free cloud model (Gemini 2.5 Flash) is genuinely good — losing it hurts. Boost gives you the same quality all month plus a smarter model. Pro/Max unlock the complex automation that changes careers.

The agent doesn't do less over time to force upgrades. It does MORE — it learns the workflow, automates more tasks, naturally consumes more tokens. Value and cost grow together.

---

## Cost to Serve — Free Tier

### Two User Populations

**Cloud-only users (8GB machines):** Burn full allocation on Gemini 2.5 Flash. ~$0.46/user/month.

**Local-first users (16GB+ machines):** Most work goes through DeepSeek-R1 locally at $0. Cloud tokens used only for complex tasks and "try harder." Estimated ~$0.15-0.25/user/month cloud cost.

**Blended cost depends on hardware mix.** If 60% of users are on 8GB Windows machines (cloud-only when we hit v0.2) and 40% are on 16GB+ Macs (local-first), blended cost is roughly $0.33/user/month.

### Cost Table (worst case: all cloud-only)

| Scale | Monthly cost (all cloud) | Monthly cost (60/40 blend) |
|-------|------------------------|---------------------------|
| 10K free users | $4,600 | $3,300 |
| 50K free users | $23,000 | $16,500 |
| 100K free users | $46,000 | $33,000 |

### Cost Guardrail: Waitlist > Degradation

Never water down the free experience. If costs hit threshold ($50K/month configurable), new free signups go to waitlist. Existing users keep full allocation. Referrals from paid users get priority activation.

### Why Gemini 2.5 Flash (Not Cheaper)

We considered Gemini 2.5 Flash Lite ($0.10/$0.40 per 1M, ~$0.10/user/month). The math:
- Cheap free tier converts at 2-3% (industry standard)
- Wow free tier converts at 10%+ (our thesis)
- At 3% of 100K: 3K paid × $10 = $30K GMV. Subsidy: $10K. Thin.
- At 10% of 100K: 10K paid × $10 = $100K GMV. Subsidy: $46K. Healthy.

Higher free tier cost is an investment in conversion rate.

---

## Revenue Model

### Cowork Credits (Primary)

```
User buys $100 in credits
→ Charged $105.50 (5.5% service fee)
→ Cowork sends $105.50 to OpenRouter (covers OR's 5.5% top-up)
→ $100 usable credits
→ Cowork keeps $5.50 per $100 in inference

Net margin: 5.5% of GMV
```

### The Margin Problem

At 5.5%, credit pass-through doesn't cover free tier costs at scale:

| Scale | Paid users (10%) | Monthly GMV | Cowork Rev (5.5%) | Free tier cost | Net |
|-------|-----------------|-------------|-------------------|----------------|-----|
| 10K | 1,000 | $10K | $550 | $3,300 | -$2,750 |
| 50K | 5,000 | $75K | $4,125 | $16,500 | -$12,375 |
| 100K | 10,000 | $200K | $11,000 | $33,000 | -$22,000 |

**5.5% is the entry play, not the endgame.** Revenue shifts to:
1. Higher credit margin (10-15% as scale justifies)
2. Platform subscription ($5-10/month base fee for paid tiers)
3. Enterprise per-seat pricing
4. Data insights / analytics as separate SaaS

### The Improving Unit Economics

Three axes improve simultaneously:
1. Inference costs drop ~50% every 6-12 months → free tier gets cheaper
2. Conversion rate improves as product matures → more paid users per free user
3. Revenue per user grows → users upgrade as agent does more

Even at thin margins, a user who goes from Boost ($10/mo) to Pro ($50/mo) over 12 months = $360 GMV = $19.80 at 5.5%, $54 at 15%.

### BYOK (Secondary)

Power users bring own OpenRouter API key. Cowork earns $0 on inference. Value: stickiness, usage data, eventual conversion. Not the mass market path.

---

## Worker Economics

### Target User

Filipino remote worker. CX agent, VA, data entry, dev support. $500-1,500/month income. Windows laptop (8-16GB), stable internet. Not technical. Very price-sensitive — $10/month is meaningful.

### ROI by Tier

| Tier | Cost | Time Saved | Extra Earning Potential | ROI |
|------|------|-----------|----------------------|-----|
| Boost ($10/mo) | $10 | 1-2 hrs/day | $100-200/month | 10-20x |
| Pro ($50/mo) | $50 | 3-4 hrs/day | $400/month (second gig) | 8x |
| Max ($200/mo) | $200 | Agent handles volume | $800-1,300/month | 4-6x |

### The Framing

Workers think in money, not tokens:
- "Spend $10, make $200 more."
- "Spend $50, basically get a second job."
- "Spend $200, make more than your manager."

Surface this math everywhere: dashboards, upgrade prompts, monthly summaries.

---

## Competitive Position

### vs ChatGPT/Claude Direct

| | ChatGPT/Claude | Cowork.ai |
|---|---|---|
| Context | Doesn't know your ticket/SOP/last call | Always watching (with consent) |
| Action | Copy-paste text | Drafts IN Zendesk, files email, updates sheet |
| Continuity | Resets every conversation | Persistent workflow model |
| Local | Nothing runs locally | Whisper + DeepSeek = free daily assistance |
| Cost | $20/mo flat for one model | $0-10/mo, scales with value |

The pitch: "You're paying $20/month to copy-paste. Pay $10/month to have it actually do the work."

---

## Key Metrics

| Metric | Target |
|--------|--------|
| Free tier quota utilization | >70% hit quota |
| Days to quota exhaustion | Trending shorter |
| Free → Boost conversion | >10% |
| Boost → Pro upgrade | >15% of Boost users |
| Monthly churn (Boost) | <5% |
| Monthly churn (Pro/Max) | <3% |
| Cost per free user | <$0.50/month |
| Local vs cloud usage split | Track — shows DeepSeek value |

---

## Open Assumptions

1. **10% free-to-paid achievable.** Industry is 2-5%. Gap week + wow free tier may push higher. Unproven.

2. **Workers pay with own money.** If employer pays, worker doesn't feel the gap. Enterprise tier needs different model (per-seat, worker gets Boost, pays personally for Pro/Max upgrades).

3. **DeepSeek-R1 8B is "good enough" locally.** Needs validation with real CX workflows. If local feels like a toy, "prefer local" users won't engage enough to convert.

4. **Quality gap between tiers stays meaningful.** If Gemini 2.5 Flash becomes "good enough" for everything, upgrade incentive weakens. Conversion then relies purely on time gap (1 week vs all month).

5. **250 interactions/month is right free quota.** Could be too generous or too stingy. A/B test.

6. **Thinking tokens won't blow up costs.** Gemini 2.5 Flash and 3 Flash support thinking modes (3-5x more tokens). Current assumption: thinking is user-triggered, not auto-routed.

7. **5.5% margin is temporarily acceptable.** It's not sustainable — see revenue projections. Plan the transition to higher margins before 100K+ free users.

---

## Changelog

**v4 (Feb 12, 2026):**
- Added "Two Brains" framework (local DeepSeek + cloud tiered)
- Added onboarding choice and its economic impact on blended cost
- Free tier: Gemini 2.5 Flash (was 2.0 Flash — deprecated March 31)
- Updated cost tables with blended estimates
- Revenue projections show margin problem explicitly
- Added DeepSeek-R1 validation as open assumption

**v3 (Feb 12, 2026):** Initial strategy document

---

*v4 — February 12, 2026*
*Living document. Update when new data arrives.*
*Companion: "Cowork.ai — LLM Architecture Spec" covers engineering implementation.*
