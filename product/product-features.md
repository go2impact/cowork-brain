# Cowork.ai Sidecar — Product Features & Capabilities

**Version:** Concept 0.6 (capability map, not a release scope contract)
**Last Updated:** 2026-02-13
**Purpose:** Detailed feature set, capabilities, and user stories for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries.
**Scope Note:** This document defines the full product concept across phases. Current release scope is tracked in [`product-overview.md`](./product-overview.md) and implementation sequencing in [`../architecture/llm-architecture.md`](../architecture/llm-architecture.md).

**First wedge:** CX agent + Zendesk Approval Queue + Clone Mode. One persona, one app, one loop. If this combination doesn't deliver 80%+ approve-without-editing within two weeks, nothing else in this document matters.

---

## How This Document Was Created

This document was synthesized from multiple sources: repo docs, GitHub discussions, and direct product input from the founder. Everything here traces back to an identified source — written documents, GitHub threads, or explicit verbal/written input from Rustan. It's a consolidation of existing thinking into one coherent product features spec.

**From this repo:**
- [`product/product-overview.md`](./product-overview.md) — existing product positioning, two-brain architecture description, target users, and roadmap. Provided the foundation for who the product is for and the overall product thesis.
- [`design/design-system.md`](../design/design-system.md) — the correct feature set (Chat, Apps, App Gallery, MCP Browser, Automations, Context Card), the three-state interaction model (Closed / SideSheet / Detail Canvas), and the "NOT a focus score" positioning. This defined the feature categories and UI surface model.
- [`design/prototype-brief.md`](../design/prototype-brief.md) — the 6-screen demo flow and mock data. The Zendesk approval flow (Approve / Edit / Reject on AI-drafted responses) directly informed the Approval Queue feature description. The action set was expanded to five actions (Approve, Edit & Approve, Dismiss, Snooze, Reject & Teach) to cover the full range of queue behaviors.

**From GitHub:**
- [Issue #3](https://github.com/go2impact/cowork-brain/issues/3) (`Add Product Overview / PRD document`) — the original gap analysis identifying that no single document clearly explained what Cowork.ai does, its features, or where it's going.
- [Scott's comment on Issue #3](https://github.com/go2impact/cowork-brain/issues/3#issuecomment-3891009312) — positioning guardrails that shaped the "What Cowork.ai Is NOT" section. Key framing: "AI coworker that actually does things in your apps," not "smart Time Doctor." No focus scores, no productivity gamification, no surveillance framing. The surface should show "here's what I handled for you" and "here's what you can now take on" — not activity logs.

**From direct product input (Rustan):**
- The "Cowork.ai Sidecar" product description that introduced three features not yet documented anywhere in the repo:
  - **Approval Queue** ("Shadow Work" Manager) — AI monitors Slack, Gmail, Zendesk and drafts replies using Clone Mode for one-tap approval or teaching.
  - **Live Agent** (Native Audio) — real-time voice-first interface using Gemini's native audio capabilities.
  - **Task Automation & Focus Mode** — deep work protection barriers, notification interception, and background agents (Inbox Zero, Ticket Triage, Slack Digest, Calendar Guardian).
- The "second brain" framing and core thesis: the sidecar handles your secondary tasks (communication, scheduling, support) so you can focus on primary work.
- **Hybrid execution model directive:** the AI doesn't just call APIs behind the scenes — it also operates inside the user's actual apps via browser automation (Playwright), with activity capture feeding context and the user watching live and coaching in real-time. MCP for speed and background work, browser for visible execution and coaching, both together for complex tasks. The user is always in control and can intervene at any moment.

**What was composed from multiple sources (not pulled from a single document):**
- The four user stories (CX Agent, Virtual Assistant, Operations Specialist, Dev Support Engineer) — written to illustrate the features in realistic workflows, based on the target personas described in the product overview.
- The "Success Looks Like" section — derived from the positioning guardrails (Scott's comment) and the product thesis.
- Clone Mode as a standalone feature section — the concept came from Rustan's "Clone Mode (personalized instructions)" reference; the detailed breakdown of what it captures and how it learns was extrapolated from the Approval Queue's "teach" mechanic.

---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI workspace — your "second brain" — that works inside your apps alongside you, handling your secondary work (communication, scheduling, support) while you watch, coach, and stay focused on your primary work.

---

## Core Design Principle

The sidecar doesn't replace your tools. It sits on top of them *and* operates inside them. You still use Zendesk, Gmail, Slack, and your calendar — but the AI watches them via API, drafts responses, catches what you missed, and when you approve an action, it can execute visibly in your browser while you watch.

You are always in control. The AI works in the open — never behind a curtain. You can intervene at any moment: pause it, correct it, take over, or hand back. MCP handles data retrieval and background work. The browser handles visible execution and coached actions. Both work together for complex tasks.

The sidecar only expands when you need it.

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

**Cold-start & ramp-up:**

- Onboarding creates a usable default profile in ~5 minutes (role, tone, 3–5 key rules). This is enough for the AI to start drafting — not perfectly, but close enough that Edit & Approve is faster than writing from scratch.
- **Bootstrap from history (opt-in):** With the user's explicit consent, the AI can analyze recent sent messages from connected apps (e.g., last 50 Zendesk replies, last 30 sent emails) to extract tone patterns and common response structures. This accelerates profile quality without manual setup. The user reviews and approves the extracted profile before it's activated.
- **Shadow mode (first week):** For the first N interactions (configurable, default 20), the AI drafts but the Approval Queue highlights these as "learning drafts" — signaling that quality is still ramping. The Teach action is more prominent during this period to accelerate feedback loops.
- Draft quality improves with each Teach interaction. Target: usable drafts within 1 day of active use, 80%+ approve-without-editing within 1–2 weeks for routine items.

**Profile portability:**

Clone Mode profiles are stored locally (see [Privacy & Data Boundaries](#privacy--data-boundaries)). To protect against data loss during hardware changes or reinstalls, profiles can be exported as portable files and re-imported on a new device. Export/import is manual and user-initiated — profiles are never synced to the cloud automatically.

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

Example execution log (mixed MCP + browser entries):

```
10:32:04  MCP   zendesk.tickets.get #4521        → read ticket content
10:32:05  MCP   zendesk.users.get "sarah@acme"   → fetched customer context
10:32:06  MCP   clone-mode.draft                  → generated reply draft
10:32:08  QUEUE item #4521 approved (browser mode)
10:32:09  BROWSER navigate zendesk.com/tickets/4521
10:32:11  BROWSER click "Reply" button
10:32:12  BROWSER type draft into reply field
10:32:14  VOICE  user: "make the opening more empathetic"
10:32:15  BROWSER clear + retype opening paragraph
10:32:18  BROWSER click "Submit as Solved"
10:32:19  MCP   action-log.write                  → logged completed action
```

This isn't hidden. The Execution Viewer is always accessible. If the AI does something you don't understand, open the viewer and trace its reasoning. Transparency builds trust; trust enables autonomy.

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

| Stream | What's captured | Default state | Used for |
|--------|----------------|---------------|----------|
| **Window & app tracking** | Active app name and path, window title, focused UI element, active browser URL — via macOS Accessibility API (AXUIElement) with CGWindowList fallback. Polled every 500ms. Browser URLs extracted via AppleScript for Safari, Chrome, Edge, Brave, and Opera. | On (can be disabled) | Context Card, flow-state detection, Focus Mode auto-activation |
| **Keystroke & input capture** | Raw keystrokes (key code, characters, modifiers, timing) and mouse clicks (button, screen coordinates, click count, modifiers) via global CGEventTap. Raw input is stored locally and processed into communication patterns (common phrases, tone markers, structural habits) for Clone Mode. Target architecture: raw input is processed and then discarded; currently raw data persists in local storage pending the extraction pipeline. | Off (opt-in only, separate consent dialog, requires macOS Input Monitoring permission) | Clone Mode profile training, communication pattern extraction |
| **Focus detection** | Extended single-app sessions, deep work patterns — derived from window tracking data | On (can be disabled) | Focus Mode auto-activation, Context Card |
| **Browser activity** | Pages visited, actions taken by the AI during agent sessions | On during agent sessions only | Execution Viewer, audit trail |
| **Screen recording** | Full-screen video capture (15fps, H.264) of all active displays — requires macOS Screen Recording permission. Used for visual context during coached agent sessions. | Off (opt-in only, separate consent dialog) | Visual context for agent coaching, execution review |
| **Clipboard monitoring** | Text content on copy/paste events, with action type and timing. | Off (opt-in only, not yet active in current build) | Context enrichment for active tasks |

**Data flow:**

Raw capture → local SQLite database → (planned) local processing → structured context (embeddings via local RAG pipeline) → agents query for relevant context at action time.

All capture and storage happens on-device in a local SQLite database (GRDB, WAL mode). **Current state:** captured data (activity, keystrokes, mouse events) is stored locally in raw form. **Target architecture:** a local processing pipeline will extract structured context (embeddings, communication patterns) from raw data and discard the raw input within configurable retention windows. The processing pipeline is not yet implemented — raw data currently persists locally until manually cleared or retention enforcement is built.

**Clone Mode integration:**

When keystroke capture is enabled, the system captures raw input and (once the processing pipeline is built) extracts communication patterns to feed into Clone Mode profiles — common phrases, tone markers, sentence structure, greeting/sign-off habits. This means the AI learns your actual writing style from real usage, not just from explicit teaching. **Target architecture:** the extraction pipeline processes raw keystrokes into patterns and discards raw input within minutes. **Current state:** raw keystroke data (key codes, characters, modifiers, timing) is captured and stored locally in SQLite; the pattern extraction pipeline is not yet implemented, so raw data persists locally. Password fields and sensitive form inputs will be excluded from capture (detected via input field type attributes) — this exclusion logic is planned but not yet enforced in the current build. See [Privacy & Data Boundaries](#privacy--data-boundaries) for full details.

**Retention defaults (configurable):**

| Data type | Default retention | Current state |
|-----------|------------------|---------------|
| Window & app tracking | Rolling 7 days | Stored in local SQLite; retention enforcement not yet implemented |
| Raw keystroke & mouse data | Target: discarded within minutes of processing | Stored in local SQLite; processing pipeline not yet built |
| Keystroke patterns (processed) | Rolling 30 days | Not yet generated (pending extraction pipeline) |
| Focus detection | Rolling 7 days | Derived from window tracking data |
| Screen recordings | Rolling 30 days | Module built but not active in current build |
| Clipboard data | Rolling 7 days | Module built but not active in current build |
| Browser session recordings | Rolling 30 days | Active during agent sessions only |

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

**How promotion happens:**

Boundary changes are always user-initiated — the AI never auto-promotes an action category to autonomous. However, the AI may *suggest* a promotion when it detects a pattern: "You've approved 47 routine status update replies this month without editing. Would you like me to handle these automatically?" The user must explicitly confirm. Suggestions appear as a non-urgent queue item, not as an interrupting prompt. The user can dismiss the suggestion or tell the AI to stop suggesting for that category.

### The Audit Trail

Regardless of autonomy level or execution mode, every action is logged in the Execution Viewer. Approved or autonomous, API or browser — the worker can always see what the AI did, when, and why. This is non-negotiable. Transparency isn't just for approval-required actions; it covers everything.

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

## User Stories

*These stories demonstrate the full range of capabilities including browser coaching for complex or high-stakes tasks. In daily use, most approved actions execute instantly via API in the background — the worker approves and moves on. Browser mode is the exception for supervised or novel work, not the default for routine items.*

### As a CX Agent (Zendesk + Slack)

> I handle 40–60 Zendesk tickets per day for a US-based SaaS company. Half of them are routine: password resets, billing questions, feature requests I've answered a hundred times.
>
> With Cowork.ai, I open my sidecar in the morning and the Approval Queue has 15 drafted replies ready. I approve 12 via API — instant, gone. For the 2 that need tweaks, I edit and approve. For 1 tricky escalation, I switch to browser mode: the AI opens the ticket in Zendesk, types the reply I approved, and I watch. Halfway through, I say "add a note that we're waiving the fee as a one-time courtesy." The AI adjusts the reply on the spot, I confirm, and it sends. That took 10 minutes instead of 45.
>
> When I go heads-down on a complex escalation, Focus Mode holds my Slack pings. When I come up for air, I get a summary instead of 47 unread messages.

### As a Virtual Assistant (Gmail + Calendar + Slack)

> I manage communications for three executives. My day is constant context-switching: draft an email for one, schedule a call for another, respond to Slack messages for the third.
>
> Clone Mode has separate profiles for each exec — different tones, different rules, different sign-offs. When a meeting request comes in for Executive A, the AI knows to check their "no meetings on Wednesday" rule before drafting a response. The Calendar Guardian handles 80% of scheduling back-and-forth via MCP in the background.
>
> For a complicated multi-party scheduling request, I switch to browser mode and watch the AI navigate Google Calendar — checking availability across time zones, finding a room, and drafting the invite. I coach it: "Move that to Thursday — she's traveling Wednesday." The AI adjusts and sends. I went from spending 3 hours/day on scheduling to 30 minutes.

### As an Operations Specialist (Multi-tool)

> I do a bit of everything: data entry, report generation, inbox management, process documentation. My work spans 6 different tools.
>
> The App Gallery lets me connect all of them. The Context Card shows me a unified view: what's pending in each tool, what's overdue, what the AI already handled. I used to start my day with 20 minutes of "checking in" across tools. Now I start with the Context Card and go straight to what matters.
>
> My custom automation — "every Monday, pull last week's metrics from three sources and compile a draft report" — runs via MCP to gather data, then opens the reporting portal in the browser to submit. I watch the submission step because the portal has a multi-step form the API doesn't cover. The AI fills it in, I verify the numbers, and it submits. Saves me 2 hours every week.

### As a Dev Support Engineer (Zendesk + Linear + Slack)

> Bug reports come in through Zendesk, but I need to check Linear for existing tickets and Slack for engineering context before responding.
>
> The Ticket Triage Agent cross-references incoming Zendesk reports against open Linear issues via MCP. If there's a match, it pre-populates my draft with the relevant engineering context: "This is a known issue (LINEAR-1234), fix is in progress, ETA next release." I just approve — sends via API instantly.
>
> For new bugs, the AI drafts an acknowledgment to the customer and a Linear ticket for engineering. Both land in my Approval Queue. I approve the customer reply and then switch to browser mode for the Linear ticket — I watch the AI fill in the fields, coach it on severity ("make that a P1, not P2"), and it submits. Both reviewed and handled in under a minute.

---

## Privacy & Data Boundaries

Cowork.ai observes your work context to be useful. That observation has to be clearly bounded, or "AI assistant" becomes indistinguishable from "surveillance tool." These are the rules.

**Known implementation gaps (current build):** Raw keystroke and mouse data persists in local SQLite because the pattern extraction pipeline is not yet built (target: process into patterns, then discard raw input within minutes). Sensitive input field exclusion (password fields, etc.) is planned but not yet enforced. Retention window enforcement is not yet implemented — data persists locally until manually cleared. These gaps are called out inline throughout this section; this summary exists so they're scannable in one place.

### What the AI observes, and where it's processed

| Data type | What's collected | Processed where | Stored where | Retention |
|-----------|-----------------|----------------|-------------|-----------|
| **Connected app data** (Zendesk tickets, Gmail, Slack messages) | Content from apps you explicitly connect via MCP. Only the apps you install, only the channels/inboxes you authorize. | Local brain (on-device) or cloud brain — user's choice at onboarding | Persistent storage is on-device only. When cloud inference is used, content is sent transiently to the inference provider for processing. Cowork.ai's own servers do not store this content. Provider retention policies (OpenRouter, Anthropic, Google) are subject to their terms of service; Cowork.ai selects providers with zero-retention or minimal-retention API policies. See [inference providers](#who-can-see-what) below. | Persists locally until user deletes the app connection or clears data. |
| **Activity context** (which app is in focus, how long you've been in it) | Active application name, bundle path, window title, focused UI element, and active browser URL — collected system-wide via macOS Accessibility API (AXUIElement) with CGWindowList fallback, polled every 500ms. Browser URLs extracted via AppleScript for major browsers. This is how the Context Card knows you're "Working in Zendesk · 47 minutes" even before you connect Zendesk via MCP. Uses accessibility introspection beyond window titles — the app reads the focused window and UI element attributes to understand context. Note: window titles and URLs may contain sensitive fragments (email subjects, document names, ticket titles, URL parameters); activity context should be treated as potentially sensitive data. To mitigate this: activity context entries older than the rolling retention window are discarded entirely (not archived). Users can view and manually redact individual entries from their activity history at any time. Window titles and URLs are never included in telemetry, never sent off-device, and never used for any purpose beyond the Context Card and flow-state detection. | Local only. Never leaves the device. | On-device only (local SQLite). | Rolling window — last 7 days by default, configurable (retention enforcement not yet implemented). |
| **Keystroke & input capture** (typing and mouse activity) | Raw keystrokes (key code, characters, modifiers, timing) and mouse clicks (button, screen coordinates, click count, modifiers) captured via global CGEventTap. **Target architecture:** raw input is processed into communication patterns (phrase frequency, tone markers, structural habits) and then discarded. **Current state:** raw keystroke and mouse data is stored locally in SQLite; the pattern extraction pipeline is not yet implemented. Password field and sensitive input exclusion is planned but not yet enforced. Requires macOS Input Monitoring permission. | Local only. Never leaves the device. | On-device only (local SQLite). Raw data currently persists locally; target is extracted patterns only. | Target: raw input discarded within minutes, patterns rolling 30 days. Current: raw data persists locally pending pipeline implementation. |
| **Browser session data** (agent execution) | Pages visited, actions taken, coaching interventions — during AI-driven browser sessions only. Not a general browsing monitor. | Local only. | On-device only. | Rolling 30 days, configurable. |
| **Clone Mode profiles** (tone, rules, SOPs) | What the user explicitly provides, teaches, or what activity-informed training extracts (opt-in). | Local only. | On-device only. | Persists until user edits or deletes. |
| **Action history** (what the AI drafted, sent, dismissed) | Audit log of all AI actions via Execution Viewer. When an app is disconnected, action history entries for that app are retained as metadata (what action, when, which app, outcome) but content references (draft text, ticket content, message bodies) are purged. | Local only. | On-device only. | Metadata persists indefinitely for audit trail. Content references are purged on app disconnect (same as connected app data). |

### What the AI never collects

- **Continuous background screen recording.** Screen recording capability exists (opt-in, requires macOS Screen Recording permission) but is designed for user-initiated agent coaching sessions — not ambient desktop surveillance. When enabled, it captures at 15fps during active agent sessions. It is off by default and requires explicit opt-in via a separate consent dialog. Outside of opted-in sessions, no screen content is captured.
- **Keystroke data for surveillance or employer reporting.** Keystroke capture (when opted in) captures raw input locally for the purpose of extracting communication patterns into Clone Mode profiles. **Current state:** raw keystroke data (key codes, characters, modifiers, timing) and mouse events persist in local SQLite because the pattern extraction pipeline is not yet built. **Target architecture:** raw input is processed into patterns and discarded within minutes; only patterns persist. In both states, keystroke data never leaves the device and is never reported to employers, managers, or anyone else — it trains your AI, not a dashboard. Password field exclusion is planned but not yet enforced in the current build.
- **Browsing history or app content outside MCP and agent sessions.** Activity context uses macOS Accessibility APIs to read the focused window's title, UI element attributes, and browser URLs. This provides richer context than window titles alone but does not access full app content (document bodies, email text, message threads). Full app content is only accessible through explicitly connected MCP apps or during user-initiated browser agent sessions.
- **Idle/active time.** The AI doesn't track when you start working, when you stop, or how long you were "away." Activity context is used for flow-state detection, not attendance monitoring.
- **Audio recording or storage when not activated.** If wake word is enabled, a small on-device model continuously processes audio locally to detect the trigger phrase — this is ephemeral (not recorded, not stored, not transmitted). Once triggered, the Live Agent processes speech locally via Whisper (on-device transcription), then sends the resulting text — not audio — to the LLM for response generation. Audio never leaves the device. The response is delivered as synthesized speech locally. If a future native duplex audio mode is introduced (see [Live Agent roadmap](#3-live-agent--voice--visual-interface-phased)), the user will be informed that audio data will be sent to a cloud provider, and this mode will require explicit opt-in with a clear consent dialog. If wake word is disabled, audio processing only begins on hotkey press. No ambient audio is ever recorded, stored, or sent off-device.

### Who can see what

- **The worker** sees everything: all observed data, all AI actions, all audit logs via the Execution Viewer.
- **The employer** sees nothing by default. If the worker is on an enterprise plan, aggregate and anonymized data (team-level patterns, not individual activity) can be shared — but only with the worker's explicit opt-in consent. No per-worker dashboards for managers. No screen time reports. This includes action metadata — the audit trail of what the AI did and when is the worker's personal record. It is never surfaced to employers, even on enterprise plans, even in aggregate. Employer-visible aggregate data covers only feature adoption and team-level volume patterns, never per-worker action logs.
- **Cowork.ai (the company)** sees pseudonymized usage telemetry for product improvement. Telemetry includes: feature usage counts, error rates, model latency, and tier/plan metadata. User identifiers are pseudonymized (replaced with opaque IDs — not anonymous, but not directly identifying). Telemetry does not include: message content, ticket data, draft text, app-specific data, activity context (app names/window titles), keystroke data, or browser session data. Telemetry is aggregated for analysis but collected at the pseudonymized-user level. On enterprise plans, pseudonymized telemetry IDs are cryptographically separated from billing/account identifiers — Cowork.ai's analytics pipeline cannot join telemetry records to employer billing records or individual worker identities. This separation is enforced at the infrastructure level, not by policy alone.
- **Third-party inference providers** (OpenRouter, Anthropic, Google — depending on tier) receive connected app content transiently when cloud inference is used. Cowork.ai selects providers with zero-retention or minimal-retention API terms. The worker chooses local-only or cloud processing at onboarding and can switch anytime. When using local brain only, no content reaches any third party.

### How this is different from surveillance tools

The difference isn't just messaging — it's architecture:

- **Granular consent, not blanket surveillance.** Every MCP connection is explicitly installed by the worker. Every channel within an app is explicitly authorized. Keystroke capture, screen recording, and clipboard monitoring each require their own separate consent dialog and macOS system permissions. The default state for app content is "AI sees nothing." Activity context (app name, window title, focused element, browser URL via Accessibility API) is the one stream enabled by default — it powers core features (Context Card, flow-state detection) and can be disabled at any time.
- **Browser automation is user-initiated and user-supervised.** The AI only works in the browser when you approve an action for browser-mode execution. You watch it happen in real-time — like a screenshare with a coworker. You can pause, take over, or cancel at any point. This is not background screen control.
- **Activity capture trains your AI, not an employer dashboard.** Keystroke and input data feeds Clone Mode profiles. Window and accessibility tracking feeds the Context Card. Screen recordings and browser session data feed the audit trail. None of this data is reported to employers, managers, or anyone else. It exists to make your AI better at your job.
- **Local-first storage and processing.** Activity context, keystroke and mouse data, screen recordings, clipboard data, browser session data, and Clone Mode profiles are stored on-device only (local SQLite) and never leave the device. The cloud brain processes inference requests statelessly — there's no central server accumulating your work history.
- **Worker controls the data.** Disconnect an app, and the AI loses access immediately. App content and draft text are removed; action metadata (what happened, when) is retained for your audit trail but stripped of content. Disable keystroke capture, and input monitoring stops immediately. Disable screen recording, and capture stops. Delete your data, and it's gone. No extended retention on Cowork.ai's side.
- **Flow-state detection is heuristic, not tracking.** The AI notices you've been in a single app context for an extended period and offers to activate Focus Mode. It doesn't build a permanent timeline of your activity. Activity context is kept in a rolling 7-day window (configurable) for the Context Card and flow-state detection, then discarded. Action history — what the AI did, not what you did — persists for your own audit trail. The pattern is: observe → suggest → expire. Not: observe → accumulate → report.

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

| Version | Date | Changes |
|---------|------|---------|
| Concept 0.6 | 2026-02-13 | Codex review findings addressed: Added "First wedge" anchor (CX agent + Zendesk + Clone Mode) after scope note to clarify v0.1 bet. Added Guardrails & Incident Containment subsection — non-promotable categories (financial, first-contact, destructive), kill switch, rate limiting, best-effort recall. Fixed trust framing: "Opt-in, not opt-out" replaced with "Granular consent, not blanket surveillance" to accurately reflect activity context being on by default. Consolidated known implementation gaps (raw keystroke retention, missing sensitive field exclusion, missing retention enforcement) into a scannable callout at top of Privacy section. Added user stories framing paragraph clarifying that browser coaching is the exception, not the daily default. |
| Concept 0.5 | 2026-02-13 | Activity Capture aligned with native Mac app codebase: updated all capture streams to reflect actual implementation (Accessibility API with AXUIElement introspection, CGEventTap for global keystroke + mouse capture, browser URL extraction via AppleScript, screen recording module, clipboard monitoring module). Honest "current state vs. target architecture" annotations added throughout — raw data storage in local SQLite acknowledged where the processing/extraction pipeline is not yet built. Privacy section updated: Activity context row reflects accessibility introspection beyond window titles; Keystroke capture row reflects raw data persistence; "What the AI never collects" rewritten to acknowledge screen recording capability (opt-in, session-scoped) and current raw keystroke storage. Retention table expanded with current implementation state. Surveillance differentiation updated to include all capture streams. |
| Concept 0.4 | 2026-02-13 | App Gallery: restored the app-as-component concept — apps are components with built-in MCP connectivity. Added "What an app actually is" subsection explaining the developer mental model (import → render → MCP). Expanded "Custom" app line to reference the shared architecture. No user-facing terminology changes (apps stay "apps"). |
| Concept 0.3 | 2026-02-13 | Hybrid execution model: added browser automation (Playwright) alongside MCP for visible, coachable execution. New sections: Execution Modes (MCP + browser working together), Activity Capture & Context Engine (window tracking, keystroke capture, focus detection, browser session data). Shadow Work Manager gains Stage 3 (Execute & Watch) with per-app/per-category API vs. browser preference. Clone Mode adds activity-informed training via opt-in keystroke capture. Live Agent expanded to voice + visual coaching with take-over/hand-back. Task Automation agents now use MCP + browser as appropriate. MCP Browser renamed to Execution Viewer with dual-pane live browser view + execution log. Context Card shows live agent execution status. Autonomy Levels add execution mode preferences including "autonomous but visible." User stories rewritten to demonstrate hybrid model with coaching. Privacy section updated: keystroke capture and browser session data added to observation table; "no screenshots/no keystrokes" replaced with honest boundaries around user-initiated browser automation and opt-in pattern extraction; surveillance differentiation expanded. |
| Concept 0.2 | 2026-02-13 | Review findings resolved: Shadow Work Manager naming (F2), Monitor & Draft stage separation (F6), snooze return behavior (F7), Clone Mode cold-start & portability (F1, F13), Live Agent voice-to-primitive mapping & audio privacy path (F4, F12), Background Agents polling/webhook reality (F14), Focus Mode graceful degradation (F5), Context Card data sources (F8), autonomy promotion mechanism (F15), privacy mitigations for activity context (F9), telemetry re-identification (F10), employer metadata exclusion (F11). |
| Concept 0.1 | 2026-02-13 | Initial feature set: Approval Queue, Clone Mode, Live Agent, Focus Mode, App Ecosystem, Context Card. User stories for CX, VA, Ops, Dev Support. |
