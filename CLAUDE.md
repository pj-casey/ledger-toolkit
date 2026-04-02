# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental` — UI/UX overhaul in progress. See REDESIGN_VISION_V5.md for the full design vision.

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~2,450 lines)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. **Two modes:**

**Diagnostic Mode:** Overview, Accounts, Errors, Timeline, Network, APDU, Raw tabs. Copy Summary/Full. Error → Timeline jump. Error → Account linking. Severity accents.

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

## State variables (17)

`logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `hoveredTab`, `qualityOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`

---

## Functions (35)

Utilities (3): `copyText`, `downloadJSON`, `fmtBal`
Address (3): `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`
Tokens (1): `fetchTokenChains`
Diagnosis (1): `diagnose`
Parsing (2): `parseLogs`, `parseAppJson`
Extraction (8): `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `logQualityScore`, `extractActivity`, `extractAnalytics`
Diagnostic UI (6): `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard`
Customer View (7): `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText`
JSON Tree (2): `JTNode`, `JTTree`
App (1): `App`
Scoped: `decodeApduStatus`, `copyCustomerSummary`

---

## Implemented UX changes (experimental branch)

- ErrCard/Timeline severity left border accents
- Top bar stats clickable → navigate to relevant tab
- Overview stat cards contextual colors + clickable
- Card hover states (err-hover, acct-hover)
- Error → Timeline jump (`jumpTo`)
- Error → Account linking (`linkedAcct`, `goToAcct`)
- Timeline warning accents (yellow)
- Account error badges (`errCount`)
- Loading state for large files
- Raw tab: JSON tree viewer (JTNode, JTTree) with search, expand controls, copy path
