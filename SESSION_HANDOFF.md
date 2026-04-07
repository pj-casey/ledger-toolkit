# Session Handoff — April 6, 2026 (Session 10)

## What was built this session

### Staking Awareness — False Positive Fix + Context UI

Fixed false CHANGED alerts in CVAgentInsights caused by comparing total balance (including staked) vs live RPC data (liquid-only). Affected 18 of 96 accounts — worst case was SOL showing "-276 SOL missing" when funds were safely staked.

**Core fix (CVAgentInsights findings engine):**
- Added `custSpendable` (from `ajSpendable`), `hasStaking`, `stakedAmount`, `compareBalance`
- Comparison now uses `compareBalance = hasStaking ? custSpendable : cust` — live RPC vs liquid-only
- `deltaFiat` uses `custSpendFiatVal` (spendable-side fiat) so delta remains price-neutral
- Health dots strip uses same spendable-vs-live logic to avoid false amber dots

**Staking context UI added in 3 places:**
1. Agent Insights detail panel: 3-column card (Total / Staked / Available) before comparison bars. Chain-specific labels: "Network reserve" (XRP/XLM), "Frozen (bandwidth/energy)" (TRX), "Staked / Delegated" (others). SOL note about separate stake accounts.
2. Customer View account detail: Staked/frozen inline display (🥩 label + staked amount + %) below balance when `ajBalance ≠ ajSpendable`.
3. Findings list: 🥩 icon next to account name when finding has `hasStaking`.

---

### CVEarn — New Earn Tab in Customer View

New `CVEarn` component added as 4th Customer View tab ("Earn"). Shows staking positions and LST holdings — all from app.json, no live API calls.

**Module-scope constants added:**
- `STAKING_APY` — per-chain APY ranges (SOL, ATOM, DOT, MATIC, NEAR, ADA, TRX, XTZ, ETH)
- `STAKING_TYPE` — per-chain delegation label
- `LST_PATTERNS` — 8 liquid staking token patterns (stETH, cbETH, JITOSOL, LBTC, etc.)
- `EARN_OP_LABELS` — DELEGATE/UNDELEGATE/REWARD/etc. → human labels

**Component features:**
- Summary: total staked fiat + total LST fiat, allocation bar
- "My rewards" tab: staked positions from `ajBalance - ajSpendable`, expandable to show validators (Ledger badge for figment/ledger validators), recent rewards history
- "Earn opportunities" tab: chains with native staking available (not yet staked), sorted by APY
- Liquid staking tokens section: from `ajSubAccounts` pattern matching
- Empty state when no positions and no LSTs

---

### React Hook Crash Fix

`expandedTokenOps` useState was declared inside an IIFE in CustomerView's render body — Rules of Hooks violation causing crash on account click.

**Fix:** Moved declaration to top of `CustomerView` function alongside other state. Deleted the misplaced `React.useState` call from inside the IIFE.

---

### Customer View — Final Visual Polish

Three rendering improvements to match Ledger Wallet's UI language.

**Change 1 — Accounts list grid:**
- Layout: `gridTemplateColumns:'52px 1fr auto auto auto auto'` (icon | chain+name | crypto | fiat | % change | star)
- Icon: 40px circle with chain brand icon
- Chain name uppercase + account name below; DM badge + ✓ checkmark in header
- Crypto + fiat right-aligned in `MF` font; star column (★ gold / ☆ dim)

**Change 2 — Operation detail grid:**
- `renderOp` rewritten: `gridTemplateColumns:'40px 1fr auto'` (icon | label+detail | amount)
- Label ("Sent"/"Received") white + 600 weight, prominent
- Amount right-aligned with +/- sign prefix in `MF` font; fiat below in muted version of amount color
- Counterparty on own line below label; tx hash link integrated
- Commit: `1616022`

**Change 3 — Portfolio allocation bar:**
- `gridTemplateColumns` updated: `'2fr 1.2fr 0.8fr 1.5fr 1.5fr'` → `'2fr 1fr 1.5fr 1.5fr 1fr'`
- Allocation column: percentage + inline progress bar (chain color, 4px height, 80px max width, rounded)
- Commit: `1616022`

---

## Current file state

~7,700 lines. One file: `ledger-toolkit.html`.

## Current git state

Branch: `main`. Latest commit: `1616022` (visual polish). 11 commits ahead of remote.

Push workflow: sandbox blocks HTTPS push — use `! git push origin main` directly from prompt.

---

## Key decisions made (session 10)

- **Spendable as comparison baseline.** CVAgentInsights uses `ajSpendable` (not `ajBalance`) when staking detected. Live RPC only returns liquid balance — comparing against total would always flag staked funds as "missing".
- **Earn tab data-only.** CVEarn makes no live API calls — all data from app.json `ajBalance`/`ajSpendable`/`ajSubAccounts`. Prevents API dependency and load time.
- **Hook fix pattern.** All useState in CustomerView must be at top-level scope, never inside IIFEs or conditional branches. IIFE pattern for conditional rendering is fine — hooks must exit before the IIFE.

---

## Backlog (carried forward)

### Design language
1. **CopyBtn animation** — still too subtle. Needs icon color change + scale on copy.
2. **Guide labels completeness** — Advanced, Network, APDU, Raw JSON have no purpose labels yet.

### Tab-specific work
3. **Accounts tab brand alignment** — prompt `accounts-brand-alignment.md` written, not yet run. 7 fixes.
4. **Network/APDU/Raw JSON tab brand alignment** — not started.

### Customer View
5. **Agent Insights + Earn documentation** — guide overlays not yet updated to cover these tabs.
6. **Agent Insights: real-data testing** — needs testing with actual app.json exports.

### Other
7. **Responsiveness audit** — click-outside handlers may still have stopPropagation issues.
8. **Test with real logs** — especially firmware/desktop version checks end-to-end.
9. **Focus Mode: treemap block label on hover** — consider a persistent ticker label inside each block.

---

# Session Handoff — April 6, 2026 (Session 9)

## What was built this session

### CVAgentInsights — Fixed-Viewport Master-Detail Dashboard

Complete overhaul of the `CVAgentInsights` component inside `CustomerView`. Six commits (`1e6af71` through `0ebbee5`).

#### Architecture

- **Fixed-viewport master-detail:** Left panel (findings list, 38%) + right panel (detail, 62%). Outer wrapper uses `flex:1,minHeight:0,overflow:'hidden'` — never scrolls.
- **Flex containment for `.cv-content`:** Agent Insights is the only CV section needing exact viewport fill. When `cvSection==='agentInsights'`, `.cv-content` becomes a flex column with `overflowY:'hidden'`; the app.json banner is wrapped in `flexShrink:0`; the component uses `flex:1,minHeight:0`. Other CV sections unaffected.
- **Unified findings engine:** Single `useMemo` produces per-account AND system findings, sorted red→amber→green. System findings all include `kind:'system'`; account findings include `kind:'account'`.

#### Key features

- **Health dots strip (Row 2.5):** Per-account colored dots in the stat bar row — visual quick-scan of account status. Shows snapshot date label when available.
- **Comparison bars (detail panel):** 3-column grid: "Customer sees (YYYY-MM)" / "On-chain now" / "Δ Delta". Rate label ghost text on both fiat columns for transparency.
- **Portfolio donut (stat card):** SVG arc donut showing filled/unfilled portfolio coverage; secondary `liveMarketFiat` ghost line when CoinGecko rate differs from customer rate by >$1.
- **Scrollbar suppression:** Left panel uses `scrollbarWidth:'none'` — scrolling works if needed, track hidden when not.

#### Balance comparison logic (consistent pricing)

**Problem:** `liveFiat` was using CoinGecko rates while `custFiat` used the customer's cached rates. The delta conflated crypto balance change with price movement — even a matching balance would show a non-zero delta if prices moved.

**Fix:**
- `liveFiatAtCustRate = live * custRate.rate` — on-chain balance valued at customer's cached rate
- `liveMarketFiat = lbEntry.fiat` — CoinGecko value, shown as secondary ghost text only
- `deltaFiat = liveFiatAtCustRate - custFiatVal` — pure crypto balance signal, price-neutral
- Same fix applied to `onchainTotal` (portfolio delta): uses `cvLatestRate(appJson, cid)` not CoinGecko
- `onchainMarketTotal` preserved as separate CoinGecko aggregate for "at market" ghost line

#### Time context for snapshot vs live

- `snapshotDate` derived from latest date key in app.json rate map (`/^\d{4}-\d{2}/` pattern)
- `daysSinceSnapshot` computed from snapshot date to today
- Detail text: *"The data is N days old — differences may be from normal transactions since the export."*
- Action text: *"Ask the customer if they made transactions since the export. If not, this is a sync issue…"*
- Badge relabeled: `'MISMATCH'` → `'CHANGED'`; title: `Balance mismatch` → `Balance changed`
- Snapshot date shown in "Customer sees" column header and health dots strip

#### New helpers used
- `cvLatestRate(appJson, cid)` — `{rate, date}` for currency from customer's cached rates
- `cvGetRateMap(appJson)` — full rate map for date extraction
- `cvFiatValue(appJson, cid, rawBalance)` — fiat from customer's cached rates
- `cvFmtFiat(value, appJson)` — format fiat string

---

### Documentation Sync (Session 8)

Both embedded guide constants (`GUIDE_AGENT`, `GUIDE_TECHNICAL`) and standalone files (`agent-guide.html`, `technical-reference.html`) updated to reflect the visual overhaul work done in sessions 7–8.

**Agent Guide changes:**
- Timeline "What you'll see" description now names all TC colors in sequence: "gray startup, purple analytics, gold price fetching, orange sync burst, blue network calls, red errors at the end."
- Health tiles: updated to mention chain brand icons with ticker-text fallback.
- Focus Mode: added EVM precision paragraph — for accounts sharing an address (Ethereum + L2s), focus is chain-specific, not address-wide.
- Copy report color description corrected: orange for customer (`#FF5300`), red for errors (`#E40046`).

**Technical Reference changes:**
- Event Classification: added full TC 9-entry color table with hex values, bold color preview, and brief descriptions.
- Session Visualization: added visual polish paragraph (rounded bars, ambient gradient, tinted chips in strip header).
- Account Health Visualization: added chain brand icons mention (CDN-backed with ticker fallback).
- Issues strip: TC type colors at 35% opacity for context segments + red glow for error segments.
- New "About This Tool" section: architecture overview (React 18.3.1 + Babel, three-tier parser, 82-pattern ERR_DB, 60-chain registry, live on-chain verification, zero data transmission, fixed viewport).
- TOC entry added for "About This Tool".

---

### Prior session work (covered by session 7 handoff + summarized session)

Changes from the session preceding this handoff (separate summarized conversation):

**Font system swap:** Brut Grotesque (base64-embedded, ~194KB) → Darker Grotesque (CDN). Total bundle ~540KB, down from ~700KB. No offline capability change — CDN required for both fonts now (JetBrains Mono was already CDN).

**Chain color corrections** (`CHAINS` constant): fixed visibility and brand accuracy across multiple chains. Full list in commit `375b9b5`.

**Chain brand icons:** New `ICON_CDN` constant (jsDelivr/simplr-sh/coin-logos) + `chainIconUrl(id)` helper using `COINGECKO_IDS` mapping. Applied to Accounts health tiles and treemap blocks. Ticker-text fallback when icon unavailable. Refactored in `8d1af1c` to use `COINGECKO_IDS` instead of a separate `ICON_MAP`.

**TC bold palette:** 9 distinct colors with hex values, replacing previous muted palette. action (#9B9B9B), analytics (#C084FC), countervalues (#FBBF24), bridge (#FF8C42), network (#60A5FA), persistence (#4ADE80), walletsync (#D4A0FF), error (#FF4D6D), live-dmk-logger (#FF5300). Consistent across Timeline bars, legend chips, row badges, Issues strip.

**Timeline strip visual polish:** rounded bar segments, ambient gradient overlay, tinted type-count chips in strip header area.

**Issues strip:** TC type colors at 35% opacity for context segments; error segments use `T.error` with subtle red glow. Same `selectPulse` animation on selected marker.

**Focus Mode EVM fix:** `focusedAcct` comparison changed from address-only to `addr+cid` composite key. Ethereum and Polygon can share a wallet address — focus now targets the specific chain, not all chains at that address.

---

## Current file state

~7,600 lines. One file: `ledger-toolkit.html`.

## Current git state

Branch: `main`. Latest commit: `0ebbee5` (balance comparison logic fix). 8 commits ahead of remote (pre-commit hook blocks sandbox push — use `! git push origin main` directly).

Push workflow: sandbox blocks HTTPS push — use `! git push origin main` directly from prompt.

---

## Key decisions made (session 9)

- **Consistent pricing for delta.** `deltaFiat = live_balance × customer_cached_rate − customer_fiat`. This makes delta purely a crypto balance signal. CoinGecko rate preserved as secondary ghost text only.
- **Snapshot date from rate map.** App.json cached rates use date keys (`YYYY-MM`). Latest date = when customer last synced prices. Used as `snapshotDate` for time context in mismatch detail.
- **Flex containment per-section.** Agent Insights needs exact viewport fill; other CV sections scroll normally. Override `.cv-content` layout only when `cvSection==='agentInsights'` — zero impact on other sections.

---

## Backlog (carried forward)

### Design language
1. **CopyBtn animation** — still too subtle. Needs icon color change + scale on copy.
2. **Guide labels completeness** — Advanced, Network, APDU, Raw JSON have no purpose labels yet.

### Tab-specific work
3. **Accounts tab brand alignment** — prompt `accounts-brand-alignment.md` written, not yet run. 7 fixes.
4. **Network/APDU/Raw JSON tab brand alignment** — not started.

### Customer View
5. **Agent Insights documentation** — guide overlays not yet updated to cover Agent Insights tab.
6. **Agent Insights: real-data testing** — needs testing with actual app.json exports to verify snapshot date extraction and rate matching.

### Other
7. **Responsiveness audit** — click-outside handlers may still have stopPropagation issues.
8. **Test with real logs** — especially firmware/desktop version checks end-to-end.
9. **Focus Mode: treemap block label on hover** — currently popover only. Consider a persistent ticker label inside each block.

---

# Session Handoff — April 5, 2026 (Session 7)

## What was built this session

### Boot Splash — Crossfade + Cypherpunk Recession

Full redesign of the boot sequence to eliminate the black-screen gap between splash and React app.

#### Architecture change
- **`#splash` no longer has a background.** It's a pure flex positioning container (`position:fixed;inset:0;z-index:9999`).
- **`#splash-bg`** is a new `position:absolute;inset:0;background:#131214` child. On dismiss it fades with `transition:opacity 0.5s ease` at 0ms delay — React content becomes visible through it while splash content is still partially visible. True crossfade, not a fade-to-black.
- **`#splash-center` contracts** on dismiss: `scale(0.98)` + `blur(6px)` — the terminal recedes while the React interface sharpens into focus underneath. Depth, not flatness.
- **`#splash` itself has no `opacity` transition** — only `pointer-events:none`. Individual children handle their own fading. This prevents the "all opacities multiply → black dip" problem.

#### Dismiss sequence timing
| Element | Behavior | Duration |
|---|---|---|
| `#splash-bg` | Fades out | 0.5s, no delay |
| `#splash-log` | Fades out | 0.25s |
| `#splash-glow` | Fades out | 0.3s |
| `#splash-logo` | Fades out | 0.4s |
| `#splash-center` | scale(0.98) + blur(6px) | 0.6s / 0.5s |
| `#splash-header` | Fades in (0→1) | 0.3s, 0.05s delay |
| DOM removal | `splash.remove()` | 1200ms after dismiss |

#### HTML structure
```html
<div id="splash">
  <div id="splash-bg"></div>        <!-- separate bg layer -->
  <div id="splash-header">...</div> <!-- fades in on dismiss to bridge React header -->
  <div id="splash-center">
    <div id="splash-glow"></div>
    <svg id="splash-logo">...</svg>
    <div id="splash-log"></div>     <!-- telemetry, centered on logo -->
  </div>
</div>
```

#### Polish details carried from prior work
- Logo entrance: `scale(0.92→1)` with `cubic-bezier(0.16,1,0.3,1)` (overshoot-free deceleration)
- Glow breathes: `splashBreathe` keyframe (scale 1→1.15, opacity 1→0.7, 3s infinite)
- Telemetry lines slide in: `splashLineIn` (translateY 4px→0, 0.2s each)
- "ready" line in brand purple `#BBB0FF`
- Telemetry positioned `translate(-50%,-50%)` — centered on the logo, not offset

#### Key decisions
- **Separate bg layer** is the minimal correct solution. Trying to crossfade `#splash` as a whole fails because both layers are the same dark color — you need to fade the background independently of the content.
- **Recession effect** (`scale(0.98)+blur`) makes the transition feel like depth/layer change rather than a simple cut. The splash doesn't just disappear — it recedes while the interface emerges in front.
- **Header fade-in** bridges the gap: splash header fades from 0→1 exactly as React's header becomes visible, creating continuity.

---

## Previous session (Session 6)

## What was built this session

### Focus Mode — Full Redesign

#### Activation
- **Treemap block click** (regular click, not Shift+click) activates Focus Mode by setting `focusedAcct`. Clicking the focused block again clears it.
- Previously, Focus Mode required Shift+click on health tiles. Now the treemap is the primary activation surface.

#### Ambient Color System
When `focusedAcct` is set, the entire tool shifts to the chain's color at varying opacities:

| Surface | Effect |
|---|---|
| Root vignette | Chain color at `06` opacity (replaces brand purple) |
| Header border | `2px solid ${color}40` (was `1px solid ${T.border}`) |
| Sidebar gradient | `${color}12` fading at 50% (was purple `rgba(69,57,92,0.12)`) |
| Content area | Diagonal gradient (06→02→transparent) + `focusEngage` flash |
| Non-focused account cards | `opacity:0.12`, `I.medium` transition |
| Non-focused treemap blocks | `opacity:0.12` |
| Non-focused Issues strip bars | `opacity:0.12` |
| Non-focused Overview error tiles | `opacity:0.12` |
| Non-focused sidebar nav rows | `opacity:0.15` |

All background transitions use `transition:'background 0.4s ease'`.

#### Focus Indicator — Single Header Chip
Reduced from 4 simultaneous signals (sidebar pill, Context Ribbon chip, button label, content border) to **one persistent chip in the header bar**: colored dot + ticker + ✕ dismiss. The chip sits inline after the session date/time span. Dimming (0.12 opacity) does the visual work.

#### Sidebar Investigation Panel
When focus is active, the normal overview block is replaced by an investigation panel with a chain-colored gradient background:
- Chain identity: name + colored dot + ✕ dismiss
- Abbreviated address + CopyBtn
- Metrics row: balance, fiat, op count, error count
- Quick-action buttons: "View errors" (sets `errAcctFilter` + navigates to Issues), "View Timeline" (sets `tlAcctFilter` + navigates to Timeline), "Explorer" (chain explorer URL)

`DiagSidebar` receives two new props for this: `setErrAcctFilter` and `setTlAcctFilter`.

#### `focusEngage` keyframe (new)
```css
@keyframes focusEngage{
  0%{background-color:var(--focus-color,rgba(187,176,255,0.08));}
  100%{background-color:transparent;}
}
```
Applied to content area and investigation panel on focus activation. `--focus-color` = `${focusedAcct.color}12`.

---

### Header — Session Date/Time
The header now shows the log's session date/time span and duration inline (e.g., `2026-04-05 09:12–09:47 · 35.2m`). This moved from the removed Zone 3 footer. Visible on all tabs.

---

### Overview — Zone 3 Removed
The bottom `~50px` footer bar ("API status pills") on Overview was removed. Its contents were split:
- **Session date/time** → header bar (all tabs)
- **API status** → sidebar bottom vertical list (all tabs)

Dead vars `hasSession`, `chainCounts`, `topChains` removed.

---

### Sidebar — Vertical API Status List
Replaced the old horizontal pill row with a readable vertical list at the sidebar bottom. Always visible regardless of tab. Five rows: Manager API, GitHub, Balances, Prices, Ledger Status. Each row: 5px dot + label + status text (live/···/—). Opacity reflects confidence.

---

### Context Ribbon — Errors Only
Context Ribbon now only appears when `selectedErr != null && section === 'errors'`. It no longer shows a `focusedAcct` chip — that's handled by the header chip. The ribbon's ✕ clears `selectedErr` only.

---

### Treemap Popover — Debounce Fix
The popover disappeared when the cursor moved from a block to the popover itself (the block's `onMouseLeave` fired immediately). Fixed with:
- 150ms debounce via `hoverTimeoutRef = React.useRef(null)`
- Block `onMouseLeave` → `setTimeout(() => setHoveredAcct(null), 150)` (stored in ref)
- Block `onMouseEnter` → `clearTimeout(hoverTimeoutRef.current)` before setting state
- Popover div has matching `onMouseEnter` (clear timeout) + `onMouseLeave` (clear immediately)
- Popover is `position:fixed` to escape `overflow:hidden` containers

---

### Quick Summary Copy Button
`borderLeftColor` changed from `'#BBB0FF'` (purple) to `T.success` (green). Full Export keeps purple. This makes Quick Summary visually distinct — "go" action vs archive action.

---

### Bug Fixes

**Blank screen — `firmwareCheck` null access:** `triageBanner`'s `fwOutdated` branch accessed `firmwareCheck.current`/`.latest` when `firmwareCheck` could be null (log-derived path). Fixed by gating on `fwLive` boolean before accessing.

**Blank screen — adjacent JSX without fragment:** Error Flow conditional rendered two sibling `<div>` elements without a parent. Babel rejects adjacent JSX. Fixed by wrapping in `<>...</>`.

---

## Current file state

~6,400 lines. One file: `ledger-toolkit.html`.

## Current git state

Branch: `main`. Latest commit: `9b88624` (crossfade splash). Pushed to remote.

Push workflow: sandbox blocks HTTPS push — Claude must use `dangerouslyDisableSandbox:true` on the push Bash call, or user runs `! git push` directly.

---

## DiagSidebar props (updated)

```javascript
function DiagSidebar({ logData, section, setSection, liveBalances, focusedAcct, setFocusedAcct, versionCheck, firmwareCheck, appVersionCheck, ledgerStatus, setErrAcctFilter, setTlAcctFilter })
```

---

## Key decisions made this session

- **Single focus signal.** Four simultaneous indicators (sidebar pill, ribbon chip, button label, border) created visual noise without adding information. One header chip does the job — permanent, dismissible, low weight.
- **Treemap = focus activation.** The treemap is proportional fiat — you click what you care about most. Moving activation from Shift+click tiles to regular click treemap makes focus feel discoverable rather than hidden.
- **0.12 dimming, not 0.25.** 0.25 was too aggressive — made the tool feel broken. 0.12 recedes the background without hiding it.
- **Context Ribbon is for errors, not focus.** Two kinds of "selected thing" at once (a focused account + a selected error) would create a cluttered ribbon. Focus has its own ambient language now; the ribbon stays clean for error context.
- **Sidebar investigation panel.** When you're focused on an account, the sidebar diagnosis sentence is irrelevant. The panel replaces it with account-specific data and actions — same real estate, much higher signal.
- **API status in sidebar, not footer.** Agents check API status to understand data confidence, not as a primary task. Sidebar bottom is always-visible without stealing content space.
- **150ms debounce for popover.** Just long enough that normal cursor movement through the block doesn't close it, but instant enough to feel responsive when you actually leave.

---

## Backlog (carried forward)

### Design language
1. **CopyBtn animation** — still too subtle. Needs icon color change + scale on copy.
2. **Guide labels completeness** — Advanced, Network, APDU, Raw JSON have no purpose labels yet.

### Tab-specific work
3. **Accounts tab brand alignment** — prompt `accounts-brand-alignment.md` written, not yet run. 7 fixes.
4. **Network/APDU/Raw JSON tab brand alignment** — not started.
5. **Customer View overhaul** — backburner.

### Documentation
6. **Agent Guide + Technical Reference** — stale since Session 2, need entries for Focus Mode, design language, version checking.

### Other
7. **Responsiveness audit** — click-outside handlers may still have stopPropagation issues.
8. **Test with real logs** — especially firmware/desktop version checks end-to-end.
9. **Focus Mode: treemap block label on hover** — currently popover only. Consider a persistent ticker label inside each block.
