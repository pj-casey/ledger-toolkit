# Ledger Diagnostic Toolkit — QA Audit

Single-file React/Babel app (ledger-toolkit.html) for CS agents to 
diagnose customer issues from Ledger Wallet log exports.

## What the tool does
1. Agent drops a JSON log file onto the page
2. App parses entries, extracts device info, accounts, errors, network data
3. Overview shows: stats, device/app info, errors, session activity badges
4. Accounts tab shows all accounts with explorer links and xpub scanning
5. Errors tab shows diagnosed issues grouped by category
6. Timeline shows filterable log entries
7. Network tab shows HTTP requests
8. APDU tab shows device communication data
9. "Copy Summary" generates a plain-text diagnostic for support tickets

## Key features that MUST work
- Drag-and-drop file loading (JSON and TXT)
- Block explorer links for ALL chains (click → opens correct page)
- Cardano: hex credential → stake1 derivation → cexplorer.io link
- Stacks: informational message about address format
- BTC/LTC/DOGE: xpub address scanner with progress and stop button
- Token badges with explorer links (ERC-20, TRC-20, etc.)
- Error diagnosis with inline expandable details
- Session activity badges (Manager opened, Swap browsed, Device scan)
- Copy Summary button on Overview
- Keyboard shortcuts Ctrl+1-5 for tab switching
- Log Quality Score (collapsed by default, expandable)

## Architecture
- Everything is in ONE file: ledger-toolkit.html
- React 18 + Babel (CDN), bitcoinjs-lib, bs58, buffer
- No build step — opens directly in browser via `open ledger-toolkit.html`

## Test data
- Sample log files are *.txt files in this directory
- Load them by reading the file and verifying the parsed output
