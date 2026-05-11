# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build step. Open `ledger-toolkit.html` directly in a browser:

```
open ledger-toolkit.html
```

To inspect: use browser DevTools. There is no test suite, no package manager, no bundler. All changes are in the single HTML file.

**Git push:** The pre-commit hook blocks sandbox pushes. Peter runs `! git push origin main` directly from the prompt to push to remote.

## Architecture

`ledger-toolkit.html` (~8,762 lines) is the entire application — a single-file React 18.3.1 + Babel standalone app (no build step). It opens directly in any browser.

**Two modes:**
- **Diagnostic Mode** — fixed viewport (`height:100vh`, no page scrolling), sidebar navigation, sections: `overview`, `errors`, `accounts`, `timeline`, `network`, `apdu`, `raw`
- **Customer View** — scrollable left-nav layout, sections: `portfolio`, `accounts`, `earn`, `myLedger`, `agentInsights`

The file is organized top-to-bottom: CSS → data constants → parse/diagnostic functions → React components → `ReactDOM.render`.

## App Component

The `App` function (line ~6048) is ~2,678 lines with ~53 useState calls. Before adding new state, grep for existing related state and reuse or colocate. Do not add new useState without checking first.

## Data Layer

**Functions — DO NOT MODIFY** (logic is load-bearing, parsers and diagnostics depend on exact behavior):

`parseLogs`, `parseAppJson`, `synthesizeMobileMeta`, `extractErrors`, `extractSync`, `extractApdu`, `extractActivity`, `extractAnalytics`, `inferRequiredApps`, `diagnose`, `ERR_DB` (82 patterns), `fetchEvmBalance`, `fetchPrices`, and all version check fetch/compare logic.

**Registries — extension points** (additive only — never modify or remove existing entry semantics; add new chain/token entries by following the established pattern):

`CHAINS`, `UTXO_NETS`, `BALANCE_APIS`, `getChain`, `DECIMALS`, `TOKEN_CONTRACTS`, `TOKEN_METADATA`, `TOKEN_URLS`, `TOKEN_SEARCH`, `TOKEN_DECIMALS_PATTERNS`, `EVM_CHAIN_IDS`, `DEXSCREENER_CHAIN_IDS`, `CURRENCY_TO_APP`, `COINGECKO_IDS`, `EVM_RPCS`, `TX_EXPLORERS`, `DC`.

**Token enrichment functions — extension points** (additive only — fetchers can grow new vendors/chains, but signatures and cache shapes are load-bearing for the lazy-fetch wiring):

`fetchTokenMetadata` (CAL service, per-token, 24h localStorage cache), `fetchTokenBalances` (Multicall3 aggregate3 — one eth_call per chain for N tokens), `fetchTokenFiat` (DexScreener per-contract, concurrency 5, 200ms gap, 5min in-memory cache), `getTokenInfo`, `getTokenDecimals`.

`MULTICALL3_ADDRESS` (`0xcA11bde05977b3631167028862bE2a173976CA11`) is hardcoded and assumed deployed at the standard address on every chain in `EVM_RPCS`. If a future chain doesn't have Multicall3 at this address, gate it via a per-chain capability flag and fall back to N parallel `eth_call balanceOf` requests; do not change the constant.

**May extend (additive only):** `extractDevice`, `extractAccounts`, `extractDeviceApps` — for new log format support, without breaking existing parsers.

## Key Data Shapes

```
logData.accts          — accounts array
logData.errs           — errors array
logData.entries        — all log entries
logData.dev            — device info { appVer, appBrand, fw, modelId, targetId }
logData.quality        — quality score object

appJson.accounts       — accounts from parsed app.json
appJson.encrypted      — boolean

Error objects:  .dg.s = severity ('high'/'medium'/'low'), .dg.t = title, .dg.a = action, .dg.u = support URL
Account objects: .name, .currency, .ch (chain), .addr, .ops, .bal, .funded
Enriched accounts (CustomerView): ajBalance, ajSpendable, ajOps, ajOperations, ajSubAccounts,
  ajBlockHeight, ajStarred, ajSwapHistory, ajPendingOps, ajFreshAddressPath, enriched:true
```

**CVAgentInsights finding kinds:** `kind:'account'` (balance mismatch), `kind:'system'` (firmware/apps/drift), `kind:'drain'` (seed compromise — unshifted to top of list)

**Three version concepts (never conflate):**
| Concept | Source field | UI label |
|---|---|---|
| Desktop app | `info.appVer`, `info.appBrand` | `llLabel(dev)` → "Ledger Wallet" or "Ledger Live" |
| Firmware (SE OS) | `info.fw` | "Firmware" |
| Device coin apps | `extractDeviceApps()` | "Device Apps" |

## Design Tokens

```js
T.bg:'#1A1A1D'    T.panel:'#1A1A1D'    T.card:'#242528'
T.border:'rgba(255,255,255,0.06)'       T.text:'#FFFFFF'
T.muted:'#949494'  T.primary:'#BBB0FF'  T.success:'#7AC26C'
T.error:'#E40046'  T.warning:'#FFBD42'  T.orange:'#FF5300'
```

Two-tier elevation: bg (`#1A1A1D`) → card (`#242528`). No third tier.

**Border-radius tiers (strict):**
- `8px` — cards, panels, containers, overlays
- `6px` — buttons, inputs, filter chips
- `4px` — small badges, pills, tags

## Typography

- **Body:** `'Inter','Darker Grotesque',-apple-system,...` — Inter is primary
- **Mono (`MF` constant):** `'JetBrains Mono','SF Mono','Fira Code',Consolas,ui-monospace,monospace` — data values only: addresses, hashes, balances, timestamps, hex, derivation paths
- **`.stat-value`:** Darker Grotesque (display font for large numbers)
- **`.guide-embed`:** Darker Grotesque (documentation overlay)
- Labels/headings/nav: Inter, sentence case, no `textTransform`, no `letterSpacing` (except MOBILE badge)
- **`purposeLabel`** style: `{fontSize:11, color:'#666666', fontWeight:400, flexShrink:0}` — quiet hints

## Key Helpers

| Helper | Purpose |
|---|---|
| `MF` | Monospace font stack constant |
| `T` | Theme colors object |
| `TC` | Type badge colors (`action`, `analytics`, `network`, `error`, etc.) |
| `I` | Interaction timing (`I.fast=150ms`, `I.medium=250ms`, `I.slow=350ms`) |
| `DN` | Device name map (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `DIAG_WF` | 5 workflow categories for Diagnostic Priority Map |
| `classifyDiag(dg)` | Maps ERR_DB entry → workflow category |
| `llLabel(dev, isMobile)` | Returns "Ledger Wallet" or "Ledger Live" |
| `llText(text, dev)` | Runtime string replacement for app branding |
| `chainIconUrl(id)` | CDN URL for chain icon |
| `jumpTo(li)` | Navigate to Timeline + scroll to entry |
| `goToAcct(addr)` | Navigate to Accounts with filter |
| `sevColor(s)` | Severity → color |
| `cvFiatValue(appJson, cid, rawBalance)` | Raw balance → fiat |
| `cvFmtFiat(value, appJson)` | Formats fiat value with currency symbol |

## Sidebar Nav Pattern

`SectionHeader` component: pill shape (`borderRadius:8`, `margin:2px 8px`), flat active (`rgba(255,255,255,0.06)`), muted inactive (`#8A8A8E`), Inter 14px/500. No subtitle, no left border, no gradient. Hover: `rgba(255,255,255,0.04)`. Customer View sidebar matches this pattern exactly.

## Guides Drift Warning

`GUIDE_AGENT` (line ~263) and `GUIDE_TECHNICAL` (line ~749) inside ledger-toolkit.html must stay byte-identical to the body content of `agent-guide.html` and `technical-reference.html`. No automation enforces this. If you edit one, edit both. If unsure, ask before touching either.

## Making Changes

**Small targeted edits (1–3 changes, same logical section):** Use `Read` + `grep` + `Edit` directly. Faster than spawning an agent.

**Large architectural changes (new sections, layout restructuring):** Use a 3-agent sequential team:
1. Scaffold agent — structural skeleton and layout wiring
2. Implementation agent — fills the main component (blocked by scaffold)
3. Enhancements agent — empty states, keyboard shortcuts, polish (blocked by implementation)

Sequential agents avoid merge conflicts in a single-file codebase. Parallel agents only work if they have clearly non-overlapping line ranges.

Always give agents: design tokens, the DO NOT MODIFY list, existing helper names, and exact field names when known.
