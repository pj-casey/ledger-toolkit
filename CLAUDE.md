# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design references:** REDESIGN_VISION_V5.md (design philosophy), VIEWPORT_REDESIGN_SCOPE.md (fixed-viewport architecture), ROADMAP_LIVE_BALANCES.md (live balance architecture — now implemented)

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~4,400 lines, grows after guide embed)
- React 18.3.1 + Babel standalone 7.26.10, bitcoinjs-lib 5.2.0, bs58 4.0.1, buffer 6.0.3
- No build step — opens directly in browser
- **Fixed viewport** — root is `height:100vh, overflow:hidden`. The page never scrolls. Each view fills available space with a fixed header zone + scrollable content panel.
- **Live network calls** — fetches on-chain balances (public RPCs) and fiat prices (CoinGecko) on log load. No API keys required.

---

## What the tool does

1. Agent drops a JSON/TXT/LOG file (desktop or mobile format)
2. 3-tier parser handles minified, pretty-printed, and corrupted JSON
3. Mobile log normalization: `date→timestamp`, reverse chronological order, filename parsing for app version + platform
4. **Live balance fetching:** Auto-fetches on-chain balances for all accounts (40+ chains), then batch-fetches fiat prices from CoinGecko
5. **Two modes:**

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Health status pills (clickable). Unified Device card on Overview (app chips, merged environment + reference popover). Firmware update status. Live balances + fiat values on AcctCards. Portfolio stat card. Timeline session strip with stacked color-coded bars and interactive legend chips. Issues with interactive severity/category chips, error-prominent strip, and breadcrumbs. Copy Summary/Full/Customer with balances.

**Customer View:** Sidebar + main area + info bar. App.json enrichment (encrypted-safe). Copy Customer Summary. Click-to-copy. 45+ chain tx explorer URLs. dApp usage history.

---

## Golden rules

1. **Fixed viewport / single frame.** The page never scrolls. Each tab's content must be visible/accessible within its single window — through nested popups, pulldowns, collapsible sections, or clearly defined scrolling.

2. **Change how things LOOK, never how things WORK.** Data layer is frozen — but MAY be extended to support new log formats (e.g., LLv4/Ledger Wallet 4.0) as long as existing parsing for older logs is not broken.

3. **Every visual element must answer exactly one question.** If it answers two questions, split it. If it answers zero questions, remove it.

4. **No theatre. No dead space.** Every pixel should earn its place. But don't sacrifice information density for tidiness — "maximally informative" is the goal.

5. **Consistent color system.** TC type colors appear on Timeline bars, legend chips, row badges, Issues error strip, breadcrumb type badges. The same color for "bridge" everywhere. Agents learn it once.

---

## DO NOT MODIFY (data layer)

**Parsing:** `parseLogs()` (3-tier with brace scanner), `parseAppJson()` (encrypted-safe), `synthesizeMobileMeta()` (mobile log normalization)
**Error KB:** `ERR_DB` (82), `diagnose()`, `SEV`, `CAT`
**Chains:** `CHAINS` (60), `TX_EXPLORERS` (45+), `UTXO_NETS`, `getChain()`, `DC`, `DECIMALS`
**Address:** `cardanoCredToStake`, `stacksPubkeyToAddr`, `tonHexToAddr`, `fmtBal`
**Tokens:** `TOKEN_CONTRACTS`, `fetchTokenChains`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`
**Live Balances:** `COINGECKO_IDS`, `EVM_RPCS`, `fetchEvmBalance`, `BALANCE_APIS`, `fetchPrices`

## MAY EXTEND (for new log format support)

These were extended to support LLv4 (Ledger Wallet 4.0) logs. Extensions are additive only — old log formats still parse correctly.

**`extractDevice`** — added D-7 (target ID masking), D-5 (Manager API name), D-6 (Gen5 firmware prefix). Added `userAgent` extraction and `appBrand` derivation.
**`extractAccounts`** — added LLv4 bridge sync fallback (sets `dur` from `SyncSession finished` when no SyncSuccess analytics events exist).
**`extractDeviceApps`** — added tertiary `catalog-only` source (hw install events + apps-by-target catalog).

## CAN MODIFY (rendering/styling on experimental branch)

Rendering and styling of ALL components may be modified for UX improvements.

---

## Software naming (Ledger Live → Ledger Wallet)

Ledger Live was renamed to Ledger Wallet on October 23, 2025 (Op3n event). Desktop version numbers:

| Era | Version in logs | Product name |
|---|---|---|
| Ledger Live | 2.46 through ~2.131 | Ledger Live |
| Transitional | ~2.132 through ~2.145 | Ledger Wallet (brand), version still 2.x.x |
| Wallet 4.0 | 4.0.0 | Ledger Wallet |

**Detection:** `extractDevice` parses `userAgent` from `exportLogsMeta` for `LedgerWallet` vs `LedgerLive`. Fallback: `parseFloat(appVer) >= 2.132` → wallet. Stored as `dev.appBrand` ('wallet' or 'live').

**Helpers:** `llLabel(dev, isMobile)` returns correct product name. `llText(text, dev)` does runtime replacement of "Ledger Live" → "Ledger Wallet" in ERR_DB advice text without modifying the frozen ERR_DB array.

---

## Device identification — 7 detection paths

`extractDevice` tries these in order, stopping when `modelId` and `fw` are both found:

1. **Data field scan** — `d.modelId`, `d.deviceVersion`, etc. in entry data (pre-LLv4)
2. **D-4: Manager URL params** — `firmware_version_name=` and `device_type=` in Manager API URLs
3. **D-7: Target ID masking** — `(targetId & 0xFFFF0000)` against `TARGET_MASKS`. Same algorithm as `@ledgerhq/devices`. Scans `data.targetId`, `data.data.target_id`, and URL params. **Primary LLv4 identification.**
4. **D-5: Manager API response** — `get_device_version` response `data.data.name` (e.g., "Nano S Plus")
5. **D-6: Firmware path prefix** — `nanos+/1.5.1/...` → model + firmware. Covers `apex_p`/`apex` for Gen5.
6. **exportLogsMeta** — `env` block fields, `release`, `userAnonymousId`, `accountsIds`, `userAgent`

### TARGET_MASKS constant

```
nanoS:  0x31100000    nanoX:  0x33000000    nanoSP: 0x33100000
stax:   0x33200000    europa: 0x33300000    apex:   0x33400000
```

Source: `@ledgerhq/devices` package in LedgerHQ/ledger-live monorepo.

---

## Three version concepts (NEVER conflate)

| Concept | Log field | Extraction | UI label |
|---|---|---|---|
| Desktop app | `data.appVersion`, `release`, `userAgent` | `info.appVer`, `info.appBrand` | `llLabel(dev)` — "Ledger Wallet" or "Ledger Live" |
| Device firmware (secure element OS) | `data.deviceVersion`, `seVersion` | `info.fw` | "Firmware" |
| Device coin apps (on hardware) | `actions-manager-event`, hw install events, apps-by-target catalog | `extractDeviceApps()` | "Device Apps" |

---

## Device App Detection

**Primary:** `actions-manager-event` result with `installed[]` array → `source: 'manager-result'`. Full app list with versions and update status.
**Secondary:** `live-dmk-logger` APDU `b001` responses → `source: 'apdu-only'`. Single running app only.
**Tertiary (LLv4):** `type: "hw"` install events + `apps-by-target` API response catalog → `source: 'catalog-only'`. Shows required apps with latest available versions from catalog, confirmed installs from session.
**None:** Returns null. UI hides DEVICE APPS section entirely.
**AcctCard badges:** Only render for ⚠ outdated and ✕ missing. Green suppressed.
**Chain enrichment:** App rows show dependent chain tickers when 2+ chains require the same app.

---

## Timeline tab — current architecture

**Fixed header zone (flexShrink:0):**
- Session strip container:
  - Stacked bars (80 buckets, TC colors, height by event density). Error dots on 24px baseline below.
  - Hover: `tlHoveredBucket` state → tooltip with timestamp, event count, per-type breakdown.
  - Click bars → scroll to nearest entry. Click error dot → scroll to that error entry.
- Interactive legend chips: `sortedTypes` array, each chip shows TC color + type name + count. Hover sets `tlHighlightType` → dims non-matching bar segments. Click sets `typeFilter` (shares state with dropdown).
- Type filter dropdown + search bar + entry count + Grouped/Flat toggle.
- `tlAcctFilter` banner (when filtering by account): shows account name + chain color, "Show all" to clear.

**Scrollable panel (flex:1, overflowY:auto, ref=timelineScrollRef):**
- `groupedTimeline` useMemo: consecutive same-type within 1.5s → group headers. Errors exempt.
- Group headers: "N× type · duration · ▶ expand". Click toggles `expandedGroups[key]`.
- Individual rows: `data-li={logIndex}` attribute for scroll targeting. Click expands raw data.

**Key state:** `tlHoveredBucket`, `tlHighlightType`, `tlAcctFilter`, `tlGrouping`, `expandedGroups`

---

## Issues tab — current architecture

**Fixed header zone (flexShrink:0):**
- Row 1: Severity chips (Critical/Warning/Info — click `errSevFilter`, hover `errHoveredSev`) + Category chips (computed from enrichedErrs — click `errCatFilter`, hover `errHoveredCat`) + Copy Errors button.
- Row 2: Repeating pattern badges ("↻ title ×N" — click `errPatternFilter`). Only shows when patterns exist.
- Row 3: Error-prominent session strip (80px). TC-colored stacked bars, error segments at full opacity, context at 25%. Hover: `errStripHover` state → tooltip with time, event count, error titles. Click: selects error in master-detail. Severity/category chip hover dims non-matching error segments.
- Row 4: Affected accounts chips (from `errAcctMap`). Clickable → navigates to Accounts.
- Filter banners: `errAcctFilter` banner (from Accounts section), `errSevFilter/errCatFilter/errPatternFilter` banner with "Clear filters".

**Master-detail body (flex:1, overflow:hidden, display:flex):**
- Left 45%: `sorted` error list (severity-ranked). `filteredErrs` applies sev/cat/pattern filters to `displayErrs`.
- Right 55%: `ErrCard` detail panel for `effectiveSel` error.

**ErrCard component props:** `{err, onJump, linkedAcct, onGoAcct, dev, entries}`
- Breadcrumbs: "What happened before" — 5 preceding entries with relative timestamps, TC-colored type badges. `ctxOpen` internal state.
- "View in Timeline" button: calls `onJump(err.li)` → `jumpTo` which clears all Timeline filters + scrolls.

**Key state:** `selectedErr`, `errSevFilter`, `errCatFilter`, `errPatternFilter`, `errHoveredSev`, `errHoveredCat`, `errStripHover`, `errAcctFilter`

---

## Accounts tab — current architecture

**Fixed header zone (flexShrink:0):**
- Title bar: funded/empty/no-data counts + Copy IDs + Export buttons
- No-sync banners
- Enhanced health tiles (`position:relative` wrapper):
  - Inline total fiat + account count + "Portfolio ▸" button
  - 30px tiles per account: ticker label, chain color, opacity by status, corner dots (red=errors, purple=xpub, ?=unsupported)
  - Click toggles `acctFilter` for that chain; Shift+click sets `tlAcctFilter` + navigates to Timeline
  - Hover: `hoveredAcct`/`hoveredRef` → absolute popover below tile
  - `acctMapOpen` → absolute portfolio overlay (proportional fiat-sized tiles)
- Filter input (`acctFilter`)

**Scrollable panel (flex:1, overflowY:auto):**
- EVM grouping: same-address accounts grouped, ETH first. "SHARED ADDRESS" header + CopyBtn + chain count + group fiat.
- `AcctCard` props: `acct, errCount, deviceApps, liveBalance, onErrClick, onTimelineClick`
- Error badge → sets `errAcctFilter` + navigates to Issues
- "Timeline" button → sets `tlAcctFilter` + navigates to Timeline
- Xpub scan button on collapsed UTXO cards

---

## Landing page — guide overlays

**Buttons:** "📖 Agent Guide" and "🔬 Technical Reference" below drag/drop zone.
**Content:** Embedded as `GUIDE_AGENT` and `GUIDE_TECHNICAL` JS string constants (NOT fetched — `file://` blocks fetch).
**Overlay:** Full-screen backdrop, 900px card, header with title + search + close button. Click-outside and Escape dismiss.
**Search:** Text input highlights matches with `<mark>` replacement.
**CSS:** `.guide-embed` class scopes all guide styling within the overlay.
**State:** `guideOpen` ('agent'|'technical'|null), `guideSearch`

---

## Fixed Viewport Layout

Root: `height:100vh, overflow:hidden, display:flex, flexDirection:column`
Top bar: `height:52px, flexShrink:0`
Main area: `flex:1, overflow:hidden`
Each view: `display:flex, flexDirection:column, height:100%, overflow:hidden` → fixed header (`flexShrink:0`) + scrollable panel (`flex:1, overflowY:auto`)

Scrollable panels have: `borderTop: 1px solid #3C3C3C`, `boxShadow: inset 0 2px 6px rgba(0,0,0,0.15)`, internal padding. The page itself never scrolls.

---

## Design tokens

```
bg:#131214  panel:#1C1D1F  card:#242528  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#F57375  warning:#FFBD42
```

Three elevation levels: bg → panel → card. Cards use background, NOT borders. Fonts: Inter + JetBrains Mono.

---

## Sidebar

240px fixed width. `DiagSidebar` component. Receives `liveBalances` prop for fiat hints.

---

## No-Sync Visibility

When a log has accounts but no sync data (`!accts.some(a => a.dur != null || a.bal != null)`):
- Persistent amber banner on Overview below health pills
- Sidebar diagnosis sentence appended with "· No sync data"
- AcctCards: amber left border, "no data" badge

**LLv4 bridge sync fallback:** When no `SyncSuccess` analytics events exist, `extractAccounts` checks for `type: "bridge"` entries with `"SyncSession finished"`. If found, sets `dur` on all accounts from the total session duration. This prevents false "no sync" alerts on Ledger Wallet 4.0+ logs.

---

## Keyboard shortcuts

Ctrl+1 Overview | Ctrl+2 Issues | Ctrl+3 Accounts | Ctrl+4 Timeline | Ctrl+5 Network | Ctrl+6 APDU | Ctrl+7 Raw JSON

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `section`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `deviceExpanded`, `devDetailOpen`, `liveBalances`, `globalSearch`, `showGlobalResults`, `selectedErr`, `errSevFilter`, `errCatFilter`, `errPatternFilter`, `errHoveredSev`, `errHoveredCat`, `errStripHover`, `errAcctFilter`, `hoveredAcct`, `hoveredRef`, `acctMapOpen`, `tlHoveredBucket`, `tlHighlightType`, `tlAcctFilter`, `tlGrouping`, `expandedGroups`, `guideOpen`, `guideSearch`
**DiagSidebar:** `accountsOpen`, `advOpen`, `copyOpen`
**CustomerView:** `selectedAcct`, `summCopied`, `ajDragOver`
**JTTree:** `expanded`, `search`, `copiedPath`, `matchPaths`, `currentMatch`, `scrollContainerRef`
**ErrCard:** `detOpen`, `ctxOpen`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 10 | Theme colors (includes `card` elevation) |
| `TC` | 9 | Log type badge colors — used on Timeline bars, legend chips, Issues strip, breadcrumbs |
| `DN` | 6 | Device model names (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `TARGET_MASKS` | 6 | Target ID → device model mapping (same as @ledgerhq/devices) |
| `CHAINS` | 60 | Chain registry + explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |
| `CURRENCY_TO_APP` | 46 | Currency ID → device app name |
| `COINGECKO_IDS` | 40+ | Chain currency ID → CoinGecko coin slug |
| `EVM_RPCS` | 23 | Chain → publicnode.com RPC URL |
| `BALANCE_APIS` | 17 | Chain → custom balance fetch function (non-EVM) |
| `GUIDE_AGENT` | 1 | Embedded agent guide HTML content |
| `GUIDE_TECHNICAL` | 1 | Embedded technical reference HTML content |

---

## Helper functions

| Function | Purpose |
|---|---|
| `identifyTargetId(tid)` | Maps numeric target ID to device model ID via bitmask |
| `llLabel(dev, isMobile)` | Returns "Ledger Wallet" or "Ledger Live" based on `dev.appBrand` |
| `llText(text, dev)` | Runtime "Ledger Live" → "Ledger Wallet" replacement for ERR_DB strings |
| `jumpTo(li)` | Navigate to Timeline, clear all filters/grouping, expand entry, scroll to it |
| `goToAcct(addr)` | Navigate to Accounts with filter applied |
| `openGuide(type)` | Open guide overlay ('agent' or 'technical') |
| `sevColor(s)` | Returns severity color for high/medium/low |

---

## Spec files in repo

| File | Status | Purpose |
|---|---|---|
| `CLAUDE.md` | **Active — primary reference** | This file. Claude Code reads this before every task. |
| `SESSION_HANDOFF.md` | **Active — session memory** | Context for new Claude instances picking up the project. |
| `ROADMAP_LIVE_BALANCES.md` | **Implemented (Phase 1+2)** | Phase 3 (app.json comparison) and Phase 4 (polish) pending rethink. |
| `REDESIGN_VISION_V5.md` | Historical reference | Core principles still apply. |
| `VIEWPORT_REDESIGN_SCOPE.md` | Historical reference | Foundation implemented, layouts evolved. |
| `agent-guide.html` | **Active** | Agent-facing investigation guide. Updated to v4.1. |
| `technical-reference.html` | **Active** | Technical capabilities reference. New in v4.1. |

---

## Git tags

| Tag | Commit | What's included |
|---|---|---|
| `pre-live-balances` | `20e7760` | Mobile support + UX refinements + Issues master-detail. |
| `post-live-balances` | `9081d1c` | Live balance fetching + fiat values + no-device UX clarity. |
| `pre-device-card` | — | Checkpoint before unified Device card session. |
| `post-overview-redesign` | `01f89d4` | Unified Device card, app chips, adaptive error grid, viewport scaling. |
| `pre-llv4-compat` | — | Tag before LLv4 compatibility session. |
| `post-llv4-compat` | `bf6b0e1` | LLv4 device ID, sync detection, app catalog, branding rename. |
| `post-accounts-overhaul` | `9eeee24` | Enhanced tiles, hover popover, portfolio overlay, EVM grouping, error nav. |
| `post-timeline-issues` | — | **Tag after this session.** Timeline strip, Issues interactive filters, breadcrumbs, guide overlays. |
