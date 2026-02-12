# Cowork Brain

**The executive decision layer for Cowork.ai.** Where technical decisions, business decisions, and go-to-market strategy converge — with the reasoning behind every one.

This is not a code repo. This is not a task tracker. This is the collective intelligence behind building and distributing Cowork.ai. Think of it as the CEO/CFO/CMO brain in a format that can be audited, challenged, and built from.

---

## Why This Exists

Cowork.ai sits at the intersection of engineering tradeoffs, cost economics, and distribution strategy. Changing a model selection changes cost-to-serve. Changing cost-to-serve changes pricing. Changing pricing changes which GTM channels make sense. One decision ripples everywhere.

This repo captures those decisions and the reasoning behind them so that:
- **Anyone building** can understand not just *what* to build, but *why* it's built that way
- **Anyone reviewing** (Scott, investors, future team leads) can audit the logic and challenge assumptions
- **Any AI tool** (Claude Code, Cursor, etc.) can pull context and reason about tradeoffs without starting from scratch every conversation

---

## What Lives Here

```
├── README.md                  ← You're here
├── CONTRIBUTING.md            ← Rules for changes (read first)
│
├── architecture/              ← Technical specs
│   └── llm-architecture.md    ← Models, routing, budgets, hardware, infra
│
├── strategy/                  ← Business logic
│   └── llm-strategy.md        ← Economics, pricing, tiers, conversion, worker ROI
│
├── decisions/                 ← The audit trail
│   └── decision-log.md        ← Every significant change with full reasoning
│
├── gtm/                       ← Go-to-market & distribution
│   └── README.md              ← Distribution assets, channels, launch strategy
│
├── prototypes/                ← UI concepts & demos
│   └── README.md              ← Links to clickable prototypes, design references
│
└── repos.md                   ← Links to all Cowork code repos
```

### `/architecture` — How it works
Engineering build specs. What models, what hardware, what routing logic. Rustan's primary reference. Updated when technical reality diverges from plan.

### `/strategy` — Why it works (economically)
Cost models, tier pricing, conversion assumptions, worker ROI math. Where the money comes from, where it goes, and why the unit economics work (or don't yet). Scott's primary reference.

### `/decisions` — Why we chose what we chose
**The most important directory.** Every change that affects cost, model selection, routing, pricing, or distribution gets an entry with full reasoning. Not "changed X." Instead: "Changed X because Y, which costs Z, and we considered A/B/C before deciding." See [CONTRIBUTING.md](CONTRIBUTING.md).

### `/gtm` — How we get it to people
Distribution strategy. The assets we have (email list, LinkedIn following, sourcing team, Go2 enterprise leads), the channels we'll use, the phased rollout plan. Not marketing copy — marketing *logic*.

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

---

## The One Rule

**When you change something that affects money, functionality, or distribution — write down why.**

These decisions compound. Changing the free tier model changes cost-to-serve, which changes conversion economics, which changes revenue projections, which changes what GTM channels we can afford. One decision here ripples through everything.

If you can't explain why you changed it, don't change it. If the reasoning doesn't survive a PR review, it gets reverted.

---

## Current State

| Document | Version | Last Updated |
|----------|---------|-------------|
| LLM Architecture Spec | v4 | Feb 12, 2026 |
| LLM Strategy & Economics | v4 | Feb 12, 2026 |
| Decision Log | initialized | Feb 12, 2026 |
| GTM & Distribution | placeholder | — |
| Prototypes | placeholder | — |
