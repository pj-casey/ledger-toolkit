# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build step. Open `ledger-toolkit.html` directly in a browser:

```
open ledger-toolkit.html
```

To inspect: use browser DevTools. There is no test suite, no package manager, no bundler. All changes are in the single HTML file.

**Git push:** The pre-commit hook blocks sandbox pushes. Peter runs `! git push origin main` directly from the prompt to push to remote.

**Stale-base check (mandatory):** Before editing, confirm you are on the current file: `grep -c "slice(0,500).map" ledger-toolkit.html` must return ≥1 and `grep -ci "fr-FR"` must return 0. A previous round shipped a regression because edits landed on an outdated copy. If the check fails, stop and ask Peter for the current file.

## Architecture

`ledger-toolkit.html` (~10,200 lines) is the entire application — a single-file React 18.3.1 + Babel standalone app (no build step). It opens directly in any browser.

**Two modes, two design systems (do not mix):**
- **Diagnostic Mode** — the 2026 "BI dashboard" redesign. Fixed viewport (`height:100vh`, no page scrolling). **Tab bar** navigation (the old sidebar is gone). Sections: `overview`, `errors`, `accounts`, `timeline`, `network`, `apdu`, `raw`. Styling uses the `bi-*` class system: near-black panels, hairline borders, **2px radii**, orange `#E85A1A` accent, mono data values. CSS custom properties `--bi-*` and `--ledger-*` at the top of the stylesheet are the source of truth.
- **Customer View** — unchanged legacy layout: scrollable left-nav, sections `portfolio`, `accounts`, `earn`, `myLedger`, `agentInsights`. Still uses the legacy `T` theme object, `MF` mono constant, and 8/6/4px radius tiers. Do not port `bi-*` styles into Customer View or vice versa.

The file is organized top-to-bottom: CSS → embedded guides → data constants → parse/diagnostic functions → React components → `ReactDOM.render`.

## App Component

`function App()` (~line 7118) holds ~60 top-level useState calls (≈97 file-wide). Before adding new state, grep for existing related state and reuse or colocate.

**Reset-list rule (this codebase's #1 historical bug source):** any new state that holds log-derived or per-session data MUST be reset in BOTH places: the `handleFile` success path (~line 7340) and `clearLog` (~line 7626). Both are long setter chains — append to them. Recent example: `selectedApdu` was missed and showed a previous customer's frame in the next log.

## Data Layer — DO NOT MODIFY

These are stable, tested, and relied on by all rendering logic. Do not edit:

`parseLogs`, `parseAppJson`, `synthesizeMobileMeta`, `extractDevice`, `extractAccounts`, `extractErrors`, `extractSync`, `extractApdu`, `extractActivity`, `extractAnalytics`, `extractDeviceApps`, `inferRequiredApps`, `diagnose`, `CHAINS` (60), `TX_EXPLORERS` (45+), `UTXO_NETS`, `getChain`, `DC`, `DECIMALS`, `TOKEN_CONTRACTS`, `fetchTokenChains`, `TOKEN_URLS`, `TOKEN_SEARCH`, `EVM_CHAIN_IDS`, `CURRENCY_TO_APP`, `COINGECKO_IDS`, `EVM_RPCS`, `BALANCE_APIS`, `fetchEvmBalance`, `fetchPrices`, and all version check fetch/compare logic.

**ERR_DB (85 patterns) — extend additively, never relax these invariants:**
- User rejection is NOT an error (Ledger canonical rule): the seven rejection entries (`0x6985`/`27013`, `0x5501`/`21761`, `UserRefusedOnDevice`, `UserRefusedAddress`, `RefusedByUserDAError`) are `s:'medium'` and must never be raised to `'high'`. A reject-only log must show an amber banner, not red.
- Decimal status-word matchers always carry `w:1` (word boundary) so they can't false-match inside other numbers.
- No duplicate `m:` matchers (verified zero — keep it that way).

**May extend (additive only):** `extractDevice`, `extractAccounts`, `extractDeviceApps` — for new log format support, without breaking existing parsers.

## Formatting Invariants

- **Money:** module-scope `fmtFiat(value, full?)` (en-US, `$`) is the ONLY fiat formatter — 17 call sites including all copy-report builders. Never call `toLocaleString` on a fiat value directly; never introduce locale-dependent money formatting (a fr-FR bug once rendered €1,965 for a €1.97 wallet).
- **Durations:** `fmtDur(seconds)` → "Xm Ys". **Timestamps:** `fmtTime(ts, {ms:false})` where ms precision isn't wanted.
- **Big-list caps:** Network call log, APDU frame stream, and the Timeline event log all render at most 500 rows with a "Showing first 500 …" footer. Keep this convention for any new large list.

## Key Data Shapes

```
logData.accts          — accounts array
logData.errs           — errors array
logData.entries        — all log entries
logData.apdu           — APDU frames (.idx, .dir, .hex, timestamps)
logData.dev            — device info { appVer, appBrand, fw, modelId, targetId }
logData.quality        — quality score object

appJson.accounts       — accounts from parsed app.json
appJson.encrypted      — boolean

Error objects:  .dg.s = severity ('high'/'medium'/'low'), .dg.t = title, .dg.a = action, .dg.u = support URL
Account objects: .name, .currency, .ch (chain), .addr, .ops, .bal, .funded
Enriched accounts (CustomerView): ajBalance, ajSpendable, ajOps, ajOperations, ajSubAccounts,
  ajBlockHeight, ajStarred, ajSwapHistory (.swapId, .status, .provider), ajPendingOps,
  ajFreshAddressPath, enriched:true
```

**CVAgentInsights finding kinds:** `kind:'account'` (balance mismatch), `kind:'system'` (firmware/apps/drift), `kind:'drain'` (seed compromise — unshifted to top of list)

**Three version concepts (never conflate):**
| Concept | Source field | UI label |
|---|---|---|
| Desktop app | `info.appVer`, `info.appBrand` | `llLabel(dev)` → "Ledger Wallet" or "Ledger Live" |
| Firmware (SE OS) | `info.fw` | "Firmware" — badge says "on latest" ONLY when verified (`fwVerified`); otherwise "unverified" |
| Device coin apps | `extractDeviceApps()` | "Device Apps" |

## Diagnostic-Mode Components (post-redesign)

- `AcctRow` / `AcctDrill` / `AcctFilterChip` — replaced the old `AcctCard` (gone; responsibilities split).
- `OvTreemap` (Overview mini treemap; collapses tiles under 48px into "+N more"), `OvCopyChip` (userId copy chip), `Quad` (2×2 Customer Setup card).
- `SwapStatusCheck` — renders a copy-the-CLI-command chip ONLY for swaps with `swapId` + `provider` whose status is not in `SWAP_FINAL_STATUS`.
- **Focus Mode:** `focusedAcct` propagates via `focusAcctMatchesEntry(entry, acct)` to dim non-matching rows across Errors/Timeline/Overview; global focus banner with quick actions; Escape exits.
- **Timeline:** brushable histogram (`tlBrush` drag-to-zoom), swimlane-by-type panel, grouped event log (GAP 1500ms, MIN_GROUP 3).
- Known dead code: `DIAG_WF` / `classifyDiag` (the Diagnostic Priority Map UI was removed by product decision; the helpers have no callers — safe to delete in a cleanup pass).

## Motion System (2026 pass)

- Easing/durations: `var(--ledger-ease)` `cubic-bezier(.2,.8,.2,1)`, 120/200/400ms. No bounce.
- `useCountUp` / `<CountUp>` — KPI numbers; re-animates on value change; instant under reduced motion.
- `useSettleFlash(status)` + `.bi-skel` shimmer — loading→ok balance transitions; settle flash is NEUTRAL white, not green.
- `.bi-check-draw` — checkmark stroke draw, gated to `vsev==='ok'` ONLY. Hard rule: no playful/celebratory motion on warning/critical verdicts or anywhere drain findings render.
- Severity = motion: only critical pulses (once); amber/info stay still.
- `prefers-reduced-motion` is supported via a global CSS block + `prefersReducedMotion()` JS gate in rAF hooks. Any new animation must respect both. No `box-shadow` inside `@keyframes` (pulses use outline/opacity/transform).

## Key Helpers

| Helper | Purpose |
|---|---|
| `fmtFiat(v, full?)` | THE fiat formatter (see invariants) |
| `fmtDur(sec)` / `fmtTime(ts,{ms})` | Duration / timestamp formatting |
| `MF` / `T` | Legacy mono stack / theme object (Customer View only) |
| `DN` | Device name map (nanoS, nanoSP, nanoX, stax, europa→Flex, apex→Nano Gen5) |
| `llLabel(dev, isMobile)` / `llText(text, dev)` | "Ledger Wallet"/"Ledger Live" branding |
| `chainIconUrl(id)` | CDN URL for chain icon |
| `jumpTo(li)` | Navigate to Timeline + scroll to entry |
| `goToAcct(addr)` | Navigate to Accounts with filter |
| `sevColor(s)` | Severity → color |
| `cvFiatValue(appJson, cid, raw)` / `cvFmtFiat(v, appJson)` | Customer View fiat (app.json currency) |
| `useCountUp` / `useSettleFlash` / `prefersReducedMotion` | Motion hooks |

## Guides Drift Warning

`GUIDE_AGENT` (~line 471) and `GUIDE_TECHNICAL` (~line 949) are embedded in ledger-toolkit.html and are the single source of truth (the standalone `agent-guide.html` is deprecated — do not edit it back to life).

**Hard rule learned the expensive way: never document a feature that does not exist in the code.** Before adding any guide sentence describing UI behavior, grep for the implementing code. Two rounds of fixes were spent removing guide claims about a nonexistent Accounts-tab hover popover and EVM account grouping.

## Making Changes

**Small targeted edits (1–3 changes, same logical section):** Use `Read` + `grep` + `Edit` directly. Faster than spawning an agent.

**Large architectural changes (new sections, layout restructuring):** Use a 3-agent sequential team:
1. Scaffold agent — structural skeleton and layout wiring
2. Implementation agent — fills the main component (blocked by scaffold)
3. Enhancements agent — empty states, keyboard shortcuts, polish (blocked by implementation)

Sequential agents avoid merge conflicts in a single-file codebase. Parallel agents only work if they have clearly non-overlapping line ranges.

Always give agents: design tokens, the DO NOT MODIFY list, the formatting/ERR_DB invariants above, existing helper names, and exact field names when known. After every pass, re-run the stale-base greps plus: rejections still `medium` (7 entries), `fmtFiat(` ≥17, guide overclaims 0.
