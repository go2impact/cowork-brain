# Prototype Brief: Cowork.ai Clickable Demo v0.1

**Version:** 0.1  
**Last Updated:** 2026-02-12  
**Purpose:** Investor/partner demo, internal alignment, and system prompt reference for AI-generated rebuilds

---

## What This Prototype Is

A clickable demo that shows the Cowork.ai interaction model and feature set. It runs in a browser but simulates a desktop sidecar experience. Nothing connects to real APIs — all data is mocked, all AI responses are scripted.

**The demo must answer one question:** "What does this product actually do?"

Every previous prototype failed this test. They showed focus scores and activity timelines — things that describe Time Doctor, not an AI desktop agent with an app platform. This prototype shows the real product.

---

## Technical Stack

- **React + TypeScript** (single-file JSX artifact or Vite project)
- **Tailwind CSS** with M3 custom token config
- **Framer Motion** for spring animations
- **Lucide React** for icons
- No backend. No API keys. All interactions are client-side with mocked data.

For Claude Code or AI rebuild: read `design/design-system.md` first. Every visual decision is documented there.

---

## Demo Flow (What the User Clicks Through)

### Screen 1: Closed State

Shows a simulated macOS desktop with a menu bar. The only Cowork.ai element is the trigger button in the top-right of the menu bar.

**User action:** Click the COWORK trigger button.

### Screen 2: SideSheet Opens

The sidesheet slides in from the right (spring animation). Shows:

1. **Context Card** (top) — "Working in Zendesk · 47 minutes"  
   Not a focus score. Just what the user is doing right now and for how long.

2. **Installed App Cards** (2-column grid + full-width):
   - **Zendesk** — "12 open tickets · 3 high priority" with small queue indicator
   - **Gmail** — "5 unread · 1 flagged from VP of Sales" 
   - **Slack** — "2 unread channels · mentioned in #engineering"
   - **Calendar** — "Next: Design sync in 25 min"

3. **AI Activity Indicator** — "Drafted 2 ticket responses · Waiting for approval"  
   Shows the AI has been working in the background.

4. **Chat Trigger** (bottom, sticky) — "Ask your AI coworker..."

**User action:** Click the Zendesk app card.

### Screen 3: Detail Canvas — Zendesk App

The detail canvas opens to the left. Shows the Zendesk app in full view:

- **Ticket queue** with priority indicators
- **Selected ticket** shows customer message + AI-drafted response
- **Approve / Edit / Reject** buttons on the draft
- Status line: "Drafted by DeepSeek-R1 (local) · 3 seconds"

This is the money shot for the demo. It shows the AI doing real work — not measuring the human, augmenting them.

**User action:** Click "Approve" on the draft, then click "Chat" in the sidesheet or nav.

### Screen 4: Detail Canvas — Chat

Full chat interface. Pre-loaded conversation:

```
AI: Morning. You have 12 open Zendesk tickets. I've drafted responses for the 3 
    high-priority ones. Want me to walk through them?

User: Just handle the ones that are standard refund requests. Flag anything weird.

AI: Done. Approved and sent responses to tickets #4521 and #4523 (standard refund 
    flow). Ticket #4519 has an unusual request — customer wants to transfer their 
    subscription to a different company. Flagged for your review.
    
    [View ticket #4519 →]
```

The chat shows tool use indicators inline: `🔧 Searched Zendesk (12 tickets) → 🔧 Drafted response (#4521) → 🔧 Sent response`

**User action:** Click "App Gallery" via nav or a "Browse Apps" card.

### Screen 5: Detail Canvas — App Gallery

Shows available apps to install:

**Featured:**
- **Salesforce CRM** — "Pipeline view, contact lookup, activity logging via MCP"
- **Linear** — "Sprint board, ticket management, status updates"
- **Google Docs** — "Document editing, comment tracking, AI writing assistance"

**Categories:** Communication, CRM, Developer Tools, Productivity, Custom

Each app card shows: icon, name, one-line description, "Install" button.

Clicking "Install" on any app shows a brief connection flow (simulated), then the app card appears in the sidesheet.

**User action:** Click "MCP Browser" in nav.

### Screen 6: Detail Canvas — MCP Browser

Shows the AI's recent execution log:

```
Today · 2:34 PM
├─ User: "Handle standard refund requests"
├─ 🔍 zendesk.list_tickets(status=open, priority=high) → 3 results
├─ 📋 zendesk.get_ticket(4521) → Standard refund request
├─ ✏️ Drafting response using template: refund_approval
├─ ✅ zendesk.send_response(4521, draft) → Sent
├─ 📋 zendesk.get_ticket(4523) → Standard refund request  
├─ ✏️ Drafting response using template: refund_approval
├─ ✅ zendesk.send_response(4523, draft) → Sent
├─ 📋 zendesk.get_ticket(4519) → Subscription transfer (non-standard)
├─ ⚠️ Flagged for human review — no matching template
└─ 💬 Notified user in chat
```

This view is the transparency mechanism. Every tool call, every decision, visible.

---

## Mock Data Requirements

### Zendesk Tickets (3 high priority)
- #4521: "Refund for double charge" — standard, AI can handle
- #4523: "Cancel subscription and refund remaining month" — standard
- #4519: "Transfer subscription to sister company" — non-standard, flagged

### Gmail (5 unread)
- VP of Sales: "Q1 pipeline review deck needed by Friday"
- Support lead: "FYI: New Zendesk workflow deployed"
- 3 others (just subject lines visible)

### Slack (2 channels)
- #engineering: "@user Can you review the MCP auth PR?"
- #general: "Team lunch moved to Thursday"

### Calendar (3 events today)
- 10:00 AM — Team standup (completed)
- 2:30 PM — Design sync (in 25 min)
- 4:00 PM — 1:1 with manager

---

## What the Prototype Does NOT Include

- Real API connections (no Gemini, no OpenRouter, no MCP servers)
- Account creation or authentication
- Working AI chat (responses are pre-scripted)
- Data persistence (refresh = reset)
- Notification system
- Automation builder
- Settings or preferences
- Onboarding flow

These are all real features that will exist in the product. They don't belong in a v0.1 demo that needs to answer "what does this product do?"

---

## Build Instructions for AI (Claude Code / Claude Artifact)

1. Read `design/design-system.md` first — all visual decisions are there
2. Start with the three-state interaction model (closed → sidesheet → detail canvas)
3. Get the spring animations right before adding content
4. Mock data lives in a constants file, not inline
5. Each detail canvas view is a separate component
6. The sidesheet content scrolls; the chat trigger stays sticky at bottom
7. Navigation rail appears inside the detail canvas in State 3
8. Use M3 color tokens via Tailwind config — no hardcoded hex in components
9. Test the full click-through flow before declaring done

**Quality bar:** If someone clicks through and asks "so it's like Time Doctor with AI?" — the prototype failed. If they say "it's like having an AI assistant that can actually do things in my work apps" — it worked.

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-02-12 | Initial brief. Six-screen demo flow, mock data spec, build instructions. |
