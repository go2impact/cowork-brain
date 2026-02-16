# Cherry Studio — Technical Deep Dive & Lessons for Cowork.ai

| | |
|---|---|
| **Status** | Working draft |
| **Last Updated** | 2026-02-16 |
| **Source** | [GitHub: CherryHQ/cherry-studio](https://github.com/CherryHQ/cherry-studio) · 39.9k stars · [Website](https://www.cherry-ai.com/) |
| **Purpose** | Feature-by-feature technical reverse engineering of Cherry Studio. Each section: how they built it, ASCII diagram of the internals, what we should study/copy/skip, and how it maps to Cowork.ai's product-features.md. |

---

## Overview

Cherry Studio is an Electron + Vite desktop AI productivity studio. 300+ pre-configured assistants, MCP server support, multi-model simultaneous conversations, WebDAV sync, document processing. AGPL-3.0 licensed. Very similar tech choices to aime-chat.

**Key tech:** Electron, Vite+TypeScript, Biome (lint), Playwright (test), pnpm, MCP servers, 300+ provider support.

**Why study this:** Closest Electron+Vite tech stack to ours, multi-model simultaneous conversation UX, 300+ assistant pre-configuration, WebDAV sync pattern, MCP integration.

---

## Table of Contents

1. [Electron + Vite Architecture](codex-output.md#1-electron--vite-architecture)
2. [LLM Provider System](codex-output.md#2-llm-provider-system)
3. [MCP Server Integration](codex-output.md#3-mcp-server-integration)
4. [Multi-Model Conversations](codex-output.md#4-multi-model-conversations)
5. [Assistant System (Preset Library)](codex-output.md#5-assistant-system-preset-library)
6. [Storage & Sync (Local persistence + WebDAV pattern)](codex-output.md#6-storage--sync-local-persistence--webdav-pattern)
7. [UI Framework (component library, theming, document processing)](codex-output.md#7-ui-framework-component-library-theming-document-processing)
8. [Summary: Copy / Study / Skip](codex-output.md#8-summary-copy--study--skip-for-coworkai)
