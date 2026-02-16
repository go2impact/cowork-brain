# Cowork.ai Sidecar — Product Features & Capabilities

| | |
|---|---|
| **Status** | Working draft (capability map, not a release scope contract) |
| **Last Updated** | 2026-02-16 |
| **Purpose** | Detailed feature set and capabilities for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries. |


---

## Scope

This document covers **user-facing features and capabilities**. The following are intentionally out of scope here — they live in the architecture and strategy docs:

- **LLM execution architecture** (inference stack and thermal management) → [llm-architecture.md](../architecture/llm-architecture.md)
- **Billing and budget controls** (Cowork Credits, quotas, spend caps, BYOK) → [llm-architecture.md](../architecture/llm-architecture.md) and [llm-strategy.md](../strategy/llm-strategy.md)
- **Memory architecture** (four-layer model, data pipeline, RAG retrieval flow, context management) → [llm-architecture.md](../architecture/llm-architecture.md)

---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI that observes your work, remembers your context, and acts inside your apps — surfacing what matters, answering when you ask, and handling your secondary work while you stay focused on your primary work.

---

## Core Design Principles

**Works inside your tools, not instead of them.** You still use Zendesk, Gmail, Slack. The sidecar sits on top of them (via API) and operates inside them (via browser). Apps built in Google AI Studio are uploaded to the desktop app and given access to your work context through platform-provided MCPs.

**Observes and remembers.** The AI watches what you're working on, embeds and indexes that activity, and builds memory over time. It gets smarter the longer you use it while keeping your context anchored on-device.

**Pushes and pulls.** The platform comes to you when something is worth your attention (Chat — proactive notifications). You come to it when you have a question (Chat — on-demand conversation). Both paths lead to action.

**You are always in control.** The AI works in the open — never behind a curtain. You can intervene at any moment: pause it, correct it, take over, or hand back.

---

## Feature Summary

### Feature Set

| # | Feature | What it does |
|---|---------|-------------|
| 1 | [**Apps**](#1-apps) | Apps built in Google AI Studio are uploaded to the Cowork.ai desktop app and rendered inside it. Apps get tools, not agents — they access platform capabilities through platform-provided MCPs. Browse, search, and install apps from the App Gallery. |
| 2 | [**MCP Integrations**](#2-mcp-integrations) | Connect services (Zendesk, Gmail, Slack, etc.) via MCP. Manage connections, monitor health, configure scopes. The foundation layer that both Apps and Chat use. |
| 3 | [**Chat**](#3-chat) | Two interaction modes: on-demand conversation (you ask, the AI already knows your context) and proactive notifications (the platform surfaces what matters). Both lead to action. |
| 4 | [**MCP Browser**](#4-mcp-browser) | Watch the AI work in real time. See every tool call, every browser action, every outcome. Approval gates for sensitive actions. The trust mechanism that enables autonomy. |
| 5 | [**Automations**](#5-automations) | Rules and workflows that run without intervention. Time-based, event-based, and activity-pattern triggers. Logged and auditable. |
| 6 | [**Context**](#6-context) | Ambient awareness: five active input streams in v0.1 capture what you're working on (screen recording is v0.2). The AI remembers your past conversations, preferences, and how you communicate — all on-device. Context Card shows your current state at a glance. |


---

## Feature Set

### 1. Apps

Apps are built in Google AI Studio, uploaded to the Cowork.ai desktop app, and rendered inside it. Google AI Studio is the on-ramp: someone who's never built an MCP server can use a Cowork template and have a working app in minutes. Those apps have access to platform-provided MCPs, which are connected to data from the Context engine — giving apps access to work context, activity history, and connected service data.

**Apps get tools, not agents.** Third-party apps access platform capabilities through MCP tools and resources — they do not get direct access to the platform's agents. The platform agent stays as the single orchestrator: it owns reasoning, budget enforcement, and safety rails. If an app needs agent-level reasoning, the platform exposes it as an MCP tool (agent-as-tool pattern) — the app sees a tool call, the platform runs the agent with full control.

Each installed app shows a compact status card in the sidesheet (unread count, queue depth, next event) and expands to a full view for deep interaction.

**App permissions:**

- Each app declares what MCP tools it needs (like app permissions on a phone). The platform grants scoped access per app.
- Activity context is exposed as read-only MCP resources — apps can read context to be relevant, but cannot write to the activity store.
- Cross-app actions run only when required connections and scopes are available.

#### App Gallery

Where users find and install new apps. Browse by category, search by name, and install with one click.

- **Categories:** Productivity, Communication, CRM, Developer Tools, Custom/Community.
- **Each listing shows:** Icon, name, description, what MCP servers it connects to, install button.
- **After install:** App appears in sidesheet, begins syncing via MCP.
- **Configuration:** Per-app settings for connected accounts, notification preferences, and MCP scope permissions.

**In the SideSheet:** Each installed app gets a compact card showing its current state. "Browse Apps" link at the bottom of the app list.
**In the Detail Canvas:** Full gallery with categories, search, featured apps, and install flow.

---

### 2. MCP Integrations

MCP Integrations are the pipes to external services. Unlike Apps (which have their own UI and logic), integrations have no standalone interface — they expose data and actions that the platform agent and Apps can both use. Apps and Chat both depend on connected services; MCP Integrations is where you manage those connections.

**What it covers:**

- **Connected services** — Zendesk, Gmail, Slack, Google Calendar, Salesforce, Linear, and more. Each connected via MCP servers.
- **Available tools per service** — list_tickets, send_email, search_channels, etc. Visible so users understand what the AI can access.
- **Connection health** — status indicators, last sync time, auth state.
- **Add new connection** — OAuth flow to connect a new service.

**What happens on disconnect / auth expiry:**

- If a required source is disconnected or auth expires, the action pauses and the user is notified with the exact blocker and reconnect action.
- Partial permissions degrade gracefully (read-only summaries allowed; blocked write actions require reconnect/approval).

**Relationship to Apps:** Apps declare what MCP tools they need. MCP Integrations is where those connections are actually managed — think of it as the plumbing that apps and chat sit on top of.

**In the SideSheet:** Compact card showing connected services count, connection health (all green / X needs attention), and quick link to manage.
**In the Detail Canvas:** Connection management interface with status, tools, and add-new flow.

---

### 3. Chat

Chat is the conversational interface — both when you reach out and when the AI reaches out to you.

#### On-demand conversation

The AI already has your context — from activity capture (what you've been working on), connected services (your tickets, emails, threads), and memory (what it's learned about you over time). You don't copy-paste. You don't explain background. You ask, it knows.

**What it draws from:**

- **Activity data** — what you've been working on, how long, where. Retrieved via RAG from embedded activity data (see [Memory Architecture](../architecture/llm-architecture.md#embeddings-local-rag--memory-architecture)).
- **Connected services** via MCP — Zendesk tickets, emails, Slack threads, calendar events. Queried live.
- **Current-screen context** — active app + window title from activity capture's window tracking. "Summarize this" works because the AI knows what's on your screen.
- **Memory** — past conversations, your preferences, communication patterns, learned context.

**Key behaviors:**

- **Zero-context queries** — "What was that refund ticket I was looking at yesterday?" just works. The AI searches embedded activity data (RAG) to find the matching window title, URL, and timestamp — then pulls the ticket details from Zendesk via MCP.
- **Cross-service search** — "Find where we discussed the API rate limit issue" searches across embedded activity data and connected services simultaneously.
- **Current-screen awareness** — "Summarize this" when you're on a Zendesk ticket. The AI reads window context from activity capture.
- **Entry point from notifications** — accepting or expanding a proactive notification opens a chat thread with that context pre-loaded. You can ask questions and refine before acting.

**Example interactions:**

| You say | What happens |
|---------|-------------|
| "What was that refund ticket I was looking at yesterday?" | RAG search on embedded activity data finds the window title + URL → pulls ticket details from Zendesk via MCP |
| "Summarize my last 3 Slack threads with Sarah" | Queries connected Slack via MCP, summarizes |
| "What have I spent most of my time on today?" | Queries activity capture data, gives time breakdown |
| "Draft a reply to this email — same tone as my last few" | Reads current screen context + email history + communication patterns from memory, drafts in your voice |
| "Find where we discussed the API rate limit issue" | RAG search across embedded activity data + connected services |
| "Batch-close these 4 tickets with a thank-you message" | Chat → action handoff to MCP Browser |

**Platform-level:** One global chat across all connected services and activity data. Apps don't have their own chat interfaces; they surface status in the sidesheet.

#### Proactive notifications

The platform watches captured activity and connected service signals, then surfaces a native macOS notification when something is worth acting on — an insight, an offer to help, or a ready-to-execute action. Accepting a notification hands off to execution via the MCP Browser.

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

**Notification → execution handshake:**

1. Platform detects a trigger worth surfacing.
2. Native macOS notification appears with a title, context line, and action button(s).
3. **Accept** → hands off to execution (MCP path, browser path, or both depending on the action).
4. **Dismiss** → logged as a signal; repeated dismissals of the same trigger type reduce its future frequency.
5. **Expand** → opens a chat thread with that context pre-loaded for deeper interaction.

**Throttling:** Notifications are only useful if they don't become noise. The platform enforces priority tiers (urgent, helpful, informational — each with its own delivery rules), hourly caps per tier (urgent uncapped; helpful and informational capped, thresholds TBD from usage data), flow-state suppression (deep work detected → non-urgent notifications held until context switch), dismissal learning (consistently dismissed triggers get deprioritized), bundling (related notifications grouped into one), and cooldown periods after dismissal.

**Platform-level only:** Third-party apps cannot trigger notifications directly. Apps surface their status in the sidesheet (unread counts, queue depth, next event); the platform decides what crosses the threshold for a native notification. This keeps notification volume under platform control.

**Privacy:** Notifications are generated on-device from local activity data. No notification content leaves the device. Users can disable notifications globally, per trigger type, or per connected service.

#### Chat → action bridge

Conversation naturally escalates to execution. The AI drafts a reply → proactively suggests sending → hands off to the MCP Browser. The AI offers agentic next steps ("Want me to send this?" / "Should I batch-close these?") in addition to responding to direct requests.

---

### 4. MCP Browser

The MCP Browser is the trust mechanism. It shows everything the AI is doing — both the API calls happening in the background and the browser actions happening visually. Transparency builds trust; trust enables autonomy.

#### What you see

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

#### How execution works

Agents use custom tools to execute tasks through two paths:

**Path 1: Agent → Tools → MCP (background)**

- Fast, reliable, runs in the background.
- The agent calls custom tools, which connect to services (Zendesk, Gmail, calendars, etc.) via MCP — no browser involved.
- Best for: data retrieval, bulk operations, well-defined API endpoints, read operations, automated categorization.
- Example: reading 50 Zendesk tickets, categorizing emails, fetching calendar events.

**Path 2: Agent → Tools → Browser (visible execution)**

- The agent drives a browser session (Playwright) for visible, coachable actions — while still using custom tools + MCP for data retrieval and background operations.
- Best for: complex multi-field forms, visual verification ("does this reply look right in context?"), apps without full API support, actions the user wants to watch and coach.
- Example: composing a nuanced Zendesk reply in the actual UI, navigating a room booking portal, submitting a multi-step form.

**Both paths in one task — a Zendesk ticket reply:**

1. **MCP** — agent reads the ticket content, customer history, and related tickets (fast, background).
2. **MCP** — agent drafts the reply (fast, background).
3. Draft is reviewed and approved by the user.
4. **Browser** — agent opens the Zendesk ticket in the actual UI, pastes the approved reply, and positions for send (visible, coachable).
5. User watches, optionally coaches ("add a note about the refund timeline"), and the AI adjusts in real-time.
6. **Browser** — agent clicks Send. User sees it happen.
7. **MCP** — agent logs the completed action to the audit trail.

**Approval gates:** Sensitive actions require explicit user approval before execution. Configurable per app, per category, or per action — the user controls how much autonomy the AI gets.

**Key principle:** Agents choose the right path for each step. The user configures which path applies per app, per category, or per action — and can override on any individual item.

**Exposure to third-party apps:** Custom tools and execution capabilities are exposed to apps via platform-provided MCPs. Apps request actions through MCP tools; the platform agent decides how to execute them (MCP path, browser path, or both) with full safety rails. Apps never invoke agents directly.

---

### 5. Automations

Rules and workflows that run without user intervention for routine steps. Automations trigger across both Apps and MCP Integrations — any action available to the platform agent can be automated. Safety rails still apply: destructive actions (deleting tickets, sending emails) and money actions require approval even inside an automation. The automation handles everything up to the gate, then notifies the user for the final call.

**Examples:**

- "When a high-priority Zendesk ticket arrives, draft a response and notify me."
- "Every morning at 9am, summarize my unread emails and Slack messages."
- "When I start a Zoom call, mute Slack notifications."

**Trigger types:**

- **Time-based** — scheduled at specific times or intervals (daily digest, weekly report).
- **Event-based** — fired by incoming signals (new ticket, email received, Slack mention).
- **Activity-pattern** — triggered by observed behavior (deep work session started, context switch detected).

**Execution logs:** Every automation run is logged with what triggered it, what actions were taken, and the outcome. Users can review, pause, edit, or delete any automation.

**In the SideSheet:** "Active Automations" card showing what's running.
**In the Detail Canvas:** Automation builder/editor and execution logs.

---

### 6. Context

Context is the AI's ambient awareness — what you're working on, what it remembers about you, and what it knows right now.

#### What the AI observes

Five input streams in v0.1 provide the AI with work context. Screen recording is deferred to v0.2:

| Stream | Default | Used for |
|--------|---------|----------|
| **Window & app tracking** (active app, window title, browser URL via macOS Accessibility API) | On | Work context |
| **Keystroke & input capture** (raw input → communication patterns) | Off (opt-in) | Communication pattern extraction |
| **Focus detection** (extended single-app sessions, derived from window tracking) | On | Flow-state detection |
| **Browser activity** (pages visited, actions taken during agent sessions) | On during sessions | MCP Browser, audit trail |
| **Screen recording** (15fps during coached agent sessions) | Not in v0.1 (v0.2) | Agent coaching (v0.2) |
| **Clipboard monitoring** (text on copy/paste) | Off (opt-in) | Context enrichment |

**Data flow:** Raw capture → local SQLite (libsql, WAL mode) → local processing → structured context (embeddings via local RAG) → agents query at action time. All on-device.

**Retention:** Window/app tracking and focus detection roll X days. Keystroke patterns and browser session data roll X days. Retention enforcement not yet implemented.

#### What the AI remembers

The AI remembers your past conversations, your preferences, how you communicate, and what you've been working on. It gets smarter the longer you use it — stored on-device and continuously indexed so the AI can recall relevant context when you ask a question or when it generates a suggestion.

For the full technical architecture (four-layer memory model, data pipeline, RAG retrieval flow), see [Memory Architecture](../architecture/llm-architecture.md#embeddings-local-rag--memory-architecture).

#### Context Card

The Context Card shows what you're working on right now — a quick-glance summary in the sidesheet.

**What it shows:**

- What you're working on right now (active app, active document, active meeting).
- How long you've been in this context.
- Relevant suggestions ("You have a meeting in 15 minutes" / "3 unread Slack messages in #engineering").

**What it does NOT show:**

- "Focus score" with a gamified number.
- "Distractions" count.
- "Productive vs unproductive" categorization.
- Anything that frames the worker as a subject of measurement.

The distinction matters: "You've been in VS Code for 2 hours working on the auth module" is useful context. "Your focus score is 87, you're in the top 10%" is gamified surveillance. We do the first one.

#### Privacy

All context data is on-device. Users can view, edit, or delete any captured data or memory. Delete = gone — no hidden copies, no cloud backup.

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
| **Conversation memory** (chat history, working memory profile, observational logs) | On-device only. Never leaves device. | Chat history: session. Working memory: until user edits/deletes. Observations: long-term, compressed. |
| **Semantic embeddings** (vector representations of activity data + conversations) | On-device only. Never leaves device. | Long-term; pruned per retention policy. |
| **AI profiles** (tone, rules, SOPs) | On-device only. | Until user deletes. |
| **Action history** (audit log of all AI actions) | On-device only. | Metadata persists; content purged on app disconnect. |

### What the AI never collects

- **Continuous screen recording** — not in v0.1. If introduced in v0.2, screen capture remains opt-in, agent-session-only, not ambient surveillance.
- **Keystroke data for employer reporting** — keystroke capture trains your AI, never reported to anyone else. Never leaves device.
- **Full page content outside agent sessions** — activity context captures window titles and URLs for work context, not page content or DOM data. General browsing is not indexed.
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
| 2026-02-17 | Clarified Context scope for v0.1: screen recording moved out of v0.1 and reserved for v0.2. Updated “What the AI observes” defaults, retention text, and “What the AI never collects” wording to reflect no screen capture in v0.1. |
| 2026-02-16 | Major restructure per CEO feedback on PR #6. Six features: Apps (with App Gallery), MCP Integrations, Chat (merges On-Demand Chat + Proactive Suggestions), MCP Browser (merges Execution Viewer + Execution Modes), Automations (new), Context (merges Activity Capture + Context Card + user-facing Memory summary). Technical memory architecture moved to llm-architecture.md. |
| 2026-02-13 | Added features 5–6: On-Demand Chat (zero-context conversational interface, chat → action bridge, entry point from Proactive Suggestions) and Memory (four-layer system — conversation history, working memory, semantic recall via RAG, observational memory — covering both activity capture data and conversations). Updated Privacy section with memory data types. |
| 2026-02-13 | Added fourth feature: Proactive Suggestions — native macOS notifications triggered by activity patterns, incoming signals, time-based context, state transitions, and thresholds. Includes throttling model (priority tiers, hourly caps, flow-state suppression, dismissal learning, bundling). Platform-level only. |
| 2026-02-13 | Scoped down to three core features: App Ecosystem, Execution Modes, Activity Capture. Removed Approval Queue, Clone Mode, Task Automation, Live Agent, Context Card, Autonomy Levels, User Stories. |
| 2026-02-13 | Initial draft. |
