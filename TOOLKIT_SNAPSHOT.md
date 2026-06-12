# TOOLKIT_SNAPSHOT.md

Current state of the Ledger Diagnostic Toolkit. Last updated: June 2026, on
promotion of the BI-dashboard redesign to production (`ledger-toolkit.html`,
~10,200 lines, single-file React app, no build step).

## What it is

A client-side forensic tool for Ledger customer-support agents. An agent drops
a customer's log export (.json/.txt from Ledger Wallet / Ledger Live, desktop
or mobile) and optionally the customer's `app.json`; the tool parses, diagnoses,
and visualizes the session. Nothing is uploaded — parsing is entirely in the
browser; only account addresses leave the machine (for live balance checks
against public chain APIs).

## Modes

- **Diagnostic** (redesigned 2026): fixed-viewport BI dashboard, tab-bar nav.
  Tabs: Overview · Errors · Accounts · Timeline · Network · APDU · Raw.
- **Customer View** (legacy layout, unchanged): replicates the customer's
  Ledger Wallet UI from app.json — Portfolio, Accounts, Earn, My Ledger —
  plus Agent Insights (automated findings engine incl. drain/seed-compromise
  detection).

## Diagnostic capabilities (current)

- **85-pattern error knowledge base** with plain-English diagnosis + concrete
  next step per error, severity classification, support-article links.
  Follows Ledger's canonical rule: user rejections (6985/5501/refused classes)
  are amber "user choice," never red/critical. Covers DMK-era (LW 4.x) codes.
- **Overview**: verdict banner, 2×2 customer-setup quadrants, issues-by-impact,
  activity & infrastructure, portfolio mini-treemap + top accounts, session
  timeline with chronological key moments. Firmware/app badges only claim
  "on latest" after live verification.
- **Live enrichment**: balances and prices fetched per-account from public
  chain APIs (60-chain registry), compared against log-derived state; cached
  per-token in localStorage with TTL.
- **Focus Mode**: pick an account → every section dims non-matching evidence;
  global banner with per-account metrics and quick actions.
- **Timeline**: brushable (drag-to-zoom) density histogram, swimlane by event
  type, grouped event log with payload expansion. Lists cap at 500 rendered
  rows for performance.
- **APDU**: frame stream with decoded view (raw hex / decoded fields / paired
  command-reply / session context), rejection summary, copy hex.
- **Network**: KPI tiles (calls, failures, 5xx/4xx), hosts ranked by volume,
  call log.
- **Raw**: JSON tree with progressive search, match counter, keyboard nav,
  click-to-copy path.
- **Copy reports**: four formats (summary, full, customer-facing, errors-only)
  with locale-safe money formatting; per-swap "Check live status" chip that
  copies a ready-to-run `wallet-cli swap status` command for stuck swaps.
- **Log quality score** with per-component breakdown.
- **Embedded guides**: agent guide + technical reference live inside the app
  (single source of truth; standalone agent-guide.html is deprecated).
- **Motion system**: count-up KPIs, balance shimmer/settle, clean-session
  check draw, severity-gated pulses; full `prefers-reduced-motion` support.

## Known limitations / honest notes

- Manager catalog (installed-apps inventory) appears only when the customer's
  session had a live connection to Ledger's servers; offline sessions show
  "unavailable" with that explanation.
- Live balances cover the 60-chain registry via public APIs; rate limits are
  retried/staggered but third-party APIs can still fail.
- Network latency/size columns don't exist because Ledger logs don't record
  them.
- Dead code earmarked for removal: `DIAG_WF`/`classifyDiag` (Priority Map UI
  was removed by product decision).

## Deferred backlog (deliberate, not forgotten)

- Tier-2 ERR_DB candidates (device-busy, secure-channel approval, app-open
  timeout/cancel, multiple-devices) — pending validation against a real LW4
  log corpus before the matchers are finalized.
- "Slow device confirmation" heuristic from Ledger's canonical timing bounds
  (sign 60s, app-open 30s) — same dependency.
- Timeline cursor "datum line" (motion-spec item 4) — skipped, low value vs.
  existing hover tooltips.
- CLI balance-seeding bridge (proven viable in experiment: a log address can
  be seeded into `wallet-cli` sessions to pull Ledger-grade balances/operations
  for BTC/ETH/SOL with no device; unsupported session-file trick, fragile
  across CLI updates — optional supplement only).
- Embedded Claude assistant tab — fully specced, shelved pending API access.

## Provenance

Redesign promoted June 2026 after multi-round verification: feature-parity
A/B against the previous production build, prototype-fidelity audit against
the original design canvas (~85–90% of artboard elements built; several
exceeded), ERR_DB audit against Ledger's official `ledger-dmk-implementation`
skill (commit b8fae08), static debug pass (state-reset coverage, listener/
parse/storage guards, render caps), and a hands-on browser smoke test.
Previous production build retained in git history as rollback.
