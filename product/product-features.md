# Cowork.ai Sidecar — Product Features & Capabilities

| | |
|---|---|
| **Status** | Working draft (capability map, not a release scope contract) |
| **Last Updated** | 2026-02-13 |
| **Purpose** | Detailed feature set and capabilities for the Cowork.ai desktop sidecar. Product-focused — architecture details appear only where needed to define privacy boundaries. |


---

## The Product in One Sentence

Cowork.ai Sidecar is a persistent desktop AI that observes your work, remembers your context, and acts inside your apps — surfacing what matters, answering when you ask, and handling your secondary work while you stay focused on your primary work.

---

## Core Design Principles

**Works inside your tools, not instead of them.** You still use Zendesk, Gmail, Slack. The sidecar sits on top of them (via API) and operates inside them (via browser). Apps built in Google AI Studio are uploaded to the desktop app and given access to your work context through platform-provided MCPs.

**Observes and remembers.** The AI watches what you're working on, embeds and indexes that activity, and builds memory over time. It gets smarter the longer you use it — without sending your data off-device.

**Pushes and pulls.** The platform comes to you when something is worth your attention (Proactive Suggestions). You come to it when you have a question (On-Demand Chat). Both paths lead to action.

**You are always in control.** The AI works in the open — never behind a curtain. You can intervene at any moment: pause it, correct it, take over, or hand back.

---

## Feature Summary

### Feature Set

| # | Feature | What it does |
|---|---------|-------------|
| 1 | [**App Ecosystem**](#1-app-ecosystem) | Apps built in Google AI Studio are uploaded to the Cowork.ai desktop app and rendered inside it. Apps get tools, not agents — they access platform capabilities through platform-provided MCPs (connected to Activity Capture data and connected services). The platform agent stays as the single orchestrator. |
| 2 | [**Execution Modes**](#2-execution-modes) | Agents + custom tools execute through two paths: Tools → MCP (fast, background) or Tools → Browser (visible, coachable). Configurable per app/category/action. |
| 3 | [**Activity Capture & Context Engine**](#3-activity-capture--context-engine) | Six input streams: window/app tracking, keystroke/input capture, focus detection, browser activity, screen recording, clipboard monitoring. Local SQLite storage. This data is exposed to apps via platform-provided MCPs. Most streams off by default. |
| 4 | [**Proactive Suggestions**](#4-proactive-suggestions) | The platform watches captured activity and connected service signals, then surfaces native macOS notifications when something is worth acting on — an insight, an offer to help, or a ready-to-execute action. Accepting a suggestion hands off to Execution Modes. Platform-level only; third-party apps cannot trigger notifications directly. |
| 5 | [**On-Demand Chat**](#5-on-demand-chat) | Conversational interface where the AI already has your work context — activity history, connected services, current screen. Zero-context queries, cross-service search, current-screen awareness. Chat naturally escalates to execution via Execution Modes. Entry point from Proactive Suggestions. Platform-level. |
| 6 | [**Memory**](#6-memory) | Four-layer memory system covering both captured activity data and conversations: conversation history (short-term), working memory (persistent user profile), semantic recall (RAG over embedded activity + conversation data), observational memory (background compression of raw history into dense logs). All on-device. |


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

**Data flow:** Raw capture → local SQLite (GRDB, WAL mode) → local processing → structured context (embeddings via local RAG) → agents query at action time. All on-device. Processing pipeline that discards raw input is not yet implemented — raw data currently persists locally. See Memory (feature 6) for the full pipeline: how raw activity data gets embedded, stored in the vector store, and retrieved via RAG at query time.

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

### 5. On-Demand Chat

Activity Capture collects context. Apps act on it. Proactive Suggestions surface it. But none of those give the user a way to **pull** — to just ask a question and get an answer from an AI that already knows their work. On-Demand Chat is that interface.

The AI already has your context — from Activity Capture (what you've been working on), connected services (your tickets, emails, threads), and Memory (what it's learned about you over time). You don't copy-paste. You don't explain background. You ask, it knows.

**What it draws from:**

- **Activity Capture data** (feature 3) — what you've been working on, how long, where. Retrieved via RAG from embedded activity data (see Memory, feature 6).
- **Connected services** via MCP — Zendesk tickets, emails, Slack threads, calendar events. Queried live.
- **Current-screen context** — active app + window title from Activity Capture's window tracking. "Summarize this" works because the AI knows what's on your screen.
- **Memory** (feature 6) — past conversations, your preferences, communication patterns, learned context.

**Key behaviors:**

- **Zero-context queries** — "What was that refund ticket I was looking at yesterday?" just works. The AI searches embedded activity data (RAG) to find the matching window title, URL, and timestamp — then pulls the ticket details from Zendesk via MCP.
- **Cross-service search** — "Find where we discussed the API rate limit issue" searches across embedded activity data and connected services simultaneously.
- **Current-screen awareness** — "Summarize this" when you're on a Zendesk ticket. The AI reads window context from Activity Capture.
- **Chat → Action bridge** — conversation naturally escalates to execution. The AI drafts a reply → proactively suggests sending → hands off to Execution Modes. The AI offers agentic next steps ("Want me to send this?" / "Should I batch-close these?") in addition to responding to direct requests.
- **Entry point from Proactive Suggestions** — accepting or expanding a notification (feature 4) opens a chat thread with that context pre-loaded. You can ask questions and refine before acting.

**Example interactions:**

| You say | What happens |
|---------|-------------|
| "What was that refund ticket I was looking at yesterday?" | RAG search on embedded activity data finds the window title + URL → pulls ticket details from Zendesk via MCP |
| "Summarize my last 3 Slack threads with Sarah" | Queries connected Slack via MCP, summarizes |
| "What have I spent most of my time on today?" | Queries activity capture data, gives time breakdown |
| "Draft a reply to this email — same tone as my last few" | Reads current screen context + email history + communication patterns from memory, drafts in your voice |
| "Find where we discussed the API rate limit issue" | RAG search across embedded activity data + connected services |
| "Batch-close these 4 tickets with a thank-you message" | Chat → action handoff to Execution Modes |

**Platform-level:** One global chat across all connected services and activity data. Apps don't have their own chat interfaces; they surface status in the sidesheet.

**UI:** Not decided. Could be keyboard shortcut (Spotlight-style), sidesheet panel, floating window, or a combination. To be determined during design phase.

---

### 6. Memory

Activity Capture (feature 3) observes what you're doing. On-Demand Chat (feature 5) lets you ask questions. But without memory, every interaction starts from scratch — the AI forgets what it observed yesterday, what you discussed last week, and what it's learned about how you work. Memory is what makes the AI get smarter over time.

**Two data sources, one retrieval system:**

Memory doesn't just cover chat conversations. It covers **everything the AI has observed** — both captured activity data and conversation history. Both feed into the same pipeline.

- **Source 1: Captured activity data** (from Activity Capture, feature 3) — window titles, app names, browser URLs, focus sessions, keystroke/input patterns (opt-in), browser session actions, clipboard content (opt-in), screen recordings (opt-in).
- **Source 2: Conversation data** (from On-Demand Chat, feature 5) — user messages and AI responses, tool results and action outcomes, coaching interventions during execution.

Both sources flow through the same pipeline: raw data → local storage → embeddings → vector store → retrievable via RAG.

**Data pipeline:**

```
Captured Activity Data (feature 3)     Conversation Data (feature 5)
              │                                    │
              ▼                                    ▼
         ┌──────────────────────────────────────────────┐
         │          Local Storage (on-device DB)         │
         │          Raw data persisted locally            │
         └────────────────────┬─────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┬────────────────────┐
          ▼                   ▼                   ▼                    ▼
   Conversation         Embedding           Observational          Working
   History              Pipeline            Memory                 Memory
          │                   │                   │                    │
          ▼                   ▼                   ▼                    ▼
   Recent N messages    Local embedding     Background agents     Structured user
   kept in context      model generates     observe + compress    profile updated
   for short-term       vector represen-    raw history into      from conversation
   continuity           tations → stored    dense observation     + activity signals
                        in local vector     logs. Raw data
                        store, indexed      replaced over time;
                        for similarity      observations persist
                        search (RAG)        + get embedded too
```

**Four memory layers:**

| Layer | What it stores | Sources | Lifespan | Retrieval |
|-------|---------------|---------|----------|-----------|
| **Conversation History** | Recent messages from the current chat thread | Conversations | Current session | Direct — last N messages loaded into context |
| **Working Memory** | Structured user profile — name, preferences, goals, communication style, project context | Conversations + activity patterns | Persistent (until user edits/deletes) | Direct — always loaded into context |
| **Semantic Recall** | Past conversations and captured activity data | Both sources | Long-term | RAG — query embedded, vector similarity search retrieves relevant past context |
| **Observational Memory** | Dense compressed logs from raw conversation and activity history | Both sources | Long-term | Background compression; retrievable via semantic recall |

**How the layers work together:**

1. **Conversation History** gives the AI short-term continuity — what you just said.
2. **Working Memory** gives it persistent knowledge about you — who you are, how you work.
3. **Semantic Recall** lets it find relevant past conversations *and* activity — what you discussed before, what you were working on last week.
4. **Observational Memory** keeps long-term memory manageable — compress, don't delete.

**How retrieval works at query time (RAG):**

1. User asks a question in On-Demand Chat.
2. The query is embedded using the local embedding model.
3. Vector similarity search finds the most relevant past messages, activity data, and observations across all history.
4. Retrieved context is assembled with priority: current conversation history (highest) → working memory profile → semantically recalled past exchanges and activity → observational summaries (lowest).
5. Assembled context is fed to the model alongside the user's query.
6. If combined context exceeds the model's window, memory processors trim lowest-priority content. Trimmed content isn't lost — it stays in the vector store for future retrieval.

**Relationship to Activity Capture (feature 3):**

| | Activity Capture (feature 3) | Memory (feature 6) |
|---|---|---|
| **Role** | The **collection** layer — observes and stores raw activity data | The **intelligence** layer — embeds, compresses, and retrieves data from both activity capture and conversations |
| **Data flow** | Raw capture → local SQLite → structured context | Raw data (from both sources) → embeddings → vector store → RAG retrieval |
| **Retrieval** | MCP resources queried by apps and agents (structured queries) | RAG pipeline at query time (semantic similarity search) |

Activity Capture is the input. Memory is what makes that input searchable and persistent across sessions.

**Where raw data lives:**

All on-device:

- **Raw activity data** — local database (same local SQLite as Activity Capture).
- **Raw conversation messages** — local database.
- **Embeddings / vector store** — local vector database. Embeddings generated by local model, stored locally, searched locally. Nothing leaves the device.
- **Working memory profile** — local database (structured data, not vectors).
- **Observational memory logs** — local database (compressed summaries) + local vector store (embedded for semantic search).

No memory data leaves the device unless a query requires cloud inference — in which case only the assembled context for that specific query is sent transiently (zero-retention providers), not the full memory store.

**Context management:**

When combined memory exceeds the model's context limit, memory processors filter, trim, and prioritize. Priority order: (1) current conversation history, (2) working memory profile, (3) semantically recalled past context, (4) observational summaries. Trimmed content is not deleted — it remains in the vector store and can be retrieved by future queries. The user never needs to think about token limits.

**Privacy:** All memory is on-device. Users can view, edit, or delete any memory layer. Working memory profile is fully transparent — you can see exactly what the AI "thinks" it knows about you and correct it. Observational memory logs are viewable. Delete = gone — no hidden copies, no cloud backup.

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
| 2026-02-13 | Added features 5–6: On-Demand Chat (zero-context conversational interface, chat → action bridge, entry point from Proactive Suggestions) and Memory (four-layer system — conversation history, working memory, semantic recall via RAG, observational memory — covering both activity capture data and conversations). Updated Privacy section with memory data types. |
| 2026-02-13 | Added fourth feature: Proactive Suggestions — native macOS notifications triggered by activity patterns, incoming signals, time-based context, state transitions, and thresholds. Includes throttling model (priority tiers, hourly caps, flow-state suppression, dismissal learning, bundling). Platform-level only. |
| 2026-02-13 | Scoped down to three core features: App Ecosystem, Execution Modes, Activity Capture. Removed Approval Queue, Clone Mode, Task Automation, Live Agent, Context Card, Autonomy Levels, User Stories. |
| 2026-02-13 | Initial draft. |
