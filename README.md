# Cowork.ai

**An AI agent that lives on your desktop, runs a local model, and works inside your actual tools.**

---

## The Problem

Every AI tool for knowledge workers does the same thing: you paste context into a chat window, get text back, copy it into the app you were already using. The AI doesn't know what you're working on. It can't touch your tools. It generates text — it doesn't do work.

Remote workers — the people doing support tickets, inbox triage, CRM updates, Slack management — do the same 15 tasks 200 times a day. They don't need a better chatbot. They need something that sits next to them, understands the work, and handles the repetitive parts.

## What Cowork.ai Actually Is

A desktop sidecar that runs alongside your work apps with two capabilities nothing else combines:

**1. It runs locally.** DeepSeek-R1 runs on your machine. No API calls, no per-token cost, no data leaving the device. Handles ~70% of daily work — drafting ticket responses, summarizing threads, triaging inboxes — at zero operating cost. Cloud models (up to Claude Opus 4.6) are available when the task demands it.

**2. It connects to your real tools through MCP.** Model Context Protocol gives the agent standardized access to Zendesk, Gmail, Slack, Salesforce, Calendar — not through screenshots or copy-paste, but through actual tool calls. It reads your ticket queue, drafts a response, and posts it. It summarizes your unread Slack channels. It preps you for your next meeting using your calendar and CRM data.

The result: a worker using Cowork.ai handles more volume at higher quality without switching tabs, without copy-pasting, without the AI being a separate thing they go to.

For the full product story — who it's for, the roadmap, how it relates to Go2 — read **[product/product-overview.md](product/product-overview.md)**.

---

## The Trust Paradox

Here's the thing most AI-for-work companies get wrong.

Workers don't resist AI because they're afraid of technology. They resist it because every "AI productivity tool" they've seen is surveillance with a chatbot stapled on. Their employer buys it. It watches them. They tolerate it.

Go2 has 8 years of data on this. **When the employer owns the tool, 73% of workers resist it.** They minimize it, game the metrics, treat it as a threat. When the worker owns the tool — when it's theirs, running on their machine, with their data staying local — **92% opt in.** Same data. Same capabilities. Opposite outcome.

Cowork.ai is worker-first by design, not by marketing. The local model means no data leaves the device unless the worker explicitly chooses cloud processing. The worker installs it. The worker controls it. The worker benefits from it — not because we say so, but because the architecture makes surveillance impossible by default.

This isn't philosophy. It's the conversion mechanism. Worker ownership is why the free-to-paid conversion target is >10% vs the industry standard 2-5%.

---

## Why This Exists Now

Three things converged simultaneously. None of them were true 90 days ago:

**MCP became real.** AI agents can now connect to local tools (Slack, Gmail, Zendesk) through a standardized protocol instead of brittle, per-platform integrations. This is the difference between "AI that generates text" and "AI that does work."

**Local models crossed the reasoning threshold.** DeepSeek-R1 on consumer hardware can think through a CX ticket with contradictory information, draft a nuanced response, and explain its reasoning. Six months ago, local models were autocomplete.

**Cloud inference costs collapsed 80%+.** Giving users a free work week of cloud AI costs ~$0.46/user/month. A year ago, that same experience cost $5-10/user. You couldn't afford to give it away. Now you can.

The window is open but it won't stay open. Every major AI company is building agents. Distribution is the moat — and Go2 has it.

---

## The Distribution Advantage

Go2 is an 8-year-old subscription staffing company. ~$2M in revenue, 1,000+ workers placed across 51 countries, 250K LinkedIn followers in the Filipino remote worker space.

More importantly: **2M+ verified remote worker profiles** with full applications — CVs, hardware specs, tool proficiency, job history. Not scraped emails. Not purchased lists. Real workers who applied to work through Go2.

What that means for Cowork.ai:

- **Hardware filtering.** We know who has 16GB+ RAM and Apple Silicon. We know who can run local models and who needs cloud-only. This is in the applicant data.
- **Tool proficiency data.** We know who uses Zendesk vs Intercom vs Freshdesk. We know who lives in Slack vs Teams. App recommendations are informed by actual work history.
- **CAC is effectively zero.** The cost to reach the first 10,000 users is the cost of sending emails. Most AI startups spend $50-100 per user acquisition. We already have the relationship.
- **1,000+ new applicants daily.** Continuous inflow. Not a one-time list.

Cowork.ai is Go2's intentional transition from a staffing business (managing people across 51 countries) to a software business (higher margin, more scalable). The staffing business becomes a distribution channel for the software business.

See **[gtm/launch-gtm-strategy.md](gtm/launch-gtm-strategy.md)** for the phased rollout plan.

---

## This Repo

**The executive decision layer for Cowork.ai.** Not code, not tasks — the reasoning behind every technical, business, and distribution decision.

Why a separate repo: Cowork.ai sits at the intersection of model selection, cost economics, and distribution strategy. Changing a model changes cost-to-serve. Changing cost-to-serve changes pricing. Changing pricing changes which GTM channels work. One decision ripples everywhere. This repo captures those decisions so anyone — builder, reviewer, AI tool — can understand the reasoning without starting from scratch.

```
├── product/                   ← Start here
│   └── product-overview.md    ← What it is, who it's for, how it relates to Go2
│
├── architecture/              ← Models, routing, budgets, hardware, infra
│   └── llm-architecture.md
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
| [Decision Log](decisions/decision-log.md) | initialized | 2026-02-12 |
