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

`ledger-toolkit.html` — a single-file React/Babel app (~3,650 lines) for CS agents to diagnose customer issues from Ledger Wallet log exports. No build step, opens directly in browser. React 18.3.1 + Babel standalone.

**The data layer is FROZEN.** Never modify parsing, extraction, or diagnosis functions. Only rendering and styling can change. Rule: change how things LOOK, never how things WORK.

**Repo:** `github.com/pj-casey/ledger-toolkit`, branch `experimental`
**Primary reference:** `CLAUDE.md` in the repo (updated at end of this session to match current state)

---

## Git tags (safe checkpoints)

| Tag | What's included |
|---|---|
| `pre-live-balances` | Mobile support + UX refinements + Issues master-detail. Before any balance work. |
| `post-live-balances` | Live balance fetching + fiat values + no-device UX clarity. |

To revert: `git checkout experimental && git reset --hard <tag-name> && git push origin experimental --force`

---

## Current state of the tool (after this session)

### What was built THIS session (chronological)

1. **Mobile log support** — `synthesizeMobileMeta()` normalizes mobile logs (date→timestamp, reverse order, filename parsing for app version + platform, account IDs from bridge schedule, device info backfill). `logData.isMobile` flag. MOBILE badge in header.
2. **Mobile UX refinements** — Device summary bar "Mobile" prefix, Environment card "Ledger Wallet Mobile (iOS/Android)", copy summaries with "LW Mobile:" label, purple 💡 mobile hint in Accounts, OS derivation from platform in all 6 copy paths.
3. **Issues master-detail layout** — Left 45% compact error list, right 55% ErrCard detail panel. `selectedErr` state. First highest-severity error auto-selected. Error timeline dots update selection. Zero-errors state: single centered message with session context.
4. **Zero-scrollbar Overview attempt** — Tried pill popovers + full-width error table. Reverted after multiple failed attempts. Original two-column layout restored.
5. **Live on-chain balance fetching** — `COINGECKO_IDS`, `EVM_RPCS`, `fetchEvmBalance`, `BALANCE_APIS` (40+ chains), `fetchPrices`. Auto-fires on log load with 150ms stagger. AcctCard shows "live ···" → "X.XXXX TICKER · $Y.YY".
6. **Portfolio stat card** — Replaces Operations. Shows total fiat from live balances. "···" while loading, "$X,XXX" when done.
7. **Copy summaries with balances** — `fmtAcctLine()` helper used by all 5 copy paths. Account lines include balance + fiat. PORTFOLIO total appended.
8. **Sidebar fiat hints** — Right-aligned green dollar values next to funded accounts.
9. **No-device UX clarity** — DEVICE APPS hidden when no Manager data. Device disclosure auto-collapses. "no data" vs "empty" badges. Sidebar "no balance data" label.
10. **No-sync visibility** — Comprehensive amber treatment: persistent banner, sidebar indicators, stat card labels, AcctCard borders, health square colors, copy summary warnings.

### Architecture
Fixed-viewport dashboard. `height:100vh, overflow:hidden`. Page never scrolls. Each view has a fixed header + scrollable panel. 240px sidebar + main content area. Live balance fetching on log load.

### Design tokens
```
bg:#131214  panel:#1C1D1F  card:#242528  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

### Overview — 3-zone layout
**Zone 1 (fixed):** Health pills + Copy Summary/Full + 4 stat cards (Entries/Accounts/Portfolio/Issues) + activity badges + no-sync amber banner (conditional).
**Zone 2 (adaptive):**
- WITH errors: Two columns. Left 40%: device disclosure + environment card. Right 60%: issues preview + hint banners.
- WITHOUT errors: Single column. Device disclosure + green banner + environment card.
**Zone 3 (fixed bottom, 56px):** Session info left. Quality score ring + Details popover right.

### Issues — Master-Detail
**Header:** Severity counts, error timeline strip, affected accounts, repeating patterns.
**With errors:** Left 45% compact list + Right 55% ErrCard detail. `selectedErr` state.
**No errors:** Single centered message with session context line.

### Accounts
Health squares (live-balance-aware coloring), filter, funded/empty/no-data counts, Copy IDs, Export. AcctCards with live balances, app badges, amber borders for no-sync.

### Live Balance System
`BALANCE_APIS` covers: 4 UTXO (wrapping existing), 23 EVM (one pattern), 17 individual L1s. `fetchPrices` batch call to CoinGecko. `liveBalances` state. `fmtAcctLine()` for copy output. Portfolio stat card. Sidebar fiat hints.

### Customer View
Three-panel layout. NOT modified this session. Marked for future dedicated redesign.

---

## Mistakes made this session

### Zero-scrollbar Overview disaster
Attempted to eliminate scrolling on Overview via pill popovers and full-width error table. Three iterations failed — left column stretched with blank space, quality details overlapping, hint banners clipped. **Reverted to original two-column layout.** Lesson: the two-column layout with scrollable left panel was the right design. Scrolling within a panel is fine — it's page-level scrolling that's banned.

### Live balances initial break
First implementation of live balances caused a break (unclear what exactly). Reverted to `pre-live-balances` tag. Second attempt with both live balances AND no-device UX changes together succeeded. Lesson: always tag before big changes. The tagging practice saved us.

### Context window fog
By end of session, the conversation was very long and context compression caused missed details and repeated questions. Lesson: for long sessions, handoff to a fresh instance sooner. Don't push past the point where quality degrades.

---

## What's planned / next

### Immediate pending prompts (drafted but may not be applied yet)
- **no-sync-visibility.md** — 8-change comprehensive amber treatment for no-sync logs
- **issues-zero-errors-fix.md** — Single centered message for zero-errors instead of awkward split layout

### Live balances remaining phases
- **Phase 3:** App.json balance comparison (show delta between live and app.json — diagnostic gold for "my balance is wrong")
- **Phase 4:** Polish (caching, "Refresh balances" button, staleness display, Ledger node endpoints as primary)

### Ledger internal API optimization
Research found Ledger runs `{chain}.coin.ledger.com` blockchain node proxies (visible in mobile logs). Could be tried as primary balance endpoints with public RPCs as fallback. CoinGecko confirmed as safe bet for prices. Not yet implemented.

### Customer View redesign (backburner)
Deferred as dedicated project. The Diagnostic side is the priority.

### CLAUDE.md and agent-guide.html updates
CLAUDE.md updated this session. agent-guide.html still needs updating for mobile log support and live balances feature.

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
2. Check if the no-sync-visibility and issues-zero-errors-fix prompts have been applied yet
3. If live balances polish: Phase 3 (app.json comparison) is the next big win
4. If UX work: the Overview zero-scrollbar problem is unsolved but deprioritized
5. The data layer is FROZEN — never touch parsing, extraction, or diagnosis logic
6. Every prompt needs a verification checklist and a "What NOT to change" section
7. Tag before big changes: `git tag <name> && git push origin <name>`
