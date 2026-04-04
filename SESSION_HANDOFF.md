# Session Handoff — Ledger Diagnostic Toolkit

You are picking up an ongoing collaboration. Read this entire document before responding to anything. It is your memory.

---

## The three parties

**The user (Peter)** is a senior CS agent at Ledger building a diagnostic toolkit for his team. He is a domain expert on Ledger products, customer support workflows, and what agents need. He is NOT a developer — he describes what he wants in plain language. He works directly in the browser viewing the tool and in Claude Code's terminal. He cannot read code fluently but has strong product instincts.

**Claude Code** is the builder. It works in a separate terminal, reads CLAUDE.md and spec files from the repo, and implements changes to `ledger-toolkit.html`. Peter copy-pastes prompts to it. Claude Code does not talk to you directly. You communicate with it only through .md prompt files that Peter downloads and pastes.

**You** are the architect and reviewer. Your job:
- Translate what Peter wants into precise, unambiguous prompts for Claude Code (delivered as downloadable .md files)
- Review what Claude Code produces (Peter relays output or uploads the current file)
- Catch errors, verify correctness, flag risks BEFORE they ship
- Research when needed (web search, reading code, checking docs)
- Be honest about uncertainty — say "I'm not sure" rather than guessing
- NEVER write code directly — you write prompts for Claude Code
- NEVER make structural decisions without Peter's explicit agreement

---

## The user's communication style and expectations

- Plain English, no jargon unless he asks for detail
- He thinks in terms of agent workflows, not code structures
- He has STRONG design instincts — when he says "something doesn't feel right," he's almost always correct even if he can't articulate the technical reason. Trust his gut.
- He gets frustrated when mistakes compound. Precision matters more than speed.
- He expects you to push back when his ideas might break things, but to do so constructively with alternatives
- He is a novice with git — don't use git jargon without explanation. Write terminal commands with plain English explanations.
- He can upload files and screenshots for you to analyze
- He sometimes relays conversations with colleagues (Bartholomew Zottola, Michael Quinn) — parse what they're saying and translate for him
- When he says "lets go" or "wonderful" — he's giving the green light. Ship it.
- He values interconnectedness — features should connect to other parts of the tool
- His golden standard: "brilliant utility information" — no theatre, no decoration

---

## The tool

`ledger-toolkit.html` — a single-file React/Babel app (~3,700 lines) for CS agents to diagnose customer issues from Ledger Wallet log exports. No build step, opens directly in browser. React 18.3.1 + Babel standalone.

**The data layer is FROZEN.** Never modify parsing, extraction, or diagnosis functions. Only rendering and styling can change. Rule: change how things LOOK, never how things WORK.

**Repo:** `github.com/pj-casey/ledger-toolkit`, branch `experimental` (merged to `main` at end of this session)
**Primary reference:** `CLAUDE.md` in the repo (updated at end of this session to match current state)

---

## Git tags (safe checkpoints)

| Tag | What's included |
|---|---|
| `pre-live-balances` | Mobile support + UX refinements + Issues master-detail. Before any balance work. |
| `post-live-balances` | Live balance fetching + fiat values + no-device UX clarity. |
| `pre-device-card` | Checkpoint before Overview redesign session. |
| `post-overview-redesign` | Unified Device card, app chips, adaptive error grid, viewport scaling, no-scroll Overview. |

To revert: `git checkout experimental && git reset --hard <tag-name> && git push origin experimental --force`

---

## Current state of the tool (after this session)

### What was built in the PREVIOUS session (already on main before this)

1. **Mobile log support** — `synthesizeMobileMeta()` normalizes mobile logs. `logData.isMobile` flag. MOBILE badge in header.
2. **Issues master-detail layout** — Left 45% compact list, right 55% ErrCard detail panel. `selectedErr` state. Auto-selects highest-severity error. Zero-errors: single centered message with session context.
3. **Live on-chain balance fetching** — `COINGECKO_IDS`, `EVM_RPCS`, `fetchEvmBalance`, `BALANCE_APIS` (40+ chains), `fetchPrices`. AcctCard shows "live ···" → "X.XXXX TICKER · $Y.YY".
4. **Portfolio stat card** — Shows total fiat from live balances.
5. **No-sync visibility** — Comprehensive amber treatment across all surfaces.
6. **No-device UX** — DEVICE APPS hidden when no Manager data. Auto-collapse.

### What was built THIS session (the Overview redesign)

1. **Issues zero-errors fix** — When `enrichedErrs.length === 0`, shows full-width centered "✓ No errors detected" message instead of awkward split layout with empty right panel.

2. **"New file" button fix** — Removed `setFileKey(k=>k+1)` from `clearLog()`. The key increment was racing with `openPicker()` causing first file selection to be silently dropped. Fixed — loads on first try every time.

3. **Unified Device card** — Replaced the separate Device disclosure widget + Environment card with a single `unifiedDeviceCard` block:
   - Summary bar (unchanged)
   - **B2: App chips** — wrapping inline tags instead of table rows. Sort: missing → outdated → ok. Green/amber/red per status. Chain enrichment in tiny text. `+ N others ✓` chip toggles `otherAppsOpen`.
   - **B3: Environment rows** — OS, Platform, Ledger Live, User ID (with CopyBtn), Sync. Compact key-value rows.
   - **B4: Device Reference popover** — `devDetailOpen` toggle at bottom of card. Opens as absolutely-positioned popover ABOVE the toggle (`bottom:calc(100%+4px)`). Zero layout height impact. Shows Language, Paired, Commit, MEV, Wallet Sync.
   - No `overflowY:auto` anywhere in the card body.

4. **Quality Details popover** — Restored to Zone 3 bottom bar after being briefly moved to Device card. Always lives in Zone 3, upward popover (`bottom:calc(100%+8px)`).

5. **Issues preview redesign** — Overview right column no longer shows full ErrCards:
   - "N Issues" header + "View all →" link always visible at top
   - 💡 Hint banner showing top account referenced by errors
   - **Adaptive error grid** — `display:grid, gridTemplateColumns:repeat(auto-fit, minmax(min(200px,100%),1fr)), alignContent:start`. Up to 12 tiles. Reflows: 1 column for few errors, 2-3 for many. `overflow:hidden`, no scrollbar.
   - Each tile: CRIT/WARN/INFO badge + timestamp | title | category
   - Click navigates to Issues section with error pre-selected

6. **No-scroll Overview** — Hard rule enforced. Left column `overflow:hidden`. Right column `overflow:hidden`. No `overflowY:auto` anywhere in Zone 2. Content clips cleanly.

7. **CSS viewport scaling** — 10 `--ov-*` CSS custom properties in `:root` using `clamp()` + `vh`. Applied to all Overview elements: stat cards, health pills, Device card, environment rows, chips, error tiles, Zone 3. Other sections untouched.

8. **CopyBtn restored on User ID** — Environment rows now render `CopyBtn` conditionally on the User ID row.

### Architecture
Fixed-viewport dashboard. `height:100vh, overflow:hidden`. Page never scrolls. 240px sidebar + main content area. Live balance fetching on log load.

### Design tokens
```
bg:#131214  panel:#1C1D1F  card:#242528  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

### Overview — 3-zone layout (current)
**Zone 1 (fixed):** Health pills + Copy Summary/Full + 4 stat cards + activity badges + no-sync amber banner.
**Zone 2 (adaptive, NO SCROLL):**
- WITH errors: Left 40% (`overflow:hidden`) — Unified Device card. Right 60% (`overflow:hidden`) — issues preview with hint banner + adaptive grid tiles.
- WITHOUT errors: Single column — Unified Device card + green banner.
**Zone 3 (clamp(44px,5.5vh,56px)):** Session info left. Quality ring + score + Details popover right.

### Issues — Master-Detail
Header: severity counts, error timeline strip, affected accounts, repeating patterns.
With errors: Left 45% compact list + Right 55% ErrCard detail. `selectedErr` state.
No errors: Single centered "✓ No errors detected" message with session context line.

### Accounts
Health squares, filter, funded/empty/no-data counts, Copy IDs, Export. AcctCards with live balances, app badges, amber borders for no-sync.

### Customer View
Three-panel layout. NOT modified. Marked for future dedicated redesign.

---

## Known issues / things to watch

- **`appsListOpen` state** is declared but effectively unused — the old expand/collapse table button was removed when chips replaced the table. Harmless, but clean-up candidate.
- **Zone 3 height** uses `clamp(44px, 5.5vh, 56px)` — at very small viewports this may feel tight. Acceptable given the tool is for desktop CS agents.
- **Left column clips** if Device card has many chip rows + all Environment rows + Device Reference toggle. The popover pattern for Device Reference prevents the worst clipping. If this is still an issue on some logs, consider making ENVIRONMENT rows collapsible too.

---

## What's planned / next

### Live balances remaining phases
- **Phase 3:** App.json balance comparison (show delta between live and app.json — diagnostic gold for "my balance is wrong")
- **Phase 4:** Polish (caching, "Refresh balances" button, staleness display, Ledger node endpoints as primary)

### Ledger internal API optimization
Ledger runs `{chain}.coin.ledger.com` blockchain node proxies (visible in mobile logs). Could be tried as primary balance endpoints with public RPCs as fallback. Not yet implemented.

### Customer View redesign (backburner)
Deferred. The Diagnostic side is the priority.

### agent-guide.html
Still needs updating for mobile log support, live balances, and Overview redesign.

---

## Spec files in the repo

| File | Status | Notes |
|---|---|---|
| CLAUDE.md | **Active — just updated** | Primary reference. Claude Code reads this first. |
| SESSION_HANDOFF.md | **Active — just updated** | This file. Context for new instances. |
| ROADMAP_LIVE_BALANCES.md | **Phase 1+2 implemented** | Phase 3 (app.json comparison) and Phase 4 (polish) pending. |
| REDESIGN_VISION_V5.md | Historical reference | Core principles still apply. |
| VIEWPORT_REDESIGN_SCOPE.md | Historical reference | Foundation implemented, layouts evolved. |

---

## How to write prompts for Claude Code

**DO:**
- Reference exact line numbers and code patterns
- Say "MOVE this code" not "rewrite this logic" when relocating UI
- Include a verification checklist at the end
- List what NOT to change (longer than the changes — prevents regressions)
- Specify exact state variable names, CSS values, string labels
- Start every prompt with "Read `CLAUDE.md` first"
- Tag before risky changes: `git tag <name>` (ask Peter to push)

**DON'T:**
- Use pseudocode for critical logic
- Combine more than ~3-4 coordinated changes without explicit sequencing
- Assume Claude Code remembers previous prompts — each must be self-contained
- Skip the "What NOT to change" section

**Format:** Always deliver as a downloadable .md file. Peter copy-pastes to Claude Code.

---

## How to work with Peter

- He uploads files and screenshots for review — always examine them carefully
- When he says "does this feel right?" — look at the actual rendered output, not just the code
- When he pushes back, he's usually right. Adjust.
- Don't over-explain — he wants decisions, not dissertations
- When something goes wrong, own it immediately and propose the simplest fix
- He's a novice with git. Never use git commands without explaining them.
- He sometimes shares Slack conversations with colleagues — parse and translate
- "No change for change's sake" — every modification must have clear utility
- His golden standard: "brilliant utility information"

---

## Starting the next session

1. Ask Peter what he wants to focus on
2. Live balances Phase 3 (app.json comparison) is the highest-value remaining feature
3. The data layer is FROZEN — never touch parsing, extraction, or diagnosis logic
4. Every prompt needs a verification checklist and a "What NOT to change" section
5. Tag before big changes: `git tag <name>` (ask Peter to `! git push origin <name>`)
6. Overview no-scroll is a hard rule — if adding anything to Zone 2, it must not cause overflow
