# Chatbox — Technical Deep Dive & Lessons for Cowork.ai

| | |
|---|---|
| **Status** | Working draft |
| **Last Updated** | 2026-02-16 |
| **Source** | [GitHub: chatboxai/chatbox](https://github.com/chatboxai/chatbox) · 38.5k stars · [Website](https://chatboxai.app/) |
| **Purpose** | Feature-by-feature technical reverse engineering of Chatbox. Each section: how they built it, ASCII diagram of the internals, what we should study/copy/skip, and how it maps to Cowork.ai's product-features.md. |

---

## Overview

Chatbox is a clean, lean Electron desktop AI client that also ships on iOS/Android (Capacitor) and web. Uses libsql/SQLite for local storage — same DB choice as Cowork.ai. Multi-provider support, MCP plugins, Markdown/LaTeX rendering. GPL-3.0 licensed.

**Key tech:** Electron, Vite+React, TypeScript, Tailwind CSS, libsql/SQLite, Capacitor (mobile), MCP plugins.

**Why study this:** Uses libsql (same as us), lean architecture, cross-platform via Capacitor, clean multi-provider abstraction, good reference for keeping things simple.

---

## Table of Contents

1. [Electron + Vite Architecture (main/renderer split, IPC, build)](codex-output.md#1-electron--vite-architecture-mainrenderer-split-ipc-build)
2. [LLM Provider Abstraction (multi-provider, unified interface)](codex-output.md#2-llm-provider-abstraction-multi-provider-unified-interface)
3. [libsql/SQLite Storage (schema design, local-first patterns)](codex-output.md#3-libsqlsqlite-storage-schema-design-local-first-patterns)
4. [MCP Plugin System (plugin connection + tool calling)](codex-output.md#4-mcp-plugin-system-plugin-connection--tool-calling)
5. [Cross-Platform Strategy (Electron desktop + Capacitor mobile + web)](codex-output.md#5-cross-platform-strategy-electron-desktop--capacitor-mobile--web)
6. [UI Framework (React, Tailwind, Markdown/LaTeX)](codex-output.md#6-ui-framework-react-tailwind-markdownlatex)
7. [Team Collaboration Features (how multi-user works)](codex-output.md#7-team-collaboration-features-how-multi-user-works)
8. [Summary: Copy / Study / Skip](codex-output.md#8-summary-copy--study--skip-table)
