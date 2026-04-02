# Ledger Diagnostic Toolkit — Codebase Snapshot
*Experimental branch — April 2026*

## File
- **Line count:** ~2,450
- **Stack:** React 18.3.1 + Babel 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- **No build step** — single HTML file

---

## Functions (35)

| Category | Functions |
|---|---|
| Utilities (3) | `copyText`, `downloadJSON`, `fmtBal` |
| Address (3) | `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr` |
| Tokens (1) | `fetchTokenChains` |
| Diagnosis (1) | `diagnose` |
| Parsing (2) | `parseLogs` (3-tier), `parseAppJson` (encrypted-safe) |
| Extraction (8) | `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `logQualityScore`, `extractActivity`, `extractAnalytics` |
| Diagnostic UI (6) | `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard` |
| Customer View (7) | `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText` |
| JSON Tree (2) | `JTNode`, `JTTree` |
| App (1) | `App` |
| Scoped (2+) | `decodeApduStatus`, `copyCustomerSummary` |

---

## State (17 in App)

`logData`, `fileName`, `loadErr`, `tab`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `hoveredTab`, `qualityOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`

**CustomerView internal:** `selectedAcct`, `summCopied`, `ajDragOver`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 9 | Theme colors |
| `TC` | 9 | Log type badge colors |
| `DN` | 6 | Device model names |
| `CHAINS` | 60 | Chain registry + address explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns (39 hw, 31 sw, 12 server) |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |

---

## parseLogs — 3-tier parser

**Tier 1:** `JSON.parse` on full text.
**Tier 2:** Brace-depth scanner. Lookahead/lookbehind quote validation (survives Thai/Arabic binary garbage with raw 0x22). Control char stripping. Per-object independent parsing.
**Tier 3:** Line-by-line fallback.

## parseAppJson — encrypted-safe

Detects Ledger Sync encryption. Extracts settings, devices, cryptoTokens, discoverAcctIds from readable sections. Returns `{accounts, byId, encrypted, settings, cryptoTokens, discoverAcctIds}`.

---

## Diagnostic Mode (current: tab-based)

7 tabs: Overview, Timeline, Accounts, Errors, Network, APDU (conditional), Raw

### Overview
4 stat cards (clickable), activity badges, Device & App card, Environment card, issues preview, session bar, quality score (expandable), Copy Summary/Full

### Timeline
Search, type filter, export, expandable rows (500 cap), error/warning left border accents

### Accounts
Filter, funded/empty count, copy IDs, export, AcctCard components (expandable), xpub scanner, error badges

### Errors
Severity bar, diagnostic summary, repeating patterns, category-grouped ErrCards, severity left borders, linked accounts, jumpTo timeline

### Network
Method badges, clickable URLs, 500 cap

### APDU
Conditional, 9 status codes, CLA byte, export, 500 cap

### Raw
JSON tree viewer (JTTree/JTNode), collapsible tree, search with match count, expand controls (Collapse/Level 1-3/All), copy path on hover, 500-item cap per node

---

## Customer View

Sidebar (240px) + Main area. Three-panel layout:
- CVSidebar: account list with chain dots, balances, error indicators
- Main: CVPortfolioView (default) or CVAccountDetail (on click)
- CVInfoBar: device info, session stats (hidden <1024px)

App.json enrichment: drop target banner, encrypted handling, balances/tx/tokens when unencrypted. App.json as primary source when log has 0 accounts. Copy Customer Summary.

---

## CSS classes

**Diagnostic:** `ledger-tab-bar`, `ledger-tab-btn`, `err-hover`, `acct-hover`
**JSON Tree:** `jt-row`, `jt-arrow`, `jt-copy-path`
**Customer View (22):** `cv-layout`, `cv-sidebar`, `cv-sidebar-header`, `cv-main`, `cv-infobar`, `cv-acct-row`, `cv-active`, `cv-chain-dot`, `cv-sim-banner`, `cv-portfolio-grid`, `cv-portfolio-card`, `cv-detail-card`, `cv-info-row`, `cv-section-title`, `cv-mode-toggle`, `cv-mode-btn`, `cv-drop-banner`, `cv-dragover`, `cv-enriched-banner`, `cv-tx-row`, `cv-tx-type`, `cv-bal-card`

---

## Error KB: 82 patterns | Chains: 60 | TX Explorers: 45+ | Quality Score: 9 fields/100pts
## Icons: 14 Ico components | Accessibility: 10+ aria-labels

---

## Git
- Repo: `github.com/pj-casey/ledger-toolkit`
- Branch: `experimental`
