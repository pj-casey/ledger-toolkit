# Ledger Diagnostic Toolkit — Codebase Snapshot
*Verified against ledger-toolkit.html — April 2026 (parser hardening + full CV)*

## File
- **Path:** `ledger-toolkit.html` (single-file app, no build step)
- **Line count:** 2,304
- **Stack:** React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3

---

## Named Functions (33 total)

**Utilities (3):** `copyText`, `downloadJSON`, `fmtBal`
**Address derivation (3):** `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`
**Token resolution (1):** `fetchTokenChains`
**Error diagnosis (1):** `diagnose`
**Parsing (2):** `parseLogs` (3-tier), `parseAppJson` (encrypted-safe)
**Data extraction (8):** `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `logQualityScore`, `extractActivity`, `extractAnalytics`
**Diagnostic components (6):** `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard`
**Customer View components (7):** `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText`
**App (1):** `App`
**Inline/scoped (1+):** `decodeApduStatus`, `copyCustomerSummary`

---

## App State Variables (16)

| Variable | Type | Purpose |
|---|---|---|
| `logData` | object/null | Parsed log data |
| `fileName` | string | Loaded file name |
| `loadErr` | string | Parse error message |
| `tab` | string | Active diagnostic tab |
| `searchTerm` | string | Timeline search |
| `typeFilter` | string | Timeline type filter |
| `expandedRow` | number/null | Expanded timeline row |
| `dragOver` | boolean | Drag hover state |
| `acctFilter` | string | Accounts filter |
| `hoveredTab` | string/null | Tab keyboard hint |
| `qualityOpen` | boolean | Quality score expanded |
| `sumCopied` | boolean | Copy Summary feedback |
| `fullCopied` | boolean | Copy Full feedback |
| `apduExportCopied` | boolean | Export APDUs feedback |
| `viewMode` | string | `'diagnostic'` or `'customer'` |
| `appJson` | object/null | Parsed app.json data |

**CustomerView internal state:** `selectedAcct`, `summCopied`, `ajDragOver`

---

## Constants

| Constant | Entries | Description |
|---|---|---|
| `T` | 9 | Theme colors |
| `MF` | — | Mono font-family string |
| `TC` | 9 | Log type → badge color |
| `DN` | 6 | Model ID → display name |
| `CHAINS` | 60 | Chain registry with address explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URL builders |
| `DC` | — | Default chain for unknowns |
| `DECIMALS` | 60+ | Currency ID → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns |
| `SEV` | 3 | Severity definitions (high/medium/low) |
| `CAT` | 4 | Category definitions (hardware/software/server/unknown) |
| `UTXO_NETS` | 4 | BTC/LTC/DOGE/BCH scanner configs |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |

---

## parseLogs(text) — 3-tier parser

### Tier 1: Full JSON.parse
Tries `JSON.parse` on the entire trimmed text. Works for well-formed minified JSON.

### Tier 2: Brace-depth scanner (robust recovery)
Character-by-character scan of raw text:
- Tracks `{}` depth (ignoring `[]`)
- **Quote validation:** Does NOT blindly toggle on `"`. Uses lookahead (closing `"` must be followed by `:` `,` `}` `]`) and lookbehind (opening `"` must be preceded by `:` `,` `[` `{`). Rejects garbage `0x22` bytes in binary content.
- **Escape handling:** `\` only treated as escape inside strings
- **Control char stripping:** Removes `[\x00-\x08\x0b\x0c\x0e-\x1f]` before `JSON.parse` per object
- Each complete `{...}` block at depth 0 is independently `JSON.parse`d. Failed objects are silently skipped.
- Returns all successfully parsed objects with logIndex, mobile field normalization

Handles: pretty-printed JSON arrays, binary garbage in locale strings, raw control characters (`\x0f`), malformed escape sequences, unescaped quotes in Arabic/Thai text, corrupted individual objects.

### Tier 3: Line-by-line fallback
Splits on `\n`, tries `JSON.parse` per line. Non-JSON lines become `{type:'raw', message: lineText}`.

---

## parseAppJson(text) — encrypted-safe

- Detects encrypted files: `data.accounts` is a string (Ledger Sync ciphertext), not array
- **Unencrypted:** extracts full accounts with operations, subAccounts, balances. Builds `byId` lookup map.
- **Encrypted:** returns `{encrypted: true, accounts: [], settings, cryptoTokens, discoverAcctIds}`
- **Settings extracted regardless:** devices, lastDevice, counterValue, language, locale
- **Token definitions extracted:** from `data.cryptoAssets.tokens`
- **Account IDs from discover:** `data.discover.currentAccountHist` values starting with `js:2:`
- Drop handler accepts both encrypted (`parsed.encrypted`) and unencrypted (`parsed.accounts.length > 0`) files

---

## Customer View Components

### `CustomerView({ logData, appJson, onAppJson })`
Top-level. Computes `enrichedAccts` via useMemo with three cases:
1. Log has accounts → enrich with app.json matches
2. Log has 0 accounts, app.json has accounts → build entirely from app.json
3. Neither → empty

Three-state banner: drop target (yellow dashed) → encrypted warning (orange) → enriched (green with ✕ Remove). Copy Customer Summary button.

### `CVSidebar({ accts, namedAccts, selected, onSelect, errs, as, appJson })`
Left panel (240px). Shows formatted balances when enriched. Fund count prefers appJson → analytics → log data.

### `CVPortfolioView({ accts, namedAccts, port, as, appJson, onSelectAcct })`
Stats bar + responsive card grid. Formatted balances, real ops counts when enriched.

### `CVAccountDetail({ acct, errs, onBack })`
Balance card, transaction history with `TX_EXPLORERS` links, token holdings, related errors. Click-to-copy via `CVCopyText` on addresses, hashes, counterparties.

### `CVInfoBar({ dev, as, port, accts, appJson })`
Right panel (210px, hidden <1024px). Shows devices from app.json settings when log device data missing.

### `CVCopyText({ text, mono })`
Click-to-copy with "Copied!" feedback.

### `ModeToggle({ mode, onChange })`
Segmented control. `role="group"`, `aria-label="View mode"`.

---

## Customer View CSS (22 classes, `cv-` prefixed)

**Layout:** `cv-layout`, `cv-sidebar`, `cv-sidebar-header`, `cv-main`, `cv-infobar`
**Sidebar:** `cv-acct-row`, `cv-active`, `cv-chain-dot`
**Portfolio:** `cv-portfolio-grid`, `cv-portfolio-card`
**Detail:** `cv-detail-card`, `cv-info-row`, `cv-section-title`, `cv-bal-card`
**Banners:** `cv-sim-banner`, `cv-drop-banner`, `cv-dragover`, `cv-enriched-banner`
**Transactions:** `cv-tx-row`, `cv-tx-type`
**Toggle:** `cv-mode-toggle`, `cv-mode-btn`

---

## Diagnostic Mode (unchanged throughout all CV work)

Overview (4 stats, activity badges, device/env cards, issues, session, quality score 9 fields, copy buttons), Timeline (search, filter, 500 cap), Accounts (filter, explorer links, xpub scanner, tokens), Errors (severity bar, categories, repeat patterns), Network (method badges, 500 cap), APDU (conditional, 9 status codes, 500 cap)

---

## Error KB: 82 patterns (39 hardware, 31 software, 12 server)
## Chains: 60 entries, 3 with convert, 4 with xpub scanner
## Quality Score: 9 fields, max 100
## Icons: 14 Ico components
## Accessibility: 10+ aria-labels, role attributes, keyboard handlers

---

## Git / Deployment
- Repo: `https://github.com/pj-casey/ledger-toolkit`
- Branch: `main`
- No build step — single HTML file
