# Cowork.ai — Repo Inventory

All application code lives in separate repos. This file is the single source of truth for what exists, what each repo does, and its disposition in the Sidecar transition.

---

## Repos

| Repo | GitHub | Description | Disposition |
|------|--------|-------------|-------------|
| coworkai-desktop | [go2impact/coworkai-desktop](https://github.com/go2impact/coworkai-desktop) | Electron app (Electron 37.1.0, React 19, Forge packaging) | **Gut in place** — keep shell, remove tracker logic |
| coworkai-activity-capture | [go2impact/coworkai-activity-capture](https://github.com/go2impact/coworkai-activity-capture) | Native C++ addon — active window/URL detection | **Keep as-is** |
| coworkai-keystroke-capture | [go2impact/coworkai-keystroke-capture](https://github.com/go2impact/coworkai-keystroke-capture) | Native C++ addon — keystroke/mouse input capture | **Keep as-is** |
| coworkai-agent | [go2impact/coworkai-agent](https://github.com/go2impact/coworkai-agent) | Capture orchestration, agent configs, sync logic | **Reference only** — lift patterns, then archive |
| cowork-brain | [go2impact/cowork-brain](https://github.com/go2impact/cowork-brain) | Architecture, decisions, strategy, design | **Active** — the decision layer |

## Dependency Graph

```
coworkai-desktop
├── @engineering-go2/coworkai-activity-capture  (native addon)
├── @engineering-go2/coworkai-keystroke-capture  (native addon)
└── @engineering-go2/coworkai-agent             (being replaced by Mastra.ai)
```

## Build Status

Last verified: 2026-02-17

| Repo | npm install | Build | Notes |
|------|------------|-------|-------|
| coworkai-activity-capture | PASS | PASS | macOS arm64, Node 22.16.0 |
| coworkai-keystroke-capture | PASS | PASS | macOS arm64, Node 22.16.0 |
| coworkai-desktop | FAIL | — | Blocked: expired nut-tree token, missing coworkai-video-capture package |
| coworkai-agent | — | — | Reference only, not maintained |

## Key Decisions

- **Desktop framework:** Electron (Swift and Tauri evaluated and rejected)
- **Codebase origin & salvage decisions:** [system-architecture.md § Codebase Origin](architecture/system-architecture.md#codebase-origin)
- **Target architecture:** [system-architecture.md](architecture/system-architecture.md)

## Documentation Status

All 3 maintained repos now have:
- `CLAUDE.md` — AI assistant context (what it is, disposition, build, key files)
- `README.md` — Human-readable orientation with Sidecar context
