# Fixed-Viewport Dashboard Redesign — Scope

*"Everything visible on landing. Scroll happens inside panels, never on the page."*

---

## The problem

The tool is built as a scrolling document. An agent lands on the Overview and sees four stat cards + activity badges. Everything else — device info, issues, session data, quality score — is below the fold. They have to scroll to discover it exists. Every view has this problem: the Timeline is a 2000-row scrolling page. Issues are a tall stack of ErrCards. Accounts are a tall stack of AcctCards.

## The principle

The browser window is the viewport. The outer shell (52px top bar + 240px sidebar + remaining main area) fills 100vh exactly. **The page never scrolls.** Each view fills the available main area space with a deliberate layout. Content that overflows scrolls inside clearly bounded panels with visible borders. The agent always knows: "this part of the screen is fixed, this part scrolls."

## The structural change (applies to ALL views)

**Line 2719** currently: `style={{flex:1,overflowY:'auto',padding:'24px'}}`

Changes to: `style={{flex:1,overflow:'hidden',padding:0,display:'flex',flexDirection:'column',height:'calc(100vh - 52px)'}}`

Each view rendered inside this container becomes a flex child that fills the available space. Views manage their own padding (24px on fixed headers, 0 on scrollable panels that go edge-to-edge). Each scrollable panel gets:
- `flex:1` (fills remaining height)
- `overflowY:'auto'`
- A visible top border (`1px solid var(--border)`) to visually separate it from the fixed header above
- Padding inside the scrollable area

---

## View-by-view spec

### Overview (Diagnosis Detail)

**Layout: 3 horizontal zones filling the viewport**

**Zone 1 — Top strip (fixed, ~140px):**
```
┌─────────────────────────────────────────────────────────────────┐
│ Device ✓ · FW 2.6.1 ✓ · Apps 8/10 ⚠ · Sync ✓ · 12 info       │  ← NEW health pills
├──────────┬──────────┬──────────┬────────────────────────────────┤
│  2028    │    8     │    —     │     12                          │  ← existing stat cards
│ Entries  │ Accounts │   Ops   │   Issues                        │
│          │          │         │                                  │
└──────────┴──────────┴──────────┴────────────────────────────────┘
  [Manager opened] [Swap browsed] [Device scan]                      ← existing activity badges
```

The health pills row is NEW — a row of compact colored pills. Each clickable to its section:
- `Device: Nano X ✓` (green) → clicks to Device section
- `FW: 2.6.1 ✓` (green) or `FW: 2.5.0 ⚠` (yellow) → clicks to Device
- `Apps: 8/10 ⚠` (yellow) or `Apps: 10/10 ✓` (green) or `Apps: ? no data` (gray) → clicks to Device
- `Sync: active ✓` (green) or `Sync: encrypted` (yellow) or `Sync: —` (gray) → clicks to Device
- `Errors: 12 info` (blue) or `Errors: 2 critical` (red) or `Errors: 0` (green) → clicks to Issues

Stat cards become slightly more compact (less padding) to save height. Activity badges stay below stat cards.

Copy Summary / Copy Full buttons move to the right side of the health pills row (same line, right-aligned). They're less prominent but always visible.

**Zone 2 — Middle content (fills remaining height, 2-column grid):**

```
┌────────────────────────────┬────────────────────────────────────┐
│                            │                                    │
│  🔌 Device Summary         │  ⚠ Issues Preview                 │
│  Nano X · FW 2.6.1 ✓      │  ┌──────────────────────────────┐  │
│  14 apps · Sync active     │  │ [ErrCard 1]                  │  │
│  View device →             │  │ [ErrCard 2]                ↕ │  │
│                            │  │ [ErrCard 3]                  │  │
│  Environment               │  │ ...scrolls if needed         │  │
│  OS     macOS 25.2.0       │  └──────────────────────────────┘  │
│  Platform  desktop         │  View all →                        │
│  User ID   ae5a36...       │  Consider: 2 errors ref ETH...    │
│                            │                                    │
└────────────────────────────┴────────────────────────────────────┘
```

Left column (~40% width): Device summary (existing clickable row) + Environment card (existing). Both compact, no internal scroll needed. These answer "who is this customer and what device do they have?"

Right column (~60% width): Issues preview. The header and "View all →" link are fixed. The ErrCards have an internal scrollable area if there are more than fit. Bordered panel. This answers "what's wrong?"

If no issues: right column shows the green "No errors detected" message and the guidance text.

**Zone 3 — Bottom strip (fixed, ~48px):**

```
┌─────────────────────────────────────────────────────────────────┐
│ Session: Apr 2 09:14–09:16 · 5.2s · 5/8 active │ Quality: 73 ◐ │
│ ETH ×2  XTZ ×2  ATOM ×1  TRX ×1  NEAR ×1       │ Good [Det ›] │
└─────────────────────────────────────────────────────────────────┘
```

Session info + Quality score combined into a single compact bottom bar. Session on the left (with chain pills), quality on the right (with NEW small SVG ring/arc and the existing expandable details toggle). The quality details expand UPWARD (like a popover) rather than pushing content down.

---

### Issues

**Fixed header (~100px):** Severity counts bar (Critical/Warning/Info) + NEW error distribution bar (thin CSS stacked bar showing hardware/software/server proportions) + diagnostic summary sentence + repeating patterns.

**Scrollable panel (remaining height):** Category-grouped ErrCards with full detail, expandability, jumpTo, linked accounts. Bordered panel with internal scroll.

The error distribution bar is NEW: a thin horizontal bar (~8px tall, full width) with three colored segments proportional to error categories. Below it, a muted legend: "3 hardware · 5 software · 4 server". Pure CSS (three divs in a flex row with background colors).

---

### Accounts

**Fixed header (~80px):** NEW account health visual (a row of small colored squares, one per account — funded = chain color, empty = gray, hover shows name) + funded/empty count + filter input + Copy IDs / Export buttons.

**Scrollable panel (remaining height):** AcctCards list. Bordered panel with internal scroll.

The account health visual is NEW: small colored squares (~20×20px) in a flex row. Each square's background is the chain's color if funded, or `T.border` if empty. Hover tooltip shows "Ethereum 1 · 12 ops" or "Tezos 2 · empty". Clicking a square scrolls to that AcctCard in the panel below.

---

### Device

**Fixed header (~200px):** Device & App card (model, firmware ✓/⚠, language, paired, Ledger Live, commit, Wallet Sync, MEV). This is the existing card, always visible.

**Scrollable panel (remaining height):** Device Apps section (summary + expandable list + other installed + storage) then APDU Exchanges section (count + rejections + rows). Both scroll in a single panel. Bordered.

---

### Timeline

**Fixed header (~70px):** NEW density strip (a thin bar showing event distribution over time — explained below) + search bar + type filter dropdown + export button + entry count.

**Scrollable panel (remaining height):** Timeline rows. Bordered panel with internal scroll.

The density strip is NEW: divides the session timespan into ~80 tiny vertical bars packed in a horizontal row (full width, ~24px tall). Each bar's height represents event count in that time bucket. Color: gray default, red if any errors in that bucket, purple if device events. The strip creates a "minimap" of the session — agents instantly see "errors clustered at 18:03:37" as a red spike. Pure CSS (array of tiny divs with calculated heights and background colors). No click interaction in v1 — purely visual.

---

### Network

**Fixed header (~50px):** Summary line: "453 requests · N errors (X server, Y client)" + any existing controls.

**Scrollable panel (remaining height):** Network entry rows. Bordered.

---

### Raw JSON

**Fixed header (~50px):** Search input + expand controls (Collapse / Level 1 / Level 2 / Level 3 / All) + entry count + Copy raw text button.

**Scrollable panel (remaining height):** JTTree content. Bordered.

---

## Scrollable panel visual treatment

Every scrollable panel in every view uses the same visual pattern so agents learn it once:
- Top border: `1px solid var(--border)` separating it from the fixed header
- A subtle inner shadow at the top (`box-shadow: inset 0 2px 8px rgba(0,0,0,0.15)`) indicating "content behind here"
- When scrolled, the top shadow persists, reinforcing that there's content above
- Padding: `16px 24px` inside the scrollable area
- Scrollbar: the existing thin 6px scrollbar style

This is a CSS pattern, applied consistently. An agent sees a bordered panel with a slight shadow and immediately understands "this part scrolls."

---

## Sidebar enhancement

Add a small colored status dot to the Device row in the sidebar:
- 🟢 Green (6px): firmware current + all required apps installed and up to date
- 🟡 Yellow (6px): something outdated (firmware or app)
- 🔴 Red (6px): required app not installed
- No dot: no device data available

This dot sits to the right of the Device label, before the count. Same size as the Diagnosis block's severity dot but smaller (6px vs 10px). Doesn't compete with the Diagnosis block — answers a different question (device health vs session errors).

---

## Customer View

Customer View is NOT changed in this redesign. It already uses a fixed sidebar + main area pattern. The CVPortfolioView and CVAccountDetail can remain as scrollable views within their container.

---

## Phasing

| Phase | What | Risk | Size |
|---|---|---|---|
| 1 | **Foundation: fixed viewport** — change main area to overflow:hidden, make each view fill available height, create scrollable panels with consistent visual treatment | Medium — touches every view's outer container | Large |
| 2 | **Overview dashboard grid** — rearrange Overview into 3-zone layout (health pills / 2-column content / session+quality bar) | Medium — significant layout restructure of one view | Large |
| 3 | **Visual additions** — error distribution bar, timeline density strip, account health squares, quality score ring, sidebar device dot | Low per item — each is a self-contained addition | Medium |

Phase 1 is the foundation. Phase 2 makes the Overview work within the fixed viewport. Phase 3 adds the visual polish. Each phase can be tested independently.

---

## What does NOT change

- Every existing feature, button, card, function stays
- Data layer completely untouched
- Customer View untouched
- Sidebar structure untouched (except the small device dot)
- Copy outputs untouched
- Keyboard shortcuts untouched
- The drop zone (pre-log-load) stays as-is
- All existing components reused — ErrCard, AcctCard, XpubView, JTTree, CopyBtn, etc.
