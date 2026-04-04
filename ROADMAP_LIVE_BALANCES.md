# ROADMAP: Live On-Chain Balance Fetching

## Summary

Automatically fetch live blockchain balances for every account found in log files, displayed in Diagnostic Accounts. No app.json required. When app.json IS loaded, enable comparison between customer's local state (Customer View) and blockchain truth (Diagnostic Accounts).

---

## What already exists in the tool

- **CHAINS map** (~60 chains) with ticker, color, explorer URL, and decimal places (DECIMALS const)
- **UTXO_NETS** already fetches live balances for BTC, LTC, DOGE, BCH via mempool.space, blockcypher, blockchair
- **Address converters** for Cardano (hex → stake bech32), Stacks (pubkey → SP address), TON (hex → user-friendly)
- **Account data from logs**: each account has `id`, `cid` (chain ID), `addr` (address), `ch` (chain info)
- **AcctCard component** already renders account cards in Diagnostic Accounts — this is where balances would display
- **app.json enrichment** already adds `ajBalance` to accounts when loaded — comparison infrastructure is halfway there

---

## Architecture

### Data flow
```
Log loaded → accounts extracted → for each account:
  1. Resolve display address (handle xpubs, converters)
  2. Look up chain in API registry
  3. Fire async fetch (with rate limiting)
  4. Store result in state: { balance, txCount, timestamp, status }
  5. Render in AcctCard
```

### State shape
```js
const [liveBalances, setLiveBalances] = useState({});
// Key: account ID
// Value: { bal: number|null, txs: number|null, status: 'loading'|'ok'|'error'|'unsupported', err: string|null, ts: number }
```

### Trigger
- Auto-fetch on log load (with a "Fetch balances" button to re-trigger)
- Stagger requests to avoid rate limits (100-300ms between calls)
- Show loading state per account card while fetching

---

## API Registry — Chain by chain

### Tier 1: Already implemented (UTXO_NETS)
| Chain | API | Rate limit | Notes |
|---|---|---|---|
| Bitcoin | mempool.space → blockchair fallback | Generous / 30req/min | Already working |
| Litecoin | blockcypher | 200req/hr (no key) | Already working, has retry logic |
| Dogecoin | blockcypher | 200req/hr (no key) | Already working |
| Bitcoin Cash | blockchair | 30req/min | Already working |

### Tier 2: EVM chains (one pattern, many chains)
All EVM chains support the same RPC call: `eth_getBalance`. Use public RPCs or block explorers.

| Chain | Free API option | Endpoint pattern |
|---|---|---|
| Ethereum | llamarpc / cloudflare-eth / publicnode | `POST` JSON-RPC `eth_getBalance` |
| BNB Chain | publicnode | Same pattern |
| Polygon | publicnode | Same pattern |
| Arbitrum | publicnode | Same pattern |
| Optimism | publicnode | Same pattern |
| Base | publicnode | Same pattern |
| Avalanche C | publicnode | Same pattern |
| Linea | publicnode | Same pattern |
| Scroll | publicnode | Same pattern |
| Blast | publicnode | Same pattern |
| ZKsync | publicnode | Same pattern |
| Fantom | publicnode | Same pattern |
| Cronos | publicnode | Same pattern |
| Moonbeam | publicnode | Same pattern |
| Moonriver | publicnode | Same pattern |
| Celo | publicnode | Same pattern |
| Mantle | publicnode | Same pattern |
| Flare | publicnode | Same pattern |
| Sonic | publicnode | Same pattern |
| Neon EVM | publicnode | Same pattern |
| Lukso | publicnode | Same pattern |
| HyperEVM | publicnode | Same pattern |

**Implementation:** One `fetchEvmBalance(rpcUrl, address)` function that does:
```js
fetch(rpcUrl, {
  method: 'POST',
  headers: {'Content-Type':'application/json'},
  body: JSON.stringify({jsonrpc:'2.0',method:'eth_getBalance',params:[address,'latest'],id:1})
})
```
Then parse hex result and divide by 10^decimals.

Public RPC list: https://publicnode.com (free, no key) or https://chainlist.org for fallbacks.

### Tier 3: Chain-specific APIs (each unique)

| Chain | API | Endpoint | Decimals | Notes |
|---|---|---|---|---|
| Solana | Public RPC | `POST` JSON-RPC `getBalance` | 9 (lamports) | Free public RPCs available |
| Tezos | tzkt.io | `GET /v1/accounts/{addr}/balance` | 6 (mutez) | Free, generous limits |
| Cosmos | mintscan/rest | `GET /cosmos/bank/v1beta1/balances/{addr}` | 6 (uatom) | Public LCD endpoints |
| XRP | xrplcluster.com | `POST` JSON-RPC `account_info` | 6 (drops) | Free public |
| Polkadot | subscan.io | `POST /api/v2/scan/account/tokens` | 10 (planck) | Free tier available |
| TON | toncenter.com | `GET /api/v3/account?address={addr}` | 9 (nanoton) | Free, need to convert address first |
| Cardano | blockfrost or koios | `GET /addresses/{addr}` | 6 (lovelace) | Free tier with key, or koios (free) |
| NEAR | public RPC | `POST` JSON-RPC `query` (view_account) | 24 (yoctoNEAR) | Free |
| Stellar | horizon.stellar.org | `GET /accounts/{addr}` | 7 (stroops) | Free |
| Hedera | mirror.hedera.com | `GET /api/v1/accounts/{addr}` | 8 (tinybars) | Free |
| MultiversX | api.multiversx.com | `GET /accounts/{addr}` | 18 (denomination) | Free |
| Tron | trongrid.io | `POST /wallet/getaccount` | 6 (sun) | Free tier |
| Stacks | hiro.so | `GET /v2/accounts/{addr}` | 6 (microstacks) | Free, need addr conversion |
| Sui | fullnode.mainnet.sui.io | `POST` JSON-RPC `suix_getBalance` | 9 (MIST) | Free |
| Aptos | fullnode.mainnet.aptoslabs.com | `GET /v1/accounts/{addr}/resource/0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>` | 8 (octas) | Free |
| Mina | graphql.minaexplorer.com | GraphQL query | 9 (nanomina) | Free |
| Kaspa | api.kaspa.org | `GET /addresses/{addr}/balance` | 8 (sompi) | Free |

### Tier 4: Difficult / limited
| Chain | Issue |
|---|---|
| Internet Computer (ICP) | Account model differs, needs principal → account ID conversion |
| Algorand | Free APIs limited, may need Algonode |
| Vechain | Less common, connex or vethor node |
| Klaytn | Transitioning to Kaia, APIs shifting |
| dYdX/Injective/Osmosis/etc (Cosmos SDK) | Same REST pattern as Cosmos, just different LCD endpoint |

---

## Implementation phases

### Phase 1: EVM + existing UTXO (biggest coverage, one pattern)
- Build `fetchEvmBalance(rpcUrl, addr)` helper
- Map each EVM chain to a public RPC URL
- Refactor UTXO_NETS fetchers to share the new state pattern
- Add `liveBalances` state to App
- Display in AcctCard: balance + "Live" badge
- Loading/error states per card
- ~25 chains covered

### Phase 2: Major L1s
- Solana, Tezos, Cosmos, XRP, Polkadot, TON, Cardano, NEAR
- Each needs a custom fetch function but they're all simple REST/RPC
- ~8 more chains, covering probably 90%+ of real user accounts

### Phase 3: Long tail + comparison
- Remaining chains from Tier 3/4
- **Balance comparison feature**: when app.json is loaded, show delta between live balance and app.json balance on each account card
- Visual indicator: ✓ match, ⚠ mismatch with diff amount
- This is the diagnostic gold — instantly spots sync bugs

### Phase 4: Polish
- Batch fetch with smart staggering (don't fire 13 requests simultaneously)
- Cache results for session (don't re-fetch on section switch)
- "Refresh balances" button with cooldown
- Aggregate portfolio value if countervalue data available
- Error handling: distinguish "address not found" (empty/new account) from "API down"

---

## Rate limiting strategy

- **Stagger requests**: 200ms gap between each fetch (13 accounts = ~2.6s total)
- **Group by chain**: If 3 Ethereum accounts, use one RPC per account but batch where possible
- **Fallback on 429**: Retry with exponential backoff (already implemented for UTXO_NETS)
- **Cache in state**: Don't re-fetch unless user clicks "Refresh"
- **Show staleness**: Display "fetched 30s ago" timestamp

---

## UI integration in AcctCard

Current AcctCard shows: chain name, address, operations count, explorer link, app badges.

Add below the address or as a new row:
```
Live balance: 2.4851 ETH    [fetched 5s ago]
```

When app.json is also loaded:
```
Live balance: 2.4851 ETH    ✓ matches app.json
```
or:
```
Live balance: 2.4851 ETH    ⚠ app.json shows 2.3200 ETH (stale by 0.1651)
```

Loading state:
```
Live balance: ··· fetching    [shimmer animation]
```

Error state:
```
Live balance: — (API unavailable)
```

Unsupported chain:
```
[no balance row shown — don't show "unsupported"]
```

---

## Network consideration

**IMPORTANT**: The tool currently runs from a local file (file:// or localhost). Network egress must be available for API calls. The existing UTXO scanner (Xpub Scanner) already makes external API calls, so this is an established pattern. However, in environments where network is restricted (like the Claude Code sandbox), these fetches will fail gracefully with error states.

---

## Files to modify

- `ledger-toolkit.html` — all changes in single file:
  - New: API registry object mapping chainId → fetch function
  - New: `useLiveBalances` hook or inline effect in App
  - Modified: AcctCard to display live balance
  - Modified: App state to include liveBalances

---

## Dependencies

- None. All APIs are REST/RPC with fetch(). No npm packages needed.
- Existing UTXO_NETS pattern is the template.

---

## Success criteria

1. Agent loads a log → sees live balances on all supported accounts within 3-5 seconds
2. No app.json needed for basic balance visibility
3. When app.json IS loaded, mismatches are flagged automatically
4. Failed fetches show graceful error states, don't break anything
5. Works for the top 20 chains that cover 95%+ of real Ledger users
