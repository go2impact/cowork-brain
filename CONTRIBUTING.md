# Contributing to Cowork.ai Architecture

## The Process

1. **Branch** from `main` (name: `your-name/short-description`, e.g. `rustan/whisper-warm-up`)
2. **Edit** the relevant doc(s) in `product/`, `architecture/`, `strategy/`, `design/`, or `gtm/`
3. **Add an entry** to [decisions/decision-log.md](decisions/decision-log.md) (see format below)
4. **Update the changelog** at the bottom of any doc you changed
5. **PR** to `main`. Tag Scott for review on anything that affects cost or tier logic.

## What Requires a Decision Log Entry

Any change that affects one or more of:

- **Model selection** (which models we use, at which tier)
- **Cost structure** (what we pay per user, what users pay us)
- **Routing logic** (what goes to local vs cloud, complexity thresholds)
- **Tier boundaries** (what free vs paid users get)
- **Token budgets** (quotas, caps, limits)
- **Privacy/consent** (what data leaves the device, what doesn't)
- **Hardware requirements** (minimum specs, what runs where)

If your change doesn't touch any of these, you don't need a decision log entry. Typo fixes, clarifications, and formatting don't need entries.

## Decision Log Entry Format

```markdown
## YYYY-MM-DD — Short Title

**Changed:** What specifically changed (the diff in plain English)

**From → To:** Previous state → New state

**Why:** The reasoning. Not "because it's better." WHY is it better?
What problem does this solve? What data or analysis drove this?
What are the cost implications? What are the risks?

**Cost impact:** Dollar impact if applicable (per user, at scale)

**Alternatives considered:** What else did we look at and why we didn't pick it

**Approved by:** Who signed off (Scott for cost/strategy changes)
```

### Example of a GOOD entry:

```markdown
## 2026-02-12 — Free tier model upgrade: Gemini 2.0 Flash → Gemini 2.5 Flash

**Changed:** Free tier cloud model upgraded from Gemini 2.0 Flash to Gemini 2.5 Flash

**From → To:** $0.10/user/month → $0.46/user/month (at 100K users: $10K/mo → $46K/mo)

**Why:** Three reasons:
1. Gemini 2.0 Flash is deprecated March 31, 2026. Can't build on a dying model.
2. The free week must prove enough value that losing it hurts. Gemini 2.5 Flash has thinking capabilities
   and 1M context — it feels genuinely useful, not like a demo. A mediocre free
   experience converts at 2-3% (industry standard). We need 10%+.
3. At 10% conversion with 100K free users: 10K paid users × $10/mo avg = $100K GMV.
   The $46K subsidy is justified as customer acquisition cost.

**Cost impact:** ~4.6x increase in free tier cost per user. Mitigated by:
- "Prefer local" users on 16GB+ Macs cost ~$0.15-0.25/month (local handles most work)
- Waitlist system caps total free tier spend at configurable threshold
- Model prices trend down ~50% every 6-12 months

**Alternatives considered:**
- Gemini 2.5 Flash Lite ($0.10/user/month) — cheaper but not a "wow" experience
- Blended approach (Lite for simple, Flash for complex) — added routing complexity
  for marginal savings, decided against
- Keeping 2.0 Flash — deprecated, not an option

**Approved by:** Scott
```

### Example of a BAD entry:

```markdown
## 2026-02-12 — Model update

**Changed:** Updated model

**Why:** Better model available
```

This tells us nothing. Six months from now when costs are higher than projected and someone asks "why did we pick this model?", this entry is useless. Write it like you're explaining the decision to someone who wasn't in the room and needs to understand it in 30 seconds.

## Document Changelog Format

Every doc has a changelog at the bottom. When you edit a doc, add a bullet to the current version. When changes are significant enough to warrant a new version number (Scott's call), bump the version.

```markdown
## Changelog

**v4 (Feb 12, 2026):**
- [your change here with brief explanation]

**v3 (Feb 12, 2026):** Initial spec
```

## PR Review Rules

- **Cost/strategy changes:** Scott must review and approve
- **Architecture changes:** Rustan reviews. If cost implications, Scott also reviews.
- **Typo/clarification fixes:** Anyone can merge

## Using AI Tools to Edit

Using Claude Code, Cursor, or any AI tool to edit these docs is fine and encouraged. But the AI doesn't make the decision — you do. If Claude suggests changing the routing logic, you need to understand and endorse the reasoning before committing it. "Claude said so" is not valid reasoning in the decision log.
