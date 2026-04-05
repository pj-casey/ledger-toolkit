# Documentation Update Spec

Both guides are embedded in `ledger-toolkit.html` as JS template literal constants: `GUIDE_AGENT` (line ~190) and `GUIDE_TECHNICAL` (line ~549). They also exist as standalone files `agent-guide.html` and `technical-reference.html` in the repo. Updates must be applied to the embedded versions (the standalone files are for reference/editing convenience).

## Distinction between documents

**Agent Guide** = "How do I investigate with this tool?"
- Task-oriented. Organized by customer complaint type.
- Covers workflow: load log → read diagnosis → follow investigation path → resolve.
- Audience: all agents, from first-day to senior. Beginner gets step-by-step, senior gets efficiency tips.
- Tone: direct, practical, "do this, then this."

**Technical Reference** = "What can this tool do and how does it work?"
- Capability-oriented. Organized by system/feature.
- Covers architecture: what the tool extracts, computes, renders, and correlates.
- Audience: senior agents, tier 2, anyone wanting to understand the tool deeply.
- Tone: precise, thorough, "this is how the system works."

**Interlinking:** Each document should reference the other where relevant. Agent Guide says "For technical details on how error classification works, see Technical Reference → Error Analysis." Tech Ref says "For investigation workflows using this feature, see Agent Guide → Investigation Playbook."

---

## Agent Guide — What to update

### Remove or update outdated references

1. **"Health pills"** — referenced in Quick Start, Investigation Playbook (device, balance, crash sections), and Interface section. Health pills no longer exist. Replace with references to the **device status line** at the top of Overview and the **stat cards**.

2. **"Portfolio stat card"** — referenced in balance investigation playbook and Live Balances section. Portfolio card was replaced with **Network** stat card. Portfolio total is still visible in the Accounts section header (not Overview).

3. **Stat cards description** — currently says "Entries, Accounts, Portfolio, Issues". Update to "Issues (hero), Accounts, Network, Log Quality."

### Add new sections

4. **Account Focus Mode** — NEW section in The Interface area. Should cover:
   - What it does (one account, every section filters)
   - How to activate: Shift+click tile, popover Focus button, click "Affected: [account]" on error card, Focus Investigation button on Overview
   - What happens: sidebar indicator, context ribbon, adaptive dimming, auto-filter Timeline + Issues
   - How to clear: ✕ button, Escape key, Shift+click same tile
   - Tip: "Focus mode is your primary investigation tool. Most tickets center on one account — focus on it and every section becomes relevant."

5. **Context Ribbon** — brief mention within the Focus Mode section. "A ribbon at the top of the content area shows your current investigation thread. Click any piece to navigate back."

6. **Diagnostic Pathways** — add to the Issues/Breadcrumbs section. "Below the breadcrumbs, a Related Evidence section shows correlated events: network failures, APDU rejections, sync activity in the seconds before the error. Each link navigates to the relevant section."

7. **Overview restructure** — update the Interface → Diagnostic Mode → Overview description:
   - Device status line (one-line quick check)
   - Stat cards: Issues (severity-based hero), Accounts, Network (failure count), Log Quality
   - Focus Investigation button (dropdown with all accounts)
   - Guide button (📖 for accessing this guide during investigation)
   - Error tiles with linked account badges

8. **Adaptive Dimming** — brief mention in the Focus Mode section. "When focused, everything not related to your account dims to 25% — tiles, cards, sidebar rows, error strip. Your investigation path stays bright."

9. **Discoverability hints** — mention in Tips section. "Look for small hint text below interactive elements — tiles, timeline chips, issues chips — showing available interactions."

10. **Update Keyboard Shortcuts** — add:
    - Escape: clear focus mode
    - (Shift+click tile is a mouse shortcut, not keyboard — mention in Focus Mode section)

11. **Update Investigation Playbook scenarios** to reference Focus Mode:
    - "My balance is wrong" → "Focus on the affected account using the Focus Investigation button or Shift+click its tile. Check Accounts for live vs log balance."
    - "My transaction failed" → "Click the error tile on Overview — if it's linked to an account, focus activates automatically."
    - "My device won't connect" → "Check the Network stat card on Overview. 0 calls means no API activity."

### Cross-link to Technical Reference

12. Add links where appropriate:
    - Focus Mode section: "See Technical Reference → Account Focus System for implementation details"
    - Diagnostic Pathways: "See Technical Reference → Diagnostic Pathways for the evidence computation"
    - Xpub Scanner: "See Technical Reference → UTXO Extended Public Key Analysis"

---

## Technical Reference — What to update

### Update existing sections

1. **Cross-Domain Correlation → Connected Navigation** — the navigation list is outdated. Add:
   - **Focus → All sections** — `focusedAcct` state propagates to `tlAcctFilter`, `errAcctFilter`, and visual dimming across tiles, cards, sidebar rows, strip segments
   - **Overview error tile → Focus + Issues** — clicking an error tile with a linked account sets `focusedAcct` and navigates to Issues
   - **ErrCard "Affected account" → Focus** — clicking the linked account chip sets `focusedAcct` and navigates to Accounts
   - **Context Ribbon → Navigation** — each ribbon segment is a clickable navigation target (account → Accounts, error → Issues, timestamp → Timeline jumpTo)
   - **Diagnostic Pathway → Section** — "Related evidence" links navigate to Network, APDU, or Timeline with focus preserved

2. **Account Health Visualization** — update to mention adaptive dimming: "When account focus is active, non-focused tiles dim to 25% opacity. The focused tile displays a `focusPulse` CSS animation."

3. **Session Visualization** — update to mention that the Issues error strip also dims non-focused segments.

### Add new sections

4. **Account Focus System** — NEW section under Cross-Domain Correlation:
   - State: `focusedAcct` in App component (holds full account object or null)
   - Triggers: Shift+click tile, popover Focus button, ErrCard linked account click, Overview Focus Investigation dropdown, Overview error tile click (when linked)
   - Downstream effects: `useEffect` propagates to `tlAcctFilter` and `errAcctFilter`
   - Visual: sidebar indicator (purple pill), content area left border (chain color, 3px), `focusPulse` animation on focused tile
   - Adaptive dimming: tiles (0.25), AcctCards single + grouped (0.25), sidebar rows (0.4), Overview error tiles (0.25), Issues strip segments (0.2), Network entries (0.3). All transitions 0.2s.
   - Clear: Escape key, ✕ on sidebar indicator, ✕ on context ribbon, Shift+click same tile
   - Resets: cleared in `handleFile` and `clearLog`

5. **Context Ribbon** — NEW section under Cross-Domain Correlation:
   - Computed from existing state: `focusedAcct` + `selectedErr` + `section`
   - Visibility condition: `focusedAcct` is set OR (`selectedErr` is set AND section === 'errors')
   - Segments: account chip (→ Accounts), error title chip (→ Issues + select error), timestamp chip (→ jumpTo in Timeline)
   - Clear button wipes `focusedAcct` and `selectedErr`
   - No new state variable — purely computed

6. **Diagnostic Pathways** — NEW section under Error Analysis:
   - `useMemo` on ErrCard component, deps: `[entries, err.li, err.ts]`
   - Scans 5-second window before each error
   - Counts: network failures (type='network' + status>=400/level='error'/message includes 'fail'), APDU rejections (type includes 'apdu' + codes 6985/6982/6a82/6984/5515), sync entries (message includes 'sync'), other errors (level='error', excluding current)
   - Returns null if no correlated events found
   - Renders clickable links: `onGoNetwork` → Network section, `onGoApdu` → APDU section, `onJump(syncEntry.logIndex)` → Timeline

7. **Overview Architecture** — update the existing Overview description or add new section:
   - Zone 1: device status line (compact one-liner using existing computed variables), stat cards (Issues hero, Accounts, Network, Quality), action buttons (Focus Investigation dropdown, Guide), activity badges
   - Stat card computation: Issues shows severity-based label, Network counts failures, Quality shows score
   - Focus Investigation dropdown: lists all accounts with chain colors, tickers, fiat values, error counts
   - Error tiles: sorted by severity, limited to 12, show linked account badge (chain dot + ticker)

### Cross-link to Agent Guide

8. Add links where appropriate:
    - Account Focus System: "For investigation workflows, see Agent Guide → Account Focus Mode"
    - Diagnostic Pathways: "For practical use in error investigation, see Agent Guide → Investigation Playbook"
    - Overview Architecture: "For step-by-step usage, see Agent Guide → Quick Start"

---

## Implementation note

Both guides are embedded in `ledger-toolkit.html` as template literals (`const GUIDE_AGENT=\`...\`` and `const GUIDE_TECHNICAL=\`...\``). The updates should be applied to these embedded strings. The standalone HTML files should also be updated to match.

This is a large text editing task — best handled as a Claude Code prompt with `/team-build`. The HTML structure uses the existing CSS classes (.card, .tip, .warn, .badge-*, .def-list, .section-divider, .new-badge, etc.) which should be reused for consistency.
