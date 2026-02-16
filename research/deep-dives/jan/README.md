# Jan — Technical Deep Dive & Lessons for Cowork.ai

| | |
|---|---|
| **Status** | Working draft |
| **Last Updated** | 2026-02-16 |
| **Source** | [GitHub: janhq/jan](https://github.com/janhq/jan) · 40.4k stars · [Website](https://jan.ai/) |
| **Purpose** | Feature-by-feature technical reverse engineering of Jan. Each section: how they built it, ASCII diagram of the internals, what we should study/copy/skip, and how it maps to Cowork.ai's product-features.md. |

---

## Overview

Jan is the only major Tauri-based AI desktop app (73% TypeScript, 19% Rust). Runs llama.cpp natively for local inference — no Ollama dependency. Exposes an OpenAI-compatible API on localhost:1337. 100% offline-first with privacy focus. MCP support for agentic features.

**Key tech:** Tauri (Rust+TS), llama.cpp (via Cortex engine), OpenAI-compatible local API, MCP, HuggingFace model downloads.

**Why study this:** Only Tauri alternative to Electron in this space, native llama.cpp integration, local API server pattern, offline-first architecture, Rust backend patterns.

---

## Table of Contents

1. [Tauri Architecture](codex-output.md#1-tauri-architecture)
2. [Local LLM Execution (Cortex lineage, llama.cpp, model management)](codex-output.md#2-local-llm-execution-cortex-lineage-llamacpp-integration-model-management)
3. [OpenAI-Compatible Local API (localhost:1337)](codex-output.md#3-openai-compatible-local-api-localhost1337)
4. [Provider System (local vs cloud abstraction)](codex-output.md#4-provider-system-local-vs-cloud-abstraction)
5. [MCP Integration (agentic features + tool calling)](codex-output.md#5-mcp-integration-agentic-features--tool-calling)
6. [Storage & Data (local persistence)](codex-output.md#6-storage--data-local-persistence)
7. [Model Download & Management (HuggingFace, formats, quantization)](codex-output.md#7-model-download--management-huggingface-formats-quantization)
8. [Summary: Copy / Study / Skip](codex-output.md#8-summary-copy--study--skip-for-coworkai)
