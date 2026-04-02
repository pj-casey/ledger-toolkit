# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design reference:** REDESIGN_VISION_V5.md — sidebar + main area layout spec (implemented).
**Current sprint:** DEVICE_APP_VERSIONS_SPEC.md — device coin app version detection and display.

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~3,000 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Sidebar (Diagnosis block + Issues/Accounts/Timeline/Advanced sections) + main content area. Copy Summary/Full/Customer via dropdown. Error → Timeline jump. Error → Account linking. Severity accents.

**Customer View:** Sidebar + main area layout. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer links.

---

## DO NOT MODIFY (data layer)

**Parsing:** `parseLogs()` (3-tier with brace scanner), `parseAppJson()` (encrypted-safe)
**Extraction:** `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `extractActivity`, `logQualityScore`, `extractAnalytics`
**Error KB:** `ERR_DB` (82), `diagnose()`, `SEV`, `CAT`
**Chains:** `CHAINS` (60), `TX_EXPLORERS` (45+), `UTXO_NETS`, `getChain()`, `DC`, `DECIMALS`
**Address:** `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`, `fmtBal`
**Tokens:** `TOKEN_CONTRACTS`, `fetchTokenChains`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`

## CAN MODIFY (rendering/styling on experimental branch)

Rendering and styling of ALL components may be modified for UX improvements. Rule: **change how things LOOK, never how things WORK.** Data flow, parsing, extraction, and diagnosis logic must not change.

## CAN ADD (new data layer functions for device app sprint)

New extraction functions may be added alongside existing ones. See DEVICE_APP_VERSIONS_SPEC.md for full spec. Key additions:
- `extractDeviceApps(entries, apduData)` — parse APDU B001 responses and DMK logger entries for coin app names/versions
- `inferRequiredApps(accounts)` — map accounts to required device apps
- `CURRENCY_TO_APP` — currency ID → device app name lookup
- `LATEST_APP_VERSIONS` — hardcoded latest app versions per device family (to be added in later phase)

Once stable, these should be added to the DO NOT MODIFY list.

---

## Three version concepts (NEVER conflate)

| Concept | Log field | Extraction | UI label |
|---|---|---|---|
| Ledger Live (desktop app) | `data.appVersion`, `release` | `info.appVer` | "Ledger Live" |
| Device firmware (secure element OS) | `data.deviceVersion`, `seVersion` | `info.fw` | "Firmware" |
| Device coin apps (on hardware) | APDU B001 responses, DMK logger | `extractDeviceApps()` | "Device Apps" |

The label "App ver" must NEVER appear in the UI. Use "Ledger Live" for the desktop version.

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

## Functions

Utilities (3): `copyText`, `downloadJSON`, `fmtBal`
Address (3): `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`
Tokens (1): `fetchTokenChains`
Diagnosis (1): `diagnose`
Parsing (2): `parseLogs`, `parseAppJson`
Extraction (8): `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `logQualityScore`, `extractActivity`, `extractAnalytics`
Diagnostic UI (6): `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard`
Customer View (7): `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText`
JSON Tree (2): `JTNode`, `JTTree`
Sidebar (1): `DiagSidebar` (inline in App)
App (1): `App`
Scoped: `decodeApduStatus`, `copyCustomerSummary`, `buildSummaryText`, `buildFullText`, `buildCustomerText`, `SectionHeader`

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
