# Cowork.ai Sidecar — Product Features & Capabilities

**Status:** Working draft (capability map, not a release scope contract)
**Last Updated:** 2026-02-13
**Purpose:** Detailed feature set and capabilities for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries.
**Scope Note:** This document defines the full product concept across phases. Current release scope is tracked in [`product-overview.md`](./product-overview.md) and implementation sequencing in [`../architecture/llm-architecture.md`](../architecture/llm-architecture.md).

---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI workspace — your "second brain" — that works inside your apps alongside you, handling your secondary work (communication, scheduling, support) while you watch, coach, and stay focused on your primary work.

---

## Core Design Principle

The sidecar doesn't replace your tools. It sits on top of them *and* operates inside them. Apps built in Google AI Studio are uploaded to the desktop app, rendered inside it, and given access to platform-provided MCPs connected to your work context.

You are always in control. The AI works in the open — never behind a curtain. You can intervene at any moment: pause it, correct it, take over, or hand back.

The sidecar only expands when you need it.

---

## Feature Summary

### Feature Set

| # | Feature | What it does |
|---|---------|-------------|
| 1 | **App Ecosystem** | Apps built in Google AI Studio are uploaded to the Cowork.ai desktop app and rendered inside it. Apps get tools, not agents — they access platform capabilities through platform-provided MCPs (connected to Activity Capture data and connected services). The platform agent stays as the single orchestrator. |
| 2 | **Execution Modes** | Agents + custom tools execute through two paths: Tools → MCP (fast, background) or Tools → Browser (visible, coachable). Configurable per app/category/action. |
| 3 | **Activity Capture & Context Engine** | Six input streams: window/app tracking, keystroke/input capture, focus detection, browser activity, screen recording, clipboard monitoring. Local SQLite storage. This data is exposed to apps via platform-provided MCPs. Most streams off by default. |
| 4 | **Proactive Suggestions** | The platform watches captured activity and connected service signals, then surfaces native macOS notifications when something is worth acting on — an insight, an offer to help, or a ready-to-execute action. Accepting a suggestion hands off to Execution Modes. Platform-level only; third-party apps cannot trigger notifications directly. |


---

## Feature Set

### 1. App Ecosystem

Apps are built in Google AI Studio, uploaded to the Cowork.ai desktop app, and rendered inside it. Those apps have access to platform-provided MCPs, which are connected to data from the Activity Capture & Context Engine — giving apps access to work context, activity history, and connected service data.

**Apps get tools, not agents.** Third-party apps access platform capabilities through MCP tools and resources — they do not get direct access to the platform's agents. The platform agent stays as the single orchestrator: it owns reasoning, routing (local vs. cloud brain), budget enforcement, and safety rails. If an app needs agent-level reasoning, the platform exposes it as an MCP tool (agent-as-tool pattern) — the app sees a tool call, the platform runs the agent with full control.

Each installed app shows a compact status card in the sidesheet (unread count, queue depth, next event) and expands to a full view for deep interaction.

**App permissions:**

- Each app declares what MCP tools it needs (like app permissions on a phone). The platform grants scoped access per app.
- Activity context is exposed as read-only MCP resources — apps can read context to be relevant, but cannot write to the activity store.
- Cross-app actions run only when required connections and scopes are available.
- If a required source is disconnected or auth expires, the action pauses and the user is notified with the exact blocker and reconnect action.
- Partial permissions degrade gracefully (read-only summaries allowed; blocked write actions require reconnect/approval).

**Execution Viewer — Full Transparency:**

The Execution Viewer is a dual-pane interface that shows everything the AI is doing — both the API calls happening in the background and the browser actions happening visually.

- **Live browser view** — Watch the AI's browser session in real-time. See it navigate apps, fill forms, compose replies, and submit actions. Controls: Pause, Take Over (switch to your own mouse/keyboard), Resume. The browser view is only active during browser-mode execution.
- **Execution log** — A unified timeline of all AI activity: MCP calls, browser actions, coaching interventions, and outcomes. Every entry shows what happened, when, in which app, and the result.

Example execution log:

```
10:32:04  MCP     zendesk.tickets.get #4521      → read ticket
10:32:06  MCP     ai.draft-reply                 → generated reply
10:32:09  BROWSER navigate zendesk.com/tickets/4521
10:32:12  BROWSER type draft into reply field
10:32:14  USER    edited reply in browser
10:32:18  BROWSER click "Submit as Solved"
```

The Execution Viewer is always accessible. Transparency builds trust; trust enables autonomy.

---

### 2. Execution Modes

Agents use custom tools to execute tasks through two paths:

**Path 1: Agent → Tools → MCP (background)**

- Fast, reliable, runs in the background.
- The agent calls custom tools, which connect to services (Zendesk, Gmail, calendars, etc.) via MCP — no browser involved.
- Best for: data retrieval, bulk operations, well-defined API endpoints, read operations, automated categorization.
- Example: reading 50 Zendesk tickets, categorizing emails, fetching calendar events, checking Linear for existing issues.

**Path 2: Agent → Tools → Browser (visible execution)**

- The agent drives a browser session (Playwright) for visible, coachable actions — while still using custom tools + MCP for data retrieval and background operations.
- Best for: complex multi-field forms, visual verification ("does this reply look right in context?"), apps without full API support, actions the user wants to watch and coach.
- Example: composing a nuanced Zendesk reply in the actual UI, navigating a room booking portal, submitting a multi-step form in an internal tool.

**Exposure to third-party apps:** Custom tools and execution capabilities are exposed to third-party apps (built in Google AI Studio) via platform-provided MCPs. Apps request actions through MCP tools; the platform agent decides how to execute them (local vs. cloud brain, MCP vs. browser) with full safety rails. Apps never invoke agents directly.

**Both paths in one task — a Zendesk ticket reply:**

1. **MCP** — agent reads the ticket content, customer history, and related tickets (fast, background).
2. **MCP** — agent drafts the reply (fast, background).
3. Draft is reviewed and approved by the user.
4. **Browser** — agent opens the Zendesk ticket in the actual UI, pastes the approved reply, and positions for send (visible, coachable).
5. User watches, optionally coaches ("add a note about the refund timeline"), and the AI adjusts in real-time.
6. **Browser** — agent clicks Send. User sees it happen.
7. **MCP** — agent logs the completed action to the audit trail.

**Key principle:** Agents choose the right path for each step. Tools + MCP for data and background work. Browser for visible execution and user-supervised actions. The user configures which path applies per app, per category, or per action — and can override on any individual item.

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

**Retention:** Window/app tracking and focus detection roll X days. Keystroke patterns and browser/screen recordings roll X days. Retention enforcement not yet implemented.

---

### 4. Proactive Suggestions

Activity Capture collects context. Apps query it on demand. But context sitting in a database is inert — nobody benefits until something acts on it. Proactive Suggestions close that gap: the platform watches captured activity and connected service signals, and surfaces a native macOS notification when something is worth your attention.

Each suggestion is an entry point to action. Accepting one hands off to Execution Modes — the notification is the consent handshake before the agent acts.

**Trigger types:**

| Trigger | Example |
|---------|---------|
| **Activity pattern** | You've been on the same Zendesk ticket for 10+ minutes — "Want me to draft a reply?" |
| **Incoming signal** | 3 new high-priority tickets just landed — "Review queue?" |
| **Time-based context** | Your next meeting is in 5 minutes with the same customer from ticket #4521 — "Here's the context." |
| **State transition** | You just switched from Slack to your IDE — "Hold non-urgent notifications?" |
| **Threshold** | Email queue hit 20 unread — "Want me to triage and categorize?" |

**What gets surfaced:**

- **Insight** — "You've handled 12 tickets today, 3 are pending follow-up."
- **Offer** — "Want me to draft a reply to this ticket?"
- **Ready action** — "3 tickets are ready to close — batch-resolve?"
- **Context bridge** — "Your 2pm is with the same customer from ticket #4521 — here's the thread history."

**Notification → Execution handshake:**

1. Platform detects a trigger worth surfacing.
2. Native macOS notification appears with a title, context line, and action button(s).
3. **Accept** → hands off to Execution Modes (MCP path, browser path, or both depending on the action).
4. **Dismiss** → logged as a signal; repeated dismissals of the same trigger type reduce its future frequency.
5. **Expand** → opens the relevant app card in the sidesheet for deeper interaction.

**Throttling & notification fatigue:**

Proactive suggestions are only useful if they don't become noise. The platform enforces:

- **Priority tiers** — Urgent (time-sensitive, high-impact), Helpful (saves effort, not time-critical), Informational (FYI, lowest priority). Each tier has its own delivery rules.
- **Hourly caps** — Urgent: uncapped (these matter). Helpful: capped per hour. Informational: capped lower. Exact thresholds TBD — will be tuned from usage data.
- **Flow-state suppression** — If Activity Capture's focus detection identifies an extended single-app session (deep work), non-urgent suggestions are held and delivered when the user switches context.
- **Dismissal learning** — The platform tracks accept vs. dismiss rates per trigger type. Triggers that are consistently dismissed get deprioritized; triggers that are consistently accepted get promoted.
- **Bundling** — Related suggestions (e.g., "5 tickets are ready to close") are grouped into a single notification, not five separate ones.
- **Cooldown** — After a dismissal, the same trigger type won't re-fire for a configurable cooldown period.

**Platform-level only:**

Third-party apps cannot trigger notifications directly. Apps surface their status in the sidesheet (unread counts, queue depth, next event); the platform decides what crosses the threshold for a native notification. This keeps notification volume under platform control and prevents app spam.

**Privacy:** Suggestions are generated on-device from local activity data. No suggestion content leaves the device. Users can disable suggestions globally, per trigger type, or per connected service.

---

## Privacy & Data Boundaries

Cowork.ai observes your work context to be useful. That observation has to be clearly bounded, or "AI assistant" becomes indistinguishable from "surveillance tool." These are the rules.

### What the AI observes

| Data type | Stored where | Retention |
|-----------|-------------|-----------|
| **Connected app data** (Zendesk, Gmail, Slack) — only apps/channels you explicitly connect | On-device. Cloud inference is transient (zero-retention providers). | Until user disconnects app. |
| **Activity context** (active app, window title, browser URL via Accessibility API) | On-device only. Never leaves device. | Rolling X days. |
| **Keystroke & input capture** (opt-in, raw → communication patterns) | On-device only. | Target: raw discarded within minutes; patterns roll X days. |
| **Browser session data** (agent execution only, not general browsing) | On-device only. | Rolling X days. |
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
| 2026-02-13 | Added fourth feature: Proactive Suggestions — native macOS notifications triggered by activity patterns, incoming signals, time-based context, state transitions, and thresholds. Includes throttling model (priority tiers, hourly caps, flow-state suppression, dismissal learning, bundling). Platform-level only. |
| 2026-02-13 | Scoped down to three core features: App Ecosystem, Execution Modes, Activity Capture. Removed Approval Queue, Clone Mode, Task Automation, Live Agent, Context Card, Autonomy Levels, User Stories. |
| 2026-02-13 | Initial draft. |
