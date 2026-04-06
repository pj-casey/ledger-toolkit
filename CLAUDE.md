# Ledger Diagnostic Toolkit — Claude Code Guide

Single-file React/Babel app (`ledger-toolkit.html`) for CS agents to diagnose customer issues from Ledger Wallet log exports.

**Branch:** `main`
**Design references:** REDESIGN_VISION_V5.md, VIEWPORT_REDESIGN_SCOPE.md, ROADMAP_LIVE_BALANCES.md

> **Maintenance rule:** This file describes *what the tool IS now* — current state only. When adding documentation for new features, compress or remove one existing section to stay under 40k chars. History and decisions belong in SESSION_HANDOFF.md.

---

## Architecture

- **ONE file**: `ledger-toolkit.html` (~6,300 lines)
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

**Diagnostic Mode:** Fixed-viewport dashboard. Sidebar navigation + main content area. Device status line + stat cards (Issues, Accounts, Network, Log Quality) on Overview. Unified Device card (app chips, merged environment + reference popover). Firmware update status. Live balances + fiat values on AcctCards. Help center article links on errors. Timeline session strip with stacked color-coded bars and interactive legend chips. Issues with interactive severity/category chips, error-prominent strip, and breadcrumbs. Copy report dropdown (Summary/Full/Customer/Errors) on Overview action bar. Focus Mode: click any account tile or treemap block to enter focus — entire tool shifts chain color, sidebar becomes investigation panel, non-focused accounts dim.

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
**Version Checking:** All fetch/compare logic for app catalog, firmware, and desktop app checks

## MAY EXTEND (for new log format support)

These were extended to support LLv4 (Ledger Wallet 4.0) logs. Extensions are additive only — old log formats still parse correctly.

**`extractDevice`** — added D-7 (target ID masking), D-5 (Manager API name), D-6 (Gen5 firmware prefix). Added `userAgent` extraction and `appBrand` derivation.
**`extractAccounts`** — added LLv4 bridge sync fallback (sets `dur` from `SyncSession finished` when no SyncSuccess analytics events exist).
**`extractDeviceApps`** — added tertiary `catalog-only` source (hw install events + apps-by-target catalog).

## CAN MODIFY (rendering/styling)

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

## Device identification

`extractDevice` uses 7 detection paths (D-1 through D-7), stopping when `modelId` and `fw` are both found. Primary LLv4 path is D-7: target ID masking via `(targetId & 0xFFFF0000)` against `TARGET_MASKS`. Also stores `info.targetId` (full numeric) for Manager API calls. Post-D-7 fallback scan sets `targetId` from URL params, `data.targetId`, or `data.data.target_id` even when model was identified via a different path. Path D-6 covers `apex_p`/`apex` for Gen5.

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
- Row 3: Error-prominent session strip (80px). Uniform grey (#4A4A4A) for context activity, T.error for error segments (simplified from full TC colors — Timeline keeps the full palette). Hover: `errStripHover` state → tooltip with time, event count, error titles. Click: selects error in master-detail. Severity/category chip hover dims non-matching error segments. Selected error: vertical line + triangle marker at error's timestamp on strip, pulses with `selectPulse` animation. Both strip marker AND selected error row in the list pulse the same severity color simultaneously.
- Strip labels: "SESSION · CLICK TO NAVIGATE" top-right (MF font, muted). "N errors highlighted" bottom-left when errors present.
- Strip legend: Two-item legend below strip — grey "ACTIVITY" + red "ERRORS".
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
- Two-panel layout: **health tiles** (left, `position:relative` wrapper) + **portfolio treemap** (right, proportional fiat-sized blocks):
  - Health tiles: ticker label, chain color, opacity by status, corner dots (red=errors, purple=xpub, ?=unsupported). Click = `acctFilter` toggle. Hairline `outline` border (Focus Mode uses separate border state so `outline` avoids clobbering it).
  - Treemap blocks: proportional fiat area. **Regular click** sets `focusedAcct` (activates Focus Mode); click focused block clears focus. Hover shows fixed-position popover (debounced 150ms via `hoverTimeoutRef`). Popover has `onMouseEnter`/`onMouseLeave` to prevent disappearing when cursor moves from block to popover.
  - `acctMapOpen` opens the portfolio overlay (separate from inline treemap).
- Filter input (`acctFilter`)

**Scrollable panel (flex:1, overflowY:auto):**
- EVM grouping: same-address accounts grouped, ETH first. "SHARED ADDRESS" header + CopyBtn + chain count + group fiat.
- `AcctCard` props: `acct, errCount, deviceApps, liveBalance, versionCheck, onErrClick, onTimelineClick`
- **App status badges** (plain language): "Missing" (red) when required apps not installed. "Outdated · N errors" (red) when outdated + has errors. "Outdated" (amber) when outdated only. No badge when current (green suppressed to reduce noise).
- Error badge → sets `errAcctFilter` + navigates to Issues
- "Timeline" button → sets `tlAcctFilter` + navigates to Timeline
- Xpub scan button on collapsed UTXO cards
- Non-focused account cards dim to `opacity:0.12` when Focus Mode is active (`I.medium` transition)

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

**Header bar:** Shows file name + session date/time span + duration (e.g., `2026-04-05 09:12–09:47 · 35.2m`). When `focusedAcct` is set, a **focus chip** appears inline in the header: colored dot + ticker label + ✕ dismiss. Header `borderBottom` shifts from `1px solid ${T.border}` → `2px solid ${focusedAcct.color}40` when focus active.

**Context Ribbon:** Shown only when `selectedErr != null && section === 'errors'`. Displays the active error's severity color, title, and a ✕ Clear button that calls `setSelectedErr(null)`. Does NOT appear for `focusedAcct` (focus is indicated by the header chip and ambient color instead).

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

### CSS animations

| Keyframe | Duration | Used by |
|---|---|---|
| `focusPulse` | infinite | Accounts health tile focus mode breathing |
| `selectPulse` | `I.pulse` = 0.5s ease-out | Issues strip selected-error marker, selected error row glow, active filter chips, Timeline legend chips, expanded rows, stat card click |
| `confirmFlash` | 0.4s ease-out | CopyBtn green flash on successful copy |
| `statFadeIn` | 0.3s | Stat card value entrance |
| `iconPulse` | 1.5s infinite | Drop zone icon breathing on drag hover |
| `stripDraw` | 0.8s ease-out | Timeline + Issues strips reveal left-to-right on tab enter (`clip-path: inset()`) |
| `listFade` | 0.3s ease-out | Timeline rows, Issues error list, Accounts list fade on filter change (React `key` remount) |
| `filterRemind` | 0.6s ease-out | Active filter chips glow when returning to a tab with a filter still active (`sectionChanged` 600ms window) |
| `focusEngage` | 0.4s ease-out | Content area + investigation panel flash to chain color on focus activation (`--focus-color` CSS var) |

### Hover patterns

Standardized hover behaviors (unified under the `I` interaction constants):
- **Data rows** (timeline, network, APDU, device info): `background → I.hover` (`rgba(255,255,255,0.04)`)
- **Cards** (ErrCard, AcctCard): existing hover classes (`.err-hover`, `.acct-hover`)
- **Ghost buttons** (outlined, transparent bg): `borderColor → T.primary, color → T.primary, background → I.hoverAccent` on hover
- **All transitions:** normalized to `I.fast` (150ms) or `I.medium` (250ms) — no hardcoded `transition` values

---

## Sidebar

240px fixed width. `DiagSidebar` component. Props: `logData`, `section`, `setSection`, `liveBalances`, `focusedAcct`, `setFocusedAcct`, `versionCheck`, `firmwareCheck`, `appVersionCheck`, `ledgerStatus`, `setErrAcctFilter`, `setTlAcctFilter`.

**Investigation panel (top, shown when `focusedAcct` is set):** Replaces the normal overview header block. Chain-colored gradient background. Shows: chain name + colored dot + ✕ dismiss, abbreviated address + CopyBtn, metrics row (balance/fiat/tx count/error count), quick-action buttons (View errors → sets `errAcctFilter` + navigates to Issues; View Timeline → sets `tlAcctFilter` + navigates to Timeline; Explorer → opens chain explorer in new tab). Animated in with `focusEngage` keyframe.

**Normal state:** Overview header (severity-colored left border), section list (Issues, Accounts expandable, Timeline, Advanced expandable).

**API status list (bottom, always visible):** Vertical list of 5 data sources. Each row: colored dot + label + live/···/— status. Sources: Manager API, GitHub, Balances, Prices, Ledger Status.

**Gradient treatment:** Shifts to chain-color gradient (`${focusedAcct.color}12` → `#1C1D1F` at 50%) when focus active. Normally: vertical purple `rgba(69,57,92,0.12)` gradient. The "Diagnosis" overview block uses a stronger `rgba(69,57,92,0.35)` gradient at 135deg.

---

## Customer View

Shares the same visual language as Diagnostic mode. CSS classes `.cv-sidebar`, `.cv-portfolio-card`, `.cv-detail-card`, `.cv-section-title`, `.cv-info-row`, `.cv-acct-row.cv-active`, `.cv-infobar` all use matching gradients, radii, and font treatments. CV sidebar header, stat card labels, info bar labels use `fontFamily:MF`. Portfolio cards have chain-colored left borders with inset glow (matching AcctCards). `.cv-layout` uses `height:calc(100vh - 52px)`.

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

**Copy report dropdown:** On Overview action bar (not sidebar). 4 options with color-coded left borders: Quick Summary (green `T.success`), Full Export (purple `T.primary`), Customer Summary (orange `T.orange`), Copy Errors (crimson `T.error`). Standalone builder functions at module scope.

---

## State variables

**App:** `logData`, `fileName`, `loadErr`, `section`, `searchTerm`, `typeFilter`, `expandedRow`, `dragOver`, `acctFilter`, `qualityOpen`, `otherAppsOpen`, `sumCopied`, `fullCopied`, `apduExportCopied`, `viewMode`, `appJson`, `parsing`, `fileKey`, `showMore`, `deviceExpanded`, `devDetailOpen`, `liveBalances`, `globalSearch`, `showGlobalResults`, `selectedErr`, `errSevFilter`, `errCatFilter`, `errPatternFilter`, `errHoveredSev`, `errHoveredCat`, `errStripHover`, `errAcctFilter`, `hoveredAcct`, `hoveredRef`, `acctMapOpen`, `tlHoveredBucket`, `tlHighlightType`, `tlAcctFilter`, `tlGrouping`, `expandedGroups`, `guideOpen`, `guideSearch`, `copyOpen`, `focusDropOpen`, `ledgerStatus`, `focusedAcct`, `versionCheck`, `firmwareCheck`, `desktopCheck`
**Refs:** `hoverTimeoutRef` (treemap popover debounce — 150ms delay before clearing `hoveredAcct`/`hoveredTmRect`)
**DiagSidebar:** `accountsOpen`, `advOpen`
**CustomerView:** `selectedAcct`, `summCopied`, `ajDragOver`
**JTTree:** `expanded`, `search`, `copiedPath`, `matchPaths`, `currentMatch`, `scrollContainerRef`
**ErrCard:** `detOpen`, `ctxOpen`

---

## Constants

| Constant | Count | Purpose |
|---|---|---|
| `T` | 11 | Theme colors (includes `card` elevation and `orange:#FF5300`) |
| `MF` | 1 | Monospace font stack constant |
| `I` | 11 | Interaction constants — timing (`fast`, `medium`, `slow`), hover backgrounds (`hover`, `hoverStrong`, `hoverAccent`), selection (`selectedBg`, `selectedBorder`), feedback (`confirmColor`, `pulse`), resting affordance (`interactiveBorder`) |
| `TC` | 9 | Log type badge colors — bold palette: action grey, analytics purple, countervalues amber, bridge orange, network blue, persistence green, walletsync pink, error crimson, live-dmk-logger purple |
| `DN` | 6 | Device model names (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `TARGET_MASKS` | 6 | Target ID → device model mapping (same as @ledgerhq/devices) |
| `CHAINS` | 60 | Chain registry + explorer URLs |
| `TX_EXPLORERS` | 45+ | Chain → tx hash explorer URLs |
| `DECIMALS` | 60+ | Currency → decimal places |
| `ERR_DB` | 82 | Error diagnosis patterns (33 with `u` → support article URL). Error color: `T.error` (`#E40046`). |
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
| `buildSummaryText(logData, liveBalances, ledgerStatus, versionCheck, firmwareCheck)` | Quick Summary copy text |
| `buildFullText(logData, liveBalances, ledgerStatus, versionCheck, firmwareCheck)` | Full Export copy text |
| `buildCustomerText(logData, liveBalances)` | Customer Summary copy text |
| `buildErrorText(logData)` | Error export with severity, actions, causes, help URLs |
| `inferRequiredApps(accounts)` | Maps account currencies → required device apps via `CURRENCY_TO_APP` |

---

## Live Version Checking (all phases)

Three independent checks run on log load. All fetch/compare logic is in the frozen data layer — only the UI rendering may be modified.

### Phase 1 — App Catalog

Fetches current app catalog from Manager API (`apps/by-target` endpoint). Requires `dev.targetId`, `dev.fw`, `dev.modelId`, `livecommonversion` (extracted from log, fallback `'1.0.0'`).

**State:** `versionCheck` — `{status:'ok'|'loading'|'error', apps:[{name, installed, latest, outdated}], catalogSize}`

**UI — App chips (manager-result path only):** Green `✓` = current. Red `⬆` = outdated with version path. Gray `✕` = missing. Live data overrides log-derived status. Header badge: LIVE / CHECKING / OFFLINE. Advice box shows counts when outdated/missing. Others button turns crimson when hidden apps are outdated.

### Phase 2 — Firmware

3-call chain to Manager API (`get_device_version` → `get_firmware_version` → `get_osu_version`). Optimizes by scanning log for cached responses first. Steps try `/api/` then `/api/v2/` fallback. `get_osu_version` returns 404 = current, 200 = outdated.

**State:** `firmwareCheck` — `{status:'ok'|'loading'|'error', current:string, latest:string|null, outdated:boolean}`

**UI:** Health pill green/red. Compact line: `FW x.x.x ✓` / `FW x.x.x ⬆ y.y.y` / `FW x.x.x …`. Red-tinted advice box when outdated. Overrides log-derived `fwLatest`/`fwOk` when available.

### Phase 3 — Desktop App

Fetches GitHub releases for `LedgerHQ/ledger-live`, filters for `@ledgerhq/live-desktop@X.Y.Z` tags.

**State:** `desktopCheck` — `{status:'ok'|'loading'|'error', current, latest, outdated}`

**UI:** Amber "UPDATE → X.Y.Z" badge when outdated. Green "LATEST" when current.

**Copy reports:** All three checks included in VERSION CHECK section of Summary and Full exports when data available.

---

## Design Language System

Four-layer system for consistent, legible interactivity across all tabs. All styling-only.

### Layer 1 — Clickable Signal

Every non-button interactive element has a hairline resting border (`I.interactiveBorder` = `rgba(255,255,255,0.06)`) so agents can distinguish clickable from static. Overview error tiles: hairline on top/right/bottom + 3px severity accent on left. Accounts health tiles: `outline` (avoids disrupting filter/focus border states). Timeline type badges: hairline + `cursor:pointer` + `onClick` → sets `typeFilter`. Issues strip: hairline (`borderTop` at 0.08). `cursor:pointer` on all clickable elements.

### Layer 2 — Guide Layer

Six `purposeLabel` annotations (10px JetBrains Mono, `T.muted`, uppercase, 0.06em tracking). Defined as `purposeLabel` constant inside `App()` after `sectionChanged`. Locations: Overview error grid, Accounts health tiles, Timeline strip, Timeline legend, Issues severity chips, Issues category chips. All use "· click to filter" or "· hover for detail" format.

### Layer 3 — Consistent Motion

All transitions normalized to `I` constants (defined at module scope, line ~189). Three timing tiers: `I.fast` (150ms), `I.medium` (250ms), `I.slow` (350ms). No hardcoded `transition` values remain. All `selectPulse` uses `I.pulse` (0.5s).

### Layer 4 — Delight Layer

Three animation behaviors: **`stripDraw`** — strips reveal left-to-right via `clip-path` on tab enter. **`listFade`** — content fades in on filter change via React `key` remount (key includes all active filter state). **`filterRemind`** — active filter chips glow on tab return via `sectionChanged` state (600ms window, `prevSectionRef` tracks previous tab).

---

## Focus Mode System

Ambient investigation mode that transforms the entire tool to center around one account. All styling-only.

### Activation

- **Treemap block click:** regular click sets `focusedAcct({addr, cid, ticker, name, color})`. Click again to clear.
- **Escape key:** clears `focusedAcct`.
- **Header chip ✕:** inline dismiss.

### Visual transformation (when `focusedAcct` is set)

| Surface | Effect |
|---|---|
| Root vignette | Shifts from brand purple to chain color at `06` opacity |
| Header border | `2px solid ${focusedAcct.color}40` |
| Sidebar gradient | `${focusedAcct.color}12` (top 50%) |
| Content area | Diagonal chain color gradient (06→02→transparent) + `focusEngage` flash |
| Non-focused cards/blocks/tiles/nav | `opacity:0.12` (nav rows: 0.15), `I.medium` transition |
| Header focus chip | Colored dot + ticker + ✕ |

All `transition:'background 0.4s ease'` on root and sidebar for smooth entry/exit.

### Sidebar investigation panel

When `focusedAcct` is set, the normal overview block is replaced by an investigation panel (chain-colored gradient background, `focusEngage` animation): chain identity + ✕, abbreviated address + CopyBtn, metrics (balance/fiat/ops/errors), quick actions (View errors → `setErrAcctFilter` + Issues, View Timeline → `setTlAcctFilter` + Timeline, Explorer → new tab).

---

## Git tags

Latest: `post-session6-focus` (`61c16af`) — Focus Mode, ambient color, sidebar investigation, header chip, API status sidebar, Zone 3 removal, popover debounce.

Full tag history is in SESSION_HANDOFF.md.
