# Cowork.ai — Product Overview

**Version:** 1.2
**Last Updated:** 2026-02-16
**Purpose:** What Cowork.ai is, what it does, who it's for, and where it's going. Start here if you're new.

---

## What Cowork.ai Is

Cowork.ai is a desktop AI agent for remote workers. It sits alongside their work applications as a lightweight sidecar, sees what they're working on (with their consent), and helps them do more of it — faster.

It's not a chatbot you copy-paste from. It's not a time tracker that measures how productive you are. It's an AI coworker that can actually do things: draft a Zendesk response, summarize unread Slack channels, triage your inbox, prepare you for your next meeting — inside the real tools you use, not in a separate window.

The core insight: remote workers — especially in customer experience, virtual assistance, and operations — do a huge amount of repetitive, pattern-based work that AI can handle. But they can't use ChatGPT effectively because it doesn't know their context: their tickets, their SOPs, their calendar, their active conversations. Cowork.ai knows all of that because it runs on their machine and connects to their tools via MCP (Model Context Protocol).

---

## How It Works

### The Two-Brain Architecture

The worker sees one AI assistant. Under the hood, there are two inference paths:

**Local brain** — DeepSeek-R1 runs directly on the user's machine. Handles ~70% of daily work: ticket summaries, draft responses, email triage, SOP lookups. No cost per query. No latency. No data leaves the device. Requires 16GB+ RAM (Apple Silicon preferred).

**Cloud brain** — Smarter models (Gemini 2.5 Flash on free tier, up to Claude Opus 4.6 on top tier) handle complex tasks: long documents, multi-step workflows, nuanced analysis. Metered by usage, quality scales with what the user pays.

The user chooses at onboarding: "Run on my machine" (free, private, uses RAM) or "Run in the cloud" (faster, costs tokens). Can switch anytime. Users with lower-spec machines default to cloud-only.

### The Desktop Sidecar

Cowork.ai runs as an Electron desktop application, not a browser tab. It presents as a sidecar panel that slides in from the right edge of the screen. Three states:

1. **Closed** — Only a small trigger in the menu bar. The worker's screen is fully theirs.
2. **SideSheet** — A 360px panel showing installed app states, current context, and a chat trigger. Quick glance, 5 seconds.
3. **Full workspace** — The sidesheet stays; a detail canvas opens to the left for deep engagement: full chat, app views, MCP execution browser, automations.

### MCP Integration

Cowork.ai connects to work applications through MCP (Model Context Protocol) servers. Each connected app (Zendesk, Gmail, Slack, Salesforce, Linear, etc.) exposes its data and actions to the AI agent. This means the AI can:

- **Read:** See your ticket queue, email inbox, Slack channels, calendar
- **Write:** Draft responses, update ticket status, schedule meetings, send messages
- **Act:** Execute multi-step workflows (triage → draft → send → log)

Workers can also build custom MCP integrations for tools specific to their workflow.

---

## What Users Can Do

### Install and use apps
Browse the App Gallery and install MCP-connected applications: Zendesk, Gmail, Slack, Calendar, CRM, and more. Each app shows a compact state in the sidesheet (unread count, queue depth, next event) and opens to a full view for deep interaction. Power users can build custom apps via Google AI Studio.

### Connect services (MCP Integrations)
Connect your work tools — Zendesk, Gmail, Slack, and more — via MCP. Manage connections, monitor health, and configure what the AI can access. This is the foundation layer that apps and chat sit on top of.

### Chat with an AI coworker
Talk to the AI assistant. It knows your current context — what apps are open, what you're working on, what's in your queue. Ask it to do things, not just answer questions. The AI also comes to you: proactive notifications surface insights and actions when something is worth your attention.

### Watch the AI work (MCP Browser)
When the AI executes multi-step tasks, watch every tool call in real time. See what it searched, what it found, what it drafted, what it's waiting for approval on. Full transparency — this is how we build trust.

### Build automations
Create rules that run without intervention: "When a high-priority ticket arrives, draft a response and notify me." "Every morning at 9am, summarize my unread messages." Automations are logged and auditable.

### Context that works for you
The AI observes what you're working on (with your consent), remembers your preferences and past conversations, and shows your current work context at a glance via the Context Card — never a focus score or productivity metric.

---

## Who It's For

### Primary: Remote workers in customer-facing roles

Filipino remote workers, Latin American VAs, Indian customer support agents — anyone doing remote work for international clients. Specifically:

- **CX agents** handling Zendesk/Freshdesk/Intercom ticket queues
- **Virtual assistants** managing email, calendar, and communications
- **Operations specialists** doing data entry, reporting, and process work
- **Dev support** handling bug reports, documentation, and QA

These workers earn $500–1,500/month and are price-sensitive. $10/month is a meaningful investment. The ROI pitch: spend $10, earn $200 more by handling greater volume.

### Secondary: The businesses that employ them

Go2 has sold $60M+ in staffing services to SMBs over 8 years. Thousands of these businesses are in our CRM. When their best workers are already more productive with Cowork.ai, the sell to employers is: "Give the whole team the same advantage." Pricing shifts from personal ($10–50/month) to enterprise ($100–400/seat/month).

### Future: Enterprise workforce intelligence

Large deployments where Cowork.ai becomes the productivity analytics layer — with worker consent and data ownership built in. 92% worker consent rates (proven at Go2) because workers own their data and see the value exchange.

---

## Where It's Going

### Now (Q1 2026): Local-first MVP

Ship the desktop sidecar with DeepSeek-R1 local brain, basic MCP integrations (Zendesk, Gmail, Slack), and the free cloud tier. Get it into the hands of 50–500 workers from the Go2 database who have the right hardware. Learn what works.

### Next (Q2–Q3 2026): App platform + paid tiers

Open the app gallery. Let workers install and configure their own MCP connections. Launch Boost ($5–15/month) and Pro ($30–60/month) tiers. Begin B2B outreach to Go2 enterprise customers.

### Later (2026–2027): Workforce intelligence platform

Aggregate anonymized, consented behavioral data into workforce analytics. Show businesses how their teams actually work (not surveillance — productivity patterns, tool usage, workflow bottlenecks). This is the long-term SaaS play that scales beyond individual subscriptions.

### The Thesis

AI inference costs drop ~50% every 6–12 months. The cost of making a worker 2x productive approaches zero. The value isn't in the model — it's in the distribution (we have 2M+ worker profiles), the trust relationship (92% consent rates), and the behavioral data (billions of work events processed through Go2 over 8 years). Whoever owns the desktop agent that remote workers use daily captures that value.

---

## How Cowork.ai Relates to Go2

Cowork.ai was born from Go2's internal workforce analytics platform. Go2 is an 8-year-old subscription staffing company that has placed 1,000+ workers across 51 countries. Over that time, Go2 built systems to understand how remote workers actually work — what tools they use, how they spend their time, where they get stuck.

Cowork.ai takes that intelligence and puts it in the worker's hands, not the employer's dashboard. Instead of a surveillance tool that tracks workers for their bosses, it's a productivity tool that empowers workers to earn more. The business model works because empowered workers are more productive workers — and businesses pay more for productive workers.

Go2's distribution assets (2M+ profiles, 250K LinkedIn followers, enterprise CRM, 51-country presence) give Cowork.ai a launch channel that most AI startups spend millions to build. The cost to reach our first 10,000 users is effectively the cost of sending emails.

---

## Key Documents

For deeper context on any area, see:

| Topic | Document | What It Covers |
|-------|----------|---------------|
| Technical architecture | [architecture/llm-architecture.md](../architecture/llm-architecture.md) | Models, routing, hardware requirements, infra |
| Business economics | [strategy/llm-strategy.md](../strategy/llm-strategy.md) | Pricing tiers, cost-to-serve, conversion theory, worker ROI |
| Design system | [design/design-system.md](../design/design-system.md) | M3 tokens, interaction model, visual system, feature set |
| Go-to-market | [gtm/launch-gtm-strategy.md](../gtm/launch-gtm-strategy.md) | Distribution channels, phased rollout, messaging |
| Decision history | [decisions/decision-log.md](../decisions/decision-log.md) | Every major decision with full reasoning |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.2 | 2026-02-16 | Fixed desktop framework reference to Electron. |
| 1.1 | 2026-02-16 | Updated "What Users Can Do" to reflect all 6 product features (Apps, MCP Integrations, Chat, MCP Browser, Automations, Context). |
| 1.0 | 2026-02-12 | Initial product overview. |
