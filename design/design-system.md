# Cowork.ai Design System

**Version:** 1.0  
**Last Updated:** 2026-02-12  
**Status:** Active — all UI work must follow this spec

---

## Why Material Design 3

This is not a style preference. It's an architecture decision.

**The core problem:** Cowork.ai is a platform where users build and install custom apps via MCP. Those apps generate UI. AI generates UI. Third-party developers generate UI. If every surface uses its own design language, the product becomes incoherent within weeks.

**Why M3 specifically:**

1. **AI needs rigid systems.** When you ask an LLM to "build me a dashboard widget," it needs a design system with named tokens, not vibes. M3 has the most complete, publicly documented token system in existence. Every color, elevation, shape, and motion has a name and a spec. That means AI-generated components look native by default.

2. **Google maintains it.** The system is actively maintained, has official Figma kits, CSS token libraries, and web component implementations. We're not maintaining a custom design system with a 3-person team.

3. **Tailwind compatibility.** M3 tokens map cleanly to Tailwind utility classes. Our entire frontend stack is Tailwind-based. Custom design systems fight Tailwind; M3 works with it.

4. **Developer familiarity.** Any developer who's built an Android app or used a Google product knows M3 patterns. Lower onboarding cost for contributors and third-party app developers.

5. **Dark theme is first-class.** M3 has a complete dark theme token system, not a bolted-on afterthought. Cowork.ai is a desktop sidecar — dark theme is the default, not an option.

**What M3 does NOT mean:** We're not building a Google app. We don't use Material Components verbatim. We use M3's token system, color roles, elevation model, and motion principles. The actual components are custom, built in React + Tailwind, following M3 specs.

---

## The Interaction Model

Source: `cowork_ai-sidecar---fav` (the "fav" prototype)

This is the interaction model. Every prototype and production UI must follow this three-state pattern.

### State 1: Closed (Menu Bar Only)

The sidecar is not visible. The only indicator is a small trigger button in the OS menu bar (macOS top bar or Windows system tray equivalent).

```
┌─────────────────────────────────────────────────┐
│ Finder  File  Edit  View     🔋 📶  2:34  [COWORK] │  ← menu bar trigger
├─────────────────────────────────────────────────┤
│                                                   │
│              User's desktop / apps                │
│              (fully unobstructed)                  │
│                                                   │
└─────────────────────────────────────────────────┘
```

**Trigger button behavior:**
- Small pill shape, ~28px height
- Contains status dot (pulsing when active, dim when idle) + "COWORK" label
- Click toggles to State 2
- Keyboard shortcut: `Cmd+K` (Mac) / `Ctrl+K` (Windows)
- When sidecar is open: button gets primary-container background with glow

**Design principle:** When closed, Cowork.ai does not exist visually. No floating buttons, no dock icons, no persistent overlays. The worker's screen is theirs.

### State 2: SideSheet Open

A 360–400px panel slides in from the right edge. This is the primary interface — a scrollable dashboard of cards and actions.

```
┌───────────────────────────────────┬──────────────┐
│                                   │              │
│     User's desktop / apps         │  SideSheet   │
│     (partially visible)           │  360-400px   │
│                                   │              │
│                                   │  [widgets]   │
│                                   │  [actions]   │
│                                   │  [chat fab]  │
│                                   │              │
└───────────────────────────────────┴──────────────┘
```

**SideSheet behavior:**
- Fixed to right edge, full height minus menu bar
- Slides in with spring animation (stiffness: 300, damping: 30)
- Contains scrollable content area
- Rounded corners (24px on left side, or 28px all corners with 4px gap from edge)
- M3 surface-container background
- Elevation 2 shadow

**SideSheet content (from top):**
- Greeting + contextual status line
- Widget grid (2-column for small widgets, full-width for larger ones)
- Scrollable list of active apps, context, feeds
- Chat trigger FAB at bottom (opens to State 3)

### State 3: SideSheet + Detail Canvas

Clicking any widget card or the chat trigger opens a large detail view that fills the space to the LEFT of the sidesheet. The sidesheet remains visible for navigation.

```
┌──────────────────────────────────┬──────────────┐
│                                  │              │
│         Detail Canvas            │  SideSheet   │
│         (fills remaining space)  │  (stays)     │
│                                  │              │
│   Full chat, app view,           │  Widgets     │
│   MCP browser, settings, etc.    │  still       │
│                                  │  visible     │
│                                  │              │
└──────────────────────────────────┴──────────────┘
```

**Detail Canvas behavior:**
- Animates in with spring (same parameters) from slight offset
- Has its own header with icon, title, and close button
- Close button returns to State 2 (sidesheet only)
- Content is view-specific (chat, app detail, MCP browser, etc.)
- M3 surface-container background, rounded corners, elevation 3

**Why this model works:**
- State 1 = zero intrusion. Worker doesn't see us unless they want to.
- State 2 = quick glance. Check status, see context, scan feeds. 5 seconds.
- State 3 = deep engagement. Chat with AI, configure an app, watch an MCP agent work. Minutes.
- The sidesheet in State 3 acts as persistent navigation — you can click a different widget without closing and reopening.

---

## The Visual System

Source: `cowork_ai-sidecar---best-looking-base-ui` (the "best looking" prototype)

### Color Tokens (M3 Dark Theme)

All colors use M3 semantic roles, not raw hex values. This is mandatory — no hardcoded colors in components.

```
Surface layer (backgrounds):
  --surface:                 #141218
  --surface-container:       #1D1B20
  --surface-container-high:  #2B2930
  --surface-container-highest: #36343B

Content (text & icons):
  --on-surface:              #E6E1E5
  --on-surface-variant:      #CAC4D0
  --outline:                 #938F99

Primary (brand accent):
  --primary:                 #D0BCFF
  --on-primary:              #381E72
  --primary-container:       #4F378B
  --on-primary-container:    #EADDFF

Secondary:
  --secondary-container:     #4A4458
  --on-secondary-container:  #E8DEF8

Tertiary (status/emphasis):
  --tertiary:                #EFB8C8
  --on-tertiary:             #492532
```

**Brand gradient:** `linear-gradient(to bottom-right, var(--primary), var(--tertiary))` — used only on the brand icon and key accent moments. Not splashed everywhere.

### Typography

- **Font:** Inter (primary), system sans-serif fallback
- **Scale:** M3 type scale mapped to Tailwind classes
- **Hierarchy:** Headlines are `font-bold`, body is `font-normal`, labels are `font-medium uppercase tracking-wider text-xs`
- **Color hierarchy:** `on-surface` for primary text, `on-surface-variant` for secondary, `outline` for tertiary/timestamps

### Elevation & Surfaces

M3 dark theme uses tonal elevation (lighter surface = higher elevation), not drop shadows alone.

| Level | Surface | Use |
|-------|---------|-----|
| 0 | `surface` (#141218) | App background, behind sidesheet |
| 1 | `surface-container` (#1D1B20) | Navigation rail, secondary panels |
| 2 | `surface-container-high` (#2B2930) | Cards, widgets, interactive elements |
| 3 | `surface-container-highest` (#36343B) | Hover states, selected items |

Shadows are supplementary: `shadow-elevation-2` for the sidesheet, `shadow-elevation-3` for the detail canvas. But tonal shift does the heavy lifting.

### Shape

- **Cards/Widgets:** 20–24px border radius (M3 "extra large" shape)
- **Buttons/Pills:** Full radius (9999px) for FABs and pills, 16px for action cards
- **Navigation items:** 16px radius for rail indicators
- **Input fields:** 24px radius (rounded-3xl)
- **Detail Canvas:** 24px radius

### Motion

All animations use spring physics, not easing curves.

```
Standard spring: { type: "spring", stiffness: 300, damping: 30 }
```

- **SideSheet enter/exit:** translateX with spring
- **Detail Canvas enter/exit:** opacity + slight translateX + scale(0.95→1.0)
- **Widget hover:** scale(1.02), whileTap scale(0.98)
- **Progress bars:** duration 1s with staggered delay per item
- **State transitions:** 300ms for color/opacity, spring for position/size

### Borders

Borders are almost invisible — they're structural, not decorative.

- `border border-white/5` — standard card border
- `border-primary/20` — active/selected state
- `border-transparent` — default state (preserves layout, no visible border)

---

## Feature Set (What the App Actually Does)

This is where every previous prototype gets it wrong. Cowork.ai is NOT a time tracker. It's NOT a focus score dashboard. It's an AI-powered desktop agent with an app platform.

The sidesheet and detail canvas must expose these features:

### 1. Chat (AI Assistant)

The user talks to their local AI (DeepSeek-R1 running locally, or cloud fallback). This is the primary interaction surface.

**What it knows:**
- Current application context (what apps are open, what files are active)
- Connected MCP server capabilities (what tools are available)
- Installed app data (CRM records, ticket queues, email summaries)
- Behavioral context (work patterns, not surveillance — "you usually do deep work mornings")

**What it does:**
- Answers questions using local context + tool calls
- Executes actions via MCP (send email, update ticket, search docs)
- Suggests next actions based on context
- Runs multi-step workflows when asked

**In the SideSheet:** Compact trigger card at bottom ("Ask your AI coworker...")  
**In the Detail Canvas:** Full chat interface with message history, tool use indicators, and inline results

### 2. Apps (Installed MCP Apps)

These are not widgets. They are MCP-connected applications that provide both data and actions.

**Examples of REAL apps (not hypothetical):**
- **Zendesk Agent** — Shows ticket queue, lets AI draft responses, auto-categorizes
- **Gmail Inbox** — Summarized inbox, draft replies, schedule sends
- **Slack Digest** — Unread channel summaries, quick reply, thread context
- **Google Calendar** — Today's schedule, meeting prep, conflict detection
- **Linear/Jira** — Sprint board, ticket details, status updates via chat
- **Salesforce CRM** — Pipeline view, contact lookup, activity logging
- **Custom App** — User-built via MCP server, any data source, any action

**In the SideSheet:** Each installed app gets a compact card showing its current state (unread count, next event, queue depth, pipeline value). Clicking opens Detail Canvas with full app view.  
**In the Detail Canvas:** Full app interface — the Zendesk app shows the ticket queue with AI-suggested responses. The Calendar app shows the full day view. Etc.

### 3. App Gallery (Discovery & Install)

Where users find and install new apps. Think iOS App Store, not a settings page.

**Categories:** Productivity, Communication, CRM, Developer Tools, Custom/Community  
**Each listing shows:** Icon, name, description, what MCP servers it connects to, install button  
**After install:** App appears in sidesheet, begins syncing via MCP

**In the SideSheet:** "Browse Apps" card or link  
**In the Detail Canvas:** Full gallery with categories, search, featured apps, install flow

### 4. MCP Browser (Watch AI Work)

This is unique to Cowork.ai. When the AI executes a multi-step task via MCP, the user can watch it happen in real time.

**What it shows:**
- Current tool call chain (step 1: searched Zendesk, step 2: found ticket #4521, step 3: drafting response...)
- Each step's input and output
- Approval gates for sensitive actions ("About to send email to client. Approve?")
- Execution history (what the AI did while you weren't looking)

**Why it matters:** This is the trust mechanism. Workers see exactly what the AI is doing. No black box. This is how you get 92% consent rates — transparency, not surveillance.

**In the SideSheet:** Small "AI Activity" indicator showing current/recent actions  
**In the Detail Canvas:** Full execution log with step-by-step visibility

### 5. Tools & Connections

Settings/configuration view for connected MCP servers and available tools.

**What it shows:**
- Connected MCP servers (Zendesk, Gmail, Slack, etc.) with status
- Available tools per server (list_tickets, send_email, search_channels)
- Connection health, last sync time
- Add new connection flow

**In the SideSheet:** Doesn't need a dedicated card — accessed via settings or app management  
**In the Detail Canvas:** Connection management interface

### 6. Automations

Rules and workflows that run without user intervention.

**Examples:**
- "When a high-priority Zendesk ticket arrives, draft a response and notify me"
- "Every morning at 9am, summarize my unread emails and Slack messages"
- "When I start a Zoom call, mute Slack notifications"

**In the SideSheet:** "Active Automations" card showing what's running  
**In the Detail Canvas:** Automation builder/editor, execution logs

### 7. Context Card (NOT a Focus Score)

The replacement for the "Time Doctor" widgets. Instead of surveillance metrics, this shows **useful context**.

**What it shows:**
- What you're working on right now (active app, active document, active meeting)
- How long you've been in this context
- Relevant suggestions ("You have a meeting in 15 minutes" / "3 unread Slack messages in #engineering")

**What it does NOT show:**
- "Focus score" with a gamified number
- "Distractions" count
- "Productive vs unproductive" categorization
- Anything that frames the worker as a subject of measurement

**The distinction matters:** "You've been in VS Code for 2 hours working on the auth module" is useful context. "Your focus score is 87, you're in the top 10%" is gamified surveillance. We do the first one.

---

## Navigation Model

### SideSheet Navigation

The sidesheet is a scrollable list of cards, not a tabbed interface. Cards are grouped by type:

1. **Context card** (what you're working on now) — always at top
2. **Installed app cards** (compact state of each app) — scrollable middle
3. **Quick actions** (start automation, browse apps) — lower
4. **Chat trigger** (open AI chat) — always at bottom, sticky

Clicking any card opens the Detail Canvas for that item.

### Navigation Rail (Alternative)

For the detail canvas view, a thin navigation rail (56–80px) on the left edge of the detail canvas provides direct access to major sections:

- Chat
- Apps (installed)
- Gallery
- MCP Browser
- Automations
- Settings

This rail is only visible in State 3 (detail canvas open). It's inside the detail canvas, not a separate persistent element.

---

## Component Patterns

### Card (SideSheet Widget)

```
┌──────────────────────────────────┐
│  ● SECTION LABEL                 │  ← dot + uppercase label
│                                  │
│  [Content: metrics, list, etc.]  │
│                                  │
└──────────────────────────────────┘
```

- Background: `surface-container-high`
- Border: `border-white/5` (transparent when not hovered)
- Border on active: `border-primary`
- Radius: 20–24px
- Padding: 16–20px
- Hover: slight scale (1.02) + lighter background
- Click: slight compress (0.98) + opens Detail Canvas

### Chat Message Bubble

- **AI messages:** `surface-container-high` background, left-aligned, rounded with flat bottom-left
- **User messages:** `primary` background with `on-primary` text, right-aligned, rounded with flat bottom-right
- Avatar: 32px circle, AI gets brand gradient, user gets secondary-container

### App Card (Gallery Listing)

- 48px icon with rounded corners (12px)
- Icon background uses app-specific color at 20% opacity
- Title, one-line description, MCP server badge
- "Install" button: primary text on primary/10 background, pill shape

### Input Field

- Background: `surface-container-high`
- Border: `border-white/5`, focus: `border-primary/50`
- Rounded: 24px (3xl)
- Send button: circle, primary background

---

## What This Doc Doesn't Cover (Yet)

- **Responsive behavior:** How the sidecar adapts to different screen sizes and monitor configurations
- **Multi-monitor:** Behavior when user has multiple displays
- **Onboarding flow:** First-run experience, MCP server connection setup
- **Notification system:** How alerts surface when sidecar is closed
- **Accessibility:** Keyboard navigation, screen reader support, contrast ratios
- **Light theme:** M3 supports it, but it's not priority. Dark first.
- **Windows/Linux:** Interaction model assumes macOS menu bar. Windows system tray and Linux equivalents need separate consideration.

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-12 | Initial design system. M3 decision, interaction model, visual system, correct feature set. |

**See also:** [Product Overview](../product/product-overview.md) · [Prototype Brief](prototype-brief.md) · [LLM Architecture Spec](../architecture/llm-architecture.md)
