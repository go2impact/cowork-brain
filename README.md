# Cowork.ai

**AI-powered work tools for the workers nobody else is building for — distributed through an ecosystem we already own.**

---

## The Opportunity

There are tens of millions of remote workers — virtual assistants, support agents, account managers — doing repetitive knowledge work across Zendesk, Gmail, Slack, and CRMs. They do the same 15 tasks 200 times a day. AI could handle most of it.

But nobody is building AI tools for these workers. OpenAI and Anthropic sell to developers and enterprises. Microsoft sells to Fortune 500 IT departments. The tools that do target remote workers are employer-owned surveillance platforms that workers resist and game.

Meanwhile, Go2 has spent 8 years building an ecosystem in exactly this market: 2M+ verified remote worker profiles, thousands of small business clients, 250K LinkedIn followers, 1,000+ new applicants per day. We know these workers by name, by hardware spec, by which tools they use at work.

Cowork.ai puts AI into the hands of workers who are already in our ecosystem — and takes a rake on every token they use.

---

## What Cowork.ai Is

A desktop sidecar that runs alongside a worker's apps with two capabilities nothing else combines:

**It runs locally.** DeepSeek-R1 runs on the worker's machine. No API calls, no per-token cost, no data leaving the device. Handles ~70% of daily work — drafting ticket responses, summarizing threads, triaging inboxes — at zero operating cost. Cloud models (up to Claude Opus 4.6) are there when the task demands it.

**It connects to real tools through MCP.** Model Context Protocol gives the agent standardized access to Zendesk, Gmail, Slack, Salesforce, Calendar — not through screenshots or copy-paste, but through actual tool calls. It reads the ticket queue, drafts a response, and posts it. It summarizes unread Slack channels. It preps for the next meeting using calendar and CRM data.

The result: a worker using Cowork.ai handles more volume at higher quality. They don't go to the AI — the AI is already inside their workflow.

For the full product story — who it's for, the roadmap, how it relates to Go2 — read **[product/product-overview.md](product/product-overview.md)**.

---

## The Business Model

Workers use Cowork.ai. Most daily work runs on the free local model. When they need more — larger context, harder reasoning, autonomous workflows — they upgrade to cloud tiers.

Cloud usage routes through us. We take 5.5% on the pass-through. We don't build models. We don't compete with model companies. We sit between the worker and the inference provider, and we earn on volume.

| Tier | Price | What They Get |
|------|-------|--------------|
| Free | $0 | Local model + 1 free week/month of cloud (Gemini 2.5 Flash) |
| Boost | $5–15/mo | Full month cloud access, Gemini 3 Flash, 1M context |
| Pro | $30–60/mo | Claude Sonnet 4.5, semi-autonomous workflows |
| Max | $100–400/mo | Claude Opus 4.6, near-autonomous agent |

The free tier proves undeniable value. The wall — hitting the weekly quota — is what converts. Target: >10% free-to-paid conversion vs 2–5% industry standard.

Why the conversion rate is higher: see the next section.

See **[strategy/llm-strategy.md](strategy/llm-strategy.md)** for full cost economics and worker ROI math.

---

## The Trust Paradox

Here's the thing most AI-for-work companies get wrong.

Workers don't resist AI because they're afraid of technology. They resist it because every "AI productivity tool" they've seen is surveillance with a chatbot stapled on. Their employer buys it. It watches them. They tolerate it.

Go2 has 8 years of data on this. **When the employer owns the tool, 73% of workers resist it.** They minimize it, game the metrics, treat it as a threat. When the worker owns the tool — when it's theirs, running on their machine, with their data staying local — **92% opt in.** Same data. Same capabilities. Opposite outcome.

Cowork.ai is worker-first by architecture, not by marketing. The local model means no data leaves the device unless the worker explicitly chooses cloud processing. The worker installs it. The worker controls it. The worker benefits from it — not because we say so, but because the design makes surveillance structurally impossible.

This isn't philosophy. It's the conversion mechanism.

---

## The Distribution Advantage

Go2 spent 8 years building a staffing business. ~$2M in revenue, 1,000+ workers placed across 51 countries, $60M+ in total sales over the life of the company.

The staffing business built assets that no AI startup can replicate:

- **2M+ verified worker profiles** with full applications — CVs, hardware specs, tool proficiency, job history. Not scraped emails. Real workers who applied to work through Go2.
- **Hardware data on file.** We know who has 16GB+ RAM and Apple Silicon. We know who can run local models and who needs cloud-only.
- **Tool proficiency data.** We know who uses Zendesk vs Intercom vs Freshdesk. App recommendations come from actual work history.
- **Thousands of small business clients** in the HubSpot pipeline. These businesses already buy remote labor through Go2. They're not on OpenAI's radar.
- **250K LinkedIn followers** in the Filipino remote worker space. Organic distribution, not paid.
- **1,000+ new applicants daily.** Continuous inflow. Not a one-time list.
- **CAC is effectively zero.** The cost to reach the first 10,000 users is the cost of sending emails. Most AI startups spend $50–100 per user acquisition.

The staffing business is the distribution channel. Cowork.ai is what it distributes.

See **[gtm/launch-gtm-strategy.md](gtm/launch-gtm-strategy.md)** for the phased rollout plan.

---

## Why This Exists Now

Three things converged simultaneously. None of them were true 90 days ago:

**MCP became real.** AI agents can connect to local tools through a standardized protocol instead of brittle, per-platform integrations. This is the difference between AI that generates text and AI that does work.

**Local models crossed the reasoning threshold.** DeepSeek-R1 on consumer hardware can reason through a CX ticket with contradictory information. Six months ago, local models were autocomplete.

**Cloud inference costs collapsed 80%+.** A free work week of cloud AI costs ~$0.46/user/month. A year ago, the same experience cost $5–10/user. You couldn't afford to give it away.

These are enablers, not differentiators. Anyone can use MCP. Anyone can run DeepSeek. The differentiator is owning the ecosystem where these tools get used — and we already do.

---

## This Repo

**The executive decision layer for Cowork.ai.** Not code, not tasks — the reasoning behind every technical, business, and distribution decision.

Cowork.ai sits at the intersection of model selection, cost economics, and distribution strategy. Changing a model changes cost-to-serve. Changing cost-to-serve changes pricing. Changing pricing changes which channels work. One decision ripples everywhere. This repo captures those decisions so anyone — builder, reviewer, AI tool — can understand the reasoning without starting from scratch.

```
├── product/                   ← Start here
│   └── product-overview.md    ← What it is, who it's for, how it relates to Go2
│
├── architecture/              ← System architecture, LLM stack, apps runtime
│   ├── system-architecture.md
│   ├── capture-pipeline-architecture.md
│   └── llm-architecture.md
│
├── research/                  ← Open-source app analysis
│   └── adaptation-guides/     ← Copy/adapt/skip decisions per app
│
├── strategy/                  ← Pricing, cost-to-serve, tiers, worker ROI
│   └── llm-strategy.md
│
├── design/                    ← M3 design system, interaction model, feature set
│   ├── design-system.md
│   └── prototype-brief.md
│
├── gtm/                       ← Distribution channels, phased rollout
│   └── launch-gtm-strategy.md
│
├── decisions/                 ← Every significant change with full reasoning
│   └── decision-log.md
│
├── prototypes/                ← Links to clickable demos
│   └── README.md
│
└── repos.md                   ← Links to code repos
```

**If you're new:** Read [product/product-overview.md](product/product-overview.md), then [architecture/llm-architecture.md](architecture/llm-architecture.md), then [strategy/llm-strategy.md](strategy/llm-strategy.md).

**If you're building UI:** Read [design/design-system.md](design/design-system.md) before touching any code.

**If you're making a decision that affects cost, pricing, or distribution:** Log it in [decisions/decision-log.md](decisions/decision-log.md). See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## The One Rule

**When you change something that affects money, functionality, or distribution — write down why.**

These decisions compound. Changing the free tier model changes cost-to-serve, which changes conversion economics, which changes revenue projections, which changes what GTM channels we can afford. One decision here ripples through everything.

If you can't explain why you changed it, don't change it.

---

## Current State

| Document | Version | Last Updated |
|----------|---------|-------------|
| [Product Overview](product/product-overview.md) | v1.0 | 2026-02-12 |
| [LLM Architecture](architecture/llm-architecture.md) | v4 | 2026-02-12 |
| [LLM Strategy & Economics](strategy/llm-strategy.md) | v4 | 2026-02-12 |
| [Design System](design/design-system.md) | v1.0 | 2026-02-12 |
| [GTM & Distribution](gtm/launch-gtm-strategy.md) | v1.1 | 2026-02-12 |
| [Prototype Brief](design/prototype-brief.md) | v0.1 | 2026-02-12 |
| [Capture Pipeline Architecture](architecture/capture-pipeline-architecture.md) | v1 | 2026-02-17 |
| [Decision Log](decisions/decision-log.md) | initialized | 2026-02-12 |
