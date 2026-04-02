# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design references:** REDESIGN_VISION_V5.md (sidebar layout), VIEWPORT_REDESIGN_SCOPE.md (fixed-viewport dashboard)

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,400 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser
- **Fixed viewport** — root is `height:100vh, overflow:hidden`. The page never scrolls. Each view fills available space with a fixed header zone + scrollable content panel.

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Health status pills. Device app detection. Firmware update status. Timeline density visualization. Error distribution. APDU device communication. Copy Summary/Full/Customer.

**Customer View:** Sidebar + main area + info bar. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links. dApp usage history.

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

## Sidebar

```
Diagnosis block (severity dot + sentence + Copy dropdown)
⚠ Issues          — expandable, grouped issue titles
👛 Accounts        — expandable, funded accounts with chain dots
🔌 Device  ●       — direct nav, subtitle: model + FW, status dot (green/yellow/red)
📋 Timeline        — direct nav
⚙ Advanced         — expandable: Network, Raw JSON
```

Device status dot: green (all OK), yellow (outdated), red (apps missing), hidden (no data).

---

## Main Area Views

### Overview (Diagnosis Detail) — 3-zone dashboard grid

**Zone 1 (top, fixed):** Health summary pills (clickable: Device/FW/Apps/Sync/Errors with color-coded status) + Copy Summary/Full buttons + stat cards (Entries/Accounts/Operations/Issues) + activity badges.

**Zone 2 (middle, flex:1):** Two columns. Left 40%: device summary line (clickable to Device) + Environment card. Right 60%: issues preview with internal scroll panel.

**Zone 3 (bottom bar, 56px):** Session info (date/time/duration/active/chain pills) left. Quality score with SVG ring + expandable details (popover upward) right.

### Device
Fixed header: Device & App card (model, firmware ✓/⚠, language, paired, Ledger Live, commit, Wallet Sync status, MEV).
Scrollable: Device Apps (summary + expandable list + other installed + storage count) + APDU Exchanges.

### Issues
Fixed header: Severity counts + error distribution bar (CSS stacked bar: hardware/software/server proportions) + diagnostic summary + repeating patterns.
Scrollable: Category-grouped ErrCards.

### Accounts
Fixed header: Account health squares (colored blocks: chain color if funded, gray if empty) + funded/empty count + filter + Copy IDs / Export.
Scrollable: AcctCards.

### Timeline
Fixed header: Density strip (minimap: 80 time buckets, height = event count, color: gray/red errors/purple device) + search + type filter + export + count.
Scrollable: Timeline rows.

### Network
Fixed header: Summary/count.
Scrollable: Network entry rows.

### Raw JSON
Fixed header: Search + expand controls + count + Copy raw text.
Scrollable: JTTree.

---

## Device App Detection

**Primary:** `actions-manager-event` result with `installed[]` array. Full app list with versions and update status.
**Secondary:** `live-dmk-logger` APDU `b001` responses. Single running app only.
**None:** Returns null. UI shows guidance.
**AcctCard badges:** Only render for ⚠ outdated and ✕ missing. Green suppressed.

---

## Visual Additions

- **Health summary pills** — Overview Zone 1, colored status indicators for device/firmware/apps/sync/errors
- **Error distribution bar** — Issues view, thin CSS stacked bar showing hw/sw/server proportions
- **Timeline density strip** — Timeline view, 80-bucket minimap with red error spikes, purple device events
- **Account health squares** — Accounts view, colored blocks per account (chain color = funded, gray = empty)
- **Quality score ring** — Overview Zone 3, SVG circular progress arc beside score text
- **Sidebar device dot** — 6px colored dot on Device section (green/yellow/red)
- **Timeline purple markers** — left border on device event types (live-dmk-logger, list-apps, actions-manager-event, device-action, hw, manager)

---

## Keyboard shortcuts

Ctrl+1 Overview | Ctrl+2 Issues | Ctrl+3 Accounts | Ctrl+4 Device | Ctrl+5 Timeline | Ctrl+6 Network | Ctrl+7 Raw JSON

---

## Design tokens

```
bg:#131214  panel:#1C1D1F  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

Fonts: Inter (body) + JetBrains Mono (mono).

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `appsListOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `globalSearch`, `showGlobalResults`
**DiagSidebar:** `section`, `issuesOpen`, `accountsOpen`, `advancedOpen`, `advSub`, `copyOpen`
**CustomerView:** `selectedAcct`, `summCopied`, `ajDragOver`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 9 | Theme colors |
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

## Functions

Utilities (3): `copyText`, `downloadJSON`, `fmtBal`
Address (3): `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`
Tokens (1): `fetchTokenChains`
Diagnosis (1): `diagnose`
Parsing (2): `parseLogs`, `parseAppJson`
Extraction (8): `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `logQualityScore`, `extractActivity`, `extractAnalytics`
Device Apps (3): `extractDeviceApps`, `inferRequiredApps`, `parseGetAppAndVersionResponse`
Diagnostic UI (6): `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard`
Customer View (7): `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText`
JSON Tree (2): `JTNode`, `JTTree`
Sidebar (1): `DiagSidebar` (inline in App)
App (1): `App`
Scoped: `decodeApduStatus`, `copyCustomerSummary`, `buildSummaryText`, `buildFullText`, `buildCustomerText`, `buildDeviceAppsLines`, `SectionHeader`

---

## logData shape

- `entries`, `dev`, `accts`, `errs`, `syncDur`, `apdu`, `quality`, `activity`, `analytics`, `rawText`
- `deviceApps` — `{installed[], deviceModelId, deviceInfo, firmware, source}` or `{openedApps[], source}` or `null`
- `requiredApps` — `[{appName, currencyIds, chains}]`

---

## Quality Score (10 fields / 100 pts)

Device model 10 | Firmware 10 | Ledger Live 10 | Active account 20 | OS info 5 | User ID 5 | Sync duration 10 | No critical errors 10 | Log volume 10 | Device app info 10
