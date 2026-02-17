# Capture Stream Data Model Analysis

| | |
|---|---|
| **Status** | Decided |
| **Date** | 2026-02-17 |
| **Author** | Rustan (via Claude analysis) |
| **Decision** | Capture streams are independent peers, not parent-child. Flush decoupling follows as a consequence. |
| **Blocks** | Sprint 4 implementation (capture pipeline port) |

---

## The Question

Sprint 4 plan review surfaced an ambiguity: when keystroke accumulation hits 1000 characters, does the parent activity session flush too? Investigating that question led to a more fundamental one: **should keystroke chunks be children of activity sessions at all?**

---

## What We Found

### The old codebase uses parent-child

Source: `coworkai-agent/src/database/activity/manager.ts`, `coworkai-agent/src/database/keystroke/buffer.ts`

The `ActivityManager` holds up to 5 `ActivityBuffer` instances. Each buffer accumulates keystroke data. At 1000 chars, the entire buffer (activity + keystrokes) flushes to SQLite together. Keystrokes reference `activity_id` as a hard FK.

This existed because the sync pipeline shipped activities with nested keystrokes as payload containers:

```typescript
// coworkai-agent/src/database/sync/sync.ts
eventType: "activities_sync" | "keystrokes_sync" | "clipboards_sync"
```

Large payloads caused sync problems. The coupled flush kept payloads manageable for network transmission.

### The new architecture has no sync pipeline

| Old constraint | Still applies? |
|---------------|---------------|
| Remote sync pipeline | **No.** Local-only. |
| Payload size for network | **No.** Writes go to local libsql. |
| FK ordering (activity must exist before keystrokes) | **No.** No enforced FK in new schema. |
| Sync flag per row | **No.** No `sync` column. |

### The product describes independent streams

product-features.md defines five independently toggleable capture streams with different defaults, different data shapes, and different retention needs. The product model is **peer streams**, not a hierarchy.

### Only one consumer needs keystroke-to-activity correlation

| Consumer | Needs keystrokes tied to activity? |
|----------|------------------------------------|
| Context Card ("What are you working on?") | No — activity only |
| RAG / embeddings | No — activity sessions are embedded independently |
| Focus detection | No — derived from activity session duration |
| Communication patterns ("Draft in my tone") | **Yes** — needs to know Slack tone vs email tone |
| Proactive notifications | No — activity only |

And what that consumer actually needs is **which app was this typed in** — not a foreign key to a session row. A denormalized `app_name` on the chunk answers the question without a JOIN.

The clipboard table already does this: it has `source_app` directly on the row.

---

## The Decision

**Capture streams are independent peers.** Each table is a self-contained timeline.

### What changed in the schema

`keystroke_chunks` gains denormalized app context (same pattern clipboard already uses):

```sql
-- Added
app_name   TEXT,   -- which app was active when this was typed
bundle_id  TEXT,   -- macOS bundle ID of the active app
```

`activity_session_id` across all capture tables is now documented as an **optional correlation hint** — set by the orchestration layer when convenient, nullable, no FK constraint. Primary correlation method is overlapping timestamps.

### What this means for the write path

Each stream buffers and flushes independently. No write ordering constraints between tables.

The `CaptureBufferManager` (orchestration layer) still coordinates lifecycle events:
- Activity change → flush open keystroke chunk (clean chunk boundaries)
- Idle/sleep/shutdown → flush all streams

But this is a pipeline concern, not a schema invariant. If the orchestration layer fails to coordinate (crash mid-switch), the worst case is a keystroke chunk that spans two activity windows by a few hundred milliseconds — recoverable at read time via timestamps.

### What this means for the flush coupling

The original question — "does 1000-char keystroke accumulation flush the parent activity?" — dissolves. There is no parent. Keystroke chunks flush on their own triggers (debounce, special-key, max-length). Activity sessions end on their own triggers (focus change, idle, sleep, shutdown). Neither depends on the other at the DB level.

---

## Concrete Example

### Old behavior (parent-child, coupled flush):

```
User types in Gmail for 30 minutes without switching windows

1000 chars → activity session closes → new activity starts for same Gmail
1000 more  → same thing
1000 more  → same thing

Database: 3-4 activity_sessions rows for Gmail, 3-4 keystroke_chunks rows
AI sees: "Gmail — 3 min, Gmail — 4 min, Gmail — 2 min" (fragmented)
Focus detection: never triggers (no single session hits 15-min threshold)
```

### New behavior (independent streams):

```
User types in Gmail for 30 minutes without switching windows

1000 chars → keystroke chunk closes (with app_name='Gmail') → activity continues
1000 more  → another chunk closes → activity continues
User switches to Slack → last chunk closes → activity session closes

Database: 1 activity_sessions row (30 min), 3-4 keystroke_chunks rows (each with app_name='Gmail')
AI sees: "Gmail — 30 min" (correct)
Focus detection: fires correctly for deep work
```

---

## Documents Updated

| Document | Change |
|----------|--------|
| **database-schema.md** | Added `app_name`/`bundle_id` to `keystroke_chunks` DDL + TypeScript type. Added design note "Why capture streams are independent peers." Updated `activity_session_id` comments to "optional correlation hint" across all capture tables. Added `idx_keystroke_chunks_app_name` index. |
| **system-architecture.md** | Updated capture orchestration section and data flow diagram to reflect independent streams with denormalized app context. |
| **decision-log.md** | Added "Capture data model: independent streams" entry. |

### No changes needed:
- **product-features.md** — already describes independent streams (this decision aligns the schema to the product model)
- **COWORKAI_TECHNICAL_REFERENCE.md** — documents the old codebase (reference, not prescription)

---

## Changelog

- **2026-02-17** — Reframed from flush coupling analysis to capture stream data model decision. The real finding is that streams are independent peers (product model), not parent-child (sync-era artifact). Flush decoupling is a consequence, not the primary decision.
- **2026-02-17** — Original version analyzed flush coupling in isolation. User feedback: "got tunnel-visioned on flush." Broadened to first-principles analysis of the capture data model.
