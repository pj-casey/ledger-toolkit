# ERC-20 Token Support: Live Brief

This is the durable record of how the toolkit's ERC-20 token enrichment actually works as of 2026-05-09. It supersedes any earlier draft brief that referenced the old static `erc20.json` files or CoinGecko's `simple/token_price` batch endpoint — both of which are gone or hostile to free-tier use.

The lesson of this work: **vendor docs lie. Probe before you implement.**

## What ships

When the user expands an EVM account card, in order:

1. **CAL metadata fetch** (`crypto-assets-service.api.ledger.com/v1/tokens?id=…`) — one request per missing tokenId, in parallel via `Promise.allSettled`. Populates `TOKEN_METADATA[tokenId] = {contractAddress, ticker, name, decimals, delisted}` and the legacy `TOKEN_CONTRACTS` map. Cached in localStorage under key `ldt_tkm_v1` with 24h per-token TTL.

2. **Multicall3 balance fetch** (Multicall3 at `0xcA11bde05977b3631167028862bE2a173976CA11` via `EVM_RPCS[chain]`) — one `eth_call` returns N `balanceOf(owner)` reads in a single round trip via `aggregate3`. Encoder uses `bytes offset = 0x60` (Call3 has 3 fields); decoder uses `bytes offset = 0x40` (Result has 2 fields).

3. **DexScreener fiat fetch** (`api.dexscreener.com/latest/dex/tokens/{contract}`) — one request per token, fanned out with a session-wide concurrency cap of 5 and a 200ms minimum gap between request starts. Pair selection: filter to pairs where `chainId === DEXSCREENER_CHAIN_IDS[ledgerChain]` AND `baseToken.address === queriedContract` (lowercased), sort by `liquidity.usd` desc, take the top pair's `priceUsd`. Cache is in-memory only, keyed by `chain:lc0xContract`, with 5-minute TTL for hits and 1-minute TTL for nulls.

The token tile in `AcctCard` then renders ticker (from CAL), live balance, live USD value, and an error dot if any stage failed. The `cv-drift-tokens` finding in `CVAgentInsights` compares each token's cached `cvLatestRate` against its DexScreener live price (only when `liquidity.usd >= DEXSCREENER_DRIFT_LIQUIDITY_FLOOR`) and surfaces drifts above 2% as amber, above 10% as red.

## Why each vendor was chosen

### CAL service for metadata

- **Old design (rejected):** static `https://raw.githubusercontent.com/LedgerHQ/ledger-live/develop/libs/ledgerjs/packages/cryptoassets/src/data/evm/{chainId}/erc20.json` per-chain bulk fetches, parsed as fixed-position arrays.
- **Why rejected:** Ledger removed those files in early 2026 (LIVE-21487 "Remove CAL Initiative"). The path returns 404 on `develop`, `main`, and `master`. The toolkit's prior `fetchTokenChains` was silently broken in production.
- **New design:** per-token JSON object lookups via the CAL service. No batch lookup is supported (`id__in` returns 400, repeated `id=` returns 400). Smoke tested: 30 parallel `?id=…` requests complete in 783ms with 0 × 429.

### Multicall3 for balances

- One round trip per chain regardless of token count. Available at the standard deterministic address on all 22 chains in `EVM_RPCS`.
- Encoding spec is in the inline comment at `fetchTokenBalances`. The trick is the response Result tuple has `(success, bytes)` not `(target, allowFailure, bytes)`, so the inner `bytesOffset` is `0x40` not `0x60` — gotcha for anyone writing a from-scratch decoder.

### DexScreener for fiat (after CoinGecko was eliminated)

- **CoinGecko free tier:** rejected after empirical probing showed (a) `simple/token_price` hard-limits to 1 contract per request, (b) burst limit triggers 429 at the ~5th request and locks out for tens of seconds, (c) recovery time is unpredictable. A faithful implementation of "concurrency 5 + 200ms gap" took 61.7s wall-clock for 8 tokens because of cascading 30s back-offs.
- **DexScreener:** verified by the same probe pattern. 30 sequential requests with 200ms gap completed in 7.4s with 0 × 429. Per-contract reads are the right shape; comma-separated batching syntactically works but is functionally broken (the response is capped at 30 pairs total, so distinct tokens get truncated).

## Coverage gaps (ship as-is)

DexScreener does not index these chains (probed by querying their wrapped-native contract):

- **Lukso** — 0 pairs returned
- **Moonbeam** — 0 pairs returned
- **Klaytn / Kaia** — DS does not appear to index Kaia under any chainId tested

Tokens on these chains render with "no live price" in the tile. The chain-id map (`DEXSCREENER_CHAIN_IDS` in the source) intentionally omits them so the fetcher never fires a doomed request.

DexScreener also has a per-token edge case for forked-popular tokens. The v0 endpoint returns up to 30 pairs across all chains; for highly forked tokens (Ethereum USDT, whose contract address is shared by PulseChain and EthereumPoW forks), the 30-pair cap can fill with fork pairs and exclude the actual mainnet pairs we wanted. Those specific tokens render fiat=null even though the price is recoverable elsewhere. Acceptable; documented.

## Liquidity floors (named constants, knobs to tune)

Pulled out as named constants at the top of the DexScreener block:

```js
const DEXSCREENER_DISPLAY_LIQUIDITY_FLOOR = 1000;     // USD; pairs below this are not shown at all
const DEXSCREENER_DRIFT_LIQUIDITY_FLOOR   = 100000;   // USD; pairs below this don't qualify for cv-drift-tokens
```

If false positives or false negatives surface in real customer logs, those are the knobs. Don't bury new logic; tune the constants.

## Verification harness

Each of these probe scripts lives in `/private/tmp/claude-502/` during development and was run to ground out an empirical answer before code was written. Re-run them whenever you re-touch this code — vendor reality drifts.

### A. CAL rate-limit smoke test

```js
// 30 parallel /v1/tokens?id=… requests with real Ledger token IDs.
// Pass criteria: 0 × 429, all OK in <3s wall, response shape per CAL.
const ids = [/* real tokenIds from a customer log */];
const t0 = Date.now();
const r = await Promise.allSettled(
  ids.map(id => fetch(
    `https://crypto-assets-service.api.ledger.com/v1/tokens?id=${encodeURIComponent(id)}&limit=1&output=id,name,ticker,contract_address,decimals,delisted`
  ).then(r => r.json()))
);
console.log(`wall: ${Date.now()-t0}ms, ok: ${r.filter(x=>x.status==='fulfilled').length}/${ids.length}`);
```

Last result: 30 IDs in 783ms, 0 × 429. Slug exactness matters — guessed slugs return `[]`. In production we use the slug straight from `acct.ajSubAccounts[i].tokenId` which always originated from Ledger Live and is correct.

### B. CoinGecko rate-limit probe (preserved as evidence)

If anyone proposes "use CoinGecko for token prices" again, run this and ask them why they want to spend 60 seconds fetching 8 prices:

```js
// 5 sequential requests with no gap. Fifth typically returns 429.
const tokens = [/* USDC, USDT, DAI, WBTC, LINK */];
for (const c of tokens) {
  const t0 = Date.now();
  const r = await fetch(`https://api.coingecko.com/api/v3/simple/token_price/ethereum?contract_addresses=${c}&vs_currencies=usd`);
  console.log(`${c.slice(0,8)} ${r.status} ${Date.now()-t0}ms`);
}
```

Last result: tokens 1-4 returned 200; token 5 returned 429. Subsequent gentle retries also returned 429 for tens of seconds. Burst detection is aggressive even for sequential single-contract reads.

### C. DexScreener empirical protocol

The decisive probe — run all six tests when re-evaluating the fiat path.

1. **Single contract success:** fetch USDC on Ethereum, document response shape (returns `{schemaVersion, pairs[]}` with up to 30 pairs).
2. **Burst test:** 5 parallel requests for 5 different contracts. Last result: 5/5 OK, 268ms wall, 0 × 429.
3. **Sustained test:** 30 sequential requests with 200ms gap. Last result: 30/30 OK, 7.4s wall, 0 × 429.
4. **Batching:** try `/{c1},{c2},…`. Last result: works syntactically but the response caps at 30 pairs total — distinct tokens get truncated. Per-contract is the right shape.
5. **Coverage holes:** probe a wrapped-native token on Lukso, Moonbeam, Kaia. Last result: 0 pairs each. Skip those chains in `DEXSCREENER_CHAIN_IDS`.
6. **Bogus contract:** verify graceful empty response. Returns `{schemaVersion, pairs: []}` status 200.

### Pair-selection sanity

The `chainId AND baseToken match` filter is critical. Without it:

- USDT on Polygon returns 30 pairs in the response, all `polygon` chainId, but every single one has USDT as `quoteToken` (e.g. `BET/USDT`). `pairs[0].priceUsd` = $0.0004 (the price of BET, not USDT). Filtering to `baseToken === USDT` drops all of them and we correctly fall through to fiat=null.
- USDT on Ethereum returns 30 pairs but they're all PulseChain or EthereumPoW (forks reusing the contract address). `chainId === 'ethereum'` filter drops all of them, fiat=null.

For most real tokens (USDC, DAI, LINK, UNI, ARB, WBTC, COMP, …), the filter passes through to the highest-liquidity Uniswap/Curve/etc pair on the queried chain and `priceUsd` is correct.

## Final smoke test (manual, in-browser)

This replaces the CoinGecko-shaped smoke test from the original draft brief.

Open a real log with an EVM account holding 5+ tokens on at least 2 chains. On account expand, verify:

- CAL metadata lookups complete in under 3s (one parallel batch per account)
- Multicall3 balance fetch completes in under 1s
- DexScreener fiat fetch completes in under 3s for ~5 tokens (fewer with cache hits)
- Network tab shows no 429s and no 4xx errors
- Re-expanding the same account within 5 minutes shows fiat values immediately (cache hit, zero new requests)
- Tokens on Lukso/Moonbeam/Kaia render with "no live price" — not an error
- The `cv-drift-tokens` finding in CV mode appears when cached vs live prices diverge by >2% AND the pair has ≥$100k liquidity

If any of these fail, run probes A/B/C against the failing vendor before changing code. The fix is almost certainly empirical, not theoretical.

## Token discovery (Layer 1 + 2) — added 2026-05-09

The original brief assumed `acct.toks` would always be populated by `extractAccounts` parsing `SyncSuccess` analytics events. **That assumption was wrong for roughly half of real logs.** Mobile logs, no-sync diagnostic captures, and partial captures often lack `SyncSuccess` entirely — the toolkit still parsed the customer's accounts but rendered zero tokens, so the live-balance / fiat / drift work never fired. Two new layers fill the gap.

### Layer 1: parse token signals from the log

`discoverTokensFromLog(accts, entries)` runs immediately after `extractAccounts` in the parse pipeline. It scans raw entries for token-id mentions from sources B and D below and merges them additively into each EVM account's `toks` array. `SyncSuccess` (Source A) wins on dedupe — its entries carry richer fields (operationsLength, votesCount, parentDerivationMode).

| Source | Where | Pattern | Notes |
|---|---|---|---|
| **A** (existing) | `extractAccounts` | `SyncSuccess` analytics events with `tokens[]` array | Authoritative when present. Carries op counts and parent metadata. |
| **B** | message strings + stringified payload/data bodies | `/([a-z_][a-z0-9_]+)\/erc20\/([a-zA-Z0-9_\-]+(?:_0x[a-fA-F0-9]{40})?)/g` | Catches countervalues messages (`"USD ethereum/erc20/usd__coin@daily…"`), embedded slug arrays in payloads, etc. Newer slugs include a `_0x…` suffix. |
| **C** (informational, no parser) | network entries with `proxyetherscan` URLs | `proxyetherscan.api.live.ledger.com/v2/api/{chainId}?…` | The URL alone identifies a (chain, address) pair queried by Live but doesn't expose token IDs. Operationally handled by Layer 2 — flagging accounts here would be redundant. |
| **D** | Redux RTK-Query actions | `endpointName === 'findTokenById'` with `originalArgs.id === '{chain}/erc20/{slug}'` | Common in verbose desktop logs. Confirmed the entry shape on real logs (`type:'action'`, `payload.action.payload`). |

Each account is tagged with `tokSource: 'sync' | 'mixed' | 'discovered' | 'none'` so the UI can show a "from log" / "partial" label when the data isn't sync ground truth.

**Verified against real logs** (token counts after Layer 1 — these match the brief's predictions ±15%):

| Log | Layer 1 tokens | Comment |
|---|---|---|
| `ledgerwallet-logs-2026.04.02-…-db07b272` | 6 ETH (USDC, USDT, MATIC, DAI, USDS, …) | SyncSuccess-rich |
| `chuck/ledgerwallet-logs-2026.04.03-…-8db16532` | 10 across 3 chains (base/erc20/cgeth, ethereum/erc20/aave_*, polygon/erc20/pigcoin, …) | countervalues-only |
| `chuck/emulated/ledgerwallet-logs-2026.04.03-…-db07b272` | 60 across 5 chains | findTokenById + countervalues, large scope |
| `ledgerwallet-mob-3.111.0-(2)-2026-04-03-logs` | **0 tokens** | mobile, no token signals — Layer 2 case |

### Layer 2: discover from chain via Ledger's proxyetherscan

When `AcctCard` opens an EVM account where `acct.toks.length === 0`, fire one call per account:

```
GET https://proxyetherscan.api.live.ledger.com/v2/api/{chainId}?module=account&action=addresstokenbalance&address={addr}&tag=latest&page=1&offset=100
```

No auth — Ledger's proxy adds the API key server-side. Same backend Ledger Live itself uses. Empirically verified 2026-05-09:

- 200 OK in ~760ms for the reference address (`0x0FED…D252` on Arbitrum, chain 42161)
- 10 parallel requests across 10 (chain, address) tuples: 1159ms wall, 0 × 429
- Some chains return `status: '0'` for unsupported/empty cases — handle as empty list, not error
- Response body (verified shape, **case-sensitive field names**):

```json
{
  "status": "1",
  "message": "OK",
  "result": [
    {
      "TokenAddress": "0xa0b8…",
      "TokenName": "USD Coin",
      "TokenSymbol": "USDC",
      "TokenQuantity": "10",
      "TokenDivisor": "6",
      "TokenPriceUSD": "0.9998…"
    }
  ]
}
```

**Spam filter** (`_isLikelySpamToken` in source). Ledger's proxy does **not** filter scam/airdrop bait. Out of 12 tokens on the reference address, 8 were spam — symbols like `"Claim $stlink rewards at https://stlink.fi"` or `"merge-eth.io"`. Filter rules (in order):

1. URL anywhere in symbol or name (`https?://`)
2. Domain-style symbol (`/\.[a-z]{2,}/i` — catches `merge-eth.io`, `centreusd.org`)
3. Whitespace in symbol, or social-bait keywords (`claim`, `airdrop`, `visit`, `reward`, `gift`, `swap`, `t.me`)
4. Zero-width or formatting chars in symbol or name
5. Symbol length > 14 chars
6. Excessive whitespace padding in name (`\s{4,}`)

Spam that slips the filter still won't show fiat — its DEX liquidity is below `DEXSCREENER_DISPLAY_LIQUIDITY_FLOOR` ($1k).

Discovery results are cached per `(chain, address)` for 5 minutes in `_PROXY_CACHE`. Re-expanding the same account inside that window does not re-probe.

**For each kept token**, the AcctCard effect synthesizes a Ledger-style tokenId of the form `${chain}/erc20/${contractAddress}`, populates `TOKEN_METADATA[tid]` directly from the etherscan-derived metadata (so CAL is skipped — we already have ticker/name/decimals), and pushes the synthesized entry onto `acct.toks` with `_discovered: true`. The existing Multicall3 + DexScreener pipeline then fires unchanged.

### UI indicator

`AcctCard` renders a small chip next to the "Tokens" header when `tokSource === 'discovered'` (label: "from log") or `tokSource === 'mixed'` (label: "partial"). `tokSource === 'sync'` shows nothing — sync is the canonical source. Hover tooltip explains the provenance.

### Verification checklist results

1. **Proxyetherscan shape probe** — done; documented above. Field names are PascalCase (`TokenAddress`, not `token_address`); `TokenDivisor` is a string (parse with `parseInt`).
2. **Rate-limit probe** — done; 0 × 429 on 10× concurrent. No throttle needed in the toolkit code.
3. **Three log files** — Layer 1 verified against Apr 02, Apr 03 servlet, Apr 03 emulated; counts match brief expectations.
4. **Layer 2 fallback** — verified against the mobile log's two EVM addresses. First account returned 4 raw tokens, 2 after spam filter (HQG, USDC). Second account returned `status: '0'` (no holdings) — handled as empty list. The filter correctly drops `merge-eth.io` and `centreusd.org` while keeping legitimate tokens regardless of value.
5. **Mobile log behavior** — Layer 1 produces 0 tokens (confirmed); Layer 2 fires on EVM account expand and produces tiles via the discovered → CAL-shortcircuit → Multicall3 → DexScreener pipeline.

## Iteration log — 2026-05-11

Three layered display improvements, each independently verified against the
Apr 02 log (`ledgerwallet-logs-2026.04.02-19.15.48-db07b272.txt`, 6-token
Ethereum account: USDC, USDT, POL, DAI, USDS, USDe).

### Iteration A — clean labels from CAL

Before this pass, tile labels rendered the raw sluggified tokenId,
including the trailing `_0x<40-hex>` contract suffix on newer slugs
(e.g. `"USDS STABLECOIN 0XDC035D45D973E3EC169D2276DDAB16F1E407384F"`).

Changes:
- New helpers `_tokenSlugTail(tokenId)` and `_titleCaseSlug(s)` strip the
  trailing `_0x[a-fA-F0-9]{40}$` contract suffix and produce a clean
  title-cased fallback (`"USDS Stablecoin"` instead of the all-caps slug).
- `getTokenInfo`'s slug fallback now returns
  `{ticker: tail.toUpperCase(), name: titleCase(tail)}` so the **name** field
  exists pre-CAL — not just a single label.
- AcctCard tile renders **two lines**: ticker (CAL `ticker` > sync
  `tokenTicker` > slug first-word) and an optional name subtitle (CAL
  `name` > title-cased slug). The name line is suppressed when it would
  duplicate the ticker (e.g. `USDe` / `USDe`).
- Contract addresses never appear in the visible label. They live only in
  the expand panel and the explorer link.

**Side effect uncovered during verification:** the page CSP did not
whitelist `crypto-assets-service.api.ledger.com` (CAL),
`api.dexscreener.com`, `proxyetherscan.api.live.ledger.com`, or
`crypto-icons.ledger.com`. Every fetch silently failed in production with
"Refused to connect ... violates the document's Content Security Policy",
so the entire CAL/fiat/icon pipeline was a no-op in real browser use even
though Node-side probes (in `/tmp/...`) passed. CSP `connect-src` and
`img-src` updated to include all four hosts. **Without this fix, no other
ERC-20 work would have actually rendered.**

### Iteration B — real token icons via Ledger's CDN

Source: `https://crypto-icons.ledger.com/index.json` — a flat
`{tokenId: {icon: "FILENAME.png"}}` mapping (~2,800 keys, ~3MB raw).

Changes:
- New `fetchIconIndex()` — single per-session fetch, 24h localStorage
  cache under key `ldt_icons_v1`. Cached form is compressed to a flat
  `{tokenId → filename}` map (the `{icon: …}` wrapper is dropped). Bump
  the cache-key suffix if the cached shape changes again.
- New `getTokenIconUrl(tokenId)` — synchronous lookup. Tries the exact
  tokenId, then a normalized variant where `__` collapses to `_` (handles
  the legacy `usd__coin` ↔ modern `usd_coin` divergence observed in
  SyncSuccess vs the CDN index). Returns null on miss; caller renders a
  letter-circle fallback.
- Tile renders a 24×24 `<img>` with an `onError` swap to a chain-color
  letter circle (uppercase first char of the ticker). Same pattern as the
  existing chain-icon fallback in the account-row header.
- Icon fetch is kicked off in parallel with the CAL fetch in
  `AcctCard`'s expand effect; a second `setTokV` after the icon promise
  resolves so tiles re-render with icons even if CAL was already cached.

**Apr 02 verification:** all 6 Ethereum tokens (USDC, USDT, POL, DAI,
USDS, USDe) resolved to real Ledger CDN icons. No letter-circle fallback
fired on this log.

### Iteration C — compact row layout

The width-variable pill grid wrapped unpredictably and made it hard for a
CS rep to scan 10+ tokens. Replaced with a single-row-per-token list
inside a bordered card.

Row layout:
```
[icon] TICKER  N ops                 BALANCE   [status]  ›
       name (subtitle)               fiat
```

Changes:
- Replaced the `flex flexWrap` pill grid in `AcctCard` with a vertical
  rows container. Each row is a `grid auto 1fr auto auto` (icon, label
  column, right-aligned balance/fiat, status + chevron).
- Sort order: tokens with `operationsLength > 0 OR live balance > 0`
  render first. Discovered-zero-activity tokens are grouped below a
  small divider labeled `DISCOVERED (NO ACTIVITY IN LOG)`. Order within
  each group is preserved from `acct.toks` (already sorted by Layer 1's
  source-priority).
- Status indicators on the far right: a red dot for live-balance fetch
  failures, a small `NO PRICE` pill when balance came back but
  DexScreener returned no qualifying pair.
- Click-to-expand: each row toggles `expandedToks[tid]`. Expand panel
  shows the contract address (with copy), the combined
  `${acct.id}+${tid}` account-ID (with copy), and an "View on explorer"
  link routed through the existing `TOKEN_URLS` / `TOKEN_SEARCH`
  resolvers.
- The existing data model (`acct.toks`, `ajSubAccounts`,
  `liveTokenBalances`, `liveTok.status === 'error'`) is untouched — only
  the render code changes. The `cv-drift-tokens` finding in CV mode is
  unaffected.

**Apr 02 verification:** USDC (3 ops) and USDT (2 ops) render in the
active group at top; POL, DAI, USDS, USDe in the discovered group below
the divider. No hex anywhere; real icons; click expands to show
`0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` and
`js:2:ethereum:0x8ceD…d9FeE:+ethereum/erc20/usd__coin` as the row detail.
