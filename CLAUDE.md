# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**cowork-brain** is the executive decision layer for Cowork.ai — not a code repo. It contains architecture specs, business strategy, design system, GTM plans, and a decision log with full reasoning behind every significant choice. Application code lives in separate repos (see `repos.md`).

Cowork.ai is a desktop AI agent for remote workers. It runs as a sidecar alongside work apps, connecting to tools (Zendesk, Gmail, Slack) via MCP. Two-brain architecture: local LLM (DeepSeek-R1-8B via Ollama, free) + cloud models (via OpenRouter, paid tiers).

## The One Rule

**When you change something that affects money, functionality, or distribution — write down why.** Decisions ripple: model selection → cost-to-serve → pricing → GTM channels. If you can't explain the reasoning, don't make the change.

## Contribution Process

1. Branch from `main` using `your-name/short-description` (e.g., `rustan/whisper-warm-up`)
2. Edit the relevant doc(s)
3. Add a decision log entry to `decisions/decision-log.md` if the change affects: model selection, cost structure, routing logic, tier boundaries, token budgets, privacy/consent, or hardware requirements
4. Update the changelog at the bottom of any doc you changed
5. PR to `main` — tag Scott for cost/strategy changes, Rustan for architecture

Decision log entries must include: **Changed**, **From → To**, **Why** (with real reasoning, not "it's better"), **Cost impact**, **Alternatives considered**, **Approved by**. See `CONTRIBUTING.md` for examples.

## Repo Structure

- `product/` — What Cowork.ai is, who it's for, roadmap. **Start here if new.**
- `architecture/` — Engineering specs: models, routing, budgets, hardware (Rustan's reference)
- `strategy/` — Cost models, pricing tiers, conversion economics (Scott's reference)
- `design/` — M3 design system, three-state interaction model, prototype brief. **Read before touching any UI code.**
- `gtm/` — Distribution channels, phased rollout, Go2 asset leverage
- `decisions/` — **Most important directory.** Full audit trail with reasoning for every significant change
- `prototypes/` — Links to UI demos and design concepts

## Key Technical Context

- **Local brain:** DeepSeek-R1-Distill-Qwen-8B (4-bit, Ollama) — handles ~70% of daily work, free
- **Cloud brain:** OpenRouter gateway — Gemini 2.5 Flash (free tier), Gemini 3 Flash (Boost), Claude Sonnet 4.5 (Pro), Claude Opus 4.6 (Max)
- **Complexity router:** Rule-based, not ML — must stay under 10ms
- **Audio:** Whisper via Core ML on Apple Neural Engine, kept warm in memory on 16GB+ Macs
- **Embeddings:** Qwen3-Embedding-0.6B via Ollama, stored in SQLite-vec (same DB as activity store)
- **Desktop:** Native Swift/AppKit (Tauri was evaluated and rejected — see decision log Feb 13)
- **Design system:** Material Design 3 (M3) mandatory, dark theme first, maps to Tailwind
- **v0.1 scope:** Mac-only (16GB+ Apple Silicon for local, 8GB gets cloud-only)

## Cross-Document Consistency

Changes in one doc often require updates to others. Watch for ripple effects:
- Model change → update `architecture/llm-architecture.md` + `strategy/llm-strategy.md` + decision log
- Price change → update `strategy/` + `gtm/` + decision log
- UI/interaction change → update `design/design-system.md` + decision log

## Review Ownership

- **Scott:** Cost, strategy, pricing, tier logic
- **Rustan:** Architecture, technical decisions
- Typo/clarification fixes can be merged by anyone without decision log entries
