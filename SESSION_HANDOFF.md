# Session Handoff — April 8, 2026 (Session 12)

## Current state

- **Branch:** `main` (working branch)
- **Local HEAD:** `da6f310` — Merge remote-tracking branch 'origin/main'
- **origin/main:** in sync with local
- **origin/experimental:** ahead of main (experimental-only commits), but all shipped features are on main
- All three files (`ledger-toolkit.html`, `agent-guide.html`, `technical-reference.html`) are identical across local main and origin/main

---

## What was built this session — Drain Detection

All changes inside `function CVAgentInsights`. Data layer untouched.

### Drain classification (per-account)
- `isDrained` flag: live balance dropped >90% vs cached, account had real value (>$50 fiat or >0.001 crypto)
- Staking guard: sub-account fiat sum ≥50% of account fiat → not a drain (liquid staking protection: stETH, rETH, etc.)
- Native staking already handled via `hasStaking` → `compareBalance = custSpendable`
- Drained accounts: badge = DRAINED, title = "DRAINED — name", drain-specific detail + escalation action

### Drain summary finding (`kind:'drain'`)
- Cross-account aggregation after forEach loop — `f.unshift()` to top of findings
- Extracts OUT operations from `ajOperations`, scans token sub-accounts
- Deduplicates recipient addresses case-insensitively; filters customer's own addresses
- Single drain → "DRAIN DETECTED"; 2+ drains → "SEED COMPROMISE"
- Forensic copy report: customer identity, per-account drain txs with explorer URLs, attacker addresses

### Drain detail panel (renderDetail early return for kind:'drain')
- Red header: total loss, chain summary, drain time window
- Escalation action box (red)
- Staking chain warning (amber, 🥩): shown when drained chains support native staking (STAKING_APY lookup). Solana-specific stake account note included.
- Fund flow visualization: left (victim accounts, chain-colored) → gradient arrows → right (attacker addresses, red-tinted). Multi-chain convergence highlighted. Tx + address explorer links.
- "Funds sent to" full address list
- Per-account breakdown with drain transactions
- "Copy drain report for escalation" button

### Guide updates (both embedded + standalone files)
- GUIDE_AGENT: new playbook scenario "I was hacked / drained", drain detection paragraph in Agent Insights section, TOC entry
- GUIDE_TECHNICAL: new Drain Detection section, `kind:'drain'` in finding types, TOC entry
- Both guides: cross-guide href anchor links converted to prose references (broken in separate overlay instances)
- Stale content fixed: "ambient gradient", line count (~8,400), popover description

---

## Previous session (Session 11 — April 8 earlier) built:
- Complete Diagnostic view reskin: Inter font, flat surfaces, pill nav, no decorative gradients
- Color unification: `#1A1A1D` everywhere for bg/panel
- SectionHeader rewritten as pill nav item
- All `textTransform:'uppercase'` removed (except MOBILE badge)
- `fontFamily:MF` removed from all non-data-value elements

---

## Architecture reminder (for fresh context)

- Single file: `ledger-toolkit.html` (8,477 lines), React 18.3.1 + Babel, no build step
- **Data layer is frozen** — parseLogs, ERR_DB, CHAINS, all fetch/compare logic — do not touch
- Two modes: `viewMode='diagnostic'` (Diagnostic) and `viewMode='customer'` (Customer View)
- Fixed viewport: root `height:100vh,overflow:hidden`, page never scrolls

## Workflow note
To bring changes from experimental → main:
1. `git checkout main`
2. `git checkout experimental -- <files>`
3. `git add <files> && git commit`
4. `git checkout experimental`
5. User runs `! git push origin main` (and `! git push origin experimental` if needed)
