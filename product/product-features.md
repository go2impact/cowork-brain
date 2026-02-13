# Cowork.ai Sidecar — Product Features & Capabilities

**Status:** Working draft (capability map, not a release scope contract)
**Last Updated:** 2026-02-13
**Purpose:** Detailed feature set, capabilities, and user stories for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries.
**Scope Note:** This document defines the full product concept across phases. Current release scope is tracked in [`product-overview.md`](./product-overview.md) and implementation sequencing in [`../architecture/llm-architecture.md`](../architecture/llm-architecture.md).

**First wedge:** CX agent + Zendesk. One persona, one app, one loop.

---

## Sources

Synthesized from: [`product-overview.md`](./product-overview.md), [`design-system.md`](../design/design-system.md), [`prototype-brief.md`](../design/prototype-brief.md), [Issue #3](https://github.com/go2impact/cowork-brain/issues/3) + [Scott's positioning guardrails](https://github.com/go2impact/cowork-brain/issues/3#issuecomment-3891009312), and direct product input from Rustan (hybrid execution model, second-brain thesis). User stories and "Success Looks Like" were composed from multiple sources.

---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI workspace — your "second brain" — that works inside your apps alongside you, handling your secondary work (communication, scheduling, support) while you watch, coach, and stay focused on your primary work.

---

## Core Design Principle

The sidecar doesn't replace your tools. It sits on top of them *and* operates inside them. You still use Zendesk, Gmail, Slack, and your calendar — but the AI watches them via API, drafts responses, catches what you missed, and when you approve an action, it can execute visibly in your browser while you watch.

You are always in control. The AI works in the open — never behind a curtain. You can intervene at any moment: pause it, correct it, take over, or hand back. MCP handles data retrieval and background work. The browser handles visible execution and coached actions. Both work together for complex tasks.

The sidecar only expands when you need it.

---

## Feature Summary

### Feature Set

| # | Feature | What it does |
|---|---------|-------------|
| 1 | **App Ecosystem** | App Gallery — pre-built integrations (Slack, Gmail, Zendesk, Salesforce, etc.). Apps connect to Cowork.ai's exposed MCP features for data retrieval and action execution. |
| 2 | **Execution Modes (Hybrid Model)** | MCP (API, fast, background) + Browser/Playwright (visible, coachable). Work together on complex tasks. Configurable per app/category/action. |
| 3 | **Activity Capture & Context Engine** | Six input streams: window/app tracking, keystroke/input capture, focus detection, browser activity, screen recording, clipboard monitoring. Local SQLite storage. Most streams off by default. |

### Cross-Cutting Sections

| Section | What it covers |
|---------|---------------|
| **Autonomy Levels** | External = approval required, internal = can be autonomous. Promotion/demotion of categories. Execution mode preference (API vs browser). |
| **Guardrails** | Non-promotable categories (financial, first-contact, destructive). Kill switch, rate limiting, best-effort recall. |
| **Privacy & Data Boundaries** | What's collected, where it's stored, retention defaults, what's never collected, who sees what, surveillance-tool differentiation. |
| **User Stories** | CX Agent (first wedge). |
| **"What Cowork.ai Is NOT"** | Not a time tracker, not a chatbot, not employer monitoring, not a tool replacement. |

---

## Feature Set

### 1. App Ecosystem

**App Gallery:**

A library of pre-built integrations. Install the apps that match your workflow:

- **Communication:** Slack, Gmail, Microsoft Teams
- **Support:** Zendesk, Freshdesk, Intercom
- **CRM:** Salesforce, HubSpot
- **Project Management:** Linear, Asana, Jira
- **Calendar:** Google Calendar, Outlook

Apps connect to Cowork.ai's exposed MCP features for data retrieval and action execution. Each installed app shows a compact status card in the sidesheet (unread count, queue depth, next event) and expands to a full view for deep interaction.

**Cross-app boundaries:**

- Cross-app automations run only when required connections and scopes are available.
- If a required source is disconnected or auth expires, the automation pauses and creates a queue item with the exact blocker and reconnect action.
- Partial permissions degrade gracefully (read-only summaries allowed; blocked write actions require reconnect/approval).

**Execution Viewer — Full Transparency:**

The Execution Viewer is a dual-pane interface that shows everything the AI is doing — both the API calls happening in the background and the browser actions happening visually.

- **Live browser view** — Watch the AI's browser session in real-time. See it navigate apps, fill forms, compose replies, and submit actions. Controls: Pause, Take Over (switch to your own mouse/keyboard), Resume, and voice coaching. The browser view is only active during browser-mode execution.
- **Execution log** — A unified timeline of all AI activity: MCP calls, browser actions, coaching interventions, and outcomes. Every entry shows what happened, when, in which app, and the result.

Example execution log:

```
10:32:04  MCP     zendesk.tickets.get #4521      → read ticket
10:32:06  MCP     clone-mode.draft               → generated reply
10:32:09  BROWSER navigate zendesk.com/tickets/4521
10:32:12  BROWSER type draft into reply field
10:32:14  VOICE   user: "make it more empathetic"
10:32:18  BROWSER click "Submit as Solved"
```

The Execution Viewer is always accessible. Transparency builds trust; trust enables autonomy.

---

### 2. Execution Modes

Cowork.ai uses a hybrid execution model: MCP (API calls) and browser automation (Playwright) work simultaneously, not as alternatives. The AI picks the right tool for each step of a task, and the user can override the default.

**MCP (API execution):**

- Fast, reliable, runs in the background.
- Best for: data retrieval, bulk operations, well-defined API endpoints, read operations, automated categorization.
- Example: reading 50 Zendesk tickets, categorizing emails, fetching calendar events, checking Linear for existing issues.

**Browser (Playwright execution):**

- Visible, coachable, user-supervised.
- Best for: complex multi-field forms, visual verification ("does this reply look right in context?"), apps without full API support, actions the user wants to watch and coach.
- Example: composing a nuanced Zendesk reply in the actual UI, navigating a room booking portal, submitting a multi-step form in an internal tool.

**How they work together — a Zendesk ticket reply:**

1. **MCP** reads the ticket content, customer history, and related tickets (fast, background).
2. **MCP** drafts the reply (fast, background).
3. Draft is reviewed and approved by the user.
4. **Browser** opens the Zendesk ticket in the actual UI, pastes the approved reply, and positions for send (visible, coachable).
5. User watches, optionally coaches ("add a note about the refund timeline"), and the AI adjusts in real-time.
6. **Browser** clicks Send. User sees it happen.
7. **MCP** logs the completed action to the audit trail.

**Key principle:** MCP handles data and background work. The browser handles visible execution and user-supervised actions. The user configures which mode applies per app, per category, or per action — and can override on any individual item.

---

### 3. Activity Capture & Context Engine

The AI needs context to be useful. Activity capture provides that context — what you're working on, how you communicate, where you spend your time. This data feeds the AI's ability to act relevantly.

**Input streams:**

| Stream | Default | Used for |
|--------|---------|----------|
| **Window & app tracking** (active app, window title, browser URL via macOS Accessibility API) | On | Work context |
| **Keystroke & input capture** (raw input → communication patterns) | Off (opt-in) | Communication pattern extraction |
| **Focus detection** (extended single-app sessions, derived from window tracking) | On | Flow-state detection |
| **Browser activity** (pages visited, actions taken during agent sessions) | On during sessions | Execution Viewer, audit trail |
| **Screen recording** (15fps during coached agent sessions) | Off (opt-in) | Agent coaching |
| **Clipboard monitoring** (text on copy/paste) | Off (opt-in) | Context enrichment |

**Data flow:** Raw capture → local SQLite (GRDB, WAL mode) → local processing → structured context (embeddings via local RAG) → agents query at action time. All on-device. Processing pipeline that discards raw input is not yet implemented — raw data currently persists locally.

**Retention:** Window/app tracking and focus detection roll 7 days. Keystroke patterns and browser/screen recordings roll 30 days. Retention enforcement not yet implemented.

---

## Autonomy Levels & The Approval Boundary

The features above describe two different modes of AI action: drafting for approval and acting autonomously. This isn't a contradiction — it's a spectrum that the user controls.

### The Default Rule

**External-facing actions require approval. Internal actions can be autonomous.**

| Action Type | Default | Examples |
|-------------|---------|----------|
| Send a message to a customer | Requires approval | Zendesk ticket reply, client email, support chat response |
| Send a message to a coworker | Requires approval | Slack DM, email to teammate |
| Organize your own workspace | Autonomous | Archive email, categorize tickets, prepare meeting briefs, compile summaries |
| Block or redirect your own time | Autonomous | Auto-decline conflicting meetings, hold notifications during deep work |
| Escalate to you personally | Autonomous | Push notification to your phone, break through for emergencies |

The line is: **anything that another human will see as coming from you requires your approval by default. Anything that only affects your own workspace can be pre-authorized.**

### Execution Mode Preference

When you approve an action, you also control *how* it executes:

| Execution preference | What happens | When to use it |
|---------------------|-------------|----------------|
| **API (instant)** | Action executes via MCP in the background. Fastest option. | Routine items you trust the AI to handle. |
| **Browser (visible)** | Action executes in the browser. You can watch and coach. | Items you want to verify visually or coach on. |
| **Autonomous but visible** | Action runs without approval, but executes in the browser so you can watch. | High-trust categories where you still want oversight. |

Execution preference is configurable per app, per category, per contact, or per individual item. The default is API for internal actions and browser for the first N external actions (configurable, default 10) — shifting to API as trust builds.

### How Users Move the Boundary

The default is conservative. Over time, as trust builds, users can expand what the AI handles autonomously:

- **Promote to auto-send.** After approving 50 standard Zendesk status updates without editing, you might tell the AI: "Handle routine status updates automatically. Just log them." (Note: some categories — financial commitments, first-contact messages, destructive actions — cannot be promoted. See Guardrails below.)
- **Demote to approval-required.** If the AI auto-declines a meeting you actually wanted, you can pull that action category back: "Always ask me before declining meetings."
- **Per-channel rules.** "Auto-send in Slack #general, but queue everything in #client-escalations."
- **Per-contact rules.** "Auto-respond to internal team, but always queue anything from the VP of Sales."
- **Execution mode shift.** "I trust refund replies now — switch those to API mode so I don't have to watch."

Boundary changes are always user-initiated — the AI may *suggest* a promotion but never auto-promotes. Every action, regardless of autonomy level or execution mode, is logged in the Execution Viewer.

### Guardrails & Incident Containment

**Non-promotable categories (hard limits):**

These categories always require explicit approval, regardless of trust level or autonomy settings. They cannot be promoted to autonomous:

- Financial commitments — refunds, credits, pricing promises, payment adjustments.
- First contact with a new external party — the AI never auto-sends to someone you haven't communicated with before.
- Destructive actions — data deletion, account closure, permission revocation.

**Kill switch:** A global "pause all autonomous actions" control, accessible via the sidecar UI or voice command ("stop everything"). When activated, all pending and queued autonomous actions are halted. Actions already submitted to an external platform (API call sent, message delivered) cannot be recalled from the kill switch — use platform-level undo where available (see Recall below). Unpause resumes from where it stopped — nothing is auto-retried without your confirmation.

**Rate limiting:** Autonomous actions are capped per hour per app (configurable, sensible defaults TBD during v0.1 validation). If the cap is hit, remaining actions queue for approval and the user is notified immediately. This prevents stale SOPs from generating unchecked volume.

**Recall (best-effort):** For platforms that support undo or recall (Gmail undo-send window, Slack message edit window), the Execution Viewer offers a time-limited recall button after autonomous sends. For platforms without recall support, the action is logged but not reversible — this limitation is disclosed when the user promotes a category to autonomous.

---

## User Story — First Wedge

### As a CX Agent (Zendesk + Slack)

> I handle 40–60 Zendesk tickets per day. Half are routine: password resets, billing questions, feature requests I've answered a hundred times.
>
> With Cowork.ai, I open my sidecar in the morning and 15 drafted replies are ready. I approve 12 via API — instant, gone. For the 2 that need tweaks, I edit and approve. For 1 tricky escalation, I switch to browser mode: the AI opens the ticket in Zendesk, types the reply I approved, and I watch. Halfway through, I say "add a note that we're waiving the fee as a one-time courtesy." The AI adjusts on the spot, I confirm, and it sends. That took 10 minutes instead of 45.
>
> When I go heads-down on a complex escalation, the AI holds my Slack pings. When I come up for air, I get a summary instead of 47 unread messages.

*Additional user stories (Virtual Assistant, Operations Specialist, Dev Support Engineer) are available in earlier revisions of this document.*

---

## Privacy & Data Boundaries

Cowork.ai observes your work context to be useful. That observation has to be clearly bounded, or "AI assistant" becomes indistinguishable from "surveillance tool." These are the rules.

### What the AI observes

| Data type | Stored where | Retention |
|-----------|-------------|-----------|
| **Connected app data** (Zendesk, Gmail, Slack) — only apps/channels you explicitly connect | On-device. Cloud inference is transient (zero-retention providers). | Until user disconnects app. |
| **Activity context** (active app, window title, browser URL via Accessibility API) | On-device only. Never leaves device. | Rolling 7 days. |
| **Keystroke & input capture** (opt-in, raw → communication patterns) | On-device only. | Target: raw discarded within minutes; patterns roll 30 days. |
| **Browser session data** (agent execution only, not general browsing) | On-device only. | Rolling 30 days. |
| **AI profiles** (tone, rules, SOPs) | On-device only. | Until user deletes. |
| **Action history** (audit log of all AI actions) | On-device only. | Metadata persists; content purged on app disconnect. |

### What the AI never collects

- **Continuous screen recording** — screen capture is opt-in, agent-session-only, not ambient surveillance.
- **Keystroke data for employer reporting** — keystroke capture trains your AI, never reported to anyone else. Never leaves device.
- **Browsing history outside agent sessions** — activity context reads window titles and URLs for context, not full app content.
- **Idle/active time** — no attendance monitoring. Activity context is for flow-state detection only.
- **Audio when not activated** — wake word detection is ephemeral on-device processing. Audio never leaves the device.

### Who can see what

- **The worker** — everything: all data, all actions, all audit logs.
- **The employer** — nothing by default. Enterprise plans allow opt-in aggregate data (team-level only, never per-worker action logs).
- **Cowork.ai** — pseudonymized usage telemetry only (feature counts, error rates, latency). No message content, no activity context, no keystroke data.
- **Inference providers** — connected app content transiently during cloud inference (zero-retention providers). Local-only mode sends nothing.

### How this is different from surveillance tools

The difference is architectural, not just messaging: granular per-stream consent (not blanket surveillance), browser automation is user-initiated and user-supervised (not background control), all capture trains your AI (not an employer dashboard), local-first storage (nothing accumulates on a central server), and the worker controls the data (disconnect = immediate loss of access, delete = gone).

---

## What Cowork.ai Is NOT

Getting the positioning right matters. Cowork.ai is not:

- **A time tracker.** We don't measure how many hours you worked or when you were "active." There are no focus scores, no productivity gamification, no surveillance dashboards. Activity capture feeds the AI's context, not timesheets.
- **A standalone chatbot.** You don't copy-paste context into a chat window. The AI already knows your context because it's connected to your tools.
- **An employer monitoring tool.** The worker owns their data. The worker controls what's connected. The worker decides what gets shared. Activity capture, keystroke patterns, and browser session data exist to train the worker's AI — they are never surfaced to employers, even on enterprise plans.
- **A replacement for your tools.** You still use Zendesk, Gmail, Slack. Cowork.ai sits on top of them (via API) and works inside them (via browser automation) — not instead of them.

---

## Success Looks Like

From the worker's perspective, Cowork.ai is working when:

- You open the sidecar and your queue is already handled. You just approve and move on.
- You finish a focus session and nothing fell through the cracks while you were heads-down.
- Your AI drafts are so good that you approve 80%+ without editing.
- You talk to the AI and it does what you asked on the first try.
- You handle more volume in less time, and your clients notice the improvement — not the AI.
- You watch the AI compose a Zendesk reply and it sounds exactly like you.
- You coach the AI mid-task and it adjusts instantly, like a colleague who learns your style.

The AI should be invisible when it's working well. If someone reads your replies and thinks "that person is great at their job," we won. If they think "that person is using AI," we have more work to do.

---

## See Also

| Topic | Document |
|-------|----------|
| Product overview & positioning | [product-overview.md](./product-overview.md) |
| Design system & interaction model | [../design/design-system.md](../design/design-system.md) |
| Business economics & pricing | [../strategy/llm-strategy.md](../strategy/llm-strategy.md) |
| Technical architecture | [../architecture/llm-architecture.md](../architecture/llm-architecture.md) |
| Go-to-market strategy | [../gtm/launch-gtm-strategy.md](../gtm/launch-gtm-strategy.md) |

---

## Changelog

| Date | Changes |
|------|---------|
| 2026-02-13 | Added guardrails & incident containment (non-promotable categories, kill switch, rate limiting, recall). Added first-wedge anchor. Fixed consent framing. |
| 2026-02-13 | Aligned activity capture with native Mac app codebase. Added honest "current state vs. target" annotations throughout. |
| 2026-02-13 | Clarified app-as-component developer model. |
| 2026-02-13 | Added hybrid execution model (MCP + browser). Added Activity Capture & Context Engine. Rewrote user stories. Expanded privacy section. |
| 2026-02-13 | Review findings resolved across all feature sections. |
| 2026-02-13 | Initial draft: App Ecosystem, Execution Modes, Activity Capture. |
