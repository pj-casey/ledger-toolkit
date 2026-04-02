# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design reference:** REDESIGN_VISION_V5.md — sidebar + main area layout spec (implemented, extended with Device section).

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,300 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Sidebar (Diagnosis block + Issues/Accounts/Device/Timeline/Advanced sections) + main content area. Copy Summary/Full/Customer via dropdown. Error → Timeline jump. Error → Account linking. Severity accents. Device app version detection. Firmware update status. APDU device communication.

**Customer View:** Sidebar + main area layout. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links. dApp usage history.

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

## Layout (V5 + Device section)

**Diagnostic Mode:** 240px sidebar + full-width main area.

**Sidebar sections:**
```
Diagnosis block (severity dot + sentence + Copy dropdown)
⚠ Issues          — expandable, shows grouped issue titles
👛 Accounts        — expandable, shows funded accounts with chain dots
🔌 Device          — direct nav, subtitle shows model + firmware
📋 Timeline        — direct nav
⚙ Advanced         — expandable accordion: Network, Raw JSON
```

- Issues and Accounts can be expanded simultaneously
- Advanced uses accordion (one sub-item at a time)
- Device and Timeline are direct navigation (click loads view, no expand)
- Active section gets left accent bar

**Customer View:** 240px sidebar + main area + 210px info bar.

---

## Main area views

### Diagnosis Detail (Overview — default)
- 4 stat cards (Entries, Accounts, Operations, Issues) — clickable
- Activity badges (Manager, Swap, Device scan)
- Device summary line (compact, purple left border, clickable to Device section)
- Environment card (OS, platform, user ID)
- Issues preview (top 3)
- Session info bar, Quality score (expandable)

### Device section
- Device & App card (model, firmware ✓/⚠, language, paired, Ledger Live, commit, Wallet Sync, MEV)
- Device Apps (summary always visible, list collapsed by default, auto-expands on problems, "Other installed apps" collapsible, storage app count)
- APDU Exchanges (count, rejections, rows, export)

### Issues List, Accounts List, Timeline, Network, Raw JSON
Per V5 spec. See REDESIGN_VISION_V5.md.

---

## Device App Detection

**Primary:** `actions-manager-event` result with `installed[]` array. Full app list with versions and update status.
**Secondary:** `live-dmk-logger` APDU `b001` responses. Single running app only.
**None:** Returns null. UI shows guidance.
**AcctCard badges:** Only render for ⚠ outdated and ✕ missing. Green suppressed.

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

**App:** `logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `hoveredTab`, `qualityOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`
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
