# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `experimental`
**Design references:** REDESIGN_VISION_V5.md (design philosophy), VIEWPORT_REDESIGN_SCOPE.md (fixed-viewport architecture), ROADMAP_LIVE_BALANCES.md (live balance architecture — now implemented)

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~5,700 lines)
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

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Device status line + stat cards (Issues, Accounts, Network, Log Quality) on Overview. Unified Device card (app chips, merged environment + reference popover). Firmware update status. Live balances + fiat values on AcctCards. Help center article links on errors. Timeline session strip with stacked color-coded bars and interactive legend chips. Issues with interactive severity/category chips, error-prominent strip, and breadcrumbs. Copy report dropdown (Summary/Full/Customer/Errors) on Overview action bar.

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
3. **D-7: Target ID masking** — `(targetId & 0xFFFF0000)` against `TARGET_MASKS`. Same algorithm as `@ledgerhq/devices`. Scans `data.targetId`, `data.data.target_id`, and URL params. **Primary LLv4 identification.** Now also stores `info.targetId` (full numeric target_id) for Manager API calls. Post-D-7 fallback scan sets `targetId` from URL params, `data.targetId`, or `data.data.target_id` even when model was identified via a different path.
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
- Help article link: when `dg.u` exists, renders "📄 Help article ↗" below action text (opens support.ledger.com in new tab).
- Diagnostic Pathways: `useMemo` scans 5s before error for network failures, APDU rejections, sync events. Renders clickable "Related evidence" links.

**Key state:** `selectedErr`, `errSevFilter`, `errCatFilter`, `errPatternFilter`, `errHoveredSev`, `errHoveredCat`, `errStripHover`, `errAcctFilter`

---

## Accounts tab — current architecture

**Fixed header zone (flexShrink:0):**
- Title bar: funded/empty/no-data counts + Copy IDs + Export buttons
- No-sync banners
- Enhanced health tiles (`position:relative` wrapper):
  - Inline total fiat + account count + "Portfolio ▸" button
  - 30px tiles per account: ticker label, chain color, opacity by status, corner dots (red=errors, purple=xpub, ?=unsupported)
  - Click toggles `acctFilter` for that chain; Shift+click sets `focusedAcct` (Focus Mode)
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

Ambient gradients are layered on top of this structure: root has a radial vignette, top bar has a horizontal gradient, each fixed section header has a vertical gradient, the main content container has a subtle left-edge warmth. See "Ambient gradient system" in Design tokens.

---

## Design tokens

```
bg:#131214  panel:#1C1D1F  card:#242528  border:#3C3C3C  text:#FFFFFF
muted:#949494  primary:#BBB0FF  success:#7AC26C  error:#E40046  warning:#FFBD42
orange:#FF5300
```

Three elevation levels: bg → panel → card. Cards use background, NOT borders.

### Typography

Two-font rule — applied consistently across all UI surfaces:
- **Structural labels** (section headers, stat card labels, badge text, sidebar keys, filter chips): `MF` constant = `'JetBrains Mono','SF Mono','Fira Code',Consolas,ui-monospace,monospace`
- **Values, body text, descriptions**: Brut Grotesque (300–700), embedded via Base64 @font-face

`MF` is a module-scope constant. Never inline the font stack — reference `MF` everywhere monospace is needed.

| Font | Role | Loaded from |
|---|---|---|
| **Brut Grotesque** (300–700) | All body text, headings, values, descriptions, stat numbers | Base64 embedded @font-face (4 woff2 files: Light 300, Regular 400, Medium 500, Bold 600+700) |
| **JetBrains Mono** (400–500) | Labels, tags, badges, code, addresses, hashes | Google Fonts CDN |

File size: ~700KB total (~194KB for the 4 embedded font files). No CDN dependency for body font — works offline, works on `file://`.

### Ambient gradient system

Source color: `rgba(69,57,92,α)` = Ledger brand purple `#45395C`. Radiates from top-left corner.
Eight opacity tiers from strongest to weakest:

| Surface | Direction | Opacity |
|---|---|---|
| Overview sidebar block | 135deg | 0.35 |
| Top bar | 90deg (left→right) | 0.15 |
| Fixed section headers | 180deg (top→bottom) | 0.10 |
| Device card | 135deg | 0.10 |
| Stat cards (top edge) | — | highlight only |
| Sidebar (left panel) | 180deg | 0.08 |
| Main content container (left edge) | 90deg | 0.04 |
| Root viewport radial vignette | radial from top-left | 0.06 |

### Surface physics

Cards convey hierarchy and type through left-edge treatment, not background gradients:

- **ErrCards:** 3px colored left border (`sv.c`) + `boxShadow: inset 6px 0 16px -4px ${sv.c}35` (severity color)
- **AcctCards:** 3px colored left border (`ch.color+'40'` or `ch.color+'60'` when open) + `boxShadow: inset 8px 0 24px -2px ${ch.color}40` (chain color). Outer container must NOT have `overflow:'hidden'` — it clips the inset glow.
- **SectionHeaders:** Active tab → horizontal glow using `boxShadow: inset 0 -2px 8px -2px ${T.primary}40`

### Border-radius system

Three tiers, normalized across the entire tool:
- **8px** — cards, panels, containers, overlays, popups (ErrCards, AcctCards, stat cards, device card, network container, guide overlay, portfolio overlay)
- **6px** — buttons, interactive controls, input fields, filter chips, advice boxes
- **4px** — small badges, pills, severity tags, type tags, breadcrumb chips

Do NOT use 12, 10, 5, or 3. These have been eliminated.

### Hover patterns

Three standardized hover behaviors:
- **Data rows** (timeline, network, APDU, device info): `background → rgba(255,255,255,0.02–0.03)`
- **Cards** (ErrCard, AcctCard): existing hover classes (`.err-hover`, `.acct-hover`)
- **Ghost buttons** (outlined, transparent bg): `borderColor → T.primary, color → T.primary` on hover

---

## Sidebar

240px fixed width. `DiagSidebar` component. Props: `logData`, `section`, `setSection`, `liveBalances`, `focusedAcct`, `setFocusedAcct`. Contains: Overview header (severity-colored left border), Focus indicator, section list (Issues, Accounts expandable, Timeline, Advanced expandable), contextual help articles (Layer 2) at bottom.

**Gradient treatment:** Vertical ambient gradient (top→bottom, `rgba(69,57,92,0.08)` → transparent) on the sidebar panel background. The "Diagnosis" overview block uses a stronger `rgba(69,57,92,0.35)` gradient at 135deg for the most prominent brand warmth in the tool.

---

## Customer View — Visual Alignment

Customer View shares the same visual language as Diagnostic mode:

**CSS classes updated:**
- `.cv-sidebar` — vertical purple gradient matching Diagnostic sidebar
- `.cv-portfolio-card` — 8px radius, top-edge highlight
- `.cv-detail-card` — 8px radius, top-edge highlight
- `.cv-section-title` — JetBrains Mono font (mono label treatment)
- `.cv-info-row` — hover highlight on rows
- `.cv-acct-row.cv-active` — horizontal gradient glow from left border
- `.cv-infobar` — subtle top gradient

**Inline updates:**
- CV sidebar header, stat card labels, info bar labels all use `fontFamily:MF`
- Portfolio cards have chain-colored left borders with inset glow (matching Diagnostic AcctCards)

**Layout:** `.cv-layout` uses `height:calc(100vh - 52px)` to exactly fill the space below the 52px top bar. The diagnostic main div is hidden (`display:none`) when in customer mode.

---

## No-Sync Visibility

When a log has accounts but no sync data (`!accts.some(a => a.dur != null || a.bal != null)`):
- Persistent amber banner on Overview below device status line
- Sidebar diagnosis sentence appended with "· No sync data"
- AcctCards: amber left border, "no data" badge

**LLv4 bridge sync fallback:** When no `SyncSuccess` analytics events exist, `extractAccounts` checks for `type: "bridge"` entries with `"SyncSession finished"`. If found, sets `dur` on all accounts from the total session duration. This prevents false "no sync" alerts on Ledger Wallet 4.0+ logs.

---

## Keyboard shortcuts

Ctrl+1 Overview | Ctrl+2 Issues | Ctrl+3 Accounts | Ctrl+4 Timeline | Ctrl+5 Network | Ctrl+6 APDU | Ctrl+7 Raw JSON | Escape Clear Focus

---

## Help Center Integration

**Layer 1 — ERR_DB article links:** 33 ERR_DB entries have a `u` field with a `support.ledger.com/article/{ID}` URL. `errUrl(dg)` extracts it. Rendered on ErrCards ("📄 Help article ↗"), Overview error tile 📄 icons, and in `buildErrorText` copy output.

**Layer 2 — Contextual help sidebar:** `CONTEXT_HELP` constant (12 entries). Each has `test(logData)` function checking for conditions (firmware outdated, BT errors, sync failures, APDU rejections, etc.). Two always-on entries (Ongoing Issues, Common Solutions). Rendered at bottom of DiagSidebar, max 5 links. Link color: `T.orange` (`#FF5300`) on hover.

**Layer 3 — Live status banner:** Fetches `status.ledger.com/api/v2/summary.json` on log load (no auth). `ledgerStatus` state. `STATUS_CHAIN_MAP` matches component names to customer's chains. Amber banner when chains match, muted when they don't. Included in copy exports.

**Copy report dropdown:** On Overview action bar (not sidebar). 4 options with color-coded left borders: Quick Summary (purple), Full Export (purple), Customer Summary (orange `T.orange`), Copy Errors (crimson `T.error`). Standalone builder functions at module scope.

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `section`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `deviceExpanded`, `devDetailOpen`, `liveBalances`, `globalSearch`, `showGlobalResults`, `selectedErr`, `errSevFilter`, `errCatFilter`, `errPatternFilter`, `errHoveredSev`, `errHoveredCat`, `errStripHover`, `errAcctFilter`, `hoveredAcct`, `hoveredRef`, `acctMapOpen`, `tlHoveredBucket`, `tlHighlightType`, `tlAcctFilter`, `tlGrouping`, `expandedGroups`, `guideOpen`, `guideSearch`, `copyOpen`, `focusDropOpen`, `ledgerStatus`, `focusedAcct`, `versionCheck`
**DiagSidebar:** `accountsOpen`, `advOpen`
**CustomerView:** `selectedAcct`, `summCopied`, `ajDragOver`
**JTTree:** `expanded`, `search`, `copiedPath`, `matchPaths`, `currentMatch`, `scrollContainerRef`
**ErrCard:** `detOpen`, `ctxOpen`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 11 | Theme colors (includes `card` elevation and `orange:#FF5300`) |
| `MF` | 1 | Monospace font stack constant — `'JetBrains Mono','SF Mono','Fira Code',Consolas,ui-monospace,monospace` |
| `TC` | 9 | Log type badge colors — used on Timeline bars, legend chips, Issues strip, breadcrumbs. Same color for same type everywhere. |
| `DN` | 6 | Device model names (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `TARGET_MASKS` | 6 | Target ID → device model mapping (same as @ledgerhq/devices) |
| `CHAINS` | 60 | Chain registry + explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns (33 entries have `u` field → support article URL). Error colors use `T.error` (`#E40046`). |
| `EVM_CHAIN_IDS` | 22 | Chain → EVM chain ID |
| `TOKEN_URLS` | 23 | Token page URL builders |
| `TOKEN_SEARCH` | 8 | Token search fallbacks |
| `CURRENCY_TO_APP` | 46 | Currency ID → device app name |
| `COINGECKO_IDS` | 40+ | Chain currency ID → CoinGecko coin slug |
| `EVM_RPCS` | 23 | Chain → publicnode.com RPC URL |
| `BALANCE_APIS` | 17 | Chain → custom balance fetch function (non-EVM) |
| `GUIDE_AGENT` | 1 | Embedded agent guide HTML content |
| `GUIDE_TECHNICAL` | 1 | Embedded technical reference HTML content |
| `CONTEXT_HELP` | 12 | Condition → support.ledger.com article mappings for sidebar help links |
| `STATUS_CHAIN_MAP` | ~24 | Statuspage component name → CHAINS name for live status banner |
| `dropItemStyle` | 1 | Shared style object for Copy report dropdown items |

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
| `errUrl(dg)` | Returns support.ledger.com URL from ERR_DB `u` field, or null |
| `buildSummaryText(logData, liveBalances, ledgerStatus)` | Quick Summary copy text (standalone function) |
| `buildFullText(logData, liveBalances, ledgerStatus)` | Full Export copy text (standalone function) |
| `buildCustomerText(logData, liveBalances)` | Customer Summary copy text (standalone function) |
| `buildErrorText(logData)` | Error export with severity, actions, causes, help URLs |
| `inferRequiredApps(accounts)` | Maps account currencies → required device apps via `CURRENCY_TO_APP` |

---

## Brand Alignment — v1 (2026-04)

Full pass bringing the tool's visual language into alignment with Ledger's current brand identity. All changes are styling-only — data layer and logic are untouched.

**Color changes:**
- `T.orange = '#FF5300'` — Ledger Safety Orange. Used for: Customer Summary copy border, contextual help link hover, ErrCard help article link, drag-over accent on landing page drop zone.
- `T.error = '#E40046'` — Ledger brand crimson (was #F57375). Replaces all error red uses: `SEV.high`, `TC` error type, CSS `--error`, all `rgba(245,115,117,...)` alpha variants.

**Typography changes:**
- Body font: Brut Grotesque (300–700) embedded via Base64 @font-face (replaces previous web fonts).
- Structural labels: JetBrains Mono (`MF` constant) on all section headers, stat card labels, badge text, sidebar keys, device info row keys, filter/legend chips.
- `.stat-value` letter-spacing: `-0.02em`.

**Ambient gradient system:** Brand purple `#45395C` radiates from top-left across 8 surfaces at opacity tiers 0.04–0.35. See Design tokens for full tier table.

**Surface physics:** ErrCards and AcctCards use left-edge inset glow (not background gradients) to convey severity/chain color. Background gradients don't work visually at card widths.

**Other polish:**
- Root viewport: radial vignette (top-left, brand purple, 0.06 opacity).
- Top bar: horizontal gradient (left→right, 0.15).
- Fixed section headers: vertical gradient (top→bottom, 0.10).
- Stat cards: top-edge highlight via `borderTop`.
- Device card: 135deg diagonal gradient (0.10).
- Empty states, parsing screen, network rows, portfolio overlay: all receive ambient gradient treatment.

---

## Live Version Checking (Phase 1)

On log load, the tool fetches the current app catalog from Ledger's Manager API and compares against the customer's installed/required apps.

**API:** `GET https://manager.api.live.ledger.com/api/v2/apps/by-target?target_id={tid}&firmware_version_name={fw}&device_type={model}&livecommonversion={lcv}&provider=1`

**Required from log:** `dev.targetId` (full 32-bit, stored by D-7 path), `dev.fw`, `dev.modelId`. `livecommonversion` extracted from Manager API URLs in log entries.

**State:** `versionCheck` — `{status:'ok'|'loading'|'error', apps:[{name, installed, latest, outdated}], catalogSize}`

**Data layer extension:** `extractDevice` now stores `info.targetId` (full numeric target_id, not just the bitmask). Post-D-7 fallback scan checks URL params, `data.targetId`, and `data.data.target_id`.

**App chip visual system (manager-result path only):**
- Green `✓` = installed, up to date (live-confirmed when available)
- Red `⬆` = outdated, shows version path `⬆ Bitcoin v2.4.5→v2.4.6`
- Gray `✕` = missing (not installed)
- Tooltip on red chips: "Installed vX → latest vY (update available)"
- Live data overrides log-derived status when `versionCheck.status==='ok'`

**Header:** "DEVICE APPS" label left. Right badge (one of):
- Green `LIVE` = catalog fetched, chips reflect current data
- `CHECKING` = fetch in progress
- `OFFLINE` = fetch failed, log-derived status shown
- Fallback count ("8/10 up to date") only when `versionCheck` is null

**Advice box:** Red-tinted, shows specific counts ("2 outdated, 1 missing"). Disappears if live data confirms everything current.

**Others button:** `+ N others ✓` (muted) normally; turns crimson `+ N others (M ⬆)` when hidden apps are outdated.

**Copy reports:** `buildSummaryText` and `buildFullText` accept `versionCheck` as 4th parameter. Include VERSION CHECK section with outdated apps when available.

**Phases remaining:**
- Phase 2: Firmware outdated detection (POST /api/v2/get_latest_firmware)
- Phase 3: Ledger Wallet version check (GitHub releases API)

---

## Spec files in repo

| File | Status | Purpose |
|---|---|---|
| `CLAUDE.md` | **Active — primary reference** | This file. Claude Code reads this before every task. |
| `SESSION_HANDOFF.md` | **Active — session memory** | Context for new Claude instances picking up the project. |
| `ROADMAP_LIVE_BALANCES.md` | **Implemented (Phase 1+2)** | Phase 3 (app.json comparison) and Phase 4 (polish) pending rethink. |
| `REDESIGN_VISION_V5.md` | Historical reference | Core principles still apply. |
| `VIEWPORT_REDESIGN_SCOPE.md` | Historical reference | Foundation implemented, layouts evolved. |
| `agent-guide.html` | **Active** | Agent-facing investigation guide. Updated to v4.2. |
| `technical-reference.html` | **Active** | Technical capabilities reference. Updated to v4.2. |
| `ROADMAP_LIVE_VERSIONS.md` | **Active — Phase 1 implemented** | Phase 2 (firmware) and Phase 3 (Ledger Wallet) pending. |

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
| `post-timeline-issues` | — | Timeline strip, Issues interactive filters, breadcrumbs, guide overlays. |
| `post-helpcenter-integration` | — | Help center Layers 1-3, doc updates, copy report relocation. |
| `post-brand-alignment-v1` | `3913328` | **Tag after brand alignment session.** Safety Orange, brand crimson, ambient gradient system, surface physics, typography pass. |
| `post-version-check-p1` | — | Live version checking Phase 1, Brut Grotesque font, visual unification sweep, Customer View unification, app chip redesign. |
