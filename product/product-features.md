# Cowork.ai Sidecar — Product Features & Capabilities

**Status:** Working draft (capability map, not a release scope contract)
**Last Updated:** 2026-02-13
**Purpose:** Detailed feature set, capabilities, and user stories for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries.
**Scope Note:** This document defines the full product concept across phases. Current release scope is tracked in [`product-overview.md`](./product-overview.md) and implementation sequencing in [`../architecture/llm-architecture.md`](../architecture/llm-architecture.md).

**First wedge:** CX agent + Zendesk Approval Queue + Clone Mode. One persona, one app, one loop. If this combination doesn't deliver 80%+ approve-without-editing within two weeks, nothing else in this document matters.

---

## Sources

Synthesized from: [`product-overview.md`](./product-overview.md), [`design-system.md`](../design/design-system.md), [`prototype-brief.md`](../design/prototype-brief.md), [Issue #3](https://github.com/go2impact/cowork-brain/issues/3) + [Scott's positioning guardrails](https://github.com/go2impact/cowork-brain/issues/3#issuecomment-3891009312), and direct product input from Rustan (Approval Queue, Live Agent, Task Automation, Clone Mode, hybrid execution model, second-brain thesis). User stories and "Success Looks Like" were composed from multiple sources.

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
| 1 | **Shadow Work Manager (Approval Queue)** | Three-stage pipeline: Monitor & Triage → Draft & Queue → Execute & Watch. A single prioritized feed across all connected apps. Five user actions: Approve, Edit & Approve, Dismiss, Snooze, Reject & Teach. Sources: Zendesk, Gmail, Slack, Calendar. Prioritization by urgency/importance, not arrival time. |
| 2 | **Clone Mode (AI Personality Profiles)** | Captures tone/voice, response rules, SOPs, personal preferences. Supports scoped profiles (per executive, per client, per channel, per role). Auto-selects profile based on context. Learning via onboarding, Teach feedback, history bootstrap (opt-in), and keystroke-informed training (opt-in). Shadow mode for first ~20 interactions. |
| 3 | **Live Agent (Voice & Visual Interface)** | Voice commands, conversational context, visual coaching during browser execution, take-over/hand-back, hands-free mode. Voice maps to the same action primitives as the queue. Phased: v0.x uses streaming STT+LLM+TTS, native duplex audio is future roadmap. |
| 4 | **Task Automation & Focus Mode** | Focus Mode — notification gatekeeper during deep work, emergency breakthrough, end-of-session summary. Background Agents — Inbox Zero, Ticket Triage, Slack Digest, Calendar Guardian. Custom Automations — user-defined rules for role-specific workflows. |
| 5 | **App Ecosystem & Execution Viewer** | App Gallery — pre-built integrations (Slack, Gmail, Zendesk, Salesforce, etc.) + custom apps via MCP. App-as-component model — each app = component + MCP connection + AI integration. Execution Viewer — dual-pane (live browser view + execution log), full audit trail. |
| 6 | **Execution Modes (Hybrid Model)** | MCP (API, fast, background) + Browser/Playwright (visible, coachable). Work together on complex tasks. Configurable per app/category/action. |
| 7 | **Activity Capture & Context Engine** | Six input streams: window/app tracking, keystroke/input capture, focus detection, browser activity, screen recording, clipboard monitoring. Local SQLite storage. Feeds Clone Mode, Context Card, and Focus Mode. Most streams off by default. |
| 8 | **Context Card** | Plain-language summary of current work state. Shows queue status, unread messages, next meeting, agent activity. Draws from MCP app data + activity context + live agent status. |

### Cross-Cutting Sections

| Section | What it covers |
|---------|---------------|
| **Autonomy Levels** | External = approval required, internal = can be autonomous. Promotion/demotion of categories. Execution mode preference (API vs browser). |
| **Guardrails** | Non-promotable categories (financial, first-contact, destructive). Kill switch, rate limiting, best-effort recall. |
| **Privacy & Data Boundaries** | What's collected, where it's stored, retention defaults, what's never collected, who sees what, surveillance-tool differentiation. |
| **User Stories** | CX Agent, Virtual Assistant, Operations Specialist, Dev Support Engineer. |
| **"What Cowork.ai Is NOT"** | Not a time tracker, not a chatbot, not employer monitoring, not a tool replacement. |

---

## Feature Set

### 1. Shadow Work Manager (Approval Queue)

Shadow work is everything that isn't your actual job but eats your day: responding to Slack threads, triaging email, replying to low-priority support tickets, confirming calendar invites. The Approval Queue makes this work disappear.

**How it works:**

The pipeline has three stages:

1. **Monitor & Triage** — The AI watches your connected channels (Slack, Gmail, Zendesk — and eventually Calendar, CRM, and others) via read-only MCP access. It identifies items that need a response or action from you, categorizes them, and surfaces them in the sidecar.
2. **Draft & Queue** — For items it can handle, the AI drafts a response using your **Clone Mode** profile (see below) — matching your tone, following your rules, applying your SOPs. Drafts land in the Approval Queue: a single, prioritized feed inside the sidecar.
3. **Execute & Watch** — For approved items, the AI executes the action. Depending on your configuration, execution happens via API (instant, background) or via browser automation (visible, coachable). In browser mode, you watch the AI work in real-time and can coach or intervene mid-execution.

Stages 1 and 2 are separable. In early use or with a new app connection, the AI may surface items for triage before it's confident enough to draft. Stage 3 execution mode is configurable per app and per category.

- You review each item and take one action:

  | Action | What happens | When to use it |
  |--------|-------------|----------------|
  | **Approve** | Send the draft. Executes via API (instant) or via browser (visible) based on your per-app/per-category preference. One tap. | The AI nailed it. |
  | **Edit & Approve** | Tweak the draft, then send. Same execution mode options as Approve. | Close but needs a small fix. |
  | **Dismiss** | Remove the item from the queue. Nothing is sent. | This doesn't need a reply, or you'll handle it manually. |
  | **Snooze** | Move the item out of the queue temporarily. Returns after a set time (30 min, 1 hour, end of day, custom). Snoozed items return at their recalculated priority position (not pinned to the top). On return, the AI checks for staleness — if the underlying item was resolved while snoozed (e.g., someone else replied), the item is auto-dismissed with a note in the audit log. | Not now — but don't let me forget. |
  | **Reject & Teach** | Discard the draft *and* tell the AI what it got wrong. The AI updates the relevant Clone Mode profile so it handles similar items better next time. | The draft is wrong and the AI should learn from the mistake. |

**What shows up in the queue:**

| Source | Example Items |
|--------|---------------|
| Zendesk | Drafted ticket replies, suggested status changes, escalation recommendations |
| Gmail | Drafted email responses, meeting confirmations, follow-up reminders |
| Slack | Suggested thread replies, channel summaries with proposed responses, DM drafts |
| Calendar | Scheduling conflict resolutions, meeting prep summaries, availability responses |

**Key behaviors:**

- Items are ranked by urgency and importance, not by arrival time.
- The queue consolidates across all channels — you don't context-switch between apps to catch up.
- External-facing actions (sending a customer reply, emailing a client, responding to a DM) always require your explicit approval by default. The AI drafts; you decide.
- Internal and organizational actions (archiving low-priority email, categorizing tickets, preparing meeting briefs) can be pre-authorized — see [Autonomy Levels](#autonomy-levels--the-approval-boundary) below.
- All actions — approved or autonomous — are logged so you can audit what was sent, what was handled, and when.
- **Watch-live mode** is opt-in and configurable per app. When enabled, approved actions execute in the browser with a live view. When disabled, actions execute via API in the background. You can change this at any time.

**Prioritization principles (concept-level):**

- User-defined rules outrank system defaults.
- Time-critical work outranks non-urgent work (SLA risk, meeting conflicts, overdue follow-ups).
- Relationship-critical sources outrank general sources (manager/client/escalation contacts before FYI channels).
- Business-impact work outranks low-impact work (revenue, retention, incident severity).
- Recency is a tie-breaker, not the primary rank signal.

---

### 2. Clone Mode — Your AI Personality Profiles

Clone Mode is how the AI learns to sound like *you* — or like you acting on behalf of someone else. It's the difference between generic AI drafts and responses your coworkers can't distinguish from ones you actually wrote.

**What a Clone Mode profile captures:**

- **Tone & voice** — Formal vs. casual, emoji usage, greeting/sign-off patterns, language preferences.
- **Response rules** — "Always CC my manager on escalations." "Never promise a timeline without checking with engineering." "Use the customer's first name."
- **SOPs & playbooks** — Standard operating procedures for the context. How to handle refund requests. How to triage P1 vs. P2 tickets. What qualifies as an escalation.
- **Personal preferences** — "I don't schedule meetings before 10am." "Always suggest async over a call." "Use bullet points, not paragraphs."

**Profile scoping:**

You always have a **default profile** — this is you. Your tone, your rules, your preferences. For workers who operate in a single context (one client, one role), this is all they need.

But many workers operate across multiple contexts: a VA managing three executives, a CX agent handling tickets for two different clients, a support lead who switches between internal and customer-facing communication. Clone Mode supports **scoped profiles** for this.

| Scope type | Example | What changes per profile |
|------------|---------|------------------------|
| Per executive / principal | VA drafts as Executive A vs. Executive B | Tone, sign-off, scheduling rules, CC preferences |
| Per client / account | CX agent handles Client X vs. Client Y | SOPs, escalation paths, response templates, formality level |
| Per channel | Slack #engineering vs. Slack #client-updates | Tone, detail level, emoji usage |
| Per role | Support agent vs. team lead in the same tools | Authority level, what they can promise, who they escalate to |

**How profiles activate:**

- The AI selects the right profile based on context: which app, which channel, which contact, which account. A Zendesk ticket tagged "Client X" uses the Client X profile. An email to Executive A uses Executive A's profile.
- If the context is ambiguous, the Approval Queue flags which profile was used — you can correct it via Teach.
- You can also pin a profile manually: "Use Executive B's profile for the next hour."

**How profiles are created:**

- The default profile is built during onboarding.
- Additional profiles can be created from scratch ("Add a profile for Executive A") or forked from the default ("Clone my profile but make it more formal and add these scheduling rules").
- Each profile learns independently — teaching the AI on a Client X ticket updates the Client X profile, not your default.

**How it learns:**

- Starts with a guided onboarding: you answer questions about your role, communication style, and key rules. This creates your default profile.
- Improves every time you **Teach** from the Approval Queue — corrections and overrides update the relevant scoped profile (or the default if no scope applies).
- You can directly edit any profile's settings: add rules, update SOPs, adjust tone.
- **Activity-informed training (opt-in):** With explicit consent, keystroke capture can feed observed communication patterns — common phrases, typical sentence structure, greeting/sign-off habits — into Clone Mode profiles. This is supplemental to the explicit Teach mechanic, not a replacement. Keystroke capture is off by default, requires its own consent dialog, and processes patterns (not raw keystrokes) into the profile. See [Activity Capture & Context Engine](#7-activity-capture--context-engine) for details.

**Cold-start:** Onboarding creates a usable default profile in ~5 minutes. Optional history bootstrap (opt-in) analyzes recent sent messages to accelerate profile quality. For the first ~20 interactions, drafts are flagged as "learning drafts" with prominent Teach action. Target: 80%+ approve-without-editing within 1–2 weeks for routine items. Profiles are stored locally and can be exported/imported for device portability.

---

### 3. Live Agent — Voice & Visual Interface (Phased)

The Live Agent is a real-time interface for directing the AI through voice and visual coaching. Instead of typing, you talk — and the AI talks back. When the AI works in the browser, you watch and coach in real-time. Built for moments when your hands are busy, when speaking is faster than typing, or when you want to supervise the AI's work as it happens.

**Capabilities:**

- **Voice commands** — "What's in my queue?" "Draft a reply to that Zendesk ticket." "Summarize my unread Slack messages." "Block my calendar for the next hour."
- **Conversational context** — The Live Agent knows what you're looking at. If you're reviewing a ticket in the Approval Queue, you can say "make this more formal" or "add an apology" without specifying which item.
- **Visual coaching** — Watch the AI work in the browser and direct it in real-time. "Make that more formal." "Don't send that — let me rephrase." "Go back." "Skip that field." The AI adjusts instantly, like a colleague who takes direction well.
- **Take over / hand back** — At any point during browser execution, you can take over the browser directly (mouse and keyboard), make changes yourself, and then hand control back to the AI. The AI picks up where you left off, incorporating your changes.
- **Real-time responses (v0.x)** — Low-latency voice interaction via streaming speech-to-text + LLM + speech output. This delivers practical voice-first workflows without waiting for full native duplex audio.
- **Native duplex roadmap** — Full native audio (simultaneous speech-in/speech-out) is a future upgrade once reliability, cost, and device support meet production targets.
- **Hands-free mode** — Activate via wake word or hotkey. Useful during calls, while multitasking, or for accessibility.

**How voice maps to the system:**

Voice commands are not a separate action system — they invoke the same primitives used by the queue, browser execution, and Execution Viewer. "Approve that draft" triggers the same Approve action as tapping the button. "Draft a reply to that ticket" creates a queue item via the same drafting pipeline. Coaching commands during browser execution ("make that more formal", "stop", "go back") invoke the same correction primitives as editing a draft in the queue. All voice-initiated actions appear in the Execution Viewer audit log identically to manual actions, tagged with `source: voice` for traceability.

**When to use it:**

- Morning briefing: "What do I need to handle today?"
- Mid-task: "What did Sarah say in that Slack thread?"
- Queue actions: "Approve that draft" / "Dismiss this one" / "Snooze it for an hour"
- End of day: "Anything I missed?"
- Coaching during browser execution: "That reply is too casual — make it more professional." "Add the refund amount." "Actually, skip this ticket — I'll handle it."
- Supervising complex tasks: "Walk me through what you're doing." "Why did you choose that template?"

---

### 4. Task Automation & Focus Mode

Focus Mode is how the sidecar protects your deep work time. When you're in focus, the AI becomes your gatekeeper.

**Focus Mode:**

- Activate manually ("I'm focusing for the next 2 hours") or automatically (the AI detects you've been in a single app context for an extended period and suggests activating Focus Mode — see [Privacy & Data Boundaries](#privacy--data-boundaries) for what "activity context" means and doesn't mean). If you've disabled activity context collection (see [How this is different from surveillance tools](#how-this-is-different-from-surveillance-tools)), auto-activation is unavailable — Focus Mode can only be activated manually or on a schedule. The feature degrades gracefully; it doesn't silently stop working.
- Non-urgent notifications are intercepted and held. The AI triages incoming items and only breaks through for genuine emergencies (P1 tickets, messages from your manager, calendar conflicts).
- Breakthrough decisions use the same prioritization principles as the Approval Queue so interruption behavior stays predictable.
- When Focus Mode ends, you get a summary: "While you were focused, I handled 12 items. 3 need your review."

**Background Agents:**

Persistent automation agents that run continuously without manual triggers ("continuously" means the agents check for new items on a regular cadence — typically polling connected apps every few minutes via MCP, not maintaining persistent open connections. The polling interval is configurable per app. When webhook support is available from the connected service, the agent reacts to push events instead of polling, reducing latency and resource usage). Each agent uses MCP and browser automation as appropriate for the task:

- **Inbox Zero Agent** — Continuously processes your email: categorizes and archives low-priority (autonomous via MCP), drafts responses for items that need one (queued for approval). When approved, replies are sent via API (instant) or composed in the browser (visible) based on your preference. Goal: when you open your inbox, it's already organized.
- **Ticket Triage Agent** — Monitors your support queue: auto-categorizes by type and urgency via MCP (autonomous), drafts first responses for common patterns (queued for approval), and escalates anything outside your SOP. For complex ticket forms that require multi-field submissions or internal tool navigation, the agent uses browser automation with user supervision.
- **Slack Digest Agent** — Watches your channels via MCP, filters signal from noise, and delivers periodic summaries instead of constant pings (autonomous). Knows which channels matter to you and which are FYI-only. Browser fallback for workspaces with restricted API scopes.
- **Calendar Guardian Agent** — Prepares meeting briefs before each call (autonomous via MCP), flags scheduling conflicts for review, and auto-declines low-priority meetings that conflict with focus blocks (autonomous — user can demote to approval-required). For complex scheduling UIs (e.g., room booking portals, external scheduling tools), the agent uses browser automation.

**Custom Automations:**

Background Agents are opinionated presets that ship with the product — they work out of the box with sensible defaults and learn from your Clone Mode profile. Custom Automations are user-defined rules that you create from scratch for workflows specific to your role. Custom automations can include both MCP steps and browser steps:

- "When a ticket is tagged 'billing', always check the customer's payment history before drafting."
- "Every Friday at 4pm, compile a summary of everything I handled this week."
- "If a Slack DM from my manager goes unanswered for 30 minutes, escalate to my phone."
- "When I approve a weekly report, open the reporting portal in the browser and submit it — I'll watch."

---

### 5. App Ecosystem & Execution Viewer

**App Gallery:**

A library of pre-built integrations. Install the apps that match your workflow:

- **Communication:** Slack, Gmail, Microsoft Teams
- **Support:** Zendesk, Freshdesk, Intercom
- **CRM:** Salesforce, HubSpot
- **Project Management:** Linear, Asana, Jira
- **Calendar:** Google Calendar, Outlook
- **Custom:** Build your own app components with custom MCP connections for internal tools — same architecture as built-in apps (see below)

**What an app actually is (developer mental model):**

An app is a **component with built-in MCP connectivity**. The developer mental model: import a component, render it inside the sidecar, and it can talk to MCP servers out of the box.

Every app in the gallery — Zendesk, Gmail, Slack, and the rest — is built this way. Custom apps are built the same way. The component handles three things:

- **Rendering** — compact status card in the sidesheet, full view in the detail canvas.
- **MCP connection** — data retrieval and action execution against the app's backend.
- **AI integration** — the sidecar's agents can read from and act through any installed app's MCP surface.

Building a custom app means building a component that declares which MCP servers it connects to. The sidecar handles auth, rendering context, and AI access. There is no separate "integration layer" — the app *is* the integration.

Each installed app shows a compact status card in the sidesheet (unread count, queue depth, next event) and expands to a full view for deep interaction.

**Cross-app boundaries (concept-level):**

- Cross-app automations run only when required connections and scopes are available.
- If a required source is disconnected or auth expires, the automation pauses and creates a queue item with the exact blocker and reconnect action.
- Partial permissions degrade gracefully (read-only summaries allowed; blocked write actions require reconnect/approval).
- The Execution Viewer must show source provenance for every cross-app action: what was read, what failed, and what was skipped.

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

### 6. Execution Modes

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
2. **MCP** invokes Clone Mode to draft the reply (fast, background).
3. Draft lands in the Approval Queue. User reviews and approves.
4. **Browser** opens the Zendesk ticket in the actual UI, pastes the approved reply, and positions for send (visible, coachable).
5. User watches, optionally coaches ("add a note about the refund timeline"), and the AI adjusts in real-time.
6. **Browser** clicks Send. User sees it happen.
7. **MCP** logs the completed action to the audit trail.

**Key principle:** MCP handles data and background work. The browser handles visible execution and user-supervised actions. The user configures which mode applies per app, per category, or per action — and can override on any individual item.

---

### 7. Activity Capture & Context Engine

The AI needs context to be useful. Activity capture provides that context — what you're working on, how you communicate, where you spend your time. This data feeds the Context Card, Clone Mode, and the AI's ability to act relevantly.

**Input streams:**

| Stream | Default | Used for |
|--------|---------|----------|
| **Window & app tracking** (active app, window title, browser URL via macOS Accessibility API) | On | Context Card, Focus Mode |
| **Keystroke & input capture** (raw input → communication patterns for Clone Mode) | Off (opt-in) | Clone Mode training |
| **Focus detection** (extended single-app sessions, derived from window tracking) | On | Focus Mode auto-activation |
| **Browser activity** (pages visited, actions taken during agent sessions) | On during sessions | Execution Viewer, audit trail |
| **Screen recording** (15fps during coached agent sessions) | Off (opt-in) | Agent coaching |
| **Clipboard monitoring** (text on copy/paste) | Off (opt-in) | Context enrichment |

**Data flow:** Raw capture → local SQLite (GRDB, WAL mode) → local processing → structured context (embeddings via local RAG) → agents query at action time. All on-device. Processing pipeline that discards raw input is not yet implemented — raw data currently persists locally.

**Retention:** Window/app tracking and focus detection roll 7 days. Keystroke patterns and browser/screen recordings roll 30 days. Retention enforcement not yet implemented.

---

### 8. Context Card — Your Work at a Glance

The Context Card is what you see when you open the sidesheet. It's not a focus score. It's not a productivity metric. It's a plain-language summary of your current work state:

- **"You have 4 tickets in queue, 2 are drafted and ready to send."**
- **"3 unread Slack DMs — one from your manager."**
- **"Next meeting in 45 minutes. Prep brief is ready."**
- **"Inbox Zero agent cleared 18 emails. 2 need your input."**
- **"AI is replying to Zendesk #4521 — [watch live](#5-app-ecosystem--execution-viewer)."**

The Context Card answers one question: *What should I do next?*

**Where Context Card data comes from:**

The Context Card draws from three sources: (1) **connected app data** via MCP — ticket counts, unread messages, calendar events; (2) **activity context** from the [Activity Capture & Context Engine](#7-activity-capture--context-engine) — the active app name, window title, focused UI element, browser URL, focus detection, and time-in-app data (all via macOS Accessibility API); and (3) **live agent execution status** — when the AI is actively working in the browser, the Context Card shows a live status indicator with a link to the Execution Viewer. Activity context is processed and stored locally and never leaves your device. See [Privacy & Data Boundaries](#privacy--data-boundaries) for full details.

---

## Autonomy Levels & The Approval Boundary

The features above describe two different modes of AI action: drafting for approval (Approval Queue) and acting autonomously (Background Agents, Custom Automations, Focus Mode). This isn't a contradiction — it's a spectrum that the user controls.

### The Default Rule

**External-facing actions require approval. Internal actions can be autonomous.**

| Action Type | Default | Examples |
|-------------|---------|----------|
| Send a message to a customer | Requires approval | Zendesk ticket reply, client email, support chat response |
| Send a message to a coworker | Requires approval | Slack DM, email to teammate |
| Organize your own workspace | Autonomous | Archive email, categorize tickets, prepare meeting briefs, compile summaries |
| Block or redirect your own time | Autonomous | Auto-decline conflicting meetings, hold notifications during Focus Mode |
| Escalate to you personally | Autonomous | Push notification to your phone, break through Focus Mode for emergencies |

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

- **Promote from queue to auto-send.** After approving 50 standard Zendesk status updates without editing, you might tell the AI: "Handle routine status updates automatically. Just log them." That category leaves the Approval Queue and becomes a Background Agent behavior. (Note: some categories — financial commitments, first-contact messages, destructive actions — cannot be promoted. See Guardrails below.)
- **Demote from auto to queue.** If the Calendar Guardian auto-declines a meeting you actually wanted, you can pull calendar actions back into the Approval Queue: "Always ask me before declining meetings."
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

**Kill switch:** A global "pause all autonomous actions" control, accessible from the Context Card and via voice command ("stop everything"). When activated, all pending and queued autonomous actions are halted. Actions already submitted to an external platform (API call sent, message delivered) cannot be recalled from the kill switch — use platform-level undo where available (see Recall below). Unpause resumes from where it stopped — nothing is auto-retried without your confirmation.

**Rate limiting:** Autonomous actions are capped per hour per app (configurable, sensible defaults TBD during v0.1 validation). If the cap is hit, remaining actions queue for approval and the user is notified immediately. This prevents a drifted Clone Mode profile or stale SOP from generating unchecked volume.

**Recall (best-effort):** For platforms that support undo or recall (Gmail undo-send window, Slack message edit window), the Execution Viewer offers a time-limited recall button after autonomous sends. For platforms without recall support, the action is logged but not reversible — this limitation is disclosed when the user promotes a category to autonomous.

---

## User Story — First Wedge

### As a CX Agent (Zendesk + Slack)

> I handle 40–60 Zendesk tickets per day. Half are routine: password resets, billing questions, feature requests I've answered a hundred times.
>
> With Cowork.ai, I open my sidecar in the morning and the Approval Queue has 15 drafted replies ready. I approve 12 via API — instant, gone. For the 2 that need tweaks, I edit and approve. For 1 tricky escalation, I switch to browser mode: the AI opens the ticket in Zendesk, types the reply I approved, and I watch. Halfway through, I say "add a note that we're waiving the fee as a one-time courtesy." The AI adjusts on the spot, I confirm, and it sends. That took 10 minutes instead of 45.
>
> When I go heads-down on a complex escalation, Focus Mode holds my Slack pings. When I come up for air, I get a summary instead of 47 unread messages.

*Additional user stories (Virtual Assistant, Operations Specialist, Dev Support Engineer) are available in earlier revisions of this document.*

---

## Privacy & Data Boundaries

Cowork.ai observes your work context to be useful. That observation has to be clearly bounded, or "AI assistant" becomes indistinguishable from "surveillance tool." These are the rules.

### What the AI observes

| Data type | Stored where | Retention |
|-----------|-------------|-----------|
| **Connected app data** (Zendesk, Gmail, Slack) — only apps/channels you explicitly connect | On-device. Cloud inference is transient (zero-retention providers). | Until user disconnects app. |
| **Activity context** (active app, window title, browser URL via Accessibility API) | On-device only. Never leaves device. | Rolling 7 days. |
| **Keystroke & input capture** (opt-in, raw → patterns for Clone Mode) | On-device only. | Target: raw discarded within minutes; patterns roll 30 days. |
| **Browser session data** (agent execution only, not general browsing) | On-device only. | Rolling 30 days. |
| **Clone Mode profiles** | On-device only. | Until user deletes. |
| **Action history** (audit log of all AI actions) | On-device only. | Metadata persists; content purged on app disconnect. |

### What the AI never collects

- **Continuous screen recording** — screen capture is opt-in, agent-session-only, not ambient surveillance.
- **Keystroke data for employer reporting** — keystroke capture trains your Clone Mode, never reported to anyone else. Never leaves device.
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
- Your Clone Mode drafts are so good that you approve 80%+ without editing.
- You talk to the Live Agent and it does what you asked on the first try.
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
| 2026-02-13 | Initial draft: Approval Queue, Clone Mode, Live Agent, Focus Mode, App Ecosystem, Context Card. |
