# Cowork.ai Sidecar — Product Features & Capabilities

**Version:** Concept 0.2 (capability map, not a release scope contract)
**Last Updated:** 2026-02-13
**Purpose:** Detailed feature set, capabilities, and user stories for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries.
**Scope Note:** This document defines the full product concept across phases. Current release scope is tracked in [`product-overview.md`](./product-overview.md) and implementation sequencing in [`../architecture/llm-architecture.md`](../architecture/llm-architecture.md).

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

**What was composed from multiple sources (not pulled from a single document):**
- The four user stories (CX Agent, Virtual Assistant, Operations Specialist, Dev Support Engineer) — written to illustrate the features in realistic workflows, based on the target personas described in the product overview.
- The "Success Looks Like" section — derived from the positioning guardrails (Scott's comment) and the product thesis.
- Clone Mode as a standalone feature section — the concept came from Rustan's "Clone Mode (personalized instructions)" reference; the detailed breakdown of what it captures and how it learns was extrapolated from the Approval Queue's "teach" mechanic.

---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI workspace — your "second brain" — that handles your secondary work (communication, scheduling, support) so you can stay focused on your primary work.

---

## Core Design Principle

The sidecar doesn't replace your tools. It sits *on top of them*. You still use Zendesk, Gmail, Slack, and your calendar — but the AI watches them, drafts responses, catches what you missed, and only interrupts you when something actually needs your attention. Your screen stays yours. The sidecar only expands when you need it.

---

## Feature Set

### 1. Shadow Work Manager (Approval Queue)

Shadow work is everything that isn't your actual job but eats your day: responding to Slack threads, triaging email, replying to low-priority support tickets, confirming calendar invites. The Approval Queue makes this work disappear.

**How it works:**

The pipeline has two separable stages:

1. **Monitor & Triage** — The AI watches your connected channels (Slack, Gmail, Zendesk — and eventually Calendar, CRM, and others) via read-only MCP access. It identifies items that need a response or action from you, categorizes them, and surfaces them in the sidecar.
2. **Draft & Queue** — For items it can handle, the AI drafts a response using your **Clone Mode** profile (see below) — matching your tone, following your rules, applying your SOPs. Drafts land in the Approval Queue: a single, prioritized feed inside the sidecar.

These stages are separable. In early use or with a new app connection, the AI may surface items for triage before it's confident enough to draft.

- You review each item and take one action:

  | Action | What happens | When to use it |
  |--------|-------------|----------------|
  | **Approve** | Send the draft as-is. One tap. | The AI nailed it. |
  | **Edit & Approve** | Tweak the draft, then send. | Close but needs a small fix. |
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

**Cold-start & ramp-up:**

- Onboarding creates a usable default profile in ~5 minutes (role, tone, 3–5 key rules). This is enough for the AI to start drafting — not perfectly, but close enough that Edit & Approve is faster than writing from scratch.
- **Bootstrap from history (opt-in):** With the user's explicit consent, the AI can analyze recent sent messages from connected apps (e.g., last 50 Zendesk replies, last 30 sent emails) to extract tone patterns and common response structures. This accelerates profile quality without manual setup. The user reviews and approves the extracted profile before it's activated.
- **Shadow mode (first week):** For the first N interactions (configurable, default 20), the AI drafts but the Approval Queue highlights these as "learning drafts" — signaling that quality is still ramping. The Teach action is more prominent during this period to accelerate feedback loops.
- Draft quality improves with each Teach interaction. Target: usable drafts within 1 day of active use, 80%+ approve-without-editing within 1–2 weeks for routine items.

**Profile portability:**

Clone Mode profiles are stored locally (see [Privacy & Data Boundaries](#privacy--data-boundaries)). To protect against data loss during hardware changes or reinstalls, profiles can be exported as portable files and re-imported on a new device. Export/import is manual and user-initiated — profiles are never synced to the cloud automatically.

---

### 3. Live Agent — Voice Interface (Phased)

The Live Agent is a real-time, voice-first interface. Instead of typing, you talk — and the AI talks back. Built for moments when your hands are busy or when speaking is faster than typing.

**Capabilities:**

- **Voice commands** — "What's in my queue?" "Draft a reply to that Zendesk ticket." "Summarize my unread Slack messages." "Block my calendar for the next hour."
- **Conversational context** — The Live Agent knows what you're looking at. If you're reviewing a ticket in the Approval Queue, you can say "make this more formal" or "add an apology" without specifying which item.
- **Real-time responses (v0.x)** — Low-latency voice interaction via streaming speech-to-text + LLM + speech output. This delivers practical voice-first workflows without waiting for full native duplex audio.
- **Native duplex roadmap** — Full native audio (simultaneous speech-in/speech-out) is a future upgrade once reliability, cost, and device support meet production targets.
- **Hands-free mode** — Activate via wake word or hotkey. Useful during calls, while multitasking, or for accessibility.

**How voice maps to the system:**

Voice commands are not a separate action system — they invoke the same primitives used by the queue and MCP Browser. "Approve that draft" triggers the same Approve action as tapping the button. "Draft a reply to that ticket" creates a queue item via the same drafting pipeline. All voice-initiated actions appear in the MCP Browser audit log identically to manual actions, tagged with `source: voice` for traceability.

**When to use it:**

- Morning briefing: "What do I need to handle today?"
- Mid-task: "What did Sarah say in that Slack thread?"
- Queue actions: "Approve that draft" / "Dismiss this one" / "Snooze it for an hour"
- End of day: "Anything I missed?"

---

### 4. Task Automation & Focus Mode

Focus Mode is how the sidecar protects your deep work time. When you're in focus, the AI becomes your gatekeeper.

**Focus Mode:**

- Activate manually ("I'm focusing for the next 2 hours") or automatically (the AI detects you've been in a single app context for an extended period and suggests activating Focus Mode — see [Privacy & Data Boundaries](#privacy--data-boundaries) for what "activity context" means and doesn't mean). If you've disabled activity context collection (see [How this is different from surveillance tools](#how-this-is-different-from-surveillance-tools)), auto-activation is unavailable — Focus Mode can only be activated manually or on a schedule. The feature degrades gracefully; it doesn't silently stop working.
- Non-urgent notifications are intercepted and held. The AI triages incoming items and only breaks through for genuine emergencies (P1 tickets, messages from your manager, calendar conflicts).
- Breakthrough decisions use the same prioritization principles as the Approval Queue so interruption behavior stays predictable.
- When Focus Mode ends, you get a summary: "While you were focused, I handled 12 items. 3 need your review."

**Background Agents:**

Persistent automation agents that run continuously without manual triggers ("continuously" means the agents check for new items on a regular cadence — typically polling connected apps every few minutes via MCP, not maintaining persistent open connections. The polling interval is configurable per app. When webhook support is available from the connected service, the agent reacts to push events instead of polling, reducing latency and resource usage):

- **Inbox Zero Agent** — Continuously processes your email: categorizes and archives low-priority (autonomous), drafts responses for items that need one (queued for approval), and flags anything that requires your personal attention. Goal: when you open your inbox, it's already organized.
- **Ticket Triage Agent** — Monitors your support queue: auto-categorizes by type and urgency (autonomous), drafts first responses for common patterns (queued for approval), and escalates anything outside your SOP.
- **Slack Digest Agent** — Watches your channels, filters signal from noise, and delivers periodic summaries instead of constant pings (autonomous). Knows which channels matter to you and which are FYI-only.
- **Calendar Guardian Agent** — Prepares meeting briefs before each call (autonomous), flags scheduling conflicts for review, and auto-declines low-priority meetings that conflict with focus blocks (autonomous — user can demote to approval-required).

**Custom Automations:**

Background Agents are opinionated presets that ship with the product — they work out of the box with sensible defaults and learn from your Clone Mode profile. Custom Automations are user-defined rules that you create from scratch for workflows specific to your role:

- "When a ticket is tagged 'billing', always check the customer's payment history before drafting."
- "Every Friday at 4pm, compile a summary of everything I handled this week."
- "If a Slack DM from my manager goes unanswered for 30 minutes, escalate to my phone."

---

### 5. App Ecosystem & MCP Browser

**App Gallery:**

A library of pre-built integrations. Install the apps that match your workflow:

- **Communication:** Slack, Gmail, Microsoft Teams
- **Support:** Zendesk, Freshdesk, Intercom
- **CRM:** Salesforce, HubSpot
- **Project Management:** Linear, Asana, Jira
- **Calendar:** Google Calendar, Outlook
- **Custom:** Build your own MCP integrations for internal tools

Each installed app shows a compact status card in the sidesheet (unread count, queue depth, next event) and expands to a full view for deep interaction.

**Cross-app boundaries (concept-level):**

- Cross-app automations run only when required connections and scopes are available.
- If a required source is disconnected or auth expires, the automation pauses and creates a queue item with the exact blocker and reconnect action.
- Partial permissions degrade gracefully (read-only summaries allowed; blocked write actions require reconnect/approval).
- MCP Browser must show source provenance for every cross-app action: what was read, what failed, and what was skipped.

**MCP Browser — Full Transparency:**

When the AI takes actions, you can watch every step:

- What data it read
- What it searched for
- What it drafted
- What tools it called
- What it's waiting for your approval on

This isn't hidden. The MCP Browser is always accessible. If the AI does something you don't understand, open the browser and trace its reasoning. Transparency builds trust; trust enables autonomy.

---

### 6. Context Card — Your Work at a Glance

The Context Card is what you see when you open the sidesheet. It's not a focus score. It's not a productivity metric. It's a plain-language summary of your current work state:

- **"You have 4 tickets in queue, 2 are drafted and ready to send."**
- **"3 unread Slack DMs — one from your manager."**
- **"Next meeting in 45 minutes. Prep brief is ready."**
- **"Inbox Zero agent cleared 18 emails. 2 need your input."**

The Context Card answers one question: *What should I do next?*

**Where Context Card data comes from:**

The Context Card draws from two sources: (1) **connected app data** via MCP — ticket counts, unread messages, calendar events — and (2) **activity context** — the active app name and window title, collected system-wide (not limited to MCP-connected apps). This is how the card can show "Working in Zendesk · 47 minutes" even before you connect Zendesk via MCP. Activity context is processed locally and never leaves your device. See [Privacy & Data Boundaries](#privacy--data-boundaries) for full details.

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

### How Users Move the Boundary

The default is conservative. Over time, as trust builds, users can expand what the AI handles autonomously:

- **Promote from queue to auto-send.** After approving 50 standard refund replies without editing, you might tell the AI: "Handle standard refunds automatically. Just log them." That category leaves the Approval Queue and becomes a Background Agent behavior.
- **Demote from auto to queue.** If the Calendar Guardian auto-declines a meeting you actually wanted, you can pull calendar actions back into the Approval Queue: "Always ask me before declining meetings."
- **Per-channel rules.** "Auto-send in Slack #general, but queue everything in #client-escalations."
- **Per-contact rules.** "Auto-respond to internal team, but always queue anything from the VP of Sales."

**How promotion happens:**

Boundary changes are always user-initiated — the AI never auto-promotes an action category to autonomous. However, the AI may *suggest* a promotion when it detects a pattern: "You've approved 47 standard refund replies this month without editing. Would you like me to handle these automatically?" The user must explicitly confirm. Suggestions appear as a non-urgent queue item, not as an interrupting prompt. The user can dismiss the suggestion or tell the AI to stop suggesting for that category.

### The Audit Trail

Regardless of autonomy level, every action is logged in the MCP Browser. Approved or autonomous — the worker can always see what the AI did, when, and why. This is non-negotiable. Transparency isn't just for approval-required actions; it covers everything.

---

## User Stories

### As a CX Agent (Zendesk + Slack)

> I handle 40–60 Zendesk tickets per day for a US-based SaaS company. Half of them are routine: password resets, billing questions, feature requests I've answered a hundred times.
>
> With Cowork.ai, I open my sidecar in the morning and the Approval Queue has 15 drafted replies ready. I approve 12, tweak 2, and teach the AI on 1 it got wrong. That took 8 minutes instead of 45. Now I can spend my time on the complex tickets that actually need me.
>
> When I go heads-down on a tricky escalation, Focus Mode holds my Slack pings. When I come up for air, I get a summary instead of 47 unread messages.

### As a Virtual Assistant (Gmail + Calendar + Slack)

> I manage communications for three executives. My day is constant context-switching: draft an email for one, schedule a call for another, respond to Slack messages for the third.
>
> Clone Mode has separate profiles for each exec — different tones, different rules, different sign-offs. When a meeting request comes in for Executive A, the AI knows to check their "no meetings on Wednesday" rule before drafting a response.
>
> The Calendar Guardian Agent handles 80% of scheduling back-and-forth. I review the final confirmations. I went from spending 3 hours/day on scheduling to 30 minutes.

### As an Operations Specialist (Multi-tool)

> I do a bit of everything: data entry, report generation, inbox management, process documentation. My work spans 6 different tools.
>
> The App Gallery lets me connect all of them. The Context Card shows me a unified view: what's pending in each tool, what's overdue, what the AI already handled. I used to start my day with 20 minutes of "checking in" across tools. Now I start with the Context Card and go straight to what matters.
>
> The custom automation I built — "every Monday, pull last week's metrics from three sources and compile a draft report" — saves me 2 hours every week.

### As a Dev Support Engineer (Zendesk + Linear + Slack)

> Bug reports come in through Zendesk, but I need to check Linear for existing tickets and Slack for engineering context before responding.
>
> The Ticket Triage Agent cross-references incoming Zendesk reports against open Linear issues. If there's a match, it pre-populates my draft with the relevant engineering context: "This is a known issue (LINEAR-1234), fix is in progress, ETA next release." I just approve.
>
> For new bugs, the AI drafts an acknowledgment to the customer and a Linear ticket for engineering. Both land in my Approval Queue for review before anything goes out.

---

## Privacy & Data Boundaries

Cowork.ai observes your work context to be useful. That observation has to be clearly bounded, or "AI assistant" becomes indistinguishable from "surveillance tool." These are the rules.

### What the AI observes, and where it's processed

| Data type | What's collected | Processed where | Stored where | Retention |
|-----------|-----------------|----------------|-------------|-----------|
| **Connected app data** (Zendesk tickets, Gmail, Slack messages) | Content from apps you explicitly connect via MCP. Only the apps you install, only the channels/inboxes you authorize. | Local brain (on-device) or cloud brain — user's choice at onboarding | Persistent storage is on-device only. When cloud inference is used, content is sent transiently to the inference provider for processing. Cowork.ai's own servers do not store this content. Provider retention policies (OpenRouter, Anthropic, Google) are subject to their terms of service; Cowork.ai selects providers with zero-retention or minimal-retention API policies. See [inference providers](#who-can-see-what) below. | Persists locally until user deletes the app connection or clears data. |
| **Activity context** (which app is in focus, how long you've been in it) | Active application name and window title — collected system-wide (not limited to connected MCP apps). This is how the Context Card knows you're "Working in Zendesk · 47 minutes" even before you connect Zendesk via MCP. Used for the Context Card and flow-state detection. Does not access app content beyond what appears in the window title. Note: window titles may contain sensitive fragments (email subjects, document names, ticket titles); activity context should be treated as potentially sensitive data. To mitigate this: activity context entries older than the rolling retention window are discarded entirely (not archived). Users can view and manually redact individual entries from their activity history at any time. Window titles are never included in telemetry, never sent off-device, and never used for any purpose beyond the Context Card and flow-state detection. | Local only. Never leaves the device. | On-device only. | Rolling window — last 7 days by default, configurable. |
| **Clone Mode profiles** (tone, rules, SOPs) | Only what the user explicitly provides or teaches. Not inferred from observed behavior. | Local only. | On-device only. | Persists until user edits or deletes. |
| **Action history** (what the AI drafted, sent, dismissed) | Audit log of all AI actions via MCP Browser. When an app is disconnected, action history entries for that app are retained as metadata (what action, when, which app, outcome) but content references (draft text, ticket content, message bodies) are purged. | Local only. | On-device only. | Metadata persists indefinitely for audit trail. Content references are purged on app disconnect (same as connected app data). |

### What the AI never collects

- **Screenshots or screen recordings.** The AI knows the active app name and window title, not what's on your screen.
- **Keystroke or mouse data.** No input logging of any kind.
- **Browsing history or app content outside MCP.** The AI sees the name and window title of the foreground app (activity context — see table above), but has no access to content inside apps unless they are connected via MCP.
- **Idle/active time.** The AI doesn't track when you start working, when you stop, or how long you were "away." Activity context is used for flow-state detection, not attendance monitoring.
- **Audio recording or storage when not activated.** If wake word is enabled, a small on-device model continuously processes audio locally to detect the trigger phrase — this is ephemeral (not recorded, not stored, not transmitted). Once triggered, the Live Agent processes speech locally via Whisper (on-device transcription), then sends the resulting text — not audio — to the LLM for response generation. Audio never leaves the device. The response is delivered as synthesized speech locally. If a future native duplex audio mode is introduced (see [Live Agent roadmap](#3-live-agent--voice-interface-phased)), the user will be informed that audio data will be sent to a cloud provider, and this mode will require explicit opt-in with a clear consent dialog. If wake word is disabled, audio processing only begins on hotkey press. No ambient audio is ever recorded, stored, or sent off-device.

### Who can see what

- **The worker** sees everything: all observed data, all AI actions, all audit logs via MCP Browser.
- **The employer** sees nothing by default. If the worker is on an enterprise plan, aggregate and anonymized data (team-level patterns, not individual activity) can be shared — but only with the worker's explicit opt-in consent. No per-worker dashboards for managers. No screen time reports. This includes action metadata — the audit trail of what the AI did and when is the worker's personal record. It is never surfaced to employers, even on enterprise plans, even in aggregate. Employer-visible aggregate data covers only feature adoption and team-level volume patterns, never per-worker action logs.
- **Cowork.ai (the company)** sees pseudonymized usage telemetry for product improvement. Telemetry includes: feature usage counts, error rates, model latency, and tier/plan metadata. User identifiers are pseudonymized (replaced with opaque IDs — not anonymous, but not directly identifying). Telemetry does not include: message content, ticket data, draft text, app-specific data, or activity context (app names/window titles). Telemetry is aggregated for analysis but collected at the pseudonymized-user level. On enterprise plans, pseudonymized telemetry IDs are cryptographically separated from billing/account identifiers — Cowork.ai's analytics pipeline cannot join telemetry records to employer billing records or individual worker identities. This separation is enforced at the infrastructure level, not by policy alone.
- **Third-party inference providers** (OpenRouter, Anthropic, Google — depending on tier) receive connected app content transiently when cloud inference is used. Cowork.ai selects providers with zero-retention or minimal-retention API terms. The worker chooses local-only or cloud processing at onboarding and can switch anytime. When using local brain only, no content reaches any third party.

### How this is different from surveillance tools

The difference isn't just messaging — it's architecture:

- **Opt-in, not opt-out.** Every MCP connection is explicitly installed by the worker. Every channel within an app is explicitly authorized. The default state for app content is "AI sees nothing" — activity context (app name and window title) is enabled at setup and can be disabled.
- **Local-first processing.** Activity context and Clone Mode profiles never leave the device. The cloud brain processes inference requests statelessly — there's no central server accumulating your work history.
- **Worker controls the data.** Disconnect an app, and the AI loses access immediately. App content and draft text are removed; action metadata (what happened, when) is retained for your audit trail but stripped of content. Delete your data, and it's gone. No extended retention on Cowork.ai's side.
- **Flow-state detection is heuristic, not tracking.** The AI notices you've been in a single app context for an extended period and offers to activate Focus Mode. It doesn't build a permanent timeline of your activity. Activity context is kept in a rolling 7-day window (configurable) for the Context Card and flow-state detection, then discarded. Action history — what the AI did, not what you did — persists for your own audit trail. The pattern is: observe → suggest → expire. Not: observe → accumulate → report.

---

## What Cowork.ai Is NOT

Getting the positioning right matters. Cowork.ai is not:

- **A time tracker.** We don't measure how many hours you worked or when you were "active." There are no focus scores, no productivity gamification, no surveillance dashboards.
- **A standalone chatbot.** You don't copy-paste context into a chat window. The AI already knows your context because it's connected to your tools.
- **An employer monitoring tool.** The worker owns their data. The worker controls what's connected. The worker decides what gets shared. If an employer wants team-level insights, those come from aggregated, consented data — not screen recordings.
- **A replacement for your tools.** You still use Zendesk, Gmail, Slack. Cowork.ai sits on top of them, not instead of them.

---

## Success Looks Like

From the worker's perspective, Cowork.ai is working when:

- You open the sidecar and your queue is already handled. You just approve and move on.
- You finish a focus session and nothing fell through the cracks while you were heads-down.
- Your Clone Mode drafts are so good that you approve 80%+ without editing.
- You talk to the Live Agent and it does what you asked on the first try.
- You handle more volume in less time, and your clients notice the improvement — not the AI.

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
| Concept 0.2 | 2026-02-13 | Review findings resolved: Shadow Work Manager naming (F2), Monitor & Draft stage separation (F6), snooze return behavior (F7), Clone Mode cold-start & portability (F1, F13), Live Agent voice-to-primitive mapping & audio privacy path (F4, F12), Background Agents polling/webhook reality (F14), Focus Mode graceful degradation (F5), Context Card data sources (F8), autonomy promotion mechanism (F15), privacy mitigations for activity context (F9), telemetry re-identification (F10), employer metadata exclusion (F11). |
| Concept 0.1 | 2026-02-13 | Initial feature set: Approval Queue, Clone Mode, Live Agent, Focus Mode, App Ecosystem, Context Card. User stories for CX, VA, Ops, Dev Support. |
