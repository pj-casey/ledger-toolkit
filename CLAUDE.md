# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design references:** REDESIGN_VISION_V5.md (design philosophy), VIEWPORT_REDESIGN_SCOPE.md (fixed-viewport architecture), ROADMAP_LIVE_BALANCES.md (live balance architecture — now implemented)

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,700 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser
- **Fixed viewport** — root is `height:100vh, overflow:hidden`. The page never scrolls. Each view fills available space with a fixed header zone + scrollable content panel.
- **Live network calls** — fetches on-chain balances (public RPCs) and fiat prices (CoinGecko) on log load. No API keys required.

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file (desktop or mobile format)
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. Mobile log normalization: `date→timestamp`, reverse chronological order, filename parsing for app version + platform
4. **Live balance fetching:** Auto-fetches on-chain balances for all accounts (40+ chains), then batch-fetches fiat prices from CoinGecko
5. **Two modes:**

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Health status pills (clickable). Unified Device card on Overview (app chips, merged environment + reference popover). Firmware update status. Live balances + fiat values on AcctCards. Portfolio stat card. Timeline density visualization. Error distribution. APDU device communication (under Advanced). Copy Summary/Full/Customer with balances.

**Customer View:** Sidebar + main area + info bar. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links. dApp usage history.

---

## Golden rules

1. **Fixed viewport / single frame.** The page never scrolls. Each tab's content must be visible/accessible within its single window — through nested popups, pulldowns, collapsible sections, or clearly defined scrolling.

2. **Change how things LOOK, never how things WORK.** Data layer is frozen.

3. **Every visual element must answer exactly one question.** If it answers two questions, split it. If it answers zero questions, remove it.

4. **No theatre. No dead space.** Every pixel should earn its place. But don't sacrifice information density for tidiness — "maximally informative" is the goal.

---

## DO NOT MODIFY (data layer)

**Parsing:** `parseLogs()` (3-tier with brace scanner), `parseAppJson()` (encrypted-safe), `synthesizeMobileMeta()` (mobile log normalization)
**Extraction:** `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `extractActivity`, `logQualityScore`, `extractAnalytics`
**Device Apps:** `extractDeviceApps`, `inferRequiredApps`, `CURRENCY_TO_APP`, `parseGetAppAndVersionResponse`
**Error KB:** `ERR_DB` (82), `diagnose()`, `SEV`, `CAT`
**Chains:** `CHAINS` (60), `TX_EXPLORERS` (45+), `UTXO_NETS`, `getChain()`, `DC`, `DECIMALS`
**Address:** `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`, `fmtBal`
**Tokens:** `TOKEN_CONTRACTS`, `fetchTokenChains`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`
**Live Balances:** `COINGECKO_IDS`, `EVM_RPCS`, `fetchEvmBalance`, `BALANCE_APIS`, `fetchPrices` (these are new but frozen — don't modify the fetch logic, only the display)

## CAN MODIFY (rendering/styling on experimental branch)

Rendering and styling of ALL components may be modified for UX improvements. Rule: **change how things LOOK, never how things WORK.** Data flow, parsing, extraction, and diagnosis logic must not change.

---

## Three version concepts (NEVER conflate)

| Concept | Log field | Extraction | UI label |
|---|---|---|---|
| Ledger Live (desktop app) | `data.appVersion`, `release` | `info.appVer` | "Ledger Live" |
| Device firmware (secure element OS) | `data.deviceVersion`, `seVersion` | `info.fw` | "Firmware" |
| Device coin apps (on hardware) | `actions-manager-event` result, DMK APDU | `extractDeviceApps()` | "Device Apps" |

The label "App ver" must NEVER appear in the UI. Use "Ledger Live" for the desktop version.

---

## Fixed Viewport Layout

Root: `height:100vh, overflow:hidden, display:flex, flexDirection:column`
Top bar: `height:52px, flexShrink:0`
Main area: `flex:1, overflow:hidden`
Each view: `display:flex, flexDirection:column, height:100%, overflow:hidden` → fixed header (`flexShrink:0`) + scrollable panel (`flex:1, overflowY:auto`)

Scrollable panels have: `borderTop: 1px solid #3C3C3C`, `boxShadow: inset 0 2px 6px rgba(0,0,0,0.15)`, internal padding. The page itself never scrolls.

---

## Design tokens

```
bg:#131214  panel:#1C1D1F  card:#242528  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

Three elevation levels:
- `bg` (#131214) — base/background
- `panel` (#1C1D1F) — raised surfaces (sidebar, header, content panels)
- `card` (#242528) — elevated cards/containers sitting on panel surfaces (stat cards, environment card)

Cards use background elevation (T.card) instead of borders for visual separation. Borders are reserved for structural dividers (sidebar edge, header/footer, section separators, table row dividers).

Fonts: Inter (body) + JetBrains Mono (mono).

---

## Sidebar

240px fixed width. `DiagSidebar` component. Receives `liveBalances` prop for fiat hints.

```
OVERVIEW label (uppercase, muted)
Diagnosis block (severity dot + sentence + " · No sync data" when applicable + Copy dropdown)
─── (top border divider) ───
ISSUES          — direct nav to errors section (not expandable)
─── (top border divider) ───
ACCOUNTS  96 ⚠  — expandable, first 4 funded accounts with chain dots + fiat hints + "+N more funded" + empty/no-data count
─── (top border divider) ───
TIMELINE  540   — direct nav
─── (top border divider) ───
ADVANCED  ·     — expandable: Network (count), APDU (conditional on count>0), Raw JSON
```

**Sidebar fiat hints:** Funded accounts in the expanded list show right-aligned green dollar values from live balances (e.g., "$4,531"). Only shown when `liveBalance.status==='ok'` and `fiat > 0`.

**No-sync indicator:** When no sync data, ACCOUNTS subtitle changes to "No sync — log balances unavailable" and count shows amber ⚠.

**"no data" vs "empty" distinction:** Sidebar shows "╶╶ N empty · M no data" or "╶╶ N no balance data" depending on mix.

---

## Header bar

52px. Left: Ledger logo + "DIAGNOSTIC TOOLKIT". Center: Diagnostic/Customer View toggle (prominent, bold, purple glow on active). Right: file info + "New file" button + MOBILE badge (when mobile log). Toggle only visible when a log is loaded.

---

## Mobile Log Support

Mobile logs (from Ledger Wallet iOS/Android) differ from desktop:
- `date` field instead of `timestamp` — normalized by `synthesizeMobileMeta()`
- Reverse chronological order — reversed to chronological
- No `exportLogsMeta` — account IDs extracted from bridge schedule message
- App version + platform parsed from filename pattern: `ledgerwallet-mob-{version}-{build}-{date}-logs`
- No SyncSuccess analytics — accounts have 0 ops, no balance/duration data
- `logData.isMobile` flag set, MOBILE badge shown in header bar

**Mobile-aware rendering:** Device summary bar prefixed with "Mobile". Environment section in Device card shows "Ledger Wallet Mobile (iOS/Android)" for Platform. Copy summaries use "LW Mobile:" label. Accounts hint banner explains mobile limitations.

---

## Main Area Views

### Overview (Diagnosis Detail) — 3-zone dashboard

**HARD RULE: The Overview never scrolls.** No `overflowY:auto` on any column or container in Zone 2. Content must fit; use popovers for overflow.

**Zone 1 (top, fixed):** Health summary pills (clickable — first 4 expand device card, errors pill navigates to Issues) + Copy Summary/Full buttons + stat cards (Entries/Accounts/Portfolio/Issues) + activity badges. **No-sync amber banner** appears below pills when applicable.

**Stat cards:** Entries, Accounts (amber when no-sync), **Portfolio** (replaces Operations — shows total fiat from live balances, "···" while loading, "$X,XXX" when done, "(live)" label when no-sync), Issues. All font sizes use `var(--ov-stat-num)` / `var(--ov-stat-label)` CSS variables.

**Zone 2 (middle, flex:1, adaptive layout):**

*When errors > 0 — two columns:*
Left 40% (`overflow:hidden`): Unified Device card. Right 60%: issues preview — "N Issues / View all →" header + 💡 hint banner + adaptive error grid tiles (no scroll, clips at bottom).

*When errors = 0 — single column:*
Unified Device card + "No issues detected" green banner.

**Zone 3 (bottom bar, `clamp(44px, 5.5vh, 56px)`):** Session info (date/time/duration/active/chain pills) left. Quality score SVG ring + score + label + "Details" button right. Details opens a popover **upward** (`position:absolute, bottom:calc(100%+8px)`).

#### Unified Device Card

Lives in the Overview left column. Replaces the former Device disclosure widget + separate Environment card.

**Collapsed (summary bar):** "Nano X · FW 2.6.1 ✓ · 8 apps (all up to date) · Sync active" with chevron. Click to expand (`deviceExpanded` state). Purple left border when expanded. Same text-building logic as before.

**Expanded body** (when `deviceExpanded` is true — `overflow:hidden`, no scroll):

**B1 — No-device warning** (conditional): amber banner when `!modelId && !modelName`.

**B2 — DEVICE APPS** (gated: `req.length>0 && da && da.source==='manager-result'`):
- "DEVICE APPS" uppercase label + `"N/N up to date"` / `"all up to date"` right-aligned
- **App chips** (NOT a table): `display:flex, flexWrap:wrap, gap:var(--ov-gap-chips)`. Sort order: missing → outdated → ok.
  - OK: green chip `✓ AppName v1.2.3`
  - Outdated: amber chip `⚠ AppName v1.2.3`
  - Missing: red chip `✕ AppName`
  - Chain enrichment: `· ETH POL BSC` appended in tiny text when 2+ chains share an app
  - `+ N others ✓` chip toggles `otherAppsOpen` for non-required installed apps
- Amber warning banner below chips when any apps are outdated/missing

**B3 — ENVIRONMENT** (always visible): compact key-value rows — OS, Platform, Ledger Live, User ID (with CopyBtn), Sync. Mobile-aware Platform/OS derivation.

**B4 — Device Reference** (popover on click): `devDetailOpen` state. Toggle row at bottom of card body with `▸`/`▾`. When open, renders an **absolutely-positioned popover** (`bottom:calc(100%+4px), zIndex:20`) showing Language, Paired, Commit, MEV, Wallet Sync rows. **Does NOT push content down** — zero layout height impact.

**Auto-collapse:** When no device was connected (`!modelId && !modelName && !fw`), `deviceExpanded` auto-collapses via useEffect.

**DEVICE APPS hidden when no Manager data:** Gated by `req.length>0 && da && da.source==='manager-result'`. No misleading missing-app warnings for no-device logs.

#### Issues Preview (right column, Overview only)

NOT the full Issues section. This is the Overview's compact preview:
- Fixed header: "N Issues" (color `T.error`) + "View all →" link right-aligned (always visible)
- 💡 Hint banner: shows top account referenced by errors + link to Accounts section
- **Adaptive error grid:** `display:grid, gridTemplateColumns:repeat(auto-fit, minmax(min(200px,100%),1fr)), alignContent:start`. Up to 12 errors. Each tile: severity badge (CRIT/WARN/INFO) + timestamp on top row, title below, category below. Tiles reflow — 1 column for few errors, 2-3 columns for many.
- `overflow:hidden` — tiles clip cleanly at bottom, no scrollbar. Agent clicks "View all →" for the rest.

### Issues — Master-Detail Layout

**Fixed header (compact):**
- Row 1: Severity counts inline (only non-zero) + "Copy Errors" button
- Row 2: Error timeline strip — numbered dots on session timeline, severity-colored, clickable (selects error in detail panel)
- Row 3 (conditional): Affected accounts summary
- Row 4 (conditional): Repeating patterns (only >5 errors + repeats)

**When errors > 0 — master-detail split:**
- Left panel (45%): Compact error list. Each row: severity dot + title + action text + timestamp. Selected row highlighted with severity color. Clicking a row shows its detail on the right.
- Right panel (55%): Full ErrCard for selected error. First (highest severity) error auto-selected on load.
- `selectedErr` state tracks selected error by `li` (log index). Timeline dots also update selection.

**When errors = 0 — single centered message:**
"No errors detected" + guidance text + session context line ("187 entries over 8.0s · 96 accounts · no sync data").

### Accounts
Fixed header: funded/empty/no-data counts + Copy IDs + Export + no-sync warning banner (when applicable) + account health squares (colored blocks — chain color for funded, amber tint for no-data, gray for confirmed empty, live-balance-aware) + filter input.
Scrollable: AcctCards (with app badges, live balance display, amber left border for no-sync accounts).

**AcctCard live balance display:**
- Collapsed: "live ···" while loading → "X.XXXX TICKER · $Y.YY" in green when done
- Expanded: "Live Balance" cell with loading/ok/error states + fiat USD
- "Balance (log)" cell shows sync data when available
- "no data" badge instead of "empty" when no sync data exists for the account

### Timeline
Fixed header: Density strip + search + type filter + export + count.
Scrollable: Timeline rows with purple left border on device event types.

### Network
Fixed header: Summary/count + error request warning.
Scrollable: Network entry rows. Empty state: explanation + "View Timeline →" button.

### APDU (under Advanced)
Fixed header: count + Export APDUs + rejection summary.
Scrollable: Exchange list with decoded status labels. 500 row cap.
Empty state: reconnect guidance + "Back to Overview →" button.

### Raw JSON
Fixed header: Progressive search + expand controls + count + Copy raw text.
Scrollable: JTTree with highlighted matches.

---

## Live Balance Fetching

**Architecture:**
- `COINGECKO_IDS` constant: maps 40+ chain currency IDs to CoinGecko coin slugs
- `EVM_RPCS` constant: 23 EVM chains mapped to publicnode.com RPC URLs
- `fetchEvmBalance(rpc, addr)`: JSON-RPC `eth_getBalance`, parses hex result
- `BALANCE_APIS` object: per-chain fetch functions for UTXO (wrapping existing UTXO_NETS), Solana, Tezos, XRP, Stellar, Cardano, NEAR, Tron, TON, Aptos, Sui, Stacks, Hedera, MultiversX, Kaspa, Cosmos, Algorand, Filecoin
- `fetchPrices(cgIds)`: batch CoinGecko `/simple/price` call (free, no key, 30 calls/min)
- `liveBalances` state in App component
- `useEffect` orchestrator: fires on `logData` change, staggers 150ms per account, then batch CoinGecko price fetch
- AcctCard: `liveBalance` prop, collapsed badge shows "live ···" loading / "X.XXXX TICKER · $Y.YY" / error states; expanded shows "Live Balance" cell

**Copy builders:** All 5 copy paths use `fmtAcctLine()` helper which includes live balance + fiat when available. PORTFOLIO total line appended after accounts section.

**Potential optimization:** Ledger's own blockchain nodes (`{chain}.coin.ledger.com`) could be tried as primary endpoints with public RPCs as fallback. Not yet implemented.

---

## CSS Viewport Scaling (Overview only)

The Overview uses CSS custom properties defined in `:root` for fluid scaling with viewport height via `clamp()`. Apply these **only inside the Overview section** — other sections use fixed pixel values.

```css
--ov-fs-xs: clamp(9px, 1.1vh, 11px)   /* labels, badges, timestamps */
--ov-fs-sm: clamp(10px, 1.3vh, 12px)  /* row text, banners */
--ov-fs-md: clamp(11px, 1.4vh, 13px)  /* summary bar, error titles */
--ov-fs-lg: clamp(13px, 1.6vh, 16px)  /* section headers */
--ov-pad-row: clamp(2px, 0.4vh, 4px)  /* row padding */
--ov-pad-section: clamp(6px, 1vh, 12px) /* section margin-top */
--ov-gap-chips: clamp(4px, 0.5vh, 6px) /* chip/tile gap */
--ov-gap-rows: clamp(4px, 0.6vh, 8px)  /* row gap */
--ov-stat-num: clamp(20px, 3vh, 32px)  /* stat card big number */
--ov-stat-label: clamp(10px, 1.2vh, 13px) /* stat card label */
```

Zone 3 height uses `clamp(44px, 5.5vh, 56px)`.

---

## No-Device UX

When a log has no device connection data:
- Device card auto-collapses (useEffect sets `deviceExpanded=false`)
- DEVICE APPS section hidden (gated by `da.source==='manager-result'`)
- Summary bar shows "No device connected during this session"

## No-Sync Visibility

When a log has accounts but no sync data (`!accts.some(a => a.dur != null || a.bal != null)`):
- Persistent amber banner on Overview below health pills
- Sidebar diagnosis sentence appended with "· No sync data"
- Sidebar ACCOUNTS subtitle changes, count shows amber ⚠
- Stat cards: Accounts amber, Portfolio labeled "(live)"
- AcctCards: amber left border for no-sync accounts
- Health squares: amber tint for no-data accounts, chain color for live-balance-funded
- Copy summaries: "⚠ NO SYNC DATA" warning after header
- AcctCard badges: "no data" instead of "empty"

---

## Contextual Guidance Pattern

**💡 Hint banners** (navigational tips): `background:rgba(187,176,255,0.06), border:1px solid rgba(187,176,255,0.15), borderRadius:6, padding:8px 12px` with 💡 emoji.

**⚠ Warning banners** (actionable problems): `background:rgba(255,189,66,0.06), border:1px solid rgba(255,189,66,0.25)` with ⚠.

**Empty states:** Emoji + headline + explanation + action button pointing to the next most useful section.

No raw italic muted text for guidance anywhere. Every hint has a visual container.

---

## Device App Detection

**Primary:** `actions-manager-event` result with `installed[]` array. Full app list with versions and update status.
**Secondary:** `live-dmk-logger` APDU `b001` responses. Single running app only.
**None:** Returns null. UI hides DEVICE APPS section entirely (no misleading "missing" warnings).
**AcctCard badges:** Only render for ⚠ outdated and ✕ missing. Green suppressed.
**Chain enrichment:** App rows show dependent chain tickers when 2+ chains require the same app.

---

## Keyboard shortcuts

Ctrl+1 Overview | Ctrl+2 Issues | Ctrl+3 Accounts | Ctrl+4 Timeline | Ctrl+5 Network | Ctrl+6 APDU | Ctrl+7 Raw JSON

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `appsListOpen` (declared, unused), `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `deviceExpanded`, `devDetailOpen` (Device Reference popover), `section`, `liveBalances`, `selectedErr`, `globalSearch`, `showGlobalResults`
**DiagSidebar:** `accountsOpen`, `advOpen`, `copyOpen`
**CustomerView:** `selectedAcct`, `summCopied`, `ajDragOver`
**JTTree:** `expanded`, `search`, `copiedPath`, `matchPaths`, `currentMatch`, `scrollContainerRef`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 10 | Theme colors (includes `card` elevation) |
| `TC` | 9 | Log type badge colors |
| `DN` | 6 | Device model names (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `CHAINS` | 60 | Chain registry + explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |
| `CURRENCY_TO_APP` | 46 | Currency ID → device app name |
| `COINGECKO_IDS` | 40+ | Chain currency ID → CoinGecko coin slug |
| `EVM_RPCS` | 23 | Chain → publicnode.com RPC URL |
| `BALANCE_APIS` | 17 | Chain → custom balance fetch function (non-EVM) |

---

## Spec files in repo

| File | Status | Purpose |
|---|---|---|
| `CLAUDE.md` | **Active — primary reference** | This file. Claude Code reads this before every task. |
| `ROADMAP_LIVE_BALANCES.md` | **Implemented (Phase 1+2)** | Live on-chain balance fetching. API registry, phased implementation. Phase 1 (EVM+UTXO) and Phase 2 (major L1s) done. Phase 3 (app.json comparison) and Phase 4 (polish) pending. |
| `SESSION_HANDOFF.md` | **Active — session memory** | Context for new Claude instances picking up the project. |
| `REDESIGN_VISION_V5.md` | Historical reference | Original design philosophy. Core principles still apply. Some structural specifics outdated. |
| `VIEWPORT_REDESIGN_SCOPE.md` | Historical reference | Fixed-viewport architecture. Foundation implemented, specific layouts evolved. |

---

## Git tags

| Tag | Commit | What's included |
|---|---|---|
| `pre-live-balances` | `20e7760` | Mobile support + UX refinements + Issues master-detail. Before any balance/fiat work. |
| `post-live-balances` | `9081d1c` | Live balance fetching + fiat values + no-device UX clarity. |
| `pre-device-card` | before Overview redesign | Checkpoint before unified Device card session. |
| `post-overview-redesign` | `01f89d4` | Unified Device card, app chips, adaptive error grid, viewport scaling, no-scroll Overview. |
