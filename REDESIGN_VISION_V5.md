# Ledger Diagnostic Toolkit — Redesign Vision (v5)
*"The sidebar is for finding things. The main area is for understanding them."*

---

## One rule

Every visual element must answer exactly one question. If it answers two questions, split it. If it answers zero questions, remove it.

---

## Layout

```
┌────────────────────────────────────────────────────────────────────┐
│ DIAGNOSTIC TOOLKIT     [Diagnostic | Customer View]       file… × │
│                        Tech investigation  Customer's wallet      │
├──────────────────────┬─────────────────────────────────────────────┤
│                      │                                             │
│  ┌── DIAGNOSIS ────┐ │                                             │
│  │ ● No critical   │ │  MAIN AREA                                 │
│  │   issues found  │ │                                             │
│  │ [Copy ▾]        │ │  Full-width content panel.                  │
│  └─────────────────┘ │  Shows detail for whatever is               │
│                      │  selected in the sidebar.                   │
│  ⚠ Issues        12  │                                             │
│  👛 Accounts       8  │  Default: Diagnosis Detail                  │
│  📋 Timeline    2028  │  (= current Overview tab)                   │
│  ⚙ Advanced          │                                             │
│                      │                                             │
└──────────────────────┴─────────────────────────────────────────────┘
```

Sidebar (~240px, fixed). Main area (remaining width). Same pattern as Customer View. Both panels visible at all times. Nothing scrolls off-screen in the sidebar.

---

## SIDEBAR

### Diagnosis block

The home landmark. Visually distinct — subtle background tint, Ledger bracket accents. Clicking it returns to the Diagnosis Detail view (Overview).

**Contains exactly three things:**

1. **Severity indicator + sentence.** A colored dot (🟢 green / 🟡 yellow / 🔴 red) and one sentence generated from existing `diagnose()` data:
   - `🔴 2 critical issues — device connection rejected`
   - `🟡 3 warnings during account sync`
   - `🟢 No critical issues found`
   - `🟢 Clean session — no issues detected`

2. **Copy dropdown.** `[Copy ▾]` with three options on click:
   - Quick Summary — *for ticket notes*
   - Full Export — *for escalation*
   - Customer Summary — *account-focused* (when accounts exist)

3. **Nothing else.** No session info, no domain indicators, no stat cards. Those live in the Diagnosis Detail view in the main area where there's room.

**Severity color:** The Diagnosis block's left border matches the severity dot. Red for critical, yellow for warning, green for clean. This is the ONE place in the sidebar that communicates severity. Nowhere else.

---

### Section list

Four sections, always visible. Each has: icon, label, count badge, muted subtitle.

```
⚠ Issues                   12
  Errors and warnings

👛 Accounts                  8
  Crypto accounts in this log

📋 Timeline               2028
  All events, chronological

⚙ Advanced
  Network · Device · Raw
```

**Icons** use existing `Ico.*` components (Alert, Wallet, Logs, Chip). Pattern affordance — the agent recognizes what each section contains before reading the label.

**Subtitles** are 10px muted text. New agents read them once. Experienced agents don't notice them. They add zero visual weight for people who don't need them.

**Count badges** answer "how much is in here?" — setting expectations before the agent clicks.

**That's all.** No severity coloring on sections. No status indicators. The sidebar sections are neutral navigation — they don't compete with the Diagnosis block for the agent's attention.

---

### Expanded sections

Issues and Accounts can BOTH be open simultaneously. Investigation is exploratory — agents scan issues and accounts together. Only Advanced uses accordion behavior (one sub-item at a time).

#### Issues expanded

```
⚠ Issues                   12  ▾
  Errors and warnings

    Price data unavailable  ×4
    Chart data unavailable  ×4
    Unexpected error        ×3
```

Each row: title + repeat count. That's it. No severity dots. No category dots. The row answers one question: "What issues exist?" The agent clicks a row to learn everything else (severity, category, affected account, action) in the main area.

If more than 5 unique issues, show top 5 + `+N more`.

#### Accounts expanded

```
👛 Accounts                  8  ▾
  5 funded · 3 empty

    ● Ethereum 1            ⚠2
    ● Tezos 1
    ● Cosmos 1
    ● Tron 1
    ● Tezos 2
    ╶╶ 3 empty…
```

Each row: chain color dot + name. One optional element: error badge `⚠N` when issues reference this account. The error badge answers a navigation question: "Should I click this account?" — so it belongs in the sidebar. Everything else (balance, ops, tokens, address) is in the main area.

Empty accounts collapsed under `3 empty…` to keep the list short.

#### Advanced expanded

```
⚙ Advanced                    ▾

    🌐 Network             453
    📡 APDU
    { } Raw JSON
```

Sub-items with icons and counts. APDU only shows when APDU data exists. One sub-item active at a time (accordion within Advanced only).

---

### Navigation behavior

| Sidebar click | Main area loads | Active indicator |
|---|---|---|
| Diagnosis block | Diagnosis Detail (Overview) | Block highlighted |
| Issues header | Issues List (Errors tab) | Issues header highlighted |
| Specific issue row | Issue Detail | That row highlighted |
| Accounts header | Accounts List (Accounts tab) | Accounts header highlighted |
| Specific account row | Account Detail | That row highlighted |
| Timeline | Timeline view | Timeline highlighted |
| Network | Network view | Network highlighted |
| APDU | APDU view | APDU highlighted |
| Raw JSON | Raw JSON tree | Raw JSON highlighted |

One click. Always. Active section gets a left accent bar (same visual pattern as Customer View's `cv-active`).

---

## MAIN AREA

Each view fills the full available width and height. Views use EXISTING components — ErrCard, AcctCard, XpubView, JTTree, etc. We are rearranging layout, not rebuilding content.

### Diagnosis Detail (DEFAULT)

**This IS the current Overview tab. Every element preserved:**
- 4 stat cards (Entries, Accounts, Operations, Issues) — clickable to relevant sections
- Activity badges (Manager, Swap, Device scan)
- Device & App card (model, firmware, language, paired, app, commit, sync, MEV)
- Environment card (OS, platform, user ID, sync)
- Issues preview (top 3 with "View all →")
- Session info bar
- Quality score (expandable)

**One addition:** A subtle guidance line below the issues preview, phrased as a suggestion:

```
Consider: 2 errors reference Ethereum 1 — review that account. 
For exact balances, load the customer's app.json in Customer View.
```

This appears WITHIN the Diagnosis Detail content, not as a persistent bar. It's contextual — generated from the data, shown where it's relevant, and phrased as "Consider" not "You should."

---

### Issues List

**This IS the current Errors tab. Every element preserved:**
- Severity bar (Critical / Warning / Info counts)
- Diagnostic summary sentence
- Repeating patterns section
- Category-grouped ErrCards with full detail, expandability, jumpTo, linked accounts

**One addition per ErrCard:** The "Affected account" link from Phase 2 (already implemented).

---

### Issue Detail

Focused view when a specific issue is clicked in the sidebar:
- Full ErrCard (same component)
- All occurrences listed with indices
- Affected account link
- "View in timeline" link
- "← Back to all issues"

---

### Accounts List

**This IS the current Accounts tab. Every element preserved:**
- Funded/empty count
- Filter input
- Copy all IDs, Export buttons
- Sync warning
- AcctCard components with expandability, error badges

**One addition:** When accounts exist but no sync data: *"No sync captured in this session. For exact balances, load the customer's app.json in Customer View."*

---

### Account Detail

Focused view when a specific account is clicked in the sidebar:
- Expanded AcctCard content at full width
- Explorer link, token list, related errors
- Transaction history (when app.json loaded)
- Xpub scanner (for BTC/LTC/DOGE/BCH)
- "← Back to all accounts"

---

### Timeline

**Current Timeline tab, unchanged.** Search, filter, export, expandable rows, severity accents.

**One addition to empty search state:** *"Try searching for: error, rejected, sync, timeout, or a specific account address."*

---

### Network

**Current Network tab, unchanged.**

**One addition:** If any requests have 4xx/5xx codes, a summary line at top: *"N requests returned errors (X server, Y client)."*

---

### APDU

**Current APDU tab, unchanged.**

**One addition:** If rejection codes found, a summary line: *"N device rejections. Most common: 6985 — Conditions not satisfied."*

---

### Raw JSON

**Current Raw tab with JTTree viewer, unchanged.**

---

## Empty states

Every section with no data explains why and suggests a next step. These appear in the main area content, not the sidebar.

| Section | Empty message |
|---|---|
| Issues (0) | No errors detected. If the customer reports an issue, ask them to reproduce it and export new logs immediately after. |
| Accounts (0) | No accounts in this log. Load the customer's app.json in Customer View for account information. |
| Timeline (0) | No log entries found. The file may be empty or in an unrecognized format. |
| Network (0) | No network requests captured. |
| APDU (0) | No device communication data. The device may not have been connected during this session. |

---

## Mode toggle

Labels gain one-line descriptors (tooltip or subtitle):
- **Diagnostic** → "Technical investigation"
- **Customer View** → "See the customer's wallet"

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| Ctrl+1 | Diagnosis (home) |
| Ctrl+2 | Issues |
| Ctrl+3 | Accounts |
| Ctrl+4 | Timeline |
| Ctrl+5 | Network |
| Ctrl+6 | APDU (when available) |
| Ctrl+7 | Raw JSON |

---

## What is preserved (everything)

Every feature from the current tool maps 1:1 to a location in this design. No feature is removed, hidden behind additional clicks, or degraded. The complete mapping table from v4 remains accurate — the only changes are WHERE visual indicators appear (main area instead of sidebar), not WHETHER they exist.

---

## What was deliberately removed from v4

| Removed | Why |
|---|---|
| Severity dots on issue rows | Sidebar answers "what exists?" not "how bad is it?" |
| Category dots on issue rows | Detail belongs in main area, not navigation |
| Domain indicators in Diagnosis block | Visual clutter in 240px competing with the severity sentence |
| Persistent suggestion bar | Anchoring bias risk; guidance should be contextual, not persistent |
| Single-active accordion | Investigation is exploratory; agents need Issues and Accounts visible simultaneously |

---

## Research principles satisfied

| Principle | How |
|---|---|
| 5-7 primary nav items | 4 sections + Diagnosis block = 5 |
| 2-second scanability | Icon + label + count per section. One sentence in Diagnosis. |
| Sidebar complements, doesn't compete | Navigation only. All detail in main area. |
| 4-7 pieces of info at a time | Each sidebar row: 2-3 pieces max (icon+label+count or dot+name+badge) |
| Persistent navigation | Sidebar always visible, never changes |
| Home landmark | Diagnosis block — distinct visual, always returns to overview |
| Contextual guidance not persistent | Suggestions within relevant sections, phrased as "Consider" |
| Multi-active for exploratory tasks | Issues and Accounts both expandable |
| Plain language labels | Issues, Accounts, Timeline + plain subtitles |
| Consistent with rest of app | Same layout pattern as Customer View |

---

## Implementation

Replace the tab bar + tab content in App's diagnostic mode with:
1. A sidebar component (Diagnosis block + collapsible sections with sub-items)
2. A main area that renders existing tab content based on a `section` state variable
3. Contextual guidance lines within relevant main area views
4. Empty state messages
5. Section subtitles and mode descriptors

All existing components reused as sub-renderers. Zero changes to data layer.

---

## Ledger brand alignment

Verified against Ledger's publicly available resources (April 2026):

### Design system: LDLS / Lumen (`github.com/LedgerHQ/ldls`, `github.com/LedgerHQ/lumen`)

Ledger's official open-source design system. MIT licensed. Uses Tailwind CSS presets with semantic design tokens. Key token patterns we should align with:

| LDLS token | Our current CSS var | Alignment |
|---|---|---|
| `bg-base` | `--bg: #131214` | ✓ Conceptually aligned |
| `bg-surface` | `--panel: #1C1D1F` | ✓ Conceptually aligned |
| `text-default` | `--text: #FFFFFF` | ✓ Aligned |
| `text-muted` | `--muted: #949494` | ✓ Aligned |
| `text-success` | `--success: #7AC26C` | ✓ Aligned |
| `accent` (Button appearance) | `--primary: #BBB0FF` | ✓ Purple accent confirmed |

LDLS includes AI rules for Claude (`RULES.md`). If the tool ever moves to a build system, LDLS can be integrated directly. For our single-file approach, we manually approximate their tokens — which we're already doing accurately.

### Typography (`brand.ledger.com/brand-design/typography`)

Ledger's brand fonts: **Brut Grotesque** (body), **HM ALPHA Mono** (monospace), **Ledger Mono**. Our tool uses **Inter** + **JetBrains Mono** because Brut Grotesque is likely licensed and not CDN-embeddable. Inter is an acceptable stand-in (clean geometric sans-serif). If internal font licensing is available, swapping to Brut Grotesque would make the tool feel native.

### Layout pattern

Ledger Wallet itself uses a **left sidebar with account list + main content area** — exactly our v5 pattern. Their sidebar shows chain color dots, account names, and balances. Dark mode supported via CSS custom properties. This confirms our layout is architecturally aligned with Ledger's own product.

### Brand elements already in our tool
- ✓ Ledger bracket SVG logo in top bar
- ✓ Corner bracket motifs on cards (signature brand element)
- ✓ "LEDGER DIAGNOSTIC" watermark
- ✓ Dark theme as default
- ✓ Chain color dots for account identity
- ✓ Purple accent color for primary actions
