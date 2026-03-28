# Ledger Diagnostic Toolkit — Codebase Snapshot
*For context-sharing across Claude instances*

## File
- **Path:** `ledger-toolkit.html` (single-file app, no build step)
- **Line count:** 1,561
- **Stack:** React 18.3.1 + Babel standalone (CDN), bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- **Opens directly in browser:** `open ledger-toolkit.html`

---

## Design Tokens

### CSS `:root`
```
--bg: #131214
--panel: #1C1D1F
--border: #3C3C3C
--text: #FFFFFF
--muted: #949494
--primary: #BBB0FF
--success: #7AC26C
--error: #F57375
--warning: #FFBD42
```

### JS `T` object (mirrors `:root`)
```js
bg:'#131214', panel:'#1C1D1F', border:'#3C3C3C', text:'#FFFFFF',
muted:'#949494', primary:'#BBB0FF', success:'#7AC26C',
error:'#F57375', warning:'#FFBD42'
```

### JS `TC` (log type badge colors)
```js
action:'#717070', analytics:'#BBB0FF', countervalues:'#06b6d4',
bridge:'#FFBD42', network:'#8A9EF5', persistence:'#7AC26C',
walletsync:'#ec4899', error:'#F57375', 'live-dmk-logger':'#BBB0FF'
```

---

## Fonts
- **Google Fonts:** Inter (400/500/600/700) + JetBrains Mono (400/500), `display=swap`
- **Body fallback:** `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
- **Mono fallback:** `'SF Mono', 'Fira Code', Consolas, ui-monospace, monospace`

---

## Brand Elements
- **Top bar logo:** Real Ledger bracket SVG (`viewBox="0 0 41 34"`), 24×20px, `fill="currentColor"` (white), exact Ledger mark path
- **Top bar text:** `"Diagnostic Toolkit"` — 13px/500/0.06em letter-spacing/uppercase/`T.muted`
- **Fixed watermark:** `"LEDGER DIAGNOSTIC"` bottom-right, 9px/500/0.2em letter-spacing, `opacity: 0.28`, pointer-events none
- **Corner bracket motif:** `.card-bracketed` CSS pseudo-elements — 10px brackets on all four corners; applied to Overview stat cards and device/environment panels; also used as 16px inline `<span>` elements on the drop zone
- **Tab bar:** Purple 2px underline (`var(--primary)`) animates in via `scaleX` on active tab, 150ms ease
- **Micro-interactions:** All buttons 150ms transition + `scale(0.98)` on active; stat values `statFadeIn` keyframe; drag icon `iconPulse` keyframe

---

## Tabs
Always shown: `Overview` · `Timeline` · `Accounts` · `Errors` · `Network`
Conditional: `APDU` (only when APDU exchanges detected in log)
Keyboard: `Ctrl/Cmd + 1–6`

---

## Chain Coverage
**60 CHAINS entries** (includes aliases: `osmo`/`osmosis`, `icp`/`internet_computer`, `assethub_polkadot`):

Aptos, Bitcoin, Ethereum, Tezos, Cosmos, Tron, NEAR, Solana, Polkadot, Polkadot AssetHub, TON, XRP, Cardano, BNB Chain, Polygon, Avalanche C-Chain, OP Mainnet, Arbitrum, Stacks, Cronos POS, Litecoin, Dogecoin, Bitcoin Cash, Stellar, Hedera, MultiversX, Celo, Osmosis, Sui, dYdX, Injective, Mantra, Persistence, Axelar, Quicksilver, Zenrock, Base, ZKsync, Linea, Scroll, Blast, HyperEVM, Flare, Filecoin, ICP, Vechain, Algorand, Fantom, Klaytn, Cronos EVM, Moonbeam, Moonriver, Mantle, Mina, Neon EVM, Lukso, Sonic, Kaspa

**Chain colors (corrected to brand standards):**
- BTC: `#f7931a` | ETH: `#627EEA` | LTC: `#345d9d` | DOGE: `#ba9f33` | BCH: `#8dc351`
- Sonic: `#4FC3F7` (was white-on-white, fixed)

---

## Error Knowledge Base
**82 ERR_DB patterns** across categories:
- Hardware/device (TransportStatusError, DisconnectedDevice, LockedDeviceError, GenuineCheckFailed, Bootloader, etc.)
- Bluetooth (BleError, BleDeviceNotFound, PairingNotFound)
- Software/sync (SyncError, AccountAlreadyExists, NotEnoughBalance, NotEnoughGas, InvalidAddress)
- Server/network (403, 429, 503, ECONNREFUSED, certificate, NetworkDown)
- ETH RPC (execution reverted, insufficient funds for gas, gas price too low)
- Firmware/app (AppNotFound, AppAlreadyInstalled, FirmwareUpdateFailed, EthAppPleaseEnableContractData)
- APDU status codes — both hex (0x6985, 0x6982, 0x6a82, 0x6700, 0x6d00, 0x6e00, 0x6f00, 0x5515) and decimal equivalents
- WalletSync/Trustchain (CloudSyncError, TrustchainNotFound, MemberCredentialsNotFound)
- User actions (UserRefusedOnDevice, UserRefusedAddress, UserRefusedFirmwareUpdate)

Each entry has: `m` (match string), `c` (category), `s` (severity: high/medium/low), `t` (title), `a` (agent action), `causes[]` (optional), `u` (support URL, optional)

---

## Feature List

### File Loading
- Drag-and-drop + click; accepts `.json`, `.txt`, `.log`
- Parses JSON arrays, line-delimited JSON, and mixed text
- Visible error banner on parse failure or 0 entries (not silent)
- "New file" and "Close" buttons in top bar once loaded

### Overview Tab
- **4-stat grid:** Entries, Accounts, Operations, Issues
- **Session activity badges:** Manager opened · Swap browsed · Swap browsed (not completed) · Device scan active
- **Device & App card:** Model, Firmware, Language, Paired devices, App version, Commit hash, LedgerSync, MEV protection
- **Environment card:** OS, Platform, User ID (with copy button), Sync duration
- **Issues preview:** Top 3 error cards with "View all →" link
- **Session banner:** Date/time range, duration in seconds, active/total accounts, top 5 chain chips
- **Log Quality Score:** 0–100 score across 8 fields (model, firmware, app ver, active account, OS, user ID, sync duration, no critical errors); labels Excellent/Good/Fair/Poor; collapsed by default
- **Copy Summary button:** Copies plaintext block — file, session, device, accounts (up to 10), errors (up to 8), APDU count, quality score, SESSION ACTIVITY section

### Timeline Tab
- Search by message/type; filter by log type (dropdown)
- Expandable rows showing full JSON data payload
- Search term highlighting
- 500-entry cap with notice; "Export filtered" downloads JSON
- Jump-to-entry from error cards (sets Timeline filter + scrolls to row)

### Accounts Tab
- Filter by chain name, ticker, or address
- Funded/empty counts; Export JSON; Copy all IDs
- No-sync warning when no SyncSuccess captured
- **Collapsed card row:** Chain name, token count badge, ops badge, balance (if in log), inline explorer link icon (no expand needed)
- **Expanded card:** Full address + copy, derivation mode, address path, operations, sync duration, **balance + spendable balance grid** (shows "not in log — see explorer" if absent), Account ID + copy, token badges with explorer links, XpubView scanner

### Errors Tab
- Severity bar: Critical / Warning / Info counts with colored dots
- Diagnostic summary sentence (auto-generated from error profile)
- Repeating Patterns section (deduplicated by title, shows repeat count)
- Category groups: Hardware/Device 🔌 · Software/Sync 🔄 · Server/Network ☁️ · Unknown ❓
- Each error card: severity badge, category, agent action text, common causes, repeat badge, expandable technical details

### Network Tab
- Handles both log formats: new (`data.method` field) and old (`"get https://..."` message prefix)
- Method badge (`#8A9EF5`), clickable URL if starts with `https://`
- 500-entry cap

### APDU Tab
- Direction badges: `→ IN` (`#8A9EF5`) / `← OUT` (success green)
- Status decoding: 9000 OK · 6985 Rejected · 6982 Locked · 6a82 App not open · 6700 Bad length · 6d00 Bad INS · 6e00 Wrong app · 6f00 Tech error · 5515 Locked
- CLA byte shown for IN frames
- "Export APDUs" copies tab-separated (direction, time, hex) to clipboard

### Address Derivation
- **Cardano:** hex credential → stake1... bech32 (standard BIP-173, HRP `stake`, header `0xe1`, last 28 bytes as stake key hash)
- **Stacks:** 33-byte compressed pubkey → SP... c32check (version 22, requires bitcoinjs-lib hash160/hash256 — checks `window.bitcoinjsLib`)
- **TON:** 64-char hex → UQ... base64url (tag `0x51`, workchain 0, CRC16-CCITT)

### XpubView / UTXO Scanner
- Chains: BTC (mempool.space + Blockchair fallback), LTC (BlockCypher, 429 retry up to 5×), DOGE (BlockCypher, 429 retry), BCH (Blockchair)
- Gap-20 derivation on receive (chain 0) and change (chain 1)
- Shows: balance, tx count, active addresses
- Per-address: RX/CH badge, clickable address link, copy button, balance/tx count
- Stop button cancels in-flight scan (ctx token pattern)
- Colors: BTC `#f7931a`, LTC `#345d9d`, DOGE `#ba9f33`, BCH `#8dc351`

### Token Resolution (D-2 — comprehensive)
- At page load, fetches `erc20.json` for 20 EVM chains from `raw.githubusercontent.com/LedgerHQ/ledger-live/develop/libs/ledgerjs/packages/cryptoassets/src/data/evm/{chainId}/erc20.json`
- Runs as `Promise.allSettled` (all chains in parallel)
- Contract address extracted by finding the `0x...` field in each tuple (format-agnostic)
- 24h localStorage cache (`ldt_tkc_v1`)
- Resolution priority: direct 0x address in slug → TOKEN_CONTRACTS cache → search URL fallback
- Per-chain token page URL builders for: ethereum, bsc, polygon, arbitrum, optimism, base, linea, scroll, blast, zksync, mantle, fantom, avalanche_c_chain, cronos, moonbeam, moonriver, celo, klaytn, neon_evm, lukso
- Tron, Stacks, Cardano handled separately
- Search fallback for: ethereum, bsc, polygon, arbitrum, optimism, base, fantom, avalanche_c_chain

### Accessibility (WCAG)
- `button:focus-visible` — `2px solid var(--primary)`, `outline-offset: 2px`
- `input/select:focus-visible` — same treatment
- Drop zone: `role="button"`, `tabIndex={0}`, `aria-label`, Enter/Space triggers picker
- All action buttons have `aria-label`: New file, Close, Export filtered, Copy Summary, Stop scan, Export APDUs, inline explorer link
- Error cards "Details" toggle: `role="button"`, `tabIndex={0}`, keyboard handler
- Timeline rows: `role="button"` on expandable entries, keyboard handler
- Tab bar: `role="tablist"`, `role="tab"`, `aria-selected`

### Contrast (WCAG AA)
- All `#677ce7` occurrences replaced with `#8A9EF5` (~6.5:1 on panel background)
- Affects: SEV.low badge, APDU IN badge, xpub RX badge, network method badge, errors severity bar info dot

### Error Boundary
- React `ErrorBoundary` class wraps entire app
- Shows: ⚠️ icon, "Something went wrong", "Reload" button

---

## Drop Zone (before file load)
Full-width `T.panel` card, `borderRadius: 4`, `padding: 80`, centered:
1. Four 16px corner bracket `<span>` elements (Ledger brand motif) — `T.border` at rest → `T.primary` on drag
2. Ledger bracket SVG watermark centered, 96×80px, `opacity: 0.04`
3. 40px upload arrow icon — pulses (`iconPulse` keyframe) on drag hover
4. H1: `"Drop a Ledger Wallet log file here"` — 18px/600
5. Subtext: `"Accepts .json and .txt — export from Settings → Help → Export Logs in Ledger Live"` — 14px/muted
6. Five capability chips: `Device info · Accounts · Error diagnosis · On-chain links · Network`

On drag: border + icon → `T.primary`, background → `#BBB0FF08`
On bad file: red error banner above drop zone with alert icon

---

## Device Extraction Sources (priority order)
1. `e.data` fields: `modelId`, `deviceVersion`, `firmwareVersion`, `firmware`, `seVersion`, `deviceLanguage`, `modelIdList`, `appVersion`, `osType`, `osVersion`, `platform`, `totalAccounts`, `totalOperations`, `accountsWithFunds`, `tokenWithFunds`, `ledgerSyncActivated`, `MEVProtectionActivated`
2. `exportLogsMeta` entry: `release`, `git_commit`, `userId`, `anonymousId`, `userAnonymousId`, `accountsIds`, env block fields
3. Manager API URL query params (fallback): `device_type` → `modelId`, `firmware_version_name` → `fw`

## Model name map (`DN`)
```js
nanoS:'Nano S', nanoSP:'Nano S Plus', nanoX:'Nano X',
stax:'Stax', europa:'Flex', apex:'Apex'
```

---

## Error Extraction Sources
Captures entry if any of:
- `e.error` present (uses `e.error.message`)
- `payload.status` includes `'ERROR'`
- `e.level === 'error'` or `e.type === 'error'` (catches entries without keyword match)
- `e.message` contains `rejected`/`failed`/`error` (excluding `analytics` type)
- `e.data.error` is empty object `{}` and message includes `'failed'`

## Activity Detection
- **Manager opened:** message includes `manager.api.live.ledger.com/api`
- **Swap browsed:** message includes `swap.ledger.com`
- **Swap executed:** additionally starts with `post ` or includes `/v5/swap` or `/v5/exchange`
- **Device scan:** `type === 'live-dmk-logger'` and message includes `available` or `transport`

---

## Git / Deployment
- Repo: `https://github.com/pj-casey/ledger-toolkit`
- Branch: `main`
- Latest commit: `761e985` — "Post-QA polish: all DEGRADE items resolved, comprehensive token resolution"
- No build step — single HTML file, open directly in any browser
