# Cowork.ai — Decision Log

Every significant architecture or strategy change gets an entry here. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the format and rules.

---

## 2026-02-12 — Repository initialized

**Changed:** Created this repo with v4 of both architecture and strategy docs.

**Why:** Architecture and strategy decisions were living in chat conversations and loose files. As we move to implementation, Rustan and team need a single source of truth that tracks not just what we decided but why. Changes to model selection, routing logic, or cost structure ripple through the entire business — we need the reasoning preserved so we can audit decisions when reality diverges from projections.

**Approved by:** Scott

---

## 2026-02-12 — Free tier model: Gemini 2.0 Flash → Gemini 2.5 Flash

**Changed:** Free tier cloud model upgraded from Gemini 2.0 Flash to Gemini 2.5 Flash.

**From → To:** $0.10/user/month → $0.46/user/month. At 100K users: $10K/mo → $46K/mo.

**Why:**
1. Gemini 2.0 Flash is deprecated March 31, 2026. Not an option regardless.
2. The free week must prove enough value that losing it feels like a real loss. Gemini 2.5 Flash has thinking capabilities and 1M context — it feels genuinely useful. A mediocre free experience converts at 2-3% (industry standard freemium). We need 10%+ to make the economics work.
3. At 10% conversion with 100K free users: 10K paid × $10/mo = $100K GMV. The $46K subsidy is justified as customer acquisition.
4. If costs get too high, we waitlist new free signups rather than degrade the model. 100 users who say "holy shit" beats 10,000 who say "meh."

**Cost impact:** ~4.6x increase per free user. Mitigated by local-first users costing ~$0.15-0.25/month (DeepSeek handles most work), waitlist system capping total spend, and model price trends (-50% every 6-12 months).

**Alternatives considered:**
- Gemini 2.5 Flash Lite ($0.10/user/month): Same price as 2.0 Flash, decent quality bump, but not a "wow." Rejected — marginal improvement doesn't justify rebuilding on it.
- Blended routing (Lite for simple, Flash for complex on free tier): Saves ~40% but adds routing complexity for marginal gain. Rejected — simplicity wins at this stage.

**Approved by:** Scott

---

## 2026-02-12 — Local LLM: consolidated to DeepSeek-R1 8B only

**Changed:** Removed Qwen3-4B and Qwen3-8B from local model options. DeepSeek-R1-Distill-Qwen-8B is the single local brain.

**From → To:** Three local LLM options (Qwen3-4B, Qwen3-8B, DeepSeek-R1-8B "alt") → One option (DeepSeek-R1-8B primary).

**Why:**
1. v0.1 is Mac-only. The only hardware profile that runs a local LLM is 16GB+ Apple Silicon. Qwen3-4B was for 8GB machines that can't fit the 8B — but 8GB Macs get cloud-only anyway.
2. DeepSeek-R1-Distill-Qwen-8B is built on Qwen architecture (good Tagalog/multilingual) with DeepSeek's reasoning training. You get both capabilities in one model.
3. For CX agents, reasoning capability is the difference between useful and toy. "Summarize this ticket with contradictory customer replies and suggest a response" requires thinking, not pattern matching.
4. One model = simpler onboarding, simpler Ollama management, simpler debugging. Complexity adds when Windows arrives in v0.2.

**Cost impact:** None (local inference is free). Slightly higher RAM usage than Qwen3-4B but irrelevant since 8GB machines don't run local LLM.

**Alternatives considered:**
- Dual local model (fast small + big reasoning): Gemini review suggested this. Rejected — doubles RAM pressure, doubles complexity, and the cloud tier already handles "think harder."
- Qwen3-8B as primary (better multilingual, no reasoning training): DeepSeek-R1-Distill-Qwen is literally Qwen + reasoning. Strictly better.
- MLX runtime instead of Ollama (10-15% faster on Mac): Rejected — Ollama works on Mac AND Windows. Saves a full inference layer rewrite when v0.2 ships.

**Approved by:** Scott

---

## 2026-02-12 — User chooses local vs cloud at onboarding

**Changed:** Added explicit onboarding choice for 16GB+ Mac users: "Run on my machine" vs "Run in the cloud."

**From → To:** System auto-detected hardware and defaulted to local → User makes an informed choice with clear tradeoffs explained.

**Why:**
1. Downloading a 5GB model and giving up 7GB of RAM is a meaningful tradeoff. Users should understand it.
2. "Your laptop may run warm" is an honest disclosure, not a fear tactic. Filipino remote workers on company machines may prefer cloud to avoid hardware complaints.
3. Users who choose local buy into the tradeoff and blame the app less when their laptop gets warm.
4. The choice creates a natural segmentation: local-first users cost us ~$0.15-0.25/month in cloud tokens, cloud-first users cost ~$0.46/month. Knowing the split helps forecast costs.

**Cost impact:** Reduces blended free tier cost. If 40% of users choose local, blended cost drops from $0.46 to ~$0.33/user/month.

**Approved by:** Scott

---

## 2026-02-12 — Thermal auto-detection and failover

**Changed:** Added automatic thermal state monitoring with auto-failover from local to cloud inference.

**From → To:** Manual Low-Power Mode toggle only → Auto-detection via macOS `ProcessInfo.thermalState` with automatic failover at "serious" state.

**Why:**
1. Target users (Filipino remote workers, 8-16GB machines) won't know what thermal throttling means. They'll just know their laptop is hot and slow and blame the app.
2. Auto-failover to cloud when thermals hit "serious" prevents the worst user experience (stuttering local inference on a throttled machine) without requiring the user to understand the problem.
3. Manual restore on recovery (not auto-restore) because cold-starting a 5GB model is expensive and the user may have adapted to cloud mode during the thermal event.

**Cost impact:** Minimal — thermal events are temporary. May add a few cents/month in cloud tokens during hot periods.

**Source:** Gemini technical review (Feb 12, 2026)

**Approved by:** Scott

---

## 2026-02-12 — Whisper warm-up: keep-warm daemon (v0.1), smart detection (v0.2)

**Changed:** Added Whisper pre-loading strategy to eliminate 2-second cold-start gap.

**From → To:** Load Whisper on demand (2s delay at start of transcription) → Keep Whisper warm in memory at all times on 16GB+ machines.

**Why:**
1. First impressions. If transcription starts with 2 seconds of missed audio, the product feels broken. Instant transcription feels magical.
2. v0.1 simple approach: just keep Whisper warm. Costs 1.5GB RAM on 16GB+ machines but eliminates cold-start entirely. On these machines, 1.5GB is affordable.
3. v0.2 smart approach (deferred): detect calendar events and meeting app launches, pre-load 2 min before. More RAM-efficient but requires calendar API + process monitoring. Not worth the engineering lift for v0.1.

**Cost impact:** ~1.5GB RAM permanently allocated on 16GB+ machines. Acceptable tradeoff — these machines have headroom.

**Source:** Gemini technical review (Feb 12, 2026)

**Approved by:** Scott

---

## 2026-02-12 — Vector store: SQLite-vec (not LanceDB) → 2026-02-16 superseded by libsql

**Changed:** Originally locked local vector store to SQLite-vec instead of LanceDB. **Superseded:** Database stack research led to libsql for everything — built-in vector search replaces sqlite-vec as a separate extension.

**From → To:** LanceDB or SQLite-vec → SQLite-vec only → **libsql built-in vectors** (see [DATABASE_STACK_RESEARCH.md](DATABASE_STACK_RESEARCH.md))

**Why (original):**
1. Already using SQLite for activity store. Embeddings live in the same .db file. One database, one file, zero-ops.
2. LanceDB is a separate dependency with its own process. On 8GB Windows machines (v0.2), every extra dependency is a failure point.
3. At our scale (thousands to tens of thousands of embeddings per user), SQLite-vec performance is more than sufficient.
4. Simpler backup/restore: one file.

**Why (superseded by libsql):** libsql has built-in vector search (DiskANN) that eliminates the need for sqlite-vec as a separate extension. One SDK for both structured data and vectors, native Mastra integration, no `.dylib` packaging friction. See database stack research for full analysis.

**Cost impact:** None — both are free/local.

**Source:** Gemini technical review (Feb 12, 2026); database stack research (Feb 16, 2026)

**Approved by:** Scott (original), Rustan (libsql update)

---

## 2026-02-12 — Cost guardrail: waitlist system for free tier

**Changed:** Added waitlist mechanism when free tier costs exceed configurable threshold.

**From → To:** No cost ceiling on free tier → Hard ceiling (default $50K/month) triggers waitlist for new signups.

**Why:**
1. The free experience must prove real, measurable value. Degrading model quality or reducing quotas to save money defeats the purpose.
2. At $46K/month for 100K free users, we need conversion revenue to justify the subsidy. If conversion is below target, we can't afford unlimited free signups.
3. Waitlist preserves quality for existing users while controlling volume. Priority activation for referrals from paid users creates viral incentive.
4. Better to have fewer users getting the real experience than more users getting a watered-down one.

**Cost impact:** Caps free tier at configurable threshold. Prevents runaway subsidy costs during growth.

**Approved by:** Scott

---

## 2026-02-12 — Design system: Material Design 3

**Changed:** Adopted M3 (Material Design 3) as the mandatory design system for all Cowork.ai UI surfaces.

**From → To:** No formal design system (ad hoc styling per prototype) → M3 dark theme token system, documented in `design/design-system.md`.

**Why:**
1. Cowork.ai is a platform where users install and build MCP apps. Those apps generate UI. AI generates UI. Without a rigid, well-documented token system, the product becomes visually incoherent within weeks.
2. M3 has the most complete publicly documented token system (colors, elevation, shape, motion all have named specs). When an LLM generates a component, it can follow M3 specs and produce something that looks native.
3. Google maintains M3 — we don't need to maintain a custom design system with a small team.
4. M3 tokens map cleanly to Tailwind, which is our entire frontend stack.
5. Dark theme is first-class in M3, not bolted on. Desktop sidecar = dark by default.

**Alternatives considered:**
- Custom design system: More flexible but requires dedicated design resource to maintain. Not viable at our team size.
- Shadcn/Radix: Good component libraries but no comprehensive token system for AI-generated UIs.
- Apple HIG: Strong but not well-documented for web, and would conflict with cross-platform goals (51 countries, many on Windows).

**Approved by:** Scott

---

## 2026-02-12 — Interaction model: Three-state sidecar

**Changed:** Locked the interaction model to a three-state pattern: Closed (menu bar only) → SideSheet (right panel) → SideSheet + Detail Canvas (full workspace).

**From → To:** Various prototype approaches (some with nav rail always visible, some with tab switching) → Standardized three-state model documented in `design/design-system.md`.

**Why:**
1. Workers need zero visual intrusion when they're not using us. State 1 (closed) means Cowork.ai doesn't exist on screen. This is critical for the trust model — we're not surveillance software.
2. State 2 (sidesheet) is the quick-glance dashboard. Check status, scan context, 5 seconds, back to work. The sidesheet is a scrollable card list, not a tabbed interface — tabs force navigation decisions, cards let you scan.
3. State 3 (detail canvas) opens to the LEFT of the sidesheet, keeping the sidesheet visible as persistent navigation. This means you can jump between chat, apps, and MCP browser without closing and reopening.
4. The fav prototype nailed this model. The best-looking prototype had better visual polish but used a nav rail with tabs instead of the three-state pattern. We take the interaction from fav, the visual style from best-looking.

**Approved by:** Scott

---

## 2026-02-13 — Desktop framework: Electron (Swift and Tauri rejected)

**Changed:** Evaluated native Swift (AppKit/SwiftUI), Tauri 2.0, and Electron for the desktop shell. Decision: **Electron**.

**From → To:** Native Swift/AppKit prototype → Electron with multi-process architecture (utility processes for capture and agents, sandboxed renderer).

**Why:**
1. Every major dependency (Playwright, Mastra.ai, RAG pipeline, custom N-API capture addons) is a Node.js library or N-API module. Both Swift and Tauri would require running 100% of business logic in a Node.js sidecar — the native layer would only manage windows and system tray.
2. Swift has no ecosystem for agent orchestration, browser automation, or RAG. Choosing Swift means rebuilding infrastructure that already exists in TypeScript.
3. Global input capture (CGEvent taps) requires the same Accessibility permission regardless of runtime. No native advantage.
4. The UI (dashboards, data tables, config panels, activity timelines) is exactly where web technologies excel. Even a Swift app would embed WKWebView for these, creating two rendering paradigms.
5. Electron gives one runtime, one language, one debugger, built-in IPC, and crash-isolated utility processes — no custom bridge needed.

**Alternatives considered:**
- Native Swift: Two runtimes (Swift + embedded Node.js), hand-rolled bridge, no tooling support for the bridge, slow builds, thin AI/automation ecosystem. Rejected.
- Tauri 2.0: Two runtimes (Rust + Node sidecar), sidecar lifecycle management is real engineering cost for zero user-facing value, cross-process debugging. Binary size and memory advantages are negligible given Playwright browser binaries (~200+ MB) and workload memory. Rejected.

**Risk mitigation:** All business logic lives in `core/` with zero Electron imports. Electron-specific wiring is isolated in `electron/`. If migration is ever needed, business logic lifts out cleanly.

**Full writeup:** [`decisions/DESKTOP_FRAMEWORK_DECISION.md`](DESKTOP_FRAMEWORK_DECISION.md)

**Approved by:** Rustan

---

## 2026-02-16 — Product features restructured from 6 → 6 (different 6)

**Changed:** Rewrote `product/product-features.md` to align with the design system and CEO feedback on PR #6. Moved technical memory architecture to `architecture/llm-architecture.md`. Updated `product/product-overview.md` to reflect new feature names and fix stale Electron/Tauri reference.

**From → To:**

| # | Before | After |
|---|--------|-------|
| 1 | App Ecosystem | Apps (includes App Gallery) |
| 2 | Execution Modes | MCP Integrations (new — split from App Ecosystem + Tools & Connections from design system) |
| 3 | Activity Capture & Context Engine | Chat (merges On-Demand Chat + Proactive Suggestions) |
| 4 | Proactive Suggestions | MCP Browser (merges Execution Viewer from App Ecosystem + Execution Modes) |
| 5 | On-Demand Chat | Automations (new — from design system) |
| 6 | Memory | Context (merges Activity Capture + Context Card from design system + user-facing Memory summary) |

**Why:**
1. CEO (Scott) reviewed PR #6 and identified that the previous feature list conflated Apps and MCP Integrations, was missing 5 features from the design system (App Gallery, MCP Browser, Automations, Tools & Connections, Context Card), and had overly technical Memory content that belongs in the architecture doc.
2. The feature set now maps to how users think about the product, not how the engineering is structured. Apps are what you install. MCP Integrations are what you connect. Chat is how you talk to the AI. MCP Browser is how you watch it work. Automations are rules that run without you. Context is what the AI knows about you.
3. Memory's four-layer model, data pipeline, and RAG flow are engineering internals — moved to llm-architecture.md where they belong. Product-features.md retains a user-facing summary ("the AI remembers your past conversations, preferences, how you communicate").

**Cost impact:** None. This is a documentation restructure, not a technical change.

**Approved by:** Scott

---

## 2026-02-16 — Desktop salvage strategy: gut all repos in place

**Changed:** Established the migration strategy for moving from the existing `coworkai` monitoring platform to Cowork.ai Sidecar. Three independent analyses (Gemini CLI, Codex, Claude Opus) were synthesized, then revised based on deep research into native addon replacements and the agent orchestration layer.

**From → To:**
- Four active repos (`coworkai-desktop`, `coworkai-agent`, `coworkai-activity-capture`, `coworkai-keystroke-capture`) → Gut `coworkai-desktop` and `coworkai-agent` in place, keep capture addon repos as-is
- Custom C++ native addons for capture → **Keep both** (deep research found open-source replacements have critical capability gaps — see [NATIVE_ADDON_REPLACEMENT_RESEARCH.md](NATIVE_ADDON_REPLACEMENT_RESEARCH.md))
- `coworkai-agent` tracking engine → Keep capture orchestration (activity buffers, keystroke chunking, SQLite persistence), kill timer/sync/media/employer auth. Mastra.ai handles AI agent orchestration separately.
- Tracking-oriented SQLite schema → New schema designed for context/memory/embeddings with `libsql` built-in vectors (replaced `better-sqlite3` + `sqlite-vec`)
- Single-process main architecture → Multi-process isolation (capture utility, agents/RAG utility, Playwright child, sandboxed renderer)

**Why:**
1. There is no old product to maintain separately. Forking adds complexity (new repo, new CI, new signing setup) for zero benefit. Gut in place preserves all infrastructure — CI/CD, signing certs, bundle ID, team access, git history.
2. Custom native addons (`coworkai-keystroke-capture`, `coworkai-activity-capture`) were initially planned for replacement with `uiohook-napi` + `active-win`. **Deep research reversed this:** uiohook-napi lacks character mapping, has an unresolved Electron deadlock bug, and is dormant (last release March 2024). get-windows (formerly active-win) has no Windows URL extraction, no fallback strategy, and spawns child processes. The custom addons have battle-tested capabilities that would need to be rebuilt from scratch.
3. `coworkai-agent` is more than a tracking engine — it's the capture orchestrator. `ActivityManager`, `KeystrokeBuffer`, clipboard detection, and SQLite flush patterns are reusable. Archiving it means rebuilding that orchestration from scratch. Instead, gut the tracking-specific code (timer, sync, media, employer auth) and keep the capture plumbing.
4. Three independent analyses converged on the same core decisions, differing mainly on native addon strategy (Gemini wanted to keep, Opus argued for replacement based on syscall-level comparison). Subsequent deep research proved Gemini right.

**Cost impact:** Keeping custom addons and the agent repo means maintaining three repos and their CI. Cost of maintenance is less than cost of rebuilding capabilities (character mapping, 3-tier detection, Accessibility fallback, Windows UI Automation, capture orchestration).

**Alternatives considered:**
- Fork `coworkai-desktop` (original decision): Adds complexity with no benefit — no old product to maintain, CI/CD would need re-setup. Rejected.
- Fully new repo (Codex): Loses all infrastructure. Rejected.
- ~~Replace custom native addons (Opus): Originally accepted as "same OS APIs."~~ **Reversed after deep research.** Open-source replacements have critical gaps (no character mapping, Electron deadlock, no Windows URLs). Gemini was right. Now rejected.
- ~~Archive `coworkai-agent` (original decision):~~ **Reversed.** The agent contains reusable capture orchestration that would need to be rebuilt. Gut the tracking code, keep the capture plumbing.

**Full writeup:** [`decisions/DESKTOP_SALVAGE_PLAN.md`](DESKTOP_SALVAGE_PLAN.md)

**Approved by:** Rustan

---

## 2026-02-16 — Database stack: libsql for everything (replaces better-sqlite3 + sqlite-vec)

**Changed:** Unified the database stack to `libsql` — one SDK for capture data, agent memory, and vector search. Replaces `better-sqlite3` + `sqlite-vec` (two separate native modules).

**From → To:** `better-sqlite3` (structured storage) + `sqlite-vec` (vector extension) + custom Mastra adapter → `libsql` (sync package for capture) + `@mastra/libsql` (native Mastra integration for agents). One `.db` file, one SQL dialect, one native module.

**Why:**
1. Fact-check disproved the original blocker. "libsql is unproven in Electron" was wrong — Beekeeper Studio (v4.6+, June 2024) and Outerbase Studio Desktop both ship it in production.
2. A custom Mastra adapter for better-sqlite3 + sqlite-vec is 4-6 days of work (not 2-3 as initially estimated) plus ongoing maintenance tracking Mastra interface changes. libsql gets native Mastra integration for free.
3. Two SQL dialects (sqlite-vec's `vec_f32()` vs libsql's `F32_BLOB`) means vector queries can't be shared between app code and agent code. One dialect eliminates this.
4. sqlite-vec has a `.dylib.dylib` macOS packaging bug and requires `asarUnpack` workarounds. libsql's built-in vectors have no extension packaging.
5. Electron 37.1.0 ships Node 22.16.0 — satisfies `@mastra/libsql`'s requirement (Node >= 22.13.0). Mastra runs in an Electron utility process with crash isolation via built-in IPC. No sidecar or HTTP hop needed.

**Cost impact:** None — both stacks are free/local. Saves 4-6 days of custom adapter engineering and eliminates ongoing adapter maintenance.

**Alternatives considered:**
- Stay with better-sqlite3 + sqlite-vec (original plan): Proven but requires custom Mastra adapter (~500-800 lines for vector alone), two native modules, sqlite-vec packaging friction. Rejected — libsql eliminates all of this.
- Hybrid (libsql sync for capture + @mastra/libsql for agents): This IS the recommended approach. Both access the same WAL-mode `.db` file.

**Full research:** [`decisions/DATABASE_STACK_RESEARCH.md`](DATABASE_STACK_RESEARCH.md)

**Approved by:** Rustan

---

## 2026-02-12 — Added product overview, rewrote README

**Changed:** Created `product/product-overview.md` and rewrote `README.md` to lead with what Cowork.ai is before explaining the repo structure.

**From → To:** README explained the repo but not the product. No standalone product overview existed. → README opens with product explanation and links to full product overview. New `/product` directory added.

**Why:** Issue #3. The repo had strong technical architecture (v4), business strategy (v4), design system (v1.0), and GTM (v1.1) docs, but no single document explaining what Cowork.ai is, what it does, and who it's for. A new engineer, investor, or partner had to reverse-engineer the product from docs that assumed prior context. This isn't a cost/model/routing change, but it's structural — every other doc in this repo is more useful when the reader already understands the product.

**What the product overview covers:** Plain-language product explanation, two-brain architecture summary, desktop sidecar interaction model, MCP integration, feature set (chat, apps, MCP browser, automations), target users, Go2 relationship, roadmap, and links to all detailed docs.

**What we intentionally excluded (per issue discussion):** Detailed user personas, acceptance criteria, sprint roadmaps, competitive analysis matrices. Those are product management process artifacts — they belong in Linear/GitHub Issues, not in the decision layer. Cowork-brain captures reasoning, not execution tracking.

**Approved by:** Scott

---

## 2026-02-15 — README rewrite: ecosystem and rake framing

**Changed:** Rewrote README to frame Cowork.ai around the ecosystem Go2 already owns and the rake on token pass-through, not around building an AI agent.

**From → To:** README opened with "desktop AI agent for remote workers" and led with the technical product (local model, MCP). → README opens with the market opportunity (workers nobody else is building for), adds an explicit Business Model section (5.5% rake on cloud token pass-through), and positions the AI capabilities as the mechanism, not the differentiator.

**Why:**
1. The first version put Cowork.ai in the "AI agent" lane — same positioning as OpenAI, Anthropic, and every other company with 100x more resources building agent frameworks. That's a fight we lose.
2. The actual advantage is the ecosystem: 2M+ verified worker profiles, thousands of small business clients, hardware specs on file, zero CAC. Nobody else has this distribution in this market.
3. The business model is a rake on inference, not building or hosting models. That needed to be stated explicitly — the first README never mentioned how we make money.
4. "Why Now" section now ends with "these are enablers, not differentiators" — MCP, local models, and cost collapse are available to everyone. Owning the ecosystem where they get used is not.

**What stayed:** Trust Paradox section (unchanged — it's the strongest insight in the repo), The One Rule, repo guide structure, Current State table with clickable links.

**What was added:** The Opportunity section (market framing), The Business Model section (tier table + 5.5% rake), small business clients as a distribution asset.

**What was removed:** "What Does NOT Live Here" section (defensive, unnecessary — The One Rule already covers scope).

**Approved by:** Scott

---

## 2026-02-16 — AI SDK version: v6 (not v5)

**Changed:** Decided to start on Vercel AI SDK v6 (`ai` ^6.0.x, `@ai-sdk/react` ^3.0.x) instead of matching the AIME Chat reference app's v5 (`ai` ^5.0.93).

**From → To:** Open question (v5 for reference compatibility vs v6 for current) → AI SDK v6.

**Why:**
1. Mastra 1.0+ supports both v5 and v6 natively. `@mastra/core` v1.4.0 bundles both `@ai-sdk/provider-v5` and `@ai-sdk/provider-v6` via npm aliasing. There is no Mastra constraint blocking v6.
2. The breaking changes are mechanical renames (`CoreMessage` → `ModelMessage`, `textEmbeddingModel()` → `embeddingModel()`, `@ai-sdk/react` v2 → v3, tool helper name swaps) with an automated codemod (`npx @ai-sdk/codemod v6`). They are not architectural.
3. The patterns we're copying from AIME Chat — custom `ChatTransport<UIMessage>`, Zod-validated chunk protocol, `useChat()` hook, `tool()` + Zod schemas — are stable across both versions. Same architecture, different type names.
4. Mastra hides the AI SDK version on the backend (agent orchestration, tools, embeddings all go through Mastra APIs), but `@ai-sdk/react` is a direct frontend dependency — our `IpcChatTransport` implements AI SDK's interface directly, not through Mastra. Starting on v6 avoids an eventual migration of this frontend layer.
5. v6 shipped Dec 22, 2025 and is at 6.0.86 (stable). Starting on v5 means adopting a one-major-version-behind dependency from day one.

**Cost impact:** None. Both versions are open source. Avoids future migration cost.

**Alternatives considered:**
- Start on v5 to copy AIME Chat code verbatim: We're adapting patterns, not copying files. The renames are trivial. Rejected — starting behind guarantees a migration later.

**Approved by:** Rustan

---

## 2026-02-17 — Startup failure policy: degraded boot with timeout

**Changed:** Defined explicit startup failure behavior. Main process no longer blocks indefinitely waiting for utility processes — it times out after 10s and launches the renderer in degraded mode.

**From → To:** Implicit all-or-nothing boot (if Agents fails to start, app appears dead) → Timeout + degraded mode (UI always launches, broken services shown as disabled with retry).

**Why:**
1. The Agents & RAG Utility is the riskiest startup component — DB init, provider registry, Mastra, MCP connections, embedding queue. If any step fails, the current architecture blocks the renderer forever.
2. Mastra-in-utility-process is still unproven (Open Question #1). A failed PoC attempt would leave the app unlaunchable during development.
3. Desktop apps must always show UI. A user who double-clicks the app and sees nothing will assume it's broken and uninstall. Degraded mode with a clear banner ("Agent features unavailable — reconnecting...") sets expectations correctly.
4. Service health IPC channels (`system:health`, `system:retry`) enable the renderer to react to state changes without polling.

**Cost impact:** None. Pure architecture improvement.

**Alternatives considered:**
- No timeout, rely on process auto-restart only: Still blocks renderer. Rejected — user sees nothing while retries happen.
- Launch renderer immediately (before utilities): Renderer would show empty state for all features during normal startup too. Rejected — unnecessary flash of disabled UI on happy path.

**Approved by:** Rustan

---

## 2026-02-17 — MCP credentials: Electron safeStorage (replaces plaintext) — updated from keytar

**Changed:** OAuth tokens and API keys for MCP server connections encrypted via Electron `safeStorage` API instead of plaintext JSON files on disk. Originally specified `keytar` — reversed to `safeStorage`.

**From → To:** `{userData}/.mcp/{serverUrlHash}_tokens.json` (plaintext) → encrypted via `safeStorage` API, persisted to `cowork.db` keyed by `cowork-mcp-{serverUrlHash}` (OS-backed encryption at rest). Non-secret metadata (server URL, scopes, expiry) remains in JSON file.

**Why:**
1. OAuth tokens for Zendesk, Gmail, Slack are high-value credentials. Plaintext files on disk are accessible to any app with read permissions.
2. `keytar` is deprecated and archived (since Dec 2022). No longer maintained. VS Code already migrated off it.
3. Electron `safeStorage` API has been available since Electron 15 (Sep 2021). We're on Electron 37.1.0 — 22 major versions of stability. Uses macOS Keychain / Windows DPAPI under the hood.
4. Zero additional native dependencies — `safeStorage` is built into Electron. Removes one ASAR unpack entry and one native rebuild from `forge.config.ts`.
5. The `OAuthClientProvider` from `@modelcontextprotocol/sdk` can be wrapped to use `safeStorage.encryptString()` / `safeStorage.decryptString()` — the interface stays the same.

**Cost impact:** Net negative — removes `keytar` native dependency, simplifies build pipeline.

**Alternatives considered:**
- `keytar`: Original choice. Deprecated, archived, unmaintained since 2022. Would add unnecessary native binding. Reversed.
- Encrypted file with app-generated key: Key must be stored somewhere — ends up using OS encryption anyway, adding a layer of indirection for no benefit. Rejected.

**Approved by:** Rustan

---

## 2026-02-17 — Context capture scope: screen recording deferred to v0.2

**Changed:** Screen recording was removed from v0.1 Context capture scope and explicitly deferred to v0.2.

**From → To:** v0.1 context defaults implied screen recording existed as opt-in stream (`Off (opt-in)`) → v0.1 has no screen recording stream; feature reserved for v0.2.

**Why:**
1. v0.1 is focused on the minimum stable sidecar loop (capture context, route/model decisions, MCP actions, proactive notifications). Screen recording adds media capture lifecycle complexity that is not required for v0.1 validation.
2. Screen capture has a higher trust/privacy sensitivity than the other capture streams. Deferring it reduces risk in initial rollout while preserving the same non-surveillance posture.
3. The architecture and product docs had diverged (one document marked screen capture not in v0.1 while another still listed it as active opt-in). Locking scope to v0.2 removes implementation ambiguity.

**Cost impact:** Lowers v0.1 implementation and QA cost (no screen-capture pipeline, retention path, or UI consent flow in first release). Defers those costs to v0.2.

**Alternatives considered:**
- Keep screen recording in v0.1 as opt-in: rejected — expands surface area and privacy review burden in the first ship.
- Remove screen recording entirely from roadmap: rejected — coached visible automation still benefits from optional screen context in later versions.

**Approved by:** Rustan

---

## 2026-02-17 — Apps permission boundary: tools-only (agent access via `platform_chat` tool)

**Changed:** Removed direct app-level agent permission (`chat` / `permissions.agent`) from the active schema and architecture contract. Apps now access agent reasoning only through a platform-provided MCP tool (`platform_chat`) using the same `callTool()` path as other tools.

**From → To:** Mixed model (`callTool(...)` + direct app SDK `chat(...)` with special agent permission) → Tools-only model (`callTool(...)` for all app actions, including agent-as-tool via `platform_chat`).

**Why:**
1. Product contract is explicit: "Apps get tools, not agents." The direct `chat(...)` path created a policy contradiction between product and architecture docs.
2. One permission path is easier to reason about and audit. Tool grants stay in one table (`app_permissions`) and one evaluator (preload tool permission check), reducing edge-case drift.
3. Safety and cost controls remain centralized. The platform still chooses model/routing/budget when `platform_chat` executes, but apps never receive direct agent surface area.
4. The tools-only contract simplifies installer UX (no extra "agent chat" toggle) and keeps SDK semantics consistent for template and generic app exports.

**Cost impact:** Neutral to slightly positive. No new infrastructure; removes special-case permission logic and associated QA paths.

**Alternatives considered:**
- Keep direct `chat(...)` and reinterpret it as "not really agent access": rejected — wording still conflicts with product contract and duplicates permission pathways.
- Remove agent-style access for apps entirely: rejected — apps still need platform reasoning in some flows; agent-as-tool preserves capability without expanding surface area.

**Approved by:** Rustan

---

## 2026-02-17 — Capture data model: independent streams (not parent-child)

**Changed:** Capture streams are independent peers in the database, not parent-child entities. Keystroke chunks and clipboard events carry denormalized app context (`app_name`/`bundle_id`/`source_app`) so each row is self-contained. `activity_session_id` columns across capture tables are optional correlation hints, not structural dependencies.

**From → To:**
- Parent-child model: keystroke chunks and clipboard events are children of activity sessions, linked by `activity_session_id` FK → Independent streams: each table is a self-contained timeline, correlated by overlapping timestamps at read time. `activity_session_id` is nullable and optional.
- Keystroke chunks depend on JOIN to `activity_sessions` to know which app was active → Keystroke chunks carry `app_name` and `bundle_id` directly (same pattern clipboard events already use with `source_app`).

**Why:**
1. product-features.md describes five independently toggleable capture streams. The database should mirror the product model — streams are peers, not a hierarchy.
2. The parent-child model existed because the old `coworkai-agent` sync pipeline shipped activities with nested keystrokes as payload containers. That sync pipeline is gone — data is local-only.
3. Self-contained rows mean "what was typed in which app?" is answerable from `keystroke_chunks` alone — no JOIN needed for the most common consumer query (communication pattern extraction).
4. Eliminates write-path coupling: each stream buffers and flushes independently. The orchestration layer (`CaptureBufferManager`) coordinates lifecycle events (activity change → flush keystroke chunk), but this is a pipeline concern, not a schema invariant.
5. The flush coupling problem (1000-char keystroke threshold triggering activity session flushes) dissolves entirely — there's no parent to flush.

**Schema changes:**
- Added `app_name TEXT` and `bundle_id TEXT` to `keystroke_chunks`
- Added `idx_keystroke_chunks_app_name` index
- Updated all `activity_session_id` DDL comments to "optional correlation hint"
- Added "Why capture streams are independent peers" design note to database-schema.md

**Cost impact:** None. Two additional TEXT columns on keystroke_chunks (negligible storage at our volume). Simpler write path, simpler queries.

**Full analysis:** [`decisions/CAPTURE_FLUSH_COUPLING_ANALYSIS.md`](CAPTURE_FLUSH_COUPLING_ANALYSIS.md)

**Approved by:** Rustan

---

## 2026-02-17 — Capture flush semantics: decouple keystroke max-length from activity session boundaries

**Changed:** Locked capture write semantics so 1000-char keystroke limits flush only `keystroke_chunks` (not `activity_sessions`). Added durability rules: parent-first session persistence, active-session heartbeat, one-directional end ordering, and deterministic startup recovery for stale open sessions.

**From → To:**
- 1000-char threshold could end both chunk and activity session (legacy sync-era behavior leaked into target architecture) → 1000-char threshold ends chunk only; activity sessions end only on focus/idle/sleep/shutdown/recovery-close.
- Implicit capture durability semantics → explicit invariants:
  - insert `activity_sessions` at session start (`ended_at = NULL`)
  - maintain `last_observed_at` heartbeat while active
  - on session end, flush open chunk first, then finalize session row
  - on startup, deterministically close stale open sessions from latest available signal (clamped to boot time)
- Ambiguous short-session handling → canonical rule: apply `<500ms` threshold in read-path analytics (focus/summaries), never by deleting raw capture rows on write path.

**Why:**
1. Semantic correctness: `activity_sessions` model user attention windows, not storage payload boundaries. Keystroke volume is not a valid signal that attention moved.
2. Product behavior integrity: coupled flushes fragment context into artificial micro-sessions, degrading Context Card clarity, focus detection, and RAG recall quality.
3. Data integrity under crashes: without parent-first + recovery semantics, decoupling can create orphaned chunks or indefinitely open sessions.
4. Cross-doc consistency: architecture, schema, and sprint implementation guidance must encode the same boundary rules or implementation drift is guaranteed.

**Cost impact:** Neutral to low-positive. No new external services or paid dependencies. Slightly more write activity from `last_observed_at` heartbeat updates, offset by simpler downstream analytics logic and reduced debugging cost from deterministic recovery behavior.

**Alternatives considered:**
- Keep coupled flush behavior (chunk max-length also ends activity): rejected — preserves sync-era artifact, fragments semantic sessions, weakens focus/RAG correctness.
- Decouple flushes without heartbeat/recovery rules: rejected — leaves crash edge cases unresolved (open sessions/orphan risk) and undercounts passive read-only work.
- Drop short sessions at write-time: rejected — loses raw ground-truth data and makes threshold tuning/audit harder.

**Approved by:** Rustan
