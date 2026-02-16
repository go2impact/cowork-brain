# Research: Native Addon Replacement Viability

| | |
|---|---|
| **Status** | Research complete — **reverses salvage plan decision #2** |
| **Last Updated** | 2026-02-16 |
| **Related** | [DESKTOP_SALVAGE_PLAN.md](./DESKTOP_SALVAGE_PLAN.md), [COWORKAI_TECHNICAL_REFERENCE.md](./COWORKAI_TECHNICAL_REFERENCE.md) |

---

## Context

The [Desktop Salvage Plan](./DESKTOP_SALVAGE_PLAN.md) decision #2 recommended replacing two custom native addons with open-source equivalents:

- `coworkai-keystroke-capture` → `uiohook-napi`
- `coworkai-activity-capture` → `active-win` (now `get-windows`) + thin AppleScript wrapper

This recommendation was based on Opus's analysis that both custom addons "call the same OS APIs" as their open-source replacements. **This research verifies that claim at the source-code and capability level.**

---

## Verdict

**The replacement plan is not viable.** Both proposed open-source alternatives have critical gaps, Electron integration problems, and missing capabilities that the custom addons handle. The original Opus analysis compared syscalls but not capabilities, edge case handling, or Electron-specific behavior.

---

## 1. Keystroke Capture: `coworkai-keystroke-capture` vs `uiohook-napi`

### What Opus Got Right

Both do call the same underlying OS APIs:

| API | Custom addon | uiohook-napi (via libuiohook) |
|---|---|---|
| macOS keyboard hook | `CGEventTapCreate` | `CGEventTapCreate` |
| Windows keyboard hook | `SetWindowsHookExW(WH_KEYBOARD_LL)` | `SetWindowsHookExW(WH_KEYBOARD_LL)` |
| macOS mouse hook | `CGEventTapCreate` (separate tap) | `CGEventTapCreate` (same tap, broader mask) |
| Windows mouse hook | `SetWindowsHookExW(WH_MOUSE_LL)` | `SetWindowsHookExW(WH_MOUSE_LL)` |
| Node.js bridge | `Napi::ThreadSafeFunction` | `napi_create_threadsafe_function` |

### What Opus Missed — Critical Capability Gaps

| Capability | Custom addon | uiohook-napi | Impact |
|---|---|---|---|
| **Character mapping** | Full UTF-8 via `CGEventKeyboardGetUnicodeString` (macOS) and `ToUnicodeEx` (Windows). Returns actual typed characters ("a", "é", "ñ"). | **Keycodes only.** Returns physical key position codes (like USB HID). No character/glyph mapping. `EVENT_KEY_TYPED` intentionally suppressed. | **Showstopper.** Cannot determine what the user typed. On AZERTY keyboard, pressing "A" key returns QWERTY "Q" keycode. |
| **Key repeat detection** | macOS: OS-provided via `kCGKeyboardEventAutorepeat`. Windows: manual atomic tracking with `std::atomic<bool>[256]`. | **None.** No repeat flag, no filtering. Held keys generate a stream of identical keydown events with no way to distinguish initial press from repeat. | Must build repeat detection in JS. Adds complexity and latency. |
| **CapsLock state** | Exposed as boolean in every event. | **Not exposed.** libuiohook tracks it internally (`MASK_CAPS_LOCK`) but uiohook-napi doesn't marshal it to JS. | Cannot determine toggle state. |
| **Left/right modifier distinction** | Separate keycodes for left/right shift, ctrl, alt, cmd. | **Collapsed.** `altKey` = left OR right. Cannot distinguish. Raw mask not exposed. | Minor for our use case but a capability loss. |
| **Event tap auto-recovery (macOS)** | Detects `kCGEventTapDisabledByTimeout` and re-enables automatically with logging. | libuiohook has re-enable logic but it still causes observable gaps (issue #47 — keyboard lag outside app). | Custom addon is more robust. |
| **Graceful permission handling (macOS)** | Separate `checkPermission()` and `requestPermission()` via `AXIsProcessTrusted`. App can guide user through flow. | **Crashes** if Accessibility denied. Issue #24 — `uv_mutex_destroy` on uninitialized mutex. Fix PR (#51) open since July 2024, unmerged. | Custom addon handles this gracefully. |
| **Control character mapping** | Ctrl+C → key="C" (not "\x03"). Smart fallback to base key name. | Not applicable — no character mapping at all. | Custom addon provides useful data even for control sequences. |

### uiohook-napi Electron-Specific Problems

1. **macOS deadlock in Electron (issue #23)** — libuiohook calls `dispatch_sync_f` to the main dispatch queue for Unicode key lookup. In Electron, this deadlocks because the main thread is blocked in Node.js's event loop. Multiple confirmations across macOS 12-15, Intel and ARM. **No fix merged.**

2. **Random crash on macOS (issue #50)** — FATAL ERROR in `tsfn_to_js_proxy` / `napi_call_function`. Reported on Electron 30 with ~50% frequency on startup. Likely a race condition in threadsafe function teardown.

3. **CGEventTap timeout (issue #47)** — Keyboard/mouse lag outside the app. Regression between v1.5.2 and v1.5.3.

### uiohook-napi Maintenance Concerns

| Metric | Value |
|---|---|
| Last npm release | v1.5.4, **March 2024** (~2 years ago) |
| Last commit | March 2024 |
| Maintainer | Single person (SnosMe). States "I don't own Mac" in issues. |
| Open crash fix PRs | 2 — sitting unmerged since mid-2024 |
| LGPL concern | Statically links libuiohook (LGPL-3.0). Needs legal review. |

### Recommendation: Keep `coworkai-keystroke-capture`

The custom addon provides UTF-8 character mapping, repeat detection, CapsLock state, graceful permission handling, and event tap auto-recovery — none of which uiohook-napi offers. The Electron deadlock bug alone is a showstopper. The dormant maintenance and LGPL licensing add further risk.

---

## 2. Activity Capture: `coworkai-activity-capture` vs `get-windows` (formerly `active-win`)

### What Opus Got Right

Both use `NSWorkspace.frontmostApplication` as the primary macOS app detection method. Both can extract browser URLs via AppleScript on macOS.

### What Opus Missed — Critical Capability Gaps

| Capability | Custom addon | get-windows | Impact |
|---|---|---|---|
| **macOS app detection** | **3-tier fallback**: (1) `NSWorkspace.frontmostApplication`, (2) iterate `runningApplications` checking `isActive`, (3) `CGWindowListCopyWindowInfo` layer-0 PID resolution | Single method: `NSWorkspace.frontmostApplication` only | Misses full-screen apps, system dialogs, virtual desktop edge cases. |
| **Browser URL (macOS) fallback** | AppleScript first (no permissions needed), then **Accessibility API fallback** via `AXUIElementRef` recursive traversal | AppleScript only. Fails silently if permission denied. | When AppleScript fails, custom addon has a fallback. get-windows returns nothing. |
| **Deep Chromium traversal** | `SearchForWebContent()` — recursive DFS through Accessibility tree hunting for `AXWebArea` roles | **None.** | Finds URLs in complex Chromium UI hierarchies that shallow queries miss. |
| **Browser URL (Windows)** | **Full COM UI Automation** with `IUIAutomation` — 28 localized address bar names per browser. Supports Chrome, Edge, Firefox, Brave, Opera. | **Not supported.** No URL extraction on Windows at all. | **Showstopper for v0.2** (Windows). Even for v0.1 Mac-only, this capability is lost for the future. |
| **Firefox support** | Supported on Windows via UI Automation. macOS: gets title via AppleScript, URL via Accessibility. | **Not supported on any platform.** Open issue since 2023. | Firefox is a significant browser. |
| **Architecture** | True N-API addon — runs in-process, loaded via `bindings`. Purpose-built for Electron (rebuilt against Electron ABI, unpacked from ASAR). | macOS: **spawns a Swift CLI binary** as child process. Windows: node-gyp addon. Linux: shells out to `xprop`. | Child process spawn on macOS creates permission inheritance problems in signed/sandboxed Electron apps. |
| **Electron integration** | Purpose-built. Rebuilt via forge `rebuildConfig`, unpacked via `AutoUnpackNativesPlugin`. | macOS: documented issues with permission inheritance (issues #135, #201). Windows: requires ABI rebuild (issue #143). | Multiple Electron-specific bugs in get-windows. Custom addon was designed for this. |
| **Latency** | Direct N-API function call, sub-millisecond. | macOS: process spawn + JSON parse on every call. | Higher overhead per query. |
| **Title cleaning** | Removes browser suffixes ("GitHub - Google Chrome" → "GitHub"). | Raw window title only. | Custom addon provides cleaner data. |
| **Battery impact** | Event-driven via `ActivityManager` (change-based). | Known battery drain on macOS 15+ when polling (issue #195). | Custom addon is more efficient. |
| **Special page handling** | Detects incognito/private tabs, returns `chrome://incognito`, `about:privatebrowsing`. | Not documented. | Better analytics data. |

### get-windows Maintenance Concerns

| Metric | Value |
|---|---|
| Last release | v9.2.3, August 2025 (bug fix for Linux UTF-8) |
| Open issues | 44 — many unresolved since 2022-2023 |
| Maintainer | Sindre Sorhus (maintains thousands of packages). Slow response times. |
| CI | **Only tests on macOS** — Windows and Linux not in CI. |
| JSON parsing bugs | Multiple open issues (#158, #170, #171) — special characters in window titles break JSON parsing from the Swift CLI. |

### Recommendation: Keep `coworkai-activity-capture`

The custom addon has 3-tier app detection, Accessibility API fallback, deep Chromium traversal, full Windows UI Automation with 28-name heuristic, Firefox support, in-process execution, and purpose-built Electron integration. get-windows lacks all of these. The "no Windows URL extraction" gap alone would require building a separate solution when v0.2 ships.

---

## 3. Revised Recommendation

**Reverse salvage plan decision #2.** Keep both custom native addons:

| Addon | Action | Reasoning |
|---|---|---|
| `coworkai-keystroke-capture` | **Keep and maintain** | Character mapping, repeat detection, CapsLock state, Electron-safe threading, graceful permissions. uiohook-napi lacks all of these and has an Electron deadlock bug. |
| `coworkai-activity-capture` | **Keep and maintain** | 3-tier detection, Accessibility fallback, deep Chromium traversal, full Windows UI Automation, in-process N-API. get-windows lacks all of these and spawns child processes. |

### What Changes from the Salvage Plan

| Salvage Plan (Before) | Revised (After) |
|---|---|
| Replace keystroke addon with `uiohook-napi` | **Keep** `coworkai-keystroke-capture`. Move to utility process. |
| Replace activity addon with `active-win` + AppleScript wrapper | **Keep** `coworkai-activity-capture`. Move to utility process. |
| Archive both addon repos (read-only) | **Keep active.** Continue maintenance. |
| Eliminate 2 repos + prebuild CI | **Keep** repos and CI. Cost of maintenance < cost of rebuilding capabilities. |
| `src/core/capture/` is 3 thin wrapper files (~150 lines) | `src/core/capture/` wraps existing addons, still thin but backed by battle-tested native code. |

### What Doesn't Change

- Fork-and-gut `coworkai-desktop` (decision #1) — unchanged
- Archive `coworkai-agent` (decision #3) — unchanged
- Multi-process architecture (decision #5) — unchanged, addons move to capture utility process
- New SQLite schema (decision #4) — unchanged
- Complete renderer rewrite (decision #6) — unchanged
- Local-first privacy (decision #7) — unchanged
- Kill timer, sync, auth (decision #8) — unchanged

---

## Changelog

**v1 (Feb 16, 2026):** Initial research. Deep-dive into all four codebases (two custom addons, two proposed replacements). Finding: replacement is not viable. Recommend keeping custom addons.
