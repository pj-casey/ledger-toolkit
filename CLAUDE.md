# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design reference:** REDESIGN_VISION_V5.md — sidebar + main area layout spec (implemented).
**Completed sprint:** DEVICE_APP_VERSIONS_SPEC.md + DEVICE_APP_VERSIONS_ADDENDUM.md — device coin app version detection and display (implemented).

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,100 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Sidebar (Diagnosis block + Issues/Accounts/Timeline/Advanced sections) + main content area. Copy Summary/Full/Customer via dropdown. Error → Timeline jump. Error → Account linking. Severity accents. Device app version detection and display.

**Customer View:** Sidebar + main area layout. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links.

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

## Device App Detection — How It Works

**Primary source:** `actions-manager-event` entry with `message: "result"` and `data.result.installed[]` array. Present when user opened My Ledger (Manager). Contains every installed app with `name`, `version`, `availableVersion`, `updated` (boolean). This is the richest data source — no external API or APDU parsing needed.

**Secondary source:** `live-dmk-logger` `[exchange]` entries containing GetAppAndVersion APDU responses (`b001` command). Only captures the one app that was running during the session. Returns app name + version. Filters out BOLOS (dashboard).

**No data:** When device wasn't connected. `extractDeviceApps()` returns `null`. UI shows required apps as "not detected" with guidance to connect device and re-export.

**`CURRENCY_TO_APP` mapping (46 entries):** Maps account currency IDs to verified Ledger catalog app names. Key non-obvious mappings: `tezos → 'Tezos Wallet'`, `ripple → 'XRP'`, `cardano → 'Cardano ADA'`, `internet_computer → 'InternetComputer'`. All EVM L2s (`polygon`, `arbitrum`, `base`, etc.) map to `'Ethereum'`.

**`inferRequiredApps(accounts)`:** Groups accounts by required device app. Deduplicates. Returns `[{appName, currencyIds, chains}]`.

---

## Design tokens

```
bg:#131214  panel:#1C1D1F  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

Aligned with Ledger's LDLS design system (bg-base, bg-surface, text-default, text-muted, text-success, accent).

Fonts: Inter (body) + JetBrains Mono (mono).
Helpers: `bdg(color)`, `crd`, `Ico.*` (14 icons), `DN` (6 devices), `MF`, `fmtBal()`, `copyText()`, `TX_EXPLORERS[]`

---

## Layout (V5 — implemented)

**Diagnostic Mode:** 240px sidebar + full-width main area.
- Sidebar: Diagnosis block (severity dot + sentence + Copy dropdown) → Issues section → Accounts section → Timeline → Advanced (Network/APDU/Raw)
- Main area: renders content for selected section
- Issues and Accounts can be expanded simultaneously
- Advanced uses accordion (one sub-item at a time)
- Navigation: one click → content loads in main area, active section gets left accent bar

**Customer View:** 240px sidebar + main area + 210px info bar.

See REDESIGN_VISION_V5.md for the full design rationale, navigation behavior table, and empty states spec.

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `hoveredTab`, `qualityOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`

**DiagSidebar:** `section`, `issuesOpen`, `accountsOpen`, `advancedOpen`, `advSub`, `copyOpen`

**CustomerView internal:** `selectedAcct`, `summCopied`, `ajDragOver`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 9 | Theme colors |
| `TC` | 9 | Log type badge colors |
| `DN` | 6 | Device model names (`nanoS`, `nanoSP`, `nanoX`, `stax`, `europa`→Flex, `apex`→Nano Gen5) |
| `CHAINS` | 60 | Chain registry + address explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns (39 hw, 31 sw, 12 server) |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |
| `CURRENCY_TO_APP` | 46 | Currency ID → device app name (verified against Ledger catalog) |

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

After file processing, `logData` contains:

- `entries`, `dev`, `accts`, `errs`, `syncDur`, `apdu`, `quality`, `activity`, `analytics`, `rawText`
- `deviceApps` — from `extractDeviceApps()`: `{installed[], deviceModelId, deviceInfo, source}` or `{openedApps[], source}` or `null`
- `requiredApps` — from `inferRequiredApps()`: `[{appName, currencyIds, chains}]`

---

## Quality Score (10 fields / 100 pts)

| Field | Pts |
|---|---|
| Device model | 10 |
| Firmware version | 10 |
| Ledger Live version | 10 |
| Active account (ops>0) | 20 |
| OS info | 5 |
| User ID | 5 |
| Sync duration | 10 |
| No critical errors | 10 |
| Log volume (>100 entries) | 10 |
| Device app info | 10 |

Device app info: full marks for `manager-result`, half for `apdu-only`, zero when `null`.

---

## Implemented UX (experimental branch)

- V5 sidebar + main area layout (replaces tab bar)
- Diagnosis block with severity sentence + Copy dropdown (Quick Summary / Full Export / Customer Summary)
- Expandable Issues/Accounts/Advanced sections in sidebar
- ErrCard/Timeline severity left border accents
- Stat cards clickable → navigate to relevant section
- Overview stat cards contextual colors + clickable
- Card hover states (err-hover, acct-hover)
- Error → Timeline jump (`jumpTo`)
- Error → Account linking (`linkedAcct`, `goToAcct`)
- Timeline warning accents (yellow)
- Account error badges (`errCount`)
- Loading state for large files
- Raw: JSON tree viewer (JTNode, JTTree) with search, expand controls, copy path
- Keyboard shortcuts (Ctrl+1 through Ctrl+7)
- Empty states for all sections
- Contextual guidance lines in relevant views
- `DN` `apex` → "Nano Gen5" (was incorrectly "Apex")
- "Ledger Live" label (was "App ver" — fixes all 7 UI locations + 5 copy text outputs)
- Device Apps card on Overview: three scenarios (Manager data / APDU-only / no device), collapsible "Other installed apps"
- Device app version badges on AcctCard (✓ up to date / ⚠ outdated / ✕ missing / muted detected)
- Device app data in Copy Summary/Full/Customer outputs via `buildDeviceAppsLines` helper
- Quality score: 10th field "Device app info" (10 pts, rebalanced from 9 fields)
