# Session Handoff — April 4, 2026 (Session 2)

## What was built this session

### Documentation Update (Agent Guide + Technical Reference)
Both embedded guides (`GUIDE_AGENT`, `GUIDE_TECHNICAL`) updated via `/team-build`:
- **Removed** all references to health pills, Portfolio stat card, "Stat cards: Entries, Accounts, Portfolio, Issues"
- **Updated** Overview description, investigation playbook scenarios, feature grid, keyboard shortcuts
- **Added to Agent Guide:** Account Focus Mode section, Discoverability Hints section, Diagnostic Pathways in Issues, Escape shortcut, Focus Mode references throughout playbook scenarios
- **Added to Technical Reference:** Account Focus System, Context Ribbon, Diagnostic Pathways, Adaptive Dimming, Overview Architecture sections. Connected Navigation expanded from 6 to 11 pathways.
- **Updated** Copy & Export section to reference Overview action bar location

### Copy Report Relocation
- **Moved** Copy report dropdown from sidebar bottom to Overview action bar (between Focus Investigation and Guide 📖 button)
- **Fixed bug:** click-outside handler used `setTimeout` pattern (matching Focus Investigation) instead of synchronous listener that was causing instant-close
- **Extracted** `buildSummaryText`, `buildFullText`, `buildCustomerText` as standalone module-scope functions (were inside DiagSidebar)
- `copyOpen` state moved from DiagSidebar to App
- `dropItemStyle` moved to module scope

### Copy Dropdown Enhancement
- **Added** Copy Errors as 4th option in dropdown (new `buildErrorText` standalone function)
- **Color-coded** all 4 options with left borders: purple (Quick Summary, Full Export), green (Customer Summary), red (Copy Errors)
- Hover tints background to match border color
- Copy Errors conditional on `logData.errs.length > 0`, Customer Summary conditional on `logData.accts.length > 0`

### Help Center Integration — Layer 1 (ERR_DB Article Links)
- **Added** `u` field (support.ledger.com article URL) to 30 ERR_DB entries (3 pre-existing, 33 total)
- **New** `errUrl(dg)` helper function extracts URL from diagnosis result
- **ErrCard:** "📄 Help article ↗" link renders below action text when `dg.u` exists
- **Overview error tiles:** small 📄 icon in corner when help article mapped
- **Copy Errors export:** `Help: {url}` line included per error when available

### Help Center Integration — Layer 2 (Contextual Help Sidebar)
- **New** `CONTEXT_HELP` constant (12 entries) with condition → article mappings
- Test functions scan `logData` for: firmware outdated, USB issues, Bluetooth errors, sync failures, heavy network failures, loading loop, swap issues, APDU rejections, staking, fee errors
- Two always-on entries: Ongoing Issues, Common Solutions
- **Rendered** as vertical link list at bottom of DiagSidebar (below Advanced section)
- Max 5 links, conditional first, always-on last
- APDU status word correctly extracted from `a.hex` (last 4 hex chars)

### Help Center Integration — Layer 3 (Live Status Banner)
- **Fetches** `status.ledger.com/api/v2/summary.json` on log load (fire-and-forget, no auth)
- **New** `ledgerStatus` state in App, `STATUS_CHAIN_MAP` constant (~24 entries)
- **Amber banner** on Overview when degraded Statuspage components match customer's chains
- **Muted banner** when services degraded but no chain match
- **Included** in `buildSummaryText` and `buildFullText` copy output
- Copy builders now accept `ledgerStatus` as third parameter

## What's ready to run (not yet executed)

### Layer 3b — Incident History Time-Overlap
Prompt written: `layer3b-incident-history-prompt.md`. Adds second fetch (`incidents.json`), compares log session time window against past incidents, shows blue 🕐 banner for outages that overlapped the customer's session even if now resolved. Changes `ledgerStatus` shape to `{ summary, incidents }`.

### CLAUDE.md Update
Prompt written: `update-claude-md-prompt.md`. Updates state variables, constants table, helper functions, adds Help Center Integration section, updates Overview/sidebar/copy descriptions, removes stale health pill references.

## Still on the backlog

### From Session 1 (carried forward)
1. **Sidebar Overview header fix** — Peter's #1 complaint. Overview block doesn't use SectionHeader component, looks different from Issues/Accounts/Timeline/Advanced. Spec was reviewed at start of this session but deferred.
2. **Responsiveness audit** — click-outside handlers for sidebar popovers. The Copy dropdown issue is fixed (moved to Overview), but other popovers may still have stopPropagation issues.
3. **Sweep cleanup** — dead code removal, missing state resets.

### New items
4. **Test all features with real logs** — especially Layer 3 status banner (needs active outage or mock data), Copy Errors dropdown, help article links on ErrCards.
5. **Customer View delta detection** — Peter mentioned app.json comparison for detecting delta between on-chain truth and what customer sees. Visualization not fully built out yet.
6. **Git tag** — tag after all changes committed.

## Current file state
~5,500+ lines (estimate — grew with CONTEXT_HELP, STATUS_CHAIN_MAP, expanded ERR_DB `u` fields, and guide content updates). 

## Key decisions made
- **Help center = static URL mapping, not scraping.** support.ledger.com is JS-rendered and blocked by org network policy. All article links are hardcoded URLs in ERR_DB `u` fields and CONTEXT_HELP entries. Easy to add new articles incrementally.
- **Status API = Atlassian Statuspage public endpoint.** No auth, no rate limit. One fetch per log load. Fail silently.
- **Copy report on Overview, not sidebar.** Peter wanted it visible and accessible, not buried.
- **Color-coded dropdown items.** Matches Agent Guide badge colors: purple (summary/full), green (customer), red (errors).
- **Contextual help in sidebar, not Overview.** Peter requested this — visible from every section, not just Overview.
- **Use `/team-build` for large editing tasks.** Documentation update was one prompt covering both guides.

## Article mapping — reference
35 help center articles cataloged. Full mapping in `HELPCENTER_INTEGRATION_SPEC_FINAL.md` (project knowledge). Key articles:
- USB: `115005165269-zd` | Bluetooth: `360025864773-zd` | Sync: `360012207759-zd`
- Network: `8889073163037-zd` | Firmware: `360013349800-zd` | Common: `360023518653-zd`
- Ongoing: `15158192560157-zd` | Fees: `360021039173-zd` | 0x6a82: `12434176239773-zd`

## Lessons from this session
- **Org network policy blocks all tools from support.ledger.com.** Web fetch, browser JS, page reading, navigation — all blocked. Workaround: web search indexing + Peter's domain knowledge for article mapping.
- **Track UI relocations carefully.** Copy report moved from sidebar to Overview mid-session; subsequent prompts referenced the old location. Need to update all downstream references when moving UI.
- **Peter prefers `/team-build` for big edits.** Noted for future prompts.
- **Context degradation is real.** Caught stale reference errors (Copy button location). Session should end before quality drops further.

## Priority order for next session
1. **Run CLAUDE.md update prompt** (already written)
2. **Run Layer 3b prompt** (incident history — already written)
3. **Sidebar Overview header fix** (Peter's #1 from session 1 — still pending)
4. **Responsiveness audit** (click-outside handlers)
5. **Test with real logs** (all new features)
6. **Git tag** post-help-center-integration
