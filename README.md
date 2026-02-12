# Cowork Brain

## What is Cowork.ai?

Cowork.ai is a desktop AI agent for remote workers. It runs as a lightweight sidecar alongside your work applications, connects to your tools (Zendesk, Gmail, Slack, CRM, etc.) through MCP, and helps you handle more work — drafting ticket responses, triaging inboxes, summarizing channels, preparing for meetings — inside the tools you already use.

Two things make it different from ChatGPT or Copilot: it runs a local AI model on your machine (DeepSeek-R1, free, no data leaves the device), and it connects to your actual work applications through MCP to take actions, not just generate text to copy-paste.

For the full product story — who it's for, how it works, where it's going — read **[product/product-overview.md](product/product-overview.md)**.

---

## What is This Repo?

**The executive decision layer for Cowork.ai.** This is where technical decisions, business decisions, and go-to-market strategy converge — with the reasoning behind every one.

This is not a code repo. This is not a task tracker. It's the collective intelligence behind building and distributing Cowork.ai, in a format that can be audited, challenged, and built from by humans or AI tools.

Cowork.ai sits at the intersection of engineering tradeoffs, cost economics, and distribution strategy. Changing a model selection changes cost-to-serve. Changing cost-to-serve changes pricing. Changing pricing changes which GTM channels make sense. One decision ripples everywhere. This repo captures those decisions and the reasoning behind them so that:

- **Anyone building** can understand not just *what* to build, but *why* it's built that way
- **Anyone reviewing** (Scott, investors, future team leads) can audit the logic and challenge assumptions
- **Any AI tool** (Claude Code, Cursor, etc.) can pull context and reason about tradeoffs without starting from scratch every conversation

---

## What Lives Here

```
├── README.md                  ← You're here
├── CONTRIBUTING.md            ← Rules for changes (read first)
│
├── product/                   ← What we're building
│   └── product-overview.md    ← What Cowork.ai is, who it's for, where it's going
│
├── architecture/              ← How it works (technically)
│   └── llm-architecture.md    ← Models, routing, budgets, hardware, infra
│
├── strategy/                  ← Why it works (economically)
│   └── llm-strategy.md        ← Pricing, cost-to-serve, tiers, conversion, worker ROI
│
├── design/                    ← What it looks like and why
│   ├── design-system.md       ← M3 tokens, interaction model, visual system, feature set
│   └── prototype-brief.md     ← What the clickable demo should show and how to build it
│
├── gtm/                       ← How we get it to people
│   └── launch-gtm-strategy.md ← Distribution channels, phased rollout, messaging
│
├── decisions/                 ← The audit trail
│   └── decision-log.md        ← Every significant change with full reasoning
│
├── prototypes/                ← UI concepts & demos
│   └── README.md              ← Links to clickable prototypes and design references
│
└── repos.md                   ← Links to all Cowork code repos
```

### `/product` — Start here
What Cowork.ai is, who it's for, what users can do, how it relates to Go2, and where it's going. If you're new to the project, read this first.

### `/architecture` — How it works
Engineering build specs. What models, what hardware, what routing logic. Rustan's primary reference. Updated when technical reality diverges from plan.

### `/strategy` — Why it works (economically)
Cost models, tier pricing, conversion assumptions, worker ROI math. Where the money comes from, where it goes, and why the unit economics work (or don't yet). Scott's primary reference.

### `/design` — What it looks like and why
M3 design system, interaction model, visual tokens, and the correct feature set. This is the spec that all UI work — human-built or AI-generated — must follow. Also contains the prototype brief: what the clickable demo should show and how to build it. Read this before touching any UI code.

### `/gtm` — How we get it to people
Distribution strategy. The assets we have (email list, LinkedIn following, sourcing team, Go2 enterprise leads), the channels we'll use, the phased rollout plan. Not marketing copy — marketing *logic*.

### `/decisions` — Why we chose what we chose
**The most important directory.** Every change that affects cost, model selection, routing, pricing, or distribution gets an entry with full reasoning. Not "changed X." Instead: "Changed X because Y, which costs Z, and we considered A/B/C before deciding." See [CONTRIBUTING.md](CONTRIBUTING.md).

### `/prototypes` — What it looks like
Links to UI demos, clickable prototypes, design concepts. Not the source files (those live in code repos) — the references and the reasoning behind UX decisions.

### `repos.md` — Where the code lives
Links to all Cowork code repos with descriptions. This repo points to them; they point back here for context.

---

## What Does NOT Live Here

- **Tasks, tickets, sprints.** Use whatever project tool works (Linear, GitHub Issues, Notion). This repo is concepts and reasoning, not execution tracking.
- **Marketing copy.** The GTM section captures distribution *strategy*, not blog posts or ad copy.
- **Application code.** Code repos are linked in `repos.md`. This repo is the *why*, code repos are the *how*.
- **Meeting notes.** Unless a meeting produced a decision worth logging, it doesn't go here.
- **Detailed user personas, acceptance criteria, sprint roadmaps.** Those are product management process docs — they belong in your project tool, not in the decision layer.

---

## The One Rule

**When you change something that affects money, functionality, or distribution — write down why.**

These decisions compound. Changing the free tier model changes cost-to-serve, which changes conversion economics, which changes revenue projections, which changes what GTM channels we can afford. One decision here ripples through everything.

If you can't explain why you changed it, don't change it. If the reasoning doesn't survive a PR review, it gets reverted.

---

## Current State

| Document | Version | Last Updated |
|----------|---------|-------------|
| Product Overview | v1.0 | Feb 12, 2026 |
| LLM Architecture Spec | v4 | Feb 12, 2026 |
| LLM Strategy & Economics | v4 | Feb 12, 2026 |
| Design System (M3) | v1.0 | Feb 12, 2026 |
| GTM & Distribution | v1.1 | Feb 12, 2026 |
| Prototype Brief | v0.1 | Feb 12, 2026 |
| Decision Log | initialized | Feb 12, 2026 |
