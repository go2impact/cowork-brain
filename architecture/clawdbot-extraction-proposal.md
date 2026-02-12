# Clawdbot → CoworkAI: Extraction Proposal

**Version:** 1.0
**Date:** February 13, 2026
**Status:** Awaiting review

---

## Context

Clawdbot is a mature, battle-tested codebase with ~2 years of production patterns across browser automation, shell safety, tool policy, plugin architecture, retry logic, and more. CoworkAI is building a hybrid execution model (MCP + Playwright browser + activity capture + live coaching). Rather than forking clawdbot (too much irrelevant messaging/channel code), this proposal identifies specific patterns worth extracting and where they fit in CoworkAI.

This is **not** a copy-paste exercise. Each item is a pattern to learn from and adapt — the implementations will differ because the products differ.

---

## Priority Legend

- **P0** — Blocks the hybrid execution model. Build this first.
- **P1** — High value, build within first sprint after P0.
- **P2** — Nice to have, build when touching related code.

---

## P0: Core Hybrid Execution Patterns

### 1. Playwright Session Management & Ref Caching

**What clawdbot has:** `src/browser/pw-session.ts`
Persistent CDP connection pooling with LRU ref cache (50 entries). Refs are keyed by `cdpUrl::targetId` so the AI can snapshot a page, get element refs, then act on those refs in a subsequent call without re-snapshotting.

**Why CoworkAI needs it:**
The hybrid model has the AI working inside the user's apps via browser. Without ref caching, every action requires a full re-snapshot (slow, flaky). With it, the AI can snapshot once, then click/type/fill multiple elements in sequence.

**What to extract:**
- LRU cache keyed by connection + target (not the clawdbot-specific CDP plumbing)
- Two ref modes: "aria" (Playwright's built-in `_snapshotForAI`) and "role" (custom ref parsing by ARIA role + name)
- `storeRoleRefsForTarget()` / `restoreRoleRefsForTarget()` pattern
- Timeout normalization: `Math.max(500, Math.min(60_000, opts.timeoutMs ?? 8000))`

**Where it goes in CoworkAI:**
- New module in coworkai-mastra: `src/browser/session.ts`
- Used by browser execution tools that the agent calls during visible task execution

**Estimated effort:** 2-3 days

---

### 2. Accessibility Tree Snapshots (Over Pixels)

**What clawdbot has:** `src/browser/pw-tools-core.snapshot.ts` + `src/browser/pw-role-snapshot.ts`
Two snapshot formats. The AI gets a text representation of the page (not a screenshot), with each interactive element tagged with a ref. Elements classified as interactive (button, link, textbox), content (heading, cell), or structural (group, list, table).

**Why CoworkAI needs it:**
Screenshots are expensive (tokens, bandwidth) and the AI can't reliably click pixel coordinates. Accessibility tree snapshots are small, precise, and let the AI reason about page structure. This is fundamental to the "AI works in your apps" model.

**What to extract:**
- Role classification sets (interactive/content/structural ARIA roles)
- Ref generation algorithm (role + name + nth for duplicates)
- Snapshot size limiting (default 50,000 chars)
- The pattern of offering both AI snapshot (Playwright built-in) and role snapshot (custom) — let the agent choose

**Where it goes in CoworkAI:**
- `src/browser/snapshot.ts` in coworkai-mastra
- Exposed as agent tools: `browser_snapshot`, `browser_act`

**Estimated effort:** 2-3 days

---

### 3. Progressive Screenshot Normalization

**What clawdbot has:** `src/browser/screenshot.ts`
When a screenshot *is* needed (visual verification, coaching UI), it runs progressive compression: quality grid [85,75,65,55,45,35] × size grid [2000→800px]. Returns the first combination under 5MB. Max dimension capped at 2000px.

**Why CoworkAI needs it:**
The Execution Viewer shows the user a live browser view. Screenshots need to be fast and small for real-time streaming. The user also watches AI work — screenshots are the visual feedback channel.

**What to extract:**
- The quality × size grid loop (simple, ~30 lines)
- Max dimension capping
- JPEG conversion with sharp (or equivalent)

**Where it goes in CoworkAI:**
- `src/browser/screenshot.ts` in coworkai-mastra
- Used by Execution Viewer's live feed

**Estimated effort:** 0.5 day

---

### 4. Browser Action Implementations

**What clawdbot has:** `src/browser/pw-tools-core.interactions.ts`
Complete set of Playwright actions: click, double-click, hover, drag, type, press key, fill form, scroll, wait, evaluate JS, screenshot with labeled overlays. Every action follows the same 5-step flow:
1. Get page by target ID
2. Ensure page state (listeners)
3. Restore cached refs
4. Resolve ref to Playwright locator
5. Execute action

**Why CoworkAI needs it:**
This IS the hybrid execution model. The AI needs to click buttons, fill forms, navigate pages in Zendesk/Gmail/Slack/Calendar. The action set and the consistent 5-step flow are directly reusable.

**What to extract:**
- The 5-step action pattern (not the clawdbot routing, just the action logic)
- Action type definitions (click, type, press, hover, fill, select, drag, wait, evaluate)
- Error-to-human-friendly conversion (`toAIFriendlyError`)
- Labeled screenshot overlay pattern (for visual debugging in Execution Viewer)

**Where it goes in CoworkAI:**
- `src/browser/actions.ts` in coworkai-mastra
- Registered as Mastra tools the agent can call

**Estimated effort:** 3-4 days

---

## P1: Safety & Control Patterns

### 5. Shell Command Approval & Allowlist Model

**What clawdbot has:** `src/infra/exec-approvals.ts` (~1,268 lines)
Multi-layer command approval: deny → allowlist → full. Quote-aware shell parsing that respects single/double quotes and escapes. Splits by chain operators (&&, ||, ;) separately from pipes. Rejects backgrounding (&), redirection (>/<), backticks, and $() substitution. Safe binary whitelist (jq, grep, cut, sort, etc.).

**Why CoworkAI needs it:**
The product-features.md now specifies autonomy levels including "autonomous but visible." When the AI executes shell commands (install dependencies, run scripts, git operations), it needs the same safety guardrails. The Approval Queue feature also needs a backend model for what requires approval vs. what's pre-approved.

**What to extract:**
- Quote-aware command parsing
- Allowlist with glob pattern matching
- Safe binary whitelist concept
- Three-tier security model (deny/allowlist/full)
- The "ask" modes (off / on-miss / always)

**Where it goes in CoworkAI:**
- `src/safety/exec-approvals.ts` in coworkai-mastra
- Wired into the Approval Queue and autonomy level system

**Estimated effort:** 3-4 days (adapt, don't copy — CoworkAI's approval UI is different from clawdbot's socket-based flow)

---

### 6. Environment Variable Sanitization

**What clawdbot has:** `src/infra/shell-env.ts` + various
Env vars validated before exec (shell meta-characters rejected). Sensitive values never logged. Login shell env extraction with null-byte delimiter parsing. NPM fund suppression auto-injected.

**Why CoworkAI needs it:**
Browser automation + shell execution on the user's machine means env vars could leak secrets. The AI must never expose API keys, tokens, or passwords in logs or to the LLM context.

**What to extract:**
- Env var validation (reject shell meta-chars)
- Sensitive field detection (patterns: `*_KEY`, `*_SECRET`, `*_TOKEN`, `*_PASSWORD`)
- Sanitization before logging or LLM context injection

**Where it goes in CoworkAI:**
- `src/safety/env-sanitize.ts` in coworkai-mastra
- Applied in: tool outputs before sending to agent, activity capture logging, Execution Viewer logs

**Estimated effort:** 1 day

---

### 7. Tool Policy System (Per-Profile Allowlisting)

**What clawdbot has:** `src/agents/tool-policy.ts`
Profile-based tool allowlisting: minimal, coding, messaging, full. Tool groups (group:fs, group:runtime, group:web). Per-provider overrides. Deny-always-wins resolution. Per-sender group policies for messaging channels.

**Why CoworkAI needs it:**
CoworkAI's autonomy levels (Suggest Only → Auto-pilot) map directly to tool profiles. "Suggest Only" = minimal tools (read-only). "Auto-pilot" = full tools. The product-features.md also specifies per-app/per-category configurability — that's per-provider overrides.

**What to extract:**
- Profile → allowed tools resolution
- Tool group expansion
- Deny-wins-over-allow logic
- Per-context overrides (in CoworkAI: per-app instead of per-provider)

**Where it goes in CoworkAI:**
- `src/safety/tool-policy.ts` in coworkai-mastra
- Wired to autonomy level settings from onboarding config

**Estimated effort:** 2 days

---

## P2: Infrastructure Patterns

### 8. Retry Engine with Exponential Backoff + Jitter

**What clawdbot has:** `src/infra/retry.ts` (~129 lines)
Generic `retryAsync<T>(fn, options)` with exponential backoff, jitter, `shouldRetry` callback, `retryAfterMs` extraction from response headers, and `onRetry` hooks. Provider-specific defaults (Discord: 3 attempts/500ms→30s, Telegram: 3 attempts/400ms→30s).

**Why CoworkAI needs it:**
CoworkAI already has a basic retry in `src/lib/retry.ts` in coworkai-mastra. Clawdbot's version is more mature: it handles Retry-After headers (important for OpenRouter rate limits), has jitter (prevents thundering herd), and has observable retry events.

**What to extract:**
- Retry-After header parsing
- Jitter calculation
- Observable hooks (onRetry callback for logging)
- Provider-specific shouldRetry defaults

**Where it goes in CoworkAI:**
- Enhance existing `src/lib/retry.ts` in coworkai-mastra

**Estimated effort:** 0.5 day

---

### 9. Plugin/Hook Lifecycle System

**What clawdbot has:** `src/plugins/types.ts` (~529 lines)
15 lifecycle hooks: before_agent_start, agent_end, before_tool_call, after_tool_call, tool_result_persist, message_received/sending/sent, session_start/end, before/after_compaction, gateway_start/stop. Dependency injection via context objects. Exclusive "slots" (e.g., only one memory plugin at a time).

**Why CoworkAI needs it:**
CoworkAI's product-features.md describes an App Ecosystem where users can install MCP apps. These apps need lifecycle hooks: before the AI uses a tool, after a task completes, when activity changes, when the user coaches the AI. Clawdbot's hook system is the proven pattern.

**What to extract:**
- Hook registration and dispatch pattern
- Before/after pairs with cancellation (before_tool_call can block)
- Context injection (each hook gets relevant state)
- Exclusive slot concept (for CoworkAI: one voice provider at a time, one browser controller at a time)

**Where it goes in CoworkAI:**
- `src/plugins/` in coworkai-mastra (new directory)
- Phase 2/3 feature, but the architecture should be planned now

**Estimated effort:** 3-5 days (design now, implement when App Ecosystem is built)

---

### 10. Configuration Validation Pipeline

**What clawdbot has:** `src/config/validation.ts` + Zod schema
Multi-level config hierarchy (global → agent defaults → per-agent → per-provider). Zod-based validation with custom post-validators. Legacy config detection with migration hints. Sensitive field storage with 0600 permissions.

**Why CoworkAI needs it:**
CoworkAI already has onboarding config (`user_config` table). But as the product grows (multiple autonomy levels, per-app rules, browser preferences, activity capture settings), config will get complex. A validation pipeline prevents users from getting into broken states.

**What to extract:**
- Zod schema + custom validator pattern
- Legacy detection with migration hints
- Config layering (defaults → user overrides → runtime flags)

**Where it goes in CoworkAI:**
- Enhance `src/config/onboarding.ts` in coworkai-mastra

**Estimated effort:** 1-2 days

---

### 11. Chrome Extension Relay (User's Real Browser)

**What clawdbot has:** `src/browser/extension-relay.ts` (~600 lines)
WebSocket bridge that lets Playwright control the user's real Chrome tabs (not an isolated browser). Chrome extension connects to relay, Playwright CDP clients connect to relay, relay forwards messages. Loopback-only, single extension connection, user-initiated tab attachment.

**Why CoworkAI needs it (eventually):**
The product-features.md vision is the AI working in the user's actual apps. Phase 1 might use an isolated Playwright browser, but the endgame is controlling the user's real Chrome where they're already logged into Zendesk, Gmail, etc. The extension relay is how clawdbot solved this.

**What to extract:**
- The relay architecture (extension ↔ relay ↔ Playwright)
- Security model (loopback, single connection, user-initiated)
- Tab listing, activation, and close via relay

**Where it goes in CoworkAI:**
- Phase 2/3 feature
- `src/browser/extension-relay.ts` in coworkai-mastra (or separate extension package)

**Estimated effort:** 5+ days (this is a significant feature — defer unless needed for MVP)

---

### 12. Subsystem Logging with Routing

**What clawdbot has:** `src/logging/subsystem.ts`
Subsystem-prefixed logging (`createSubsystemLogger("browser")`) routed to file + console. Ephemeral event queue (max 20 events, session-scoped, deduplicated) for non-persistent observability.

**Why CoworkAI needs it:**
CoworkAI currently uses `[MastraDebug]` console logging. As the system grows (browser automation, shell execution, activity capture, RAG pipeline, scheduler), structured subsystem logging becomes essential for debugging.

**What to extract:**
- `createSubsystemLogger(name)` pattern
- Ephemeral event queue for transient state

**Where it goes in CoworkAI:**
- `src/lib/logger.ts` in coworkai-mastra

**Estimated effort:** 0.5 day

---

## Patterns Already Present in CoworkAI (No Extraction Needed)

These exist in clawdbot but CoworkAI already has its own implementation:

| Pattern | Clawdbot | CoworkAI |
|---------|----------|----------|
| **Retry logic** | `src/infra/retry.ts` | `src/lib/retry.ts` (simpler, enhance per #8) |
| **Token budgeting** | Per-message truncation | `src/lib/context-budget.ts` (more sophisticated) |
| **Model routing** | Single model per provider | `src/scheduler/model-routing.ts` (3 tiers) |
| **Notification system** | Channel message routing | `src/notifications/` (SQLite + SSE + polling) |
| **Auth/credentials** | Cross-CLI credential sync | `AuthService.swift` (Firebase + OAuth PKCE) |
| **Activity logging** | Tool call audit trail | `ActivityCapture.swift` + SQLite + RAG pipeline |
| **Session management** | Conversation threads per channel | Mastra Memory (LibSQLStore + LibSQLVector) |

---

## Patterns CoworkAI Has That Clawdbot Doesn't

These are strengths to keep — clawdbot can't help here:

| Pattern | Where |
|---------|-------|
| **RAG pipeline** (session composition → embeddings → vector search) | coworkai-mastra `src/rag/` |
| **Proactive scheduler** (passive rules + event triggers + quiet hours) | coworkai-mastra `src/scheduler/` |
| **Context budget enforcement** (per-tier token limits) | coworkai-mastra `src/lib/context-budget.ts` |
| **Keystroke capture** (CGEventTap, 400ms debounce, segment buffering) | mac native `KeystrokeCapture.swift` |
| **Activity capture** (500ms polling, window/app/URL tracking) | mac native `ActivityCapture.swift` |
| **Watermark-based incremental sync** | coworkai-mastra `src/ingest/sqlite-sync.ts` |
| **Dynamic system prompts** (rebuilt from onboarding config) | coworkai-mastra `src/mastra/agents/coach.ts` |

---

## Implementation Order

```
Phase 1 (Hybrid Execution MVP):
  P0-1  Playwright session + ref caching
  P0-2  Accessibility tree snapshots
  P0-3  Screenshot normalization
  P0-4  Browser action implementations
  P1-6  Env var sanitization

Phase 2 (Safety & Autonomy):
  P1-5  Shell command approval model
  P1-7  Tool policy system
  P2-8  Retry engine enhancement
  P2-10 Config validation pipeline
  P2-12 Subsystem logging

Phase 3 (Ecosystem & Extensions):
  P2-9  Plugin/hook lifecycle system
  P2-11 Chrome extension relay
```

---

## What This Proposal Does NOT Cover

- **Clawdbot messaging channels** (Telegram, Discord, Slack, Signal, WhatsApp, iMessage) — not relevant to CoworkAI
- **Clawdbot CLI infrastructure** (commander.js, progress bars, terminal tables) — CoworkAI is a GUI app
- **Clawdbot gateway/daemon** (multi-node, heartbeat, pairing) — CoworkAI is single-user desktop
- **Clawdbot Canvas/A2UI** — different UI paradigm

---

## Decision Required

Review each item and mark: **Accept**, **Reject**, or **Defer**.

Items marked Accept will be scheduled for implementation. Items marked Defer go to a backlog for future consideration. Items marked Reject are dropped.

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-13 | Initial proposal based on comprehensive review of clawdbot, cowork-brain, coworkai-mac-native-desktop, and coworkai-mastra codebases |
