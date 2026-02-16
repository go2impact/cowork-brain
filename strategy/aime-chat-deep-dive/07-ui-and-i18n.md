# 7. Modern UI & Internationalization

[< Back to index](README.md)

## Modern UI — shadcn/ui

**Their claim:** "Built with shadcn/ui component library, supports light/dark theme switching"

**Maps to Cowork.ai:** Our M3 design system + Tailwind. Different component library, same rendering approach.

### Stack

| Layer | Their Choice | Our Choice |
|-------|-------------|------------|
| Component library | shadcn/ui (Radix primitives) | M3 (Material Design 3) |
| Styling | Tailwind CSS 4.1 | Tailwind CSS |
| State management | Zustand v5 + React Context | TBD |
| Routing | React Router v7 | TBD |
| Markdown rendering | react-markdown + shiki | TBD |
| Animation | Motion v12 (formerly Framer Motion) | TBD |
| Drawing | Excalidraw + tldraw | Not planned |
| Charts | Recharts v2.15 | TBD |
| Icons | Lucide React + Tabler Icons | TBD |

### UI Layout

```
┌──────────────────────────────────────────────────────────────┐
│  AppSidebar               │  Main Content Area               │
│  ┌──────────────────────┐ │  ┌──────────────────────────────┐│
│  │ Navigation            │ │  │ Breadcrumb nav               ││
│  │ ├─ Chat              │ │  │                               ││
│  │ ├─ Knowledge Base    │ │  │ ┌──────────────────────────┐ ││
│  │ ├─ Agents            │ │  │ │  Chat Conversation       │ ││
│  │ ├─ Tools             │ │  │ │  ├─ Message (content     │ ││
│  │ ├─ Projects          │ │  │ │  │   + attachments)       │ ││
│  │ └─ Settings          │ │  │ │  ├─ Reasoning (collapsible│ ││
│  │                       │ │  │ │  ├─ Tool invocations     │ ││
│  │ Task Manager (badge) │ │  │ │  ├─ Sources/citations    │ ││
│  └──────────────────────┘ │  │ │  └─ Context usage (tokens)│ ││
│                            │  │ └──────────────────────────┘ ││
│  (resizable panels)       │  │                               ││
│                            │  │ Input bar + model selector   ││
│                            │  └──────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### What We Should Do

| Action | Detail |
|--------|--------|
| **Study** | Their chat UI components — message rendering, streaming text display, tool invocation panels, reasoning collapsible sections. These are hard UI problems they've solved. |
| **Study** | Zustand for state management — lightweight, no boilerplate, works well with Electron IPC. Strong candidate for our state management choice. |
| **Skip** | shadcn/ui specifically — we're committed to M3. But the Radix primitives underneath are worth knowing about. |
| **Skip** | Excalidraw, tldraw, Recharts — not relevant for our UX. |
| **Note** | Their dark-first approach matches ours. |

---

## Internationalization

**Their claim:** "Built-in Chinese and English interfaces"

**Maps to Cowork.ai:** Not a current priority (English-first).

### Stack

- `i18next` + `react-i18next`
- Chinese (primary), English (secondary)
- Translation caching in DB (`translations` table)
- Built-in translation tool for on-the-fly translation

### What We Should Do

| Action | Detail |
|--------|--------|
| **Study (later)** | Their i18next setup if we ever need i18n. Standard approach, nothing novel. |
| **Skip for now** | Not relevant for v0.1. English-only. |
