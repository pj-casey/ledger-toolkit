# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design references:** REDESIGN_VISION_V5.md (design philosophy), VIEWPORT_REDESIGN_SCOPE.md (fixed-viewport architecture), ROADMAP_LIVE_BALANCES.md (planned: live on-chain balance fetching)

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,350 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser
- **Fixed viewport** — root is `height:100vh, overflow:hidden`. The page never scrolls. Each view fills available space with a fixed header zone + scrollable content panel.

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Health status pills (clickable, expand device disclosure). Device info + apps as inline disclosure on Overview. Firmware update status. Timeline density visualization. Error distribution. APDU device communication (under Advanced). Copy Summary/Full/Customer.

**Customer View:** Sidebar + main area + info bar. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links. dApp usage history.

---

## Golden rules

1. **Fixed viewport / single frame.** The page never scrolls. Each tab's content must be visible/accessible within its single window — through nested popups, pulldowns, collapsible sections, or clearly defined scrolling.

2. **Change how things LOOK, never how things WORK.** Data layer is frozen.

3. **Every visual element must answer exactly one question.** If it answers two questions, split it. If it answers zero questions, remove it.

4. **No theatre. No dead space.** Every pixel should earn its place. But don't sacrifice information density for tidiness — "maximally informative" is the goal.

---

## DO NOT MODIFY (data layer)

**Parsing:** `parseLogs()` (3-tier with brace scanner), `parseAppJson()` (encrypted-safe)
**Extraction:** `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `extractActivity`, `logQualityScore`, `extractAnalytics`
**Device Apps:** `extractDeviceApps`, `inferRequiredApps`, `CURRENCY_TO_APP`, `parseGetAppAndVersionResponse`
**Error KB:** `ERR_DB` (82), `diagnose()`, `SEV`, `CAT`
**Chains:** `CHAINS` (60), `TX_EXPLORERS` (45+), `UTXO_NETS`, `getChain()`, `DC`, `DECIMALS`
**Address:** `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`, `fmtBal`
**Tokens:** `TOKEN_CONTRACTS`, `fetchTokenChains`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`

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

240px fixed width. `DiagSidebar` component.

```
OVERVIEW label (uppercase, muted)
Diagnosis block (severity dot + sentence + Copy dropdown → Quick Summary / Full Export / Customer Summary)
─── (top border divider) ───
ISSUES          — direct nav to errors section (not expandable)
─── (top border divider) ───
ACCOUNTS  13 ·  — expandable, first 4 funded accounts with chain dots + "+N more funded" + empty count
─── (top border divider) ───
TIMELINE  540   — direct nav
─── (top border divider) ───
ADVANCED  ·     — expandable: Network (count), APDU (conditional on count>0), Raw JSON
```

Section headers: uppercase labels (`fontSize:12, fontWeight:600, textTransform:uppercase, letterSpacing:0.04em`), dimmer subtitles (`color:#666666`), top border dividers (`1px solid #3C3C3C`).

Accounts capped at 4 visible funded accounts in sidebar. "+N more funded" link navigates to Accounts section. Sidebar never pushes Timeline/Advanced off-screen.

Issues is a single clickable row (like Timeline) — no expandable content. Issue titles are already visible in Overview.

**No Device sidebar entry.** Device info is consolidated into the Overview as an inline disclosure widget.

Advanced subtitle is dynamic: shows "Network · APDU · Raw" when APDU data exists, "Network · Raw data" when it doesn't.

---

## Header bar

52px. Left: Ledger logo + "DIAGNOSTIC TOOLKIT". Center: Diagnostic/Customer View toggle (prominent, bold, purple glow on active). Right: file info + "New file" button. Toggle only visible when a log is loaded.

---

## Main Area Views

### Overview (Diagnosis Detail) — 3-zone dashboard

**Zone 1 (top, fixed):** Health summary pills (clickable — first 4 expand device disclosure inline, errors pill navigates to Issues) + Copy Summary/Full buttons + stat cards (Entries/Accounts/Operations/Issues on elevated `T.card` backgrounds, no borders, no bracket accents) + activity badges.

**Zone 2 (middle, flex:1, adaptive layout):**

*When errors > 0 — two columns:*
Left 40% (overflowY:auto): Device disclosure widget + Environment card. Right 60%: issues preview with internal scroll panel + hint banners (💡 account references, Customer View tip).

*When errors = 0 — single column:*
Device disclosure widget + "No issues detected" green banner + Environment card.

**Zone 3 (bottom bar, 56px):** Session info (date/time/duration/active/chain pills) left. Quality score with SVG ring + expandable details (popover upward) right.

#### Device Disclosure Widget

Lives in the Overview left column, replaces the former Device tab.

**Collapsed (summary bar):** "Nano X · FW 2.6.1 ✓ · 8 apps (all up to date) · Sync active" with chevron. Click to expand. Purple left border when expanded.

**Expanded:** Opens inline below the summary bar. Contains:
- No-device warning (when applicable)
- Device Apps section: "DEVICE APPS" uppercase label + summary stats on second line + expand/collapse app list + column headers (App / Version) + chain tickers (for apps serving 2+ chains) + "v" prefixed versions + "Not installed" for missing apps + outdated indicators with arrows (v1.2.0 → v1.3.0) + "Other installed apps" toggle
- ⚠ warning banner when apps are outdated/missing (not italic text)
- "Details" toggle → "DEVICE REFERENCE" label + reference rows: Language, Paired (shows model names via DN lookup), Commit, MEV, Wallet Sync (uses ledgerSync + appJson.encrypted + walletsync error counting)

**Auto-expand:** When app problems are detected (missing or outdated), the disclosure auto-expands on load with the app list visible.

**`deviceExpanded` defaults to `true`** — the agent lands on Overview with device info visible.

#### Environment Card
Elevated card (T.card background, no border). Shows: OS, Platform, User ID, Sync duration. Soft row dividers (rgba(255,255,255,0.06)). Note: "Sync duration" label (not "Sync") to distinguish from "Wallet Sync" in device details.

### Issues

Reorganized for maximum viewport efficiency. All errors should be visible without scrolling when count is small (≤5).

**Fixed header (compact):**
- Row 1: Severity counts inline (only non-zero severities shown, colored dots + counts) + "Copy Errors" button (right-aligned, copies formatted summary of all errors)
- Row 2: Error timeline strip — numbered dots positioned along session timeline by actual timestamp, colored by severity. Dots are clickable (scroll to corresponding error card). Start/end timestamps in mono font. Below the strip: auto-generated clustering interpretation ("All 3 errors within 0.3s — likely cascade" or "Errors spread across 30s — likely independent issues").
- Row 3 (conditional): Affected accounts summary — only when errors reference specific accounts. Compact chips with chain dots, clickable to jump to Accounts section.
- Row 4 (conditional): Repeating patterns — only shown when >5 total errors AND repeated titles exist.

**Scrollable:** Flat list of ErrCards sorted by severity (critical → warning → info). No category group headers (each card already shows its category). Each card wrapped in `<div id="errcard-{li}">` for timeline scroll targeting.

**Removed:** Distribution bar (redundant), summary card (sidebar already shows this), category group headers (redundant with card content).

### Accounts
Fixed header: funded/empty count + Copy IDs + Export + no-sync warning banner (when applicable) + account health squares (colored blocks) + filter input.
Scrollable: AcctCards (with app badges for outdated/missing apps). No-sync footer hint banner. No-accounts empty state with Customer View guidance.

### Timeline
Fixed header: Density strip (minimap: 80 time buckets, height = event count, color: gray/red errors/purple device) + search + type filter + export + count.
Scrollable: Timeline rows with purple left border on device event types.

### Network
Fixed header: Summary/count + error request warning.
Scrollable: Network entry rows. Empty state: "No network requests" + explanation + "View Timeline →" button.

### APDU (under Advanced)
Own dedicated section. Fixed header: count + Export APDUs button + rejection summary (when rejections exist).
Scrollable: Exchange list with decoded status labels (OK/Rejected/Locked/etc), CLA bytes, CopyBtn per row. 500 row cap.
Empty state: guidance to ask customer to reconnect device + "Back to Overview →" button.
Only appears in sidebar when apduCount > 0.

### Raw JSON
Fixed header: Progressive search (auto-expands tree ancestors, scrolls to first match as you type, "1/47" counter with ▲▼ prev/next navigation, Enter/Shift+Enter keyboard shortcuts) + expand controls (Collapse/Level 1/2/3/All) + count + Copy raw text.
Scrollable: JTTree with highlighted matches (current match: strong purple + left border, other matches: subtle purple tint).
Empty state: explanation + re-export instructions.

---

## Contextual Guidance Pattern

All guidance text follows consistent patterns:

**💡 Hint banners** (navigational tips): `background:rgba(187,176,255,0.06), border:1px solid rgba(187,176,255,0.15), borderRadius:6, padding:8px 12px` with 💡 emoji. Direct, imperative language. Always ends with a next step. Clickable links to other sections where relevant.

**⚠ Warning banners** (actionable problems): `background:rgba(255,189,66,0.06), border:1px solid rgba(255,189,66,0.25)` with ⚠. Specific instructions for the agent.

**Empty states:** Emoji + headline + explanation + action button pointing to the next most useful section.

No raw italic muted text for guidance anywhere. Every hint has a visual container.

---

## Device App Detection

**Primary:** `actions-manager-event` result with `installed[]` array. Full app list with versions and update status.
**Secondary:** `live-dmk-logger` APDU `b001` responses. Single running app only.
**None:** Returns null. UI shows guidance.
**AcctCard badges:** Only render for ⚠ outdated and ✕ missing. Green suppressed.
**Chain enrichment:** App rows show dependent chain tickers when 2+ chains require the same app (e.g., Ethereum row shows "ETH, MATIC, ARB, BASE").

---

## Keyboard shortcuts

Ctrl+1 Overview | Ctrl+2 Issues | Ctrl+3 Accounts | Ctrl+4 Timeline | Ctrl+5 Network | Ctrl+6 APDU | Ctrl+7 Raw JSON

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `appsListOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `deviceExpanded`, `devDetailOpen`, `section`
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

---

## Spec files in repo

| File | Status | Purpose |
|---|---|---|
| `CLAUDE.md` | **Active — primary reference** | This file. Claude Code reads this before every task. |
| `ROADMAP_LIVE_BALANCES.md` | **Planned feature** | Live on-chain balance fetching for Diagnostic Accounts. API registry, phased implementation, app.json comparison. |
| `REDESIGN_VISION_V5.md` | Historical reference | Original design philosophy. Core principles still apply. Some structural specifics outdated. |
| `VIEWPORT_REDESIGN_SCOPE.md` | Historical reference | Fixed-viewport architecture. Foundation implemented, specific layouts evolved. |
| `DEVICE_APP_VERSIONS_SPEC.md` | Implemented | Device app detection. Phases 0-6 done. Phase 7 (runtime Manager API) not done. |
| `TOOLKIT_SNAPSHOT.md` | Outdated | Superseded by this file. Can be deleted. |
