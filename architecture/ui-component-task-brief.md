# Task Brief: shadcn/ui + M3 Component Library

| | |
|---|---|
| **Status** | Not started |
| **Last Updated** | 2026-02-17 |
| **Owner** | Frontend dev (TBD) |
| **Estimated Effort** | 3-5 days |
| **Depends on** | Nothing — fully parallelizable with backend sprints |
| **Deliverable** | `src/renderer/components/ui/` folder + Tailwind config + CSS variables + preview page |

---

## Context

Cowork.ai is a desktop AI assistant (Electron app) for remote workers. It runs as a sidecar panel alongside work apps. The backend team is building the Electron multi-process architecture (capture, agents, IPC) in parallel — this task is **UI-only** and has **zero backend dependencies**.

The app follows **Material Design 3 (M3)** as its design system, implemented via **Tailwind CSS** utility classes. We're using **shadcn/ui** as the starting point — copy the component source files, then restyle them with M3 tokens.

**You are NOT building components from scratch.** You are:

1. Setting up a Tailwind config with M3 design tokens
2. Creating CSS variable files for dark + light themes
3. Copying ~15 shadcn/ui components into the project
4. Replacing shadcn's default Tailwind classes with M3 token classes
5. Building a preview page to demonstrate all components

---

## Tech Stack (locked — do not change)

| Layer | Choice |
|---|---|
| Framework | React 19 + TypeScript |
| Bundler | Vite (will be electron-vite in production, but use standalone Vite for this task) |
| Component primitives | Radix UI (via shadcn/ui) |
| CSS | Tailwind 4 |
| Icons | Lucide React |
| Animation | Motion v12 (formerly Framer Motion) |

**Do NOT add:** Ant Design, MUI, Mantine, styled-components, CSS modules, Chakra, or any other component/styling library.

---

## How The Layers Work

Understanding this is critical to doing this task correctly:

```
Layer 1: Radix UI (npm dependency — DO NOT MODIFY)
  └── Provides: accessible behavior
      - Keyboard navigation (arrow keys, ESC, Enter, Tab)
      - Focus management (focus trapping in dialogs, focus return)
      - ARIA attributes (role="menu", aria-expanded, etc.)
      - Portal rendering (dropdowns render at document root)
      - Collision detection (menus reposition to stay on screen)
  └── Provides: ZERO visual styling

Layer 2: shadcn/ui (copied source files — you OWN these)
  └── Provides: pre-written React components that wrap Radix
  └── Uses: Tailwind classes for all styling
  └── Example: <DialogContent className="bg-white rounded-lg shadow-lg p-6">
  └── These are the files you modify

Layer 3: M3 token overrides (YOUR WORK)
  └── Replace shadcn's default Tailwind classes with M3 token classes
  └── Example: <DialogContent className="bg-surface-container rounded-xl shadow-elevation-2 p-6">
  └── M3 tokens defined as CSS variables, mapped in Tailwind config
```

**The Radix behavior layer stays untouched.** You only change the Tailwind classes in the shadcn wrapper files.

---

## Part 1: Tailwind Config with M3 Tokens

Create a Tailwind config that maps M3 design tokens to utility classes. All values reference CSS variables so themes can be swapped at runtime.

**Tailwind 4 note:** Tailwind 4 supports CSS-first configuration (`@theme` in CSS) OR the traditional `tailwind.config.ts` file. Either approach works. shadcn/ui generates a `tailwind.config.ts` by default during init — use that. If you prefer the CSS-first approach, the token values are the same; just define them in your CSS with `@theme` blocks instead.

### Colors

M3 uses **semantic color roles**, not raw hex values. Every color in every component must use these roles — never hardcoded hex.

```js
// tailwind.config.js (or tailwind.config.ts)
export default {
  theme: {
    extend: {
      colors: {
        // Surface layer (backgrounds — tonal elevation)
        'surface':                    'var(--md-sys-color-surface)',
        'surface-container':          'var(--md-sys-color-surface-container)',
        'surface-container-high':     'var(--md-sys-color-surface-container-high)',
        'surface-container-highest':  'var(--md-sys-color-surface-container-highest)',

        // Content (text & icons)
        'on-surface':                 'var(--md-sys-color-on-surface)',
        'on-surface-variant':         'var(--md-sys-color-on-surface-variant)',
        'outline':                    'var(--md-sys-color-outline)',
        'outline-variant':            'var(--md-sys-color-outline-variant)',

        // Primary (brand accent)
        'primary':                    'var(--md-sys-color-primary)',
        'on-primary':                 'var(--md-sys-color-on-primary)',
        'primary-container':          'var(--md-sys-color-primary-container)',
        'on-primary-container':       'var(--md-sys-color-on-primary-container)',

        // Secondary
        'secondary':                  'var(--md-sys-color-secondary)',
        'on-secondary':               'var(--md-sys-color-on-secondary)',
        'secondary-container':        'var(--md-sys-color-secondary-container)',
        'on-secondary-container':     'var(--md-sys-color-on-secondary-container)',

        // Tertiary (status/emphasis)
        'tertiary':                   'var(--md-sys-color-tertiary)',
        'on-tertiary':                'var(--md-sys-color-on-tertiary)',

        // Error
        'error':                      'var(--md-sys-color-error)',
        'on-error':                   'var(--md-sys-color-on-error)',
        'error-container':            'var(--md-sys-color-error-container)',
        'on-error-container':         'var(--md-sys-color-on-error-container)',
      },

      borderRadius: {
        'xs':  '4px',   // M3 extra-small
        'sm':  '8px',   // M3 small
        'md':  '12px',  // M3 medium
        'lg':  '16px',  // M3 large
        'xl':  '20px',  // M3 extra-large
        '2xl': '24px',  // used for cards, input fields, detail canvas
        '3xl': '28px',  // M3 extra-extra-large
        'full': '9999px', // pills, FABs
      },

      boxShadow: {
        'elevation-1': '0 1px 2px rgba(0, 0, 0, 0.3), 0 1px 3px 1px rgba(0, 0, 0, 0.15)',
        'elevation-2': '0 1px 2px rgba(0, 0, 0, 0.3), 0 2px 6px 2px rgba(0, 0, 0, 0.15)',
        'elevation-3': '0 4px 8px 3px rgba(0, 0, 0, 0.15), 0 1px 3px rgba(0, 0, 0, 0.3)',
      },

      fontFamily: {
        sans: ['Inter', 'system-ui', '-apple-system', 'sans-serif'],
      },

      fontSize: {
        // M3 type scale
        'display-large':  ['57px', { lineHeight: '64px', letterSpacing: '-0.25px' }],
        'display-medium': ['45px', { lineHeight: '52px' }],
        'display-small':  ['36px', { lineHeight: '44px' }],
        'headline-large': ['32px', { lineHeight: '40px' }],
        'headline-medium':['28px', { lineHeight: '36px' }],
        'headline-small': ['24px', { lineHeight: '32px' }],
        'title-large':    ['22px', { lineHeight: '28px' }],
        'title-medium':   ['16px', { lineHeight: '24px', letterSpacing: '0.15px', fontWeight: '500' }],
        'title-small':    ['14px', { lineHeight: '20px', letterSpacing: '0.1px', fontWeight: '500' }],
        'body-large':     ['16px', { lineHeight: '24px', letterSpacing: '0.5px' }],
        'body-medium':    ['14px', { lineHeight: '20px', letterSpacing: '0.25px' }],
        'body-small':     ['12px', { lineHeight: '16px', letterSpacing: '0.4px' }],
        'label-large':    ['14px', { lineHeight: '20px', letterSpacing: '0.1px', fontWeight: '500' }],
        'label-medium':   ['12px', { lineHeight: '16px', letterSpacing: '0.5px', fontWeight: '500' }],
        'label-small':    ['11px', { lineHeight: '16px', letterSpacing: '0.5px', fontWeight: '500' }],
      },
    },
  },
}
```

---

## Part 2: CSS Variable Files

### Dark Theme (DEFAULT — this is the primary theme)

```css
/* globals.css or theme.css */

:root,
[data-theme="dark"] {
  /* Surface */
  --md-sys-color-surface:                    #141218;
  --md-sys-color-surface-container:          #1D1B20;
  --md-sys-color-surface-container-high:     #2B2930;
  --md-sys-color-surface-container-highest:  #36343B;

  /* Content */
  --md-sys-color-on-surface:                 #E6E1E5;
  --md-sys-color-on-surface-variant:         #CAC4D0;
  --md-sys-color-outline:                    #938F99;
  --md-sys-color-outline-variant:            #49454F;

  /* Primary */
  --md-sys-color-primary:                    #D0BCFF;
  --md-sys-color-on-primary:                 #381E72;
  --md-sys-color-primary-container:          #4F378B;
  --md-sys-color-on-primary-container:       #EADDFF;

  /* Secondary */
  --md-sys-color-secondary:                  #CCC2DC;
  --md-sys-color-on-secondary:               #332D41;
  --md-sys-color-secondary-container:        #4A4458;
  --md-sys-color-on-secondary-container:     #E8DEF8;

  /* Tertiary */
  --md-sys-color-tertiary:                   #EFB8C8;
  --md-sys-color-on-tertiary:                #492532;

  /* Error */
  --md-sys-color-error:                      #F2B8B5;
  --md-sys-color-on-error:                   #601410;
  --md-sys-color-error-container:            #8C1D18;
  --md-sys-color-on-error-container:         #F9DEDC;
}
```

### Light Theme (secondary — implement but don't prioritize)

Generate light theme values using the **Material Theme Builder** tool (https://www.figma.com/community/plugin/1034969338659738588). Use the same primary seed color (`#D0BCFF` in dark, which maps to a purple-ish seed). The tool generates the full light palette.

```css
[data-theme="light"] {
  --md-sys-color-surface:                    #FEF7FF;
  --md-sys-color-surface-container:          #F3EDF7;
  --md-sys-color-surface-container-high:     #ECE6F0;
  --md-sys-color-surface-container-highest:  #E6E0E9;

  --md-sys-color-on-surface:                 #1D1B20;
  --md-sys-color-on-surface-variant:         #49454F;
  --md-sys-color-outline:                    #79747E;
  --md-sys-color-outline-variant:            #CAC4D0;

  --md-sys-color-primary:                    #6750A4;
  --md-sys-color-on-primary:                 #FFFFFF;
  --md-sys-color-primary-container:          #EADDFF;
  --md-sys-color-on-primary-container:       #21005D;

  --md-sys-color-secondary:                  #625B71;
  --md-sys-color-on-secondary:               #FFFFFF;
  --md-sys-color-secondary-container:        #E8DEF8;
  --md-sys-color-on-secondary-container:     #1D192B;

  --md-sys-color-tertiary:                   #7D5260;
  --md-sys-color-on-tertiary:                #FFFFFF;

  --md-sys-color-error:                      #B3261E;
  --md-sys-color-on-error:                   #FFFFFF;
  --md-sys-color-error-container:            #F9DEDC;
  --md-sys-color-on-error-container:         #410E0B;
}
```

---

## Part 3: Components to Build

Copy each component from shadcn/ui (`npx shadcn@latest add <component>`), then restyle with M3 tokens. Below is the target styling for each.

### 1. Button

Variants needed:

| Variant | Background | Text | Border | Use |
|---|---|---|---|---|
| **filled** (default) | `bg-primary` | `text-on-primary` | none | Primary actions |
| **tonal** | `bg-secondary-container` | `text-on-secondary-container` | none | Secondary actions |
| **outlined** | `bg-transparent` | `text-primary` | `border border-outline` | Tertiary actions |
| **text** | `bg-transparent` | `text-primary` | none | Low-emphasis actions |
| **destructive** | `bg-error` | `text-on-error` | none | Delete/remove actions |
| **ghost** | `bg-transparent` | `text-on-surface-variant` | none | Toolbar/icon actions |

All buttons: `rounded-full` (pill shape) for standard, `rounded-lg` for icon-only. Padding: `px-6 py-2.5` for standard, `p-2.5` for icon-only.

Hover: lighten background slightly (use `hover:brightness-110` or a token step up).
Active/pressed: `active:scale-[0.98]` with `transition-transform`.

### 2. Input / Textarea

- Background: `bg-surface-container-high`
- Border: `border border-outline-variant`
- Focus: `focus:border-primary focus:ring-1 focus:ring-primary`
- Text: `text-on-surface` (input text), `text-on-surface-variant` (placeholder)
- Radius: `rounded-2xl` (24px) — this matches the sidecar's input field spec
- Padding: `px-4 py-3`

### 3. Card

This is the most important component — the sidesheet is made of cards.

- Background: `bg-surface-container-high`
- Border: `border border-white/5` (nearly invisible structural border)
- Border on hover: `hover:border-primary/20`
- Radius: `rounded-xl` (20px) to `rounded-2xl` (24px)
- Padding: `p-4` to `p-5`
- Hover effect: `hover:scale-[1.02]` with spring transition
- Active/press: `active:scale-[0.98]`
- Shadow: none by default (tonal elevation handles it), optional `shadow-elevation-1` for floating cards

Card sub-components:
- `CardHeader`: contains section dot indicator + label (`text-label-medium uppercase tracking-wider text-on-surface-variant`)
- `CardContent`: main content area
- `CardFooter`: optional action row

### 4. Dialog / AlertDialog

- Overlay: `bg-black/50 backdrop-blur-sm`
- Content background: `bg-surface-container`
- Border: `border border-outline-variant/20`
- Radius: `rounded-2xl` (24px)
- Shadow: `shadow-elevation-3`
- Title: `text-headline-small text-on-surface`
- Description: `text-body-medium text-on-surface-variant`
- Animation: scale(0.95→1.0) + opacity(0→1) with spring

### 5. Tabs

Used for view switching in the detail canvas (Chat, Context, Apps, Settings).

- Tab list background: `bg-surface-container`
- Individual tab: `text-on-surface-variant` (inactive), `text-primary` (active)
- Active indicator: `bg-primary` bar underneath (2px height, `rounded-full`)
- Hover: `hover:bg-surface-container-high`
- Radius: `rounded-lg` on individual tabs
- Padding: `px-4 py-2`

### 6. Toggle / Switch

Used in Settings view for capture stream on/off.

- Track (off): `bg-surface-container-highest`
- Track (on): `bg-primary`
- Thumb (off): `bg-outline`
- Thumb (on): `bg-on-primary`
- Border: `border-2 border-outline` (off), none (on)
- Size: 52px wide, 32px tall (M3 spec)
- Animation: thumb slides with spring transition

### 7. ScrollArea

Used for the chat message list and sidesheet card list.

- Scrollbar track: `bg-transparent`
- Scrollbar thumb: `bg-on-surface/20 hover:bg-on-surface/40`
- Thumb radius: `rounded-full`
- Thumb width: 6px (expands to 8px on hover)

### 8. Sidebar / Sheet

The sidesheet panel. Slides in from the right edge.

- Background: `bg-surface-container`
- Width: 360–400px
- Shadow: `shadow-elevation-2`
- Radius: `rounded-2xl` on left corners (right side flush with screen edge)
- Enter animation: `translateX(100% → 0)` with spring `{ stiffness: 300, damping: 30 }`
- Exit animation: reverse

### 9. Badge

Used for status indicators (capture stream status, notification counts, health).

| Variant | Background | Text |
|---|---|---|
| **default** | `bg-secondary-container` | `text-on-secondary-container` |
| **primary** | `bg-primary-container` | `text-on-primary-container` |
| **success** | `bg-tertiary/20` | `text-tertiary` |
| **error** | `bg-error-container` | `text-on-error-container` |
| **outline** | `bg-transparent border border-outline` | `text-on-surface-variant` |

All badges: `rounded-full`, `text-label-small`, `px-2.5 py-0.5`.

### 10. Tooltip

- Background: `bg-surface-container-highest`
- Text: `text-body-small text-on-surface`
- Radius: `rounded-sm` (8px)
- Shadow: `shadow-elevation-2`
- Padding: `px-3 py-1.5`
- Animation: fade + slight translateY with 200ms delay

### 11. Toast / Sonner

For notifications and error messages. Use **Sonner** (shadcn's default toast library).

- Background: `bg-surface-container-high`
- Border: `border border-outline-variant/20`
- Text: `text-body-medium text-on-surface`
- Description: `text-body-small text-on-surface-variant`
- Radius: `rounded-xl`
- Shadow: `shadow-elevation-2`
- Success accent: left border `border-l-4 border-l-tertiary`
- Error accent: left border `border-l-4 border-l-error`
- Action button: `text-primary` text button

### 12. Dropdown Menu

- Content background: `bg-surface-container`
- Border: `border border-outline-variant/20`
- Radius: `rounded-lg` (16px)
- Shadow: `shadow-elevation-2`
- Item text: `text-body-medium text-on-surface`
- Item hover: `hover:bg-surface-container-high`
- Item padding: `px-3 py-2`
- Separator: `bg-outline-variant/30`
- Item radius: `rounded-sm`

### 13. Separator

- Color: `bg-outline-variant/30`
- Default: horizontal, 1px height
- Also support vertical variant

### 14. Avatar

Used for chat message avatars.

- Size: 32px (chat), 40px (profile)
- Radius: `rounded-full`
- AI avatar: background uses brand gradient `linear-gradient(to bottom-right, var(--md-sys-color-primary), var(--md-sys-color-tertiary))`
- User avatar: `bg-secondary-container`
- Fallback text: `text-label-medium`

### 15. Skeleton

Loading placeholder used while IPC responses are in flight.

- Background: `bg-surface-container-high`
- Animation: pulse between `opacity-100` and `opacity-40`
- Radius: match the component it's replacing

---

## Part 4: Interactive States

Every interactive component must handle these states consistently. Apply these to **all** buttons, inputs, switches, tabs, menu items, etc.

### Disabled State

All disabled interactive elements:
- Opacity: `disabled:opacity-38` (M3 spec: 38% opacity for disabled content)
- Cursor: `disabled:cursor-not-allowed`
- No hover/active effects: `disabled:pointer-events-none`
- No additional color changes — the opacity reduction handles the visual dimming

```tsx
// Example: disabled button
<button
  className="bg-primary text-on-primary disabled:opacity-38 disabled:cursor-not-allowed disabled:pointer-events-none"
  disabled
>
  Submit
</button>
```

Add `opacity: { 38: '0.38' }` to the Tailwind config `extend` block for the M3 disabled opacity value.

### Focus-Visible State

M3 uses a visible focus ring for keyboard navigation only (not mouse clicks). Apply `focus-visible` (not `focus`) so the ring only shows during keyboard nav:

- Focus ring: `focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2 focus-visible:ring-offset-surface`
- Outline: `focus-visible:outline-none` (replace browser default with the ring)

Apply this to: Button, Input, Textarea, Tabs (individual tabs), Switch, Dropdown Menu items, Dialog close button.

```tsx
// Example: button with focus-visible
<button className="... focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2 focus-visible:ring-offset-surface">
```

`ring-offset-surface` ensures the focus ring gap matches the background. Add `ringOffsetColor: { surface: 'var(--md-sys-color-surface)' }` to the Tailwind config.

---

## Part 5: Borders Spec

Borders in this design system are structural, not decorative. They should be nearly invisible:

| Use | Class |
|---|---|
| Default card border | `border border-white/5` |
| Hover/focus border | `border-primary/20` |
| Active/selected border | `border-primary` |
| Inactive/default | `border-transparent` (preserves layout, no visible border) |
| Divider lines | `border-outline-variant/30` |

---

## Part 6: Motion Spec

All position/size animations use **spring physics** via Motion v12 (Framer Motion). Color/opacity transitions use CSS transitions.

```tsx
// Standard spring — use for panels, sheets, dialogs
const standardSpring = { type: "spring", stiffness: 300, damping: 30 }

// Widget hover
<motion.div whileHover={{ scale: 1.02 }} whileTap={{ scale: 0.98 }} transition={standardSpring}>

// Sheet enter/exit
<motion.div
  initial={{ x: "100%" }}
  animate={{ x: 0 }}
  exit={{ x: "100%" }}
  transition={standardSpring}
>

// Dialog enter/exit
<motion.div
  initial={{ opacity: 0, scale: 0.95 }}
  animate={{ opacity: 1, scale: 1 }}
  exit={{ opacity: 0, scale: 0.95 }}
  transition={standardSpring}
>
```

For color and opacity: `transition-colors duration-300` or `transition-opacity duration-200` (CSS, not spring).

---

## Part 7: Development Workflow

### Where you work

You work in a **standalone Vite + React project** — NOT inside the Electron app repo (`coworkai-desktop`). This keeps you unblocked from the backend team's work.

**Setup:**

```bash
# Create a standalone Vite project
npm create vite@latest cowork-ui -- --template react-ts
cd cowork-ui

# Upgrade to React 19 (Vite defaults to React 18)
npm install react@19 react-dom@19
npm install -D @types/react@19 @types/react-dom@19

# Install Tailwind 4 + Vite plugin
npm install tailwindcss @tailwindcss/vite

# Install UI dependencies
npm install lucide-react motion clsx tailwind-merge sonner

# Install Inter font
npm install @fontsource-variable/inter

# Initialize shadcn/ui (see Part 9 for details)
npx shadcn@latest init
```

**Configure Vite** — add Tailwind plugin to `vite.config.ts`:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src/renderer'),
    },
  },
})
```

**Configure TypeScript paths** — add to `tsconfig.json` (or `tsconfig.app.json`):

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/renderer/*"]
    }
  }
}
```

This is critical — shadcn generates imports like `import { cn } from "@/lib/utils"`. The `@/` alias must point to `src/renderer/` so these imports resolve correctly.

**Load Inter font** — add to your app entry point (`src/renderer/main.tsx` or equivalent):

```ts
import '@fontsource-variable/inter';
```

You develop and preview components in this standalone project using `npm run dev`. The preview page (`ComponentPreview.tsx`) runs in the browser — no Electron needed.

### How you deliver

When your work is complete, the deliverable is the following folders/files from your project:

```
src/renderer/components/ui/    ← all 15 component files
src/renderer/lib/utils.ts      ← cn() utility
src/renderer/styles/globals.css ← CSS variables (dark + light themes)
tailwind.config.ts              ← M3 token mappings
```

**Use the `src/renderer/` path prefix in your project from the start.** This matches the final folder structure in coworkai-desktop exactly, so integration is a direct copy — no path rewriting needed.

shadcn defaults to putting components in `src/components/ui/`. Override this during `npx shadcn@latest init` — when prompted for the components path, set it to `src/renderer/components/ui`. Or manually move files after generation. Either way, the final structure must use `src/renderer/`.

### How integration works (Sprint 6 — not your concern, but for context)

During Sprint 6, the backend team will:

1. Copy your `src/renderer/components/ui/` folder into `coworkai-desktop/src/renderer/components/ui/`
2. Copy your `src/renderer/lib/utils.ts` into `coworkai-desktop/src/renderer/lib/utils.ts`
3. Merge your `globals.css` into the Electron renderer's CSS entry point
4. Merge your `tailwind.config.ts` token mappings into the Electron project's Tailwind config
5. Install the same npm dependencies you used (Radix packages, lucide-react, motion, sonner, clsx, tailwind-merge)
6. Build views (Chat, Context, Settings) using your components

The preview page (`ComponentPreview.tsx`) does not get copied — it's for review only.

---

## Part 8: Folder Structure

Your project should mirror the target structure in coworkai-desktop:

```
cowork-ui/                         ← your standalone Vite project
├── src/
│   └── renderer/
│       ├── components/
│       │   └── ui/
│       │       ├── avatar.tsx
│       │       ├── badge.tsx
│       │       ├── button.tsx
│       │       ├── card.tsx
│       │       ├── dialog.tsx
│       │       ├── dropdown-menu.tsx
│       │       ├── input.tsx
│       │       ├── scroll-area.tsx
│       │       ├── separator.tsx
│       │       ├── sheet.tsx
│       │       ├── skeleton.tsx
│       │       ├── switch.tsx
│       │       ├── tabs.tsx
│       │       ├── textarea.tsx
│       │       ├── toast.tsx (sonner)
│       │       └── tooltip.tsx
│       ├── lib/
│       │   └── utils.ts           (shadcn's cn() utility — clsx + tailwind-merge)
│       ├── styles/
│       │   └── globals.css        (CSS variables for dark + light themes)
│       └── preview/
│           └── ComponentPreview.tsx (page showing all components — for review)
├── tailwind.config.ts              (M3 token mappings)
├── package.json
└── vite.config.ts
```

### Preview page structure

The `preview/ComponentPreview.tsx` should be a single scrollable page with:

1. **Theme toggle** at the top — switches `data-theme` between `"dark"` and `"light"` on the root element
2. **Sections per component** — each with a heading and all variants shown side by side
3. **Interactive demos** — buttons should be clickable, dialogs should open, switches should toggle, tooltips should appear on hover
4. **Background** — use `bg-surface` so the tonal elevation of components is visible

Example layout:

```
[Theme Toggle: Dark / Light]

## Button
[filled] [tonal] [outlined] [text] [destructive] [ghost]
[filled disabled] [tonal disabled] ...
[icon-only filled] [icon-only ghost]

## Input
[default] [with placeholder] [focused] [disabled]

## Card
[default card with header/content/footer]
[hovered state demo]

## Dialog
[Open Dialog button → triggers dialog]

## Tabs
[Chat | Context | Apps | Settings]

... (all 15 components)
```

This page is for internal review only — it does not ship in the Electron app.

---

## Part 9: shadcn/ui Setup

Initialize shadcn in the project:

```bash
npx shadcn@latest init
```

When prompted:
- Style: **Default**
- Base color: **Neutral** (we override everything anyway)
- CSS variables: **Yes**
- Components path: **src/renderer/components/ui**
- Utils path: **src/renderer/lib/utils**

Then add components one by one:

```bash
npx shadcn@latest add button input textarea card dialog tabs
npx shadcn@latest add toggle scroll-area sheet badge tooltip
npx shadcn@latest add dropdown-menu separator avatar skeleton sonner
```

After copying, restyle each component file per the specs in Part 3.

**Important:** shadcn copies files into your project. After copying, these are YOUR files. Edit them freely. You do not need to worry about shadcn updates breaking your changes.

---

## Acceptance Criteria

1. **All 15 components** are present in `src/renderer/components/ui/`
2. **Zero hardcoded hex values** in component files — all colors reference M3 Tailwind tokens
3. **Dark theme is default** — components render correctly without any `data-theme` attribute (`:root` defaults to dark)
4. **Light theme works** — adding `data-theme="light"` to a parent element switches all colors
5. **Keyboard navigation works** on all interactive components (Radix handles this — verify it wasn't broken)
6. **No additional dependencies** beyond: Radix (via shadcn), Tailwind, Lucide React, Motion, clsx, tailwind-merge, sonner
7. **Preview page** shows all components in all variants, scrollable, with theme toggle
8. **`npm run build` passes** with zero TypeScript errors

---

## What You Do NOT Need to Build or Worry About

- Electron, preload scripts, IPC — not your concern
- State management (Zustand) — integrated later
- Routing (React Router) — integrated later
- Chat message rendering (react-markdown, Shiki) — separate task
- Any backend, API, or data layer
- The sidesheet layout, detail canvas, or navigation — those use these components but are built separately
- Chat input with send button — built later using your Input + Button components

**Your scope is the component library only.** Think of it like building a Storybook of M3-styled components.

---

## Reference Documents

These live in the cowork-brain repo. Read them if you need more context:

- **design-system.md** — Full M3 design system spec including color tokens, elevation model, typography, shape, motion, and component patterns (cards, chat bubbles, input fields)
- **prototype-brief.md** — What the app looks like when assembled. Shows the interaction model and screen flows
- **phase-1b-sprint-plan.md § Renderer Stack Decisions** — Full stack decision table with rationale

---

## Changelog

**v3 (Feb 17, 2026):** Review fixes. Added Part 4 (Interactive States) — disabled state (M3 38% opacity), focus-visible ring spec. Added Vite config with `@/` path alias and tsconfig paths (critical for shadcn imports). Added React 19 explicit install step. Added Inter font install + import. Added Tailwind 4 CSS-first config note. Added preview page structure guidance. Fixed part numbering (now Parts 1-9).

**v2 (Feb 17, 2026):** Added Development Workflow — standalone Vite project (`cowork-ui`), delivery process, integration steps. Updated folder structure to mirror coworkai-desktop target paths. Updated shadcn init instructions with correct component/utils paths.

**v1 (Feb 17, 2026):** Initial task brief. 15 components, M3 token system, Tailwind config, CSS variables, motion spec, folder structure, acceptance criteria.
