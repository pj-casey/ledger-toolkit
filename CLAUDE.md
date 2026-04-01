# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to
diagnose customer issues from Ledger Wallet log exports.

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file onto the page
2. App parses entries (handles minified, pretty-printed, and corrupted JSON)
3. **Two modes** (toggled in top bar):

**Diagnostic Mode** (default):
- Overview, Accounts, Errors, Timeline, Network, APDU tabs
- Copy Summary / Copy Full for tickets

**Customer View**:
- Read-only simulation of customer's Ledger Wallet
- Sidebar account list with balances, portfolio overview, account detail with transaction history
- App.json enrichment: drop customer's `app.json` for exact balances, names, tx history
- Handles encrypted (Ledger Sync) app.json gracefully with warning banner
- App.json as primary source when logs have 0 accounts
- Copy Customer Summary, click-to-copy on addresses/hashes, correct tx explorer links

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (2,304 lines)
- React 18.3.1 + Babel standalone 7.26.10 (CDN), bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser

---

## Critical rules for Claude Code

### DO NOT MODIFY

**Parsing & extraction:**
`parseLogs()` (3-tier: JSON.parse → brace scanner → line-by-line), `extractDevice()`, `extractAccounts()`, `extractErrors()`, `extractSync()`, `extractApdu()`, `extractActivity()`, `logQualityScore()`, `extractAnalytics()`, `parseAppJson()`

**Error KB:** `ERR_DB` (82 patterns), `diagnose()`, `SEV`, `CAT`

**Chain registry:** `CHAINS` (60 entries), `TX_EXPLORERS` (45+ chains), `UTXO_NETS`, `getChain()`, `DC`, `DECIMALS`
Address derivation: `cardanoCredToStake()`, `stacksPubkeyToAddr()`, `tonHexToAddr()`
Formatting: `fmtBal()`

**Token resolution:** `TOKEN_CONTRACTS`, `fetchTokenChains()`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`

**Diagnostic components:** `CopyBtn`, `Tooltip`, `HighlightText`, `ErrCard`, `XpubView`, `AcctCard`

**Customer View components:** `ModeToggle`, `CVInfoBar`, `CVSidebar`, `CVPortfolioView`, `CVAccountDetail`, `CustomerView`, `CVCopyText`

**App infrastructure:** All 16 state variables, `handleFile()`, `clearLog()`, mode toggle, `ErrorBoundary`

**CSS:** All styles including 22 `cv-` prefixed classes, `:root` variables, animations

### SAFE TO ADD

- New state variables, new components, new `cv-` CSS classes
- New utility functions (don't modify existing ones)

---

## Design tokens

### CSS `:root` / JS `T` object
```
bg:#131214  panel:#1C1D1F  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

### Reusable
`bdg(color)`, `crd`, `Ico.*` (14 icons), `DN` (device names), `MF` (mono font), `fmtBal(raw, currencyId, ticker)`, `copyText(text)`, `TX_EXPLORERS[currencyId](hash)`

---

## Parser architecture (parseLogs — 3 tiers)

```
Tier 1: JSON.parse(full text)
  ↓ fails
Tier 2: Brace-depth scanner
  - Character-by-character scan tracking {} depth
  - String boundary detection with lookahead/lookbehind validation
    (rejects garbage " bytes not preceded/followed by JSON structural chars)
  - Escape handling (only inside strings)
  - Control char stripping before JSON.parse per object
  - Skips malformed objects, recovers the rest
  ↓ fails (0 objects)
Tier 3: Line-by-line fallback
  - Splits on \n, tries JSON.parse per line
  - Non-JSON lines become raw entries
```

The brace scanner handles: pretty-printed JSON, binary garbage in locale strings, raw control characters, malformed escape sequences, and corrupted individual objects while recovering all valid objects.

---

## Mode architecture

```
App()
├── Top bar: logo, ModeToggle, file info, New/Close
├── Padding wrapper
│   ├── Drop zone (when !logData)
│   └── Diagnostic Mode (when viewMode==='diagnostic')
├── Customer View — outside padding wrapper (when viewMode==='customer')
│   ├── CVSidebar (240px)
│   ├── Main: banner + portfolio or account detail + Copy Customer Summary
│   └── CVInfoBar (210px, hidden <1024px)
```

### Customer View data sources (priority order)
1. **App.json (unencrypted)**: exact balances, account names, full tx history, tokens
2. **App.json (encrypted/Ledger Sync)**: device list, token definitions, settings only
3. **Log analytics**: accountsWithFunds, device info, totals
4. **Log accounts**: from exportLogsMeta accountsIds + SyncSuccess events

When log has 0 accounts but app.json has accounts → builds account list from app.json directly.

---

## Doc maintenance protocol

After every prompt that changes the codebase: audit HTML → regenerate both docs → install before next prompt.

---

## File structure

```
ledger-toolkit.html    — The app (single file)
CLAUDE.md              — This file
TOOLKIT_SNAPSHOT.md    — Detailed codebase reference
app.json               — Ledger Wallet user data (schema reference)
*.txt                  — Sample log files
```
