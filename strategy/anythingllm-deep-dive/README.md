# AnythingLLM — Technical Deep Dive & Lessons for Cowork.ai

| | |
|---|---|
| **Status** | Working draft |
| **Last Updated** | 2026-02-16 |
| **Source** | [GitHub: Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm) · 54.6k stars · [Website](https://anythingllm.com/) |
| **Purpose** | Feature-by-feature technical reverse engineering of AnythingLLM. Each section: how they built it, ASCII diagram of the internals, what we should study/copy/skip, and how it maps to Cowork.ai's product-features.md. |

---

## Overview

AnythingLLM is the top RAG-first desktop AI app. Electron frontend with Vite+React, Node.js Express backend, LanceDB for vectors by default. Full MCP compatibility, no-code agent builder, multi-user support (Docker). MIT licensed.

**Key tech:** Electron, Vite+React, Node.js Express, LanceDB (default vector DB), 30+ LLM providers, MCP, Docker.

**Why study this:** RAG pipeline architecture, document processing, LanceDB vector patterns, no-code agent builder, embeddable chat widget, workspace-based organization.

---

## Table of Contents

1. [LLM Integration](codex-output.md#1-llm-integration)
2. [RAG Pipeline (core strength)](codex-output.md#2-rag-pipeline-core-strength)
3. [Vector Database (LanceDB default + alternatives)](codex-output.md#3-vector-database-lancedb-default--alternatives)
4. [Agent System (no-code builder, tools, MCP)](codex-output.md#4-agent-system-no-code-builder-tools-mcp)
5. [Electron Architecture (frontend/backend/IPC)](codex-output.md#5-electron-architecture-frontendbackendipc)
6. [Workspace & Multi-user (isolation, permissions, Docker vs desktop)](codex-output.md#6-workspace--multi-user-isolation-permissions-docker-vs-desktop)
7. [Embeddable Chat Widget](codex-output.md#7-embeddable-chat-widget)
8. [Summary: Copy / Study / Skip](codex-output.md#8-summary-copy--study--skip)
