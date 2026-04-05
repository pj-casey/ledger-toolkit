# Investigation Interconnectedness — Full System Spec

## Core principle

These three layers are ONE investigation system, not three features:
- **Context Ribbon** = where am I in this investigation?
- **Account Focus** = who am I investigating? (already built)
- **Diagnostic Pathways** = where should I look next?

Each feeds the others. The agent never needs to know about hidden gestures. They just click things that are already visible.

---

## Layer 1: Context Ribbon

**What it is:** A single line at the top of the content area showing the current investigation thread. Clickable segments. Only appears when there's context to show.

**Where it lives:** Inside the content column, above the section content. NOT in the sidebar, NOT above the sidebar. Just inside the right-side content panel. Takes ~28px when active, 0px when idle.

**What it shows (computed from existing state, no new state needed):**

| State | Ribbon shows |
|---|---|
| `focusedAcct` set | Chain color dot + ticker (e.g., "● BTC") — click goes to Accounts |
| `selectedErr` set + on Issues tab | Error title + time (e.g., "Device communication failed · 18:03:37") — click selects that error |
| `focusedAcct` + any non-Issues tab | Ticker + "focused" — click goes to Accounts |
| `focusedAcct` + `selectedErr` | Both, separated by · |
| Nothing focused, no error selected | Ribbon hidden — no space consumed |

**Clear button:** ✕ at the right edge. Clears focusedAcct AND selectedErr.

**Visual:** Background uses focused account's chain color at 6% opacity. Left border in chain color. If no account focused but error selected, uses the error's severity color. Muted text, small font (11px). Blends with the tool's design language.

---

## Layer 2: Account Focus (enhancement to existing)

**Already built:** Shift+click tile, popover Focus button, sidebar indicator, auto-filter Timeline + Issues, Escape to clear.

**The change:** Clicking "Affected: [account]" on an ErrCard now ALSO triggers focus. This is the primary discovery path. The agent clicks the most obvious thing on the error card and the whole tool narrows to that account.

**Implementation:** Where ErrCard is rendered (line ~5108), change the `onGoAcct` callback:

```jsx
onGoAcct={(addr) => {
  const acct = logData.accts.find(a => a.addr === addr);
  if (acct) {
    setFocusedAcct({addr:acct.addr, cid:acct.cid, ticker:acct.ch.ticker, name:acct.ch.name, color:acct.ch.color});
  }
  goToAcct(addr);
}}
```

No changes to ErrCard component itself.

---

## Layer 3: Diagnostic Pathways

**What it is:** Computed evidence links on ErrCards. For each error, the tool scans preceding entries within a time window and counts correlated events. Shows clickable summary lines.

**Where it lives:** On the ErrCard, below the existing "What happened before" breadcrumbs and above "Common causes."

**What it computes (for each selected error):**

Scan entries in the 5 seconds before the error. Count:
- **Network failures:** entries with type='network' and (status >= 400 or level='error')
- **APDU rejections:** entries with type matching APDU patterns and containing rejection codes (6985, 6982, 6a82, etc.)
- **Sync events:** entries with messages mentioning 'sync' in the same time window
- **Other errors:** other error-level entries in the window (excluding the current error)

**What it renders:**

```
Related evidence
  3 network failures in the 2s before → View in Network
  1 APDU rejection (6985) at −0.4s → View in APDU
  Sync active at time of error → View in Timeline
```

Only shows sections with counts > 0. If nothing found, the section doesn't render.

**Click behavior:** Each link navigates to the relevant section. If focus mode is active, the section is already filtered. The context ribbon updates to show the navigation.

---

## Implementation phases

### Phase 1: Context Ribbon + ErrCard Focus Trigger
- Add ribbon rendering in the content area
- Change onGoAcct to also trigger focus
- Ribbon computed from existing state (no new state)
- ~50 lines of JSX

### Phase 2: Diagnostic Pathways
- Add evidence computation to ErrCard
- Add "Related evidence" section rendering
- Wire pathway clicks to section navigation
- ~80 lines of JSX

### Phase 3: Polish
- Hint text below tiles: "hover · click to filter · shift+click to focus"
- Ribbon animation (fade in/out)
- Edge cases (error without linked account, network entries without identifiers)

---

## What NOT to change (across all phases)

- Data layer — FROZEN
- Existing Focus Mode Phase 1 — sidebar indicator, tile shift+click, popover Focus button, auto-filter, Escape handler. All stay.
- ErrCard component internals — unchanged in Phase 1 (pathways added in Phase 2)
- Tile popover structure — child-inside-tile with no-op handlers
- Copy/export formats — unchanged
- Customer View — unchanged
- Guide overlays — unchanged
