# Session Handoff — April 4, 2026

## What was built this session

### Interconnectedness System (all three layers shipped)
1. **Account Focus Mode** — `focusedAcct` state. Shift+click tile, popover Focus button, or click "Affected: [account]" on ErrCard. Auto-filters Timeline + Issues. Escape/✕ clears.
2. **Context Ribbon** — computed from `focusedAcct` + `selectedErr`. Shows at top of content area. Clickable segments: account → Accounts tab, error title → Issues, timestamp → Timeline jumpTo. Clear button wipes both.
3. **Diagnostic Pathways** — `useMemo` on ErrCard scans 5s before each error. Counts network failures, APDU rejections, sync events. Renders clickable "Related evidence" links that navigate to Network/APDU/Timeline.

### Adaptive Dimming
When focus is active, non-focused elements dim to 25% opacity: tiles, AcctCards (single + grouped), sidebar account rows, Overview error tiles, Issues strip segments, Network entries. Content area gets a 3px left border in the chain color. Focused tile has `focusPulse` animation (scale + opacity breathing). All transitions 0.2s.

### Discoverability Hints
Three persistent hint lines (9px, muted, 0.5 opacity):
- Tiles: "hover for details · click to filter · shift+click to focus"
- Timeline legend: "click to filter · hover to highlight"
- Issues chips: "click to filter · hover to preview on strip"

### Overview Hub Upgrade
- **Health pills removed** — replaced by compact device status line above stat cards ("Nano X · FW 2.6.1 ✓ · Apps 8/10 ✓ · Sync —")
- **Stat cards restructured**: Issues (hero, severity-colored left border), Accounts, Network (failure count), Log Quality. Portfolio removed (inaccurate, rarely relevant).
- **Focus Investigation button** — dropdown listing all accounts with chain colors, tickers, fiat values, error counts. Sits in action bar next to Guide button.
- **Error tile account badges** — chain dot + ticker on each error tile linked to an account
- **Guide button restored** — 📖 icon in action bar, opens Agent Guide overlay during investigation
- **Stat number sizing increased** — `--ov-stat-num` bumped to `clamp(28px, 4vh, 42px)`
- **Copy buttons consolidated** — sidebar Copy dropdown is the primary; redundant Overview copy removed.

## What's in progress / not yet done

### Sidebar Overview header (IMMEDIATE NEXT TASK)
Peter's top complaint: the Overview/Diagnosis block in the sidebar doesn't match the other section headers. Issues, Accounts, Timeline, Advanced all use the SectionHeader component (icon + bold label + subtitle + count). The Overview block is a custom component with different sizing, no icon, different layout. It looks smaller and feels disconnected.

**The fix:** Restructure the Overview sidebar block to match the SectionHeader visual pattern while keeping its severity-colored left border as the unique distinction. The severity dot + sentence becomes the subtitle. Add an icon. Match padding/font/spacing of other section headers. The Copy dropdown should be subordinate or relocated to the bottom of the sidebar.

### Responsiveness audit (prompts written, not yet run)
File: `responsiveness-audit.md` — click-outside handlers for Copy dropdown (sidebar), Portfolio overlay, Quality details popover, Device Reference popover. The sidebar Copy dropdown's `stopPropagation` wrapper swallows clicks meant for section headers behind it. This causes the "have to click twice" behavior Peter reported.

### Sweep cleanup (prompts written, not yet run)
File: `sweep-cleanup.md` — dead code removal (`globalSearch`, `showGlobalResults`, `showMore` + orphaned useEffect), missing state resets (`selectedErr`, `errAcctFilter`, `hoveredAcct`, `acctMapOpen` in handleFile/clearLog), stale TODO comment, popover left overflow for leftmost tiles.

## Current file state
~5,400 lines. Tagged `post-adaptive-dimming` before the Overview work. Peter committed everything and pushed to main at the end of this session.

## Key decisions made
- **Data layer is FROZEN** — never modify parsing, extraction, or diagnosis functions
- **Portfolio stat card removed** — not accurate enough, rarely relevant to tickets
- **Focus mode is account-centric only** — Peter considered broadening Focus to filter by error category, device issues, etc. We decided that belongs in a future sprint as "Investigation Scenarios" (guided playbook paths), not as Focus menu options.
- **Adaptive dimming uses opacity, not hiding** — non-focused elements dim to 25%, not hidden. Context preserved.
- **Network filtering in focus mode is informational only** — banner says "network requests are not account-specific" rather than attempting unreliable address matching.

## Design docs in project knowledge
- `interconnectedness-spec.md` — full system spec for the three layers
- `REDESIGN_VISION_V5.md` — viewport redesign vision
- `ROADMAP_LIVE_BALANCES.md` — live balance fetching roadmap

## Lessons from this session
- **Commit after every successful Claude Code run.** Peter's PC crashed mid-edit and reverted to a much older commit. Claude Code wasn't auto-committing. Always `git commit` after each change lands.
- **The Overview went through 5+ iterations.** Health pills → stat cards → mini dots → mini tiles → Focus button. Iteration is fine but each change should be tested with real logs before the next.
- **Peter's visual instincts are reliable.** When he says "this doesn't look right," he's right. Don't argue — look harder.
- **Don't repeat stale observations.** When Peter shows a new screenshot, respond to THAT screenshot, not the previous one. Context gets heavy in long sessions.

## Priority order for next session
1. **Sidebar Overview header fix** — immediate, Peter's top complaint
2. **Responsiveness audit** — the "click twice" bug on sidebar tabs
3. **Sweep cleanup** — dead code, missing resets
4. **Test with real logs** — Peter has log files to validate all features
5. **Git hygiene** — tag after each change: `git tag post-overview-final`
