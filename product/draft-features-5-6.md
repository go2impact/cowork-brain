# Draft: Features 5 & 6 — On-Demand Chat + Memory

**Status:** Superseded — content restructured across product-features.md and llm-architecture.md per CEO feedback on PR #6.
**Date:** 2026-02-13
**Note:** This file preserves the brainstorm context. The canonical version is in `product-features.md`.

---

## Feature 5: On-Demand Chat

**One-line:** A conversational interface where the AI already has your full work context — no copy-pasting, no explaining background. You ask, it knows.

**The gap it fills:**

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

## Feature 6: Memory

**One-line:** The AI remembers — past conversations, captured activity, your preferences, how you communicate. Four memory layers keep the AI useful over time without bloating its context window.

**The gap it fills:**

Activity Capture (feature 3) observes what you're doing. On-Demand Chat (feature 5) lets you ask questions. But without memory, every interaction starts from scratch — the AI forgets what it observed yesterday, what you discussed last week, and what it's learned about how you work. Memory is what makes the AI get smarter over time.

### Two data sources, one retrieval system

Memory doesn't just cover chat conversations. It covers **everything the AI has observed** — both captured activity data and conversation history. Both feed into the same pipeline.

- **Source 1: Captured activity data** (from Activity Capture, feature 3) — window titles, app names, browser URLs, focus sessions, keystroke/input patterns (opt-in), browser session actions, clipboard content (opt-in), screen recordings (opt-in).
- **Source 2: Conversation data** (from On-Demand Chat, feature 5) — user messages and AI responses, tool results and action outcomes, coaching interventions during execution.

Both sources flow through the same pipeline: raw data → local storage → embeddings → vector store → retrievable via RAG.

### Data pipeline

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

### Four memory layers

| Layer | What it stores | Sources | Lifespan | Retrieval |
|-------|---------------|---------|----------|-----------|
| **Conversation History** | Recent messages from the current chat thread | Conversations | Current session | Direct — last N messages loaded into context |
| **Working Memory** | Structured user profile — name, preferences, goals, communication style, project context | Conversations + activity patterns | Persistent (until user edits/deletes) | Direct — always loaded into context |
| **Semantic Recall** | Past conversations and captured activity data | Both sources | Long-term | RAG — query embedded, vector similarity search retrieves relevant past context |
| **Observational Memory** | Dense compressed logs from raw conversation and activity history | Both sources | Long-term | Background compression; retrievable via semantic recall |

### How the layers work together

1. **Conversation History** gives the AI short-term continuity — what you just said.
2. **Working Memory** gives it persistent knowledge about you — who you are, how you work.
3. **Semantic Recall** lets it find relevant past conversations *and* activity — what you discussed before, what you were working on last week.
4. **Observational Memory** keeps long-term memory manageable — compress, don't delete.

### How retrieval works at query time (RAG)

1. User asks a question in On-Demand Chat.
2. The query is embedded using the local embedding model.
3. Vector similarity search finds the most relevant past messages, activity data, and observations across all history.
4. Retrieved context is assembled with priority: current conversation history (highest) → working memory profile → semantically recalled past exchanges and activity → observational summaries (lowest).
5. Assembled context is fed to the model alongside the user's query.
6. If combined context exceeds the model's window, memory processors trim lowest-priority content. Trimmed content isn't lost — it stays in the vector store for future retrieval.

### Relationship to Activity Capture (feature 3)

| | Activity Capture (feature 3) | Memory (feature 6) |
|---|---|---|
| **Role** | The **collection** layer — observes and stores raw activity data | The **intelligence** layer — embeds, compresses, and retrieves data from both activity capture and conversations |
| **Data flow** | Raw capture → local SQLite → structured context | Raw data (from both sources) → embeddings → vector store → RAG retrieval |
| **Retrieval** | MCP resources queried by apps and agents (structured queries) | RAG pipeline at query time (semantic similarity search) |

Activity Capture is the input. Memory is what makes that input searchable and persistent across sessions.

### Where raw data lives

All on-device:

- **Raw activity data** — local database (same local SQLite as Activity Capture).
- **Raw conversation messages** — local database.
- **Embeddings / vector store** — local vector database. Embeddings generated by local model, stored locally, searched locally. Nothing leaves the device.
- **Working memory profile** — local database (structured data, not vectors).
- **Observational memory logs** — local database (compressed summaries) + local vector store (embedded for semantic search).

No memory data leaves the device unless a query requires cloud inference — in which case only the assembled context for that specific query is sent transiently (zero-retention providers), not the full memory store.

### Context management

When combined memory exceeds the model's context limit, memory processors filter, trim, and prioritize. Priority order: (1) current conversation history, (2) working memory profile, (3) semantically recalled past context, (4) observational summaries. Trimmed content is not deleted — it remains in the vector store and can be retrieved by future queries. The user never needs to think about token limits.

### Privacy

- All memory is on-device. Nothing leaves the device except transiently during cloud inference.
- Users can view, edit, or delete any memory layer.
- Working memory profile is fully transparent — you can see exactly what the AI "thinks" it knows about you and correct it.
- Observational memory logs are viewable — you can see what the AI has compressed and retained.
- Delete = gone. No hidden copies, no cloud backup.

---

## How features 3–6 connect

```
Activity Capture (feature 3)          Proactive Suggestions (feature 4)
  Observes your desktop                 Platform pushes notifications
  Captures raw activity data            Triggered by activity + signals
         │                                       │
         ▼                                       ▼
    Memory (feature 6)              On-Demand Chat (feature 5)
    Embeds, compresses, retrieves       User pulls — asks questions
    Both activity + conversation        Draws from Memory + Activity
    data. Makes it all searchable.      + connected services
         │                                       │
         └──────────────┬────────────────────────┘
                        ▼
              Execution Modes (feature 2)
              Acts — MCP or browser path
```

---

## Open questions

- [ ] On-Demand Chat UI: keyboard shortcut, sidesheet, floating window, or combination? (deferred to design phase)
- [ ] Memory retention policies: how long before raw data gets compressed by observational memory? Configurable?
- [ ] Memory capacity limits: how much vector storage is practical on-device? Pruning strategy for very long-term users?
- [ ] Working memory template: what fields should the default profile include? User-customizable?
