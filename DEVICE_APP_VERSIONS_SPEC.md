# Device App Versions — Feature Spec

*Sprint: "Does the customer have the right apps, and are they current?"*

---

## Problem

CS agents need to know what coin apps are installed on a customer's Ledger device and whether those apps are up to date. The toolkit currently shows "App ver: 4.0.0" which is **Ledger Live's desktop version**, not any device coin app. This is misleading and unhelpful.

The logs don't store device coin app versions as structured data. They're fragmented across APDU payloads, DMK logger entries, and Manager API network calls — when they exist at all. Most log exports happen without a device connected, so in the common case this data is absent entirely.

---

## Three version concepts (never conflate)

| What | Where in logs | Current extraction | Display label |
|---|---|---|---|
| **Ledger Live version** | `data.appVersion` in analytics, `release` in exportLogsMeta | `extractDevice` → `info.appVer` | "Ledger Live" or "App" |
| **Device firmware** | `data.deviceVersion`, `seVersion`, Manager API URL `firmware_version_name` | `extractDevice` → `info.fw` | "Firmware" |
| **Device coin apps** | APDU `B001` responses, `live-dmk-logger` entries, Manager API catalog | **NOT EXTRACTED** | "Device Apps" |

The first deliverable of this sprint is making sure labels in the UI never confuse these three.

---

## Data sources for device coin app versions

### Source A: `GetAppAndVersion` APDU responses

When Ledger Live opens a coin app, it sends APDU `B0 01 00 00 00`. The device responds with the app name and version as a TLV payload, terminated by status `9000`.

Response format: `01 <name_len> <name_bytes> 02 <ver_len> <ver_bytes> ... 9000`

Example: `01 07 426974636f696e 02 05 322e332e30 ... 9000`
→ App name: "Bitcoin", Version: "2.3.0"

**Where it appears in logs:** As an `apdu-out` / `← OUT` entry. The hex payload is in `e.data.apdu` or `e.apdu` or `e.data.hex`.

**Extraction approach:** In `extractApdu`, we already capture `{ts, dir, hex, li}` for every APDU exchange. New function `extractDeviceApps()` scans APDU OUT entries for `B001` command responses (look for IN entry with CLA `B0` INS `01`, then the immediately following OUT entry). Parse the TLV to get `{name, version, timestamp}`.

**Limitation:** Only captures whatever app was opened during the session. If the user opened the Ethereum app to sign a transaction, you'll see Ethereum's version but nothing else.

### Source B: `live-dmk-logger` entries (DMK — Device Management Kit)

Ledger Live 4.x uses the new DMK which logs device interactions via `live-dmk-logger` type entries. Relevant commands:

- `GetAppAndVersionCommand` → returns `{name, version}` for the currently running app
- `ListAppsCommand` → enumerates ALL installed apps with names, hashes, and sizes
- `GetOsVersionCommand` → returns `{seVersion, mcuSephVersion, mcuBootloaderVersion}`

**Where it appears in logs:** Entries with `type: "live-dmk-logger"` and `data.tag` containing the command name (e.g., `[GetAppAndVersionCommand]`, `[ListAppsCommand]`).

**Extraction approach:** Scan `live-dmk-logger` entries for tags matching known command names. Parse the message or data fields for app name/version data. The exact format may vary — log the raw data shape when encountered for iteration.

**Limitation:** Only present when a device was connected AND these commands actually fired. The DMK logger doesn't always emit structured result data.

### Source C: Manager API network requests

When the user opens My Ledger (Manager), Ledger Live calls:
```
GET manager.api.live.ledger.com/api/v2/apps/by-target?
    livecommonversion=...
    &target_id=...
    &firmware_version_name=2.6.1
    &device_type=nanoX
```

This returns the **catalog** of available apps for that device model + firmware — their names, latest versions, sizes, etc. This is NOT what's installed, but what COULD be installed.

**Where it appears in logs:** Entries with `type: "network"` and `message` containing `manager.api.live.ledger.com`. The code already detects Manager API access in `extractActivity()` (`managerOpened` flag) and parses `device_type` and `firmware_version_name` from the URL.

**Extraction approach:** When Manager API URLs are detected, extract the query params to determine device model + firmware. Optionally, the toolkit could fetch the catalog at analysis time (like it already does for token contracts via `fetchTokenChains`).

**Limitation:** Response bodies typically aren't logged — only the request URL. To get actual latest app versions, we'd need a runtime fetch or a hardcoded reference table.

---

## What to build

### Phase 1: Extraction (data layer — new functions)

#### `extractDeviceApps(entries, apduData)`

New extraction function. Returns:
```js
{
  openedApps: [
    { name: "Ethereum", version: "1.12.1", timestamp: "...", source: "apdu" },
    { name: "Bitcoin", version: "2.3.0", timestamp: "...", source: "dmk" }
  ],
  installedApps: [
    // From ListApps if available — usually empty
    { name: "Ethereum", hash: "...", size: 123456 }
  ],
  managerSession: {
    opened: true,
    deviceType: "nanoX",
    firmwareVersion: "2.6.1",
    targetId: "...",
    catalogUrl: "https://manager.api.live.ledger.com/api/v2/apps/by-target?..."
  } | null
}
```

**APDU parsing for GetAppAndVersion (`B001`):**
1. Find APDU IN entries where hex starts with `B001` (CLA=B0, INS=01)
2. Take the next APDU OUT entry (the response)
3. If response ends with `9000`, parse TLV:
   - Tag `01` → app name (UTF-8)
   - Tag `02` → app version (UTF-8)
4. Deduplicate by name (keep latest timestamp)

**DMK logger parsing:**
1. Filter entries where `type === 'live-dmk-logger'`
2. Look for `data.tag` containing `GetAppAndVersion` or `ListApps`
3. Extract name/version from message or data fields
4. Format may vary — be defensive, log unknowns

**Manager session extraction:**
1. Already partially done in `extractActivity()` and `extractDevice()`
2. Consolidate: find the Manager API URL, extract query params
3. Store `deviceType`, `firmwareVersion`, `targetId`

#### `inferRequiredApps(accounts)`

New function. Maps accounts to the device app they require:
```js
// Input: logData.accts (from extractAccounts)
// Output:
[
  { currencyId: "ethereum", appName: "Ethereum", chain: {...} },
  { currencyId: "bitcoin", appName: "Bitcoin", chain: {...} },
  { currencyId: "tezos", appName: "Tezos", chain: {...} },
  ...
]
```

Mapping logic: use the `CHAINS` registry. For most chains, the app name matches `chain.name`. For EVM L2s (Polygon, Arbitrum, Base, etc.), the required app is "Ethereum". For Bitcoin forks (LTC, DOGE, BCH), each has its own app. Build a `CURRENCY_TO_APP` lookup:

```js
const CURRENCY_TO_APP = {
  ethereum: 'Ethereum',
  bitcoin: 'Bitcoin',
  tezos: 'Tezos',
  cosmos: 'Cosmos',
  solana: 'Solana',
  tron: 'Tron',
  ripple: 'XRP',
  polkadot: 'Polkadot',
  near: 'NEAR',
  cardano: 'Cardano',
  ton: 'TON',
  stellar: 'Stellar',
  // EVM L2s → all need Ethereum app
  polygon: 'Ethereum',
  arbitrum: 'Ethereum',
  optimism: 'Ethereum',
  base: 'Ethereum',
  bsc: 'Ethereum',
  avalanche_c_chain: 'Ethereum',
  linea: 'Ethereum',
  scroll: 'Ethereum',
  blast: 'Ethereum',
  zksync: 'Ethereum',
  fantom: 'Ethereum',
  cronos: 'Ethereum',
  mantle: 'Ethereum',
  // UTXO
  litecoin: 'Litecoin',
  dogecoin: 'Dogecoin',
  bitcoin_cash: 'Bitcoin Cash',
  // Others
  stacks: 'Stacks',
  algorand: 'Algorand',
  hedera: 'Hedera',
  elrond: 'MultiversX',
  filecoin: 'Filecoin',
  vechain: 'VeChain',
  mina: 'Mina',
  kaspa: 'Kaspa',
  aptos: 'Aptos',
  sui: 'Sui',
  // Add as needed
};
```

Deduplicate: if user has 3 Ethereum accounts + 2 Polygon accounts, "Ethereum" app appears once.

### Phase 2: Reference data (latest versions)

#### Option A: Hardcoded lookup (simple, offline)

A `LATEST_APP_VERSIONS` constant mapping app name → latest known version per device family. Updated manually when the toolkit is updated. Example:

```js
const LATEST_APP_VERSIONS = {
  'Ethereum': { nanoS: '1.11.1', nanoSP: '1.12.1', nanoX: '1.12.1', stax: '1.12.1', europa: '1.12.1' },
  'Bitcoin': { nanoS: '2.2.3', nanoSP: '2.3.0', nanoX: '2.3.0', stax: '2.3.0', europa: '2.3.0' },
  // ...
};
```

**Pros:** Works offline, no network dependency, fast.
**Cons:** Goes stale. Someone has to update it.

#### Option B: Runtime fetch from Manager API (richer, requires network)

When device model + firmware are known, fetch the catalog:
```
GET https://manager.api.live.ledger.com/api/v2/apps/by-target?
    livecommonversion=34.0.0
    &firmware_version_name=2.6.1
    &device_type=nanoX
```

Cache in memory (like `TOKEN_CONTRACTS`). Parse the response to build a latest-version lookup.

**Pros:** Always current, authoritative source.
**Cons:** Requires network, may be rate-limited, adds latency.

#### Recommendation: Option A first, Option B as enhancement

Ship with a hardcoded table. Add a "Check for updates" button that does the runtime fetch when device info is available. Same pattern as the xpub scanner — works offline, optional network enhancement.

### Phase 3: UI — "Device Apps" card/section

#### In Diagnosis Detail (Overview)

New card below the Device & App card. Only renders when there's useful data to show (any accounts exist, or any device app data detected).

```
┌─────────────────────────────────────────────────┐
│  DEVICE APPS                                     │
│                                                   │
│  Required by accounts:                            │
│  ✅ Ethereum 1.12.1     (latest)                  │
│  ⚠️ Bitcoin  2.1.0      (2.3.0 available)         │
│  ❓ Tezos               (not detected)            │
│  ❓ Cosmos              (not detected)            │
│                                                   │
│  4 apps required · 1 detected · 1 outdated        │
│  ℹ️ Connect device in Ledger Live and re-export   │
│     logs to detect installed app versions.        │
└─────────────────────────────────────────────────┘
```

Status indicators:
- `✅` — detected and matches latest known version
- `⚠️` — detected but outdated (show installed vs latest)
- `❓` — required by account but not detected in this session
- `❌` — detected but known-incompatible (edge case, future)

When no accounts and no device apps detected: don't render the card at all.

When accounts exist but no device app data: show the required list with all `❓` and the guidance message about connecting the device.

#### In the Copy Summary / Copy Full outputs

Add a "DEVICE APPS" section:
```
DEVICE APPS (4 required)
  Ethereum    1.12.1  (latest)
  Bitcoin     2.1.0   (outdated — 2.3.0 available)
  Tezos       —       (not detected)
  Cosmos      —       (not detected)
```

#### In AcctCard (optional enhancement)

When an account's required app IS detected, show a small version badge:
```
Ethereum 1  ·  3 ops  ·  ETH app 1.12.1 ✅
```

When outdated:
```
Bitcoin 1  ·  xpub  ·  12 ops  ·  BTC app 2.1.0 ⚠️
```

---

## Fix existing label confusion

**Immediate fix (Phase 0):**

In the Device & App card on the Overview, change:
- "App ver" → "Ledger Live" (with the version number)
- Keep "Firmware" as is
- If any device app is detected, add a row: "Device app: Ethereum 1.12.1" (etc.)

In the Copy Summary/Full text outputs:
- `App ver:` → `Ledger Live:`

This is a rendering-only change — `info.appVer` stays the same internally.

---

## Integration with `logQualityScore`

Add a new quality score dimension:

```js
// Device app info present: 10pts
const hasDeviceApp = deviceApps.openedApps.length > 0;
details.push({
  field: 'Device app version',
  found: hasDeviceApp,
  note: hasDeviceApp
    ? `${deviceApps.openedApps.map(a => `${a.name} ${a.version}`).join(', ')}`
    : 'No device app version detected (device may not have been connected)'
});
if (hasDeviceApp) score += 10;
```

Rebalance existing weights so total stays 100.

---

## CLAUDE.md updates

When implementing this feature, update CLAUDE.md:

### 1. V5 redesign status
Change:
```
**Branch:** `experimental` — UI/UX overhaul in progress. See REDESIGN_VISION_V5.md for the full design vision.
```
To:
```
**Branch:** `experimental`
**Design reference:** REDESIGN_VISION_V5.md — sidebar + main area layout spec (implemented).
```

### 2. New extraction functions → add to DO NOT MODIFY once stable
```
**Device Apps:** `extractDeviceApps`, `inferRequiredApps`, `CURRENCY_TO_APP`, `LATEST_APP_VERSIONS`
```

### 3. Update function count and state
Add new functions to inventory. Add `deviceApps` to logData shape if stored there.

### 4. Add to "Implemented UX changes" list
```
- Device Apps card on Overview (required apps, detected versions, update status)
- "Ledger Live" label fix (was "App ver")
- Device app version badges on AcctCard
```

---

## DO NOT MODIFY (existing data layer)

Per CLAUDE.md: parseLogs, parseAppJson, extractDevice, extractAccounts, extractErrors, extractSync, extractApdu, extractActivity, logQualityScore, extractAnalytics, diagnose, ERR_DB, CHAINS, TX_EXPLORERS, UTXO_NETS, getChain, DC, DECIMALS, cardanoCredToStake, stacksPubkeyToAddr, tonHexToAddr, fmtBal, TOKEN_CONTRACTS, fetchTokenChains, TOKEN_URLS, TOKEN_SEARCH, EVM_CHAIN_IDS.

This spec **adds** new functions alongside these. It does not modify them (except logQualityScore gets a new scoring dimension added — this is an additive change, not a modification of existing scoring logic).

---

## Phasing

| Phase | What | Risk | Effort |
|---|---|---|---|
| 0 | Label fix: "App ver" → "Ledger Live" | None | Tiny |
| 1 | `extractDeviceApps()` + `inferRequiredApps()` + `CURRENCY_TO_APP` | Low — new code, no existing changes | Medium |
| 2 | Device Apps card on Overview | Low — rendering only | Medium |
| 3 | `LATEST_APP_VERSIONS` hardcoded table | Low but needs research | Small |
| 4 | Copy Summary/Full integration | None | Small |
| 5 | AcctCard version badges | Low | Small |
| 6 | Quality score integration | Low | Small |
| 7 | Runtime Manager API fetch (optional) | Medium — network, caching | Larger |

Phases 0–2 are the core deliverable. Phase 3–6 are fast follow-ups. Phase 7 is a stretch goal.
