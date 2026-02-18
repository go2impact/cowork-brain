---
name: contribute
description: >
  Guide for contributing to the cowork-brain repo. Use when creating new files,
  editing existing docs, adding decision log entries, or making any change to this
  repository. Covers file placement, decision log format, changelog updates,
  cross-document consistency checks, and PR review rules. Trigger on: "where should
  I put this?", "add a new doc", "log this decision", "update the spec", "create a
  decision entry", or any repo modification task.
---

# Contributing to cowork-brain

This repo is the executive decision layer for Cowork.ai — not code. It contains
architecture specs, business strategy, design system, GTM plans, and a decision log.

## File Placement

Put new files in the directory that matches their purpose:

| Directory | What goes here | Examples |
|-----------|---------------|----------|
| `product/` | What Cowork.ai is, who it's for, roadmap | product-overview.md, product-features.md |
| `architecture/` | Engineering specs: models, routing, budgets, hardware | llm-architecture.md, system-architecture.md |
| `strategy/` | Cost models, pricing tiers, conversion economics | llm-strategy.md |
| `design/` | M3 design system, interaction model, prototype brief | design-system.md |
| `gtm/` | Distribution channels, phased rollout, Go2 asset leverage | launch-gtm-strategy.md |
| `decisions/` | Decision log + deep-dive decision docs and research | decision-log.md |
| `prototypes/` | Links to UI demos and design concepts | README.md |

Root-level files: `README.md` (project overview), `CONTRIBUTING.md` (this process),
`repos.md` (map of all Cowork.ai repos), `CLAUDE.md` (Claude Code instructions).

If a new file doesn't fit an existing directory, ask before creating a new one.

## Decision Log Entry

**Required** when a change affects: model selection, cost structure, routing logic,
tier boundaries, token budgets, privacy/consent, or hardware requirements.

**Not required** for: typo fixes, clarifications, formatting.

Append to `decisions/decision-log.md` using this format:

```markdown
---

## YYYY-MM-DD — Short Title

**Changed:** What specifically changed (plain English diff)

**From → To:** Previous state → New state

**Why:** Real reasoning — not "because it's better." What problem does this solve?
What data drove this? What are the cost implications and risks?

**Cost impact:** Dollar impact if applicable (per user, at scale)

**Alternatives considered:** What else was evaluated and why it was rejected

**Approved by:** Who signed off (Scott for cost/strategy, Rustan for architecture)
```

Every entry must have enough reasoning that someone reading it 6 months later
understands the decision without asking anyone.

## Changelog Updates

Every doc has a changelog at the bottom. When editing a doc, add a bullet to the
current version section:

```markdown
## Changelog

**v4 (Feb 12, 2026):**
- [your change here with brief explanation]
```

## Cross-Document Consistency

Changes in one doc often require updates elsewhere. Check for ripple effects:

- **Model change** → update `architecture/llm-architecture.md` + `strategy/llm-strategy.md` + decision log
- **Price change** → update `strategy/` + `gtm/` + decision log
- **UI/interaction change** → update `design/design-system.md` + decision log
- **Stack/dependency change** → grep old term across entire repo, update all references

Do NOT update `decisions/COWORKAI_TECHNICAL_REFERENCE.md` — it documents the
existing codebase, not the target.

## Branch and PR Process

1. Branch from `main`: `your-name/short-description` (e.g., `rustan/whisper-warm-up`)
2. Edit the relevant doc(s)
3. Add decision log entry if required (see triggers above)
4. Update changelog at bottom of changed doc(s)
5. PR to `main`

**Review ownership:**
- Cost/strategy changes → tag **Scott**
- Architecture changes → tag **Rustan** (+ Scott if cost implications)
- Typo/clarification fixes → anyone can merge, no decision log needed
