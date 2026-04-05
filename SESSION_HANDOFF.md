# Session Handoff — April 4, 2026 (Session 3)

## What was built this session

### Brut Grotesque Font Embedding
- Embedded 4 Brut Grotesque woff2 files as Base64 `@font-face` declarations directly in the HTML
- Weights: Light 300, Regular 400, Medium 500, Bold (600+700 share one file)
- ~194KB for font data, ~700KB total file size
- No CDN dependency — works fully offline and on `file://`
- Replaces Space Grotesk (previously loaded from Google Fonts) across all body text, values, headings, descriptions

### Visual Unification Sweep (Design Normalization)
Full pass normalizing design inconsistencies introduced over multiple sessions:

**Border-radius system established:**
- 8px — cards, panels, containers, overlays, popups
- 6px — buttons, interactive controls, input fields, filter chips, advice boxes
- 4px — small badges, pills, severity tags, breadcrumb chips
- Eliminated: 12, 10, 5, 3 (all gone)

**Hover patterns standardized:**
- Data rows (timeline, network, APDU, device info): `rgba(255,255,255,0.02–0.03)`
- Cards: `.err-hover`, `.acct-hover` classes
- Ghost buttons: `borderColor → T.primary, color → T.primary`

**Other polish:** stat card `borderTop` highlight, stat card value letter-spacing, `.stat-value` class, network section container, APDU export container, popover/dropdown consistency.

### Customer View — Visual Alignment
Brought Customer View visual language into full alignment with Diagnostic mode:
- `.cv-sidebar` — vertical purple gradient (matches Diagnostic sidebar)
- `.cv-infobar` — subtle top gradient
- `.cv-portfolio-card`, `.cv-detail-card` — 8px radius, top-edge highlight border
- `.cv-section-title` — JetBrains Mono (matches all other section labels)
- `.cv-info-row` — hover highlight; keys use `fontFamily:MF` monospace treatment
- `.cv-acct-row.cv-active` — horizontal gradient glow from left border (matches AcctCards)
- Portfolio cards get chain-colored left borders with inset glow
- **Layout fix:** `.cv-layout` changed from `height:calc(100vh - 100px)` to `calc(100vh - 52px)` (the 100px was legacy from a separate mode-toggle row that no longer exists). Diagnostic main div hidden (`display:none`) in customer mode to eliminate residual gap.

### Live Version Checking — Phase 1 (App Catalog Comparison)

**Data layer extension:**
- `extractDevice` now stores `info.targetId` (full 32-bit numeric value) for Manager API calls
- D-7 block: stores targetId when model is identified via bitmask
- Post-D-7 fallback: scans `data.targetId`, `data.data.target_id`, and URL params for targetId even when model identified via other path

**API integration:**
- Fetches `GET manager.api.live.ledger.com/api/v2/apps/by-target` on log load
- Required params: `target_id`, `firmware_version_name`, `device_type`, `livecommonversion` (extracted from Manager API URLs in log entries, fallback `'1'`), `provider=1`
- Missing `livecommonversion` was the cause of initial 400 errors — fixed by scanning log entries for any Manager API URL containing the param
- `versionCheck` state: `{status:'ok'|'loading'|'error', apps:[{name, installed, latest, outdated}], catalogSize}`
- Cleared on new file load; aborted on unmount

**Copy report integration:**
- `buildSummaryText` and `buildFullText` accept `versionCheck` as 4th param
- Include VERSION CHECK section with outdated app list when available

**App chip visual system (manager-result path):**
- New 3-state system: green `✓` current, red `⬆` outdated (with `v1.0→1.1` version path), red `✕` missing
- Live catalog data overrides log-derived status when `versionCheck.status==='ok'`
- Tooltip on red chips: "Installed vX → latest vY (update available)"
- Cursor `help` on outdated chips
- Device Apps header simplified: label left, single LIVE/CHECKING/OFFLINE badge right; fallback count only when `versionCheck` is null
- Advice box: red-tinted, shows specific counts ("2 outdated, 1 missing")
- Others expanded chips: live-aware, red `⬆ name v→v` for outdated
- "+ N others" button: crimson `+ N others (M ⬆)` when hidden apps outdated

### CLAUDE.md Update
Full Session 3 documentation pass via `/team-build`. All 11 changes applied including: Brut Grotesque font table, border-radius system, hover patterns, Customer View section, Live Version Checking section, versionCheck in state variables, extractDevice targetId docs, helper functions, git tags, spec files.

---

## Current file state

~5,700+ lines. One file: `ledger-toolkit.html`.

## Current git state

Branch: `main`. Two unpushed commits on top of `origin/main`:
1. `bee7d45` — Live version checking, Brut Grotesque, visual unification, chip redesign
2. Latest (uncommitted) — Customer View layout fix (cv-layout height) + header simplification

To push: `git push origin main`

---

## Key decisions made this session

- **Brut Grotesque embedded, not CDN.** File:// compatibility and offline use require self-contained font delivery. Base64 woff2 is the right approach even at ~194KB cost.
- **Live version check uses log's own livecommonversion.** Avoids hardcoding a version number that would go stale. Extracted from first Manager API URL found in log entries.
- **Chip color system: red for outdated, not amber.** Agents think "amber = warning, fix eventually." Red makes urgency clear. Consistent with error color system elsewhere in the tool.
- **Header simplified to label + single badge.** The previous header had count text + badge text — redundant. The chips communicate status directly; the badge communicates data freshness.
- **Customer View layout: `calc(100vh - 52px)`.** The original `100px` subtraction was legacy from a separate mode-toggle row. Mode toggle is now in the top bar.
- **`/team-build` for all multi-file or large editing tasks.** Working well as a pattern.

---

## Backlog (carried forward + new)

### Carried from earlier sessions
1. **Responsiveness audit** — click-outside handlers for sidebar popovers. Copy dropdown fixed; others may still have stopPropagation issues.
2. **Test with real logs** — all new features, especially Live Version Checking (needs a log where Manager API call resolves successfully end-to-end).

### Live Version Checking — remaining phases
3. **Phase 2: Firmware check** — needs `POST /api/v2/get_latest_firmware`. Requires `device_version_id` from `get_device_version` first. Full `targetId` is now stored, enabling this.
4. **Phase 3: Ledger Wallet version** — GitHub releases API. Handle `ledger-live-desktop` vs `ledger-wallet-desktop` tag naming differences.
5. **Phase 4 polish** — per-session cache, re-check button, Agent Guide entry for version check features.

### Other
6. **Git tag** `post-version-check-p1` after push.
7. **Agent Guide + Technical Reference update** — guides haven't been updated since Session 2. Need entries for: Brut Grotesque font, Live Version Checking, Customer View alignment, new chip visual system.

---

## Lessons from this session

- **Single-file edits don't benefit from parallel agents.** All 11 CLAUDE.md edits went to one agent — the right call. Parallel agents on one file cause conflicts.
- **400 errors often have simple param causes.** The Manager API 400 was just a missing `livecommonversion`. Scanning the existing log entries for the correct param value was the cleanest fix.
- **Customer View had accumulated drift.** Gradients, fonts, hover states, border radii were all inconsistent with Diagnostic mode. One visual alignment pass fixed all of it.
- **Legacy height values accumulate silently.** The `100px` cv-layout subtraction was never wrong enough to break anything, but produced a visible black bar once the mode toggle moved into the top bar.
