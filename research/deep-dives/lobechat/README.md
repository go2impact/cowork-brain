# LobeChat — Technical Deep Dive & Lessons for Cowork.ai

| | |
|---|---|
| **Status** | Working draft |
| **Last Updated** | 2026-02-16 |
| **Source** | [GitHub: lobehub/lobe-chat](https://github.com/lobehub/lobe-chat) · 72.3k stars · [Website](https://lobechat.com/) |
| **Purpose** | Feature-by-feature technical reverse engineering of LobeChat. Each section: how they built it, ASCII diagram of the internals, what we should study/copy/skip, and how it maps to Cowork.ai's product-features.md. |

---

## Overview

LobeChat is the most popular open-source AI chat framework. Web-first (Next.js) with a desktop app added later. Heavy on MCP plugin ecosystem (10k+ tools), multi-agent groups, and modern UI. Uses Drizzle ORM for database, supports local and remote DB modes.

**Key tech:** Next.js, TypeScript, Drizzle ORM, Vitest, pnpm monorepo, Docker/Vercel deployment, desktop app (Electron).

**Why study this:** Largest community (72k stars), MCP marketplace approach, multi-agent collaboration model, agent groups concept, Drizzle ORM patterns.

---

## Table of Contents

1. [AI SDK & LLM Integration](codex-output.md#1-ai-sdk--llm-integration)
2. [Agent System (multi-agent, agent groups, collaboration)](codex-output.md#2-agent-system-multi-agent-agent-groups-collaboration)
3. [MCP Integration (marketplace + plugin architecture)](codex-output.md#3-mcp-integration-marketplace--plugin-architecture)
4. [RAG & Knowledge Base (processing, vector storage, retrieval)](codex-output.md#4-rag--knowledge-base-processing-vector-storage-retrieval)
5. [Database & Storage (Drizzle, schema design, local vs remote)](codex-output.md#5-database--storage-drizzle-schema-design-local-vs-remote)
6. [Desktop App Architecture (how Next.js becomes desktop)](codex-output.md#6-desktop-app-architecture-how-nextjs-becomes-desktop)
7. [UI Framework & Design System (components, theming, i18n)](codex-output.md#7-ui-framework--design-system-components-theming-i18n)
8. [Summary: Copy / Study / Skip](codex-output.md#8-summary-copy--study--skip-competitive-teardown)
