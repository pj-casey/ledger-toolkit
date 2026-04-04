# Ledger Diagnostic Toolkit

> One-file diagnostic dashboard for Ledger customer support. Drop a log, see everything.

**[Open the tool →](https://pj-casey.github.io/ledger-toolkit/ledger-toolkit.html)** · **[Agent Guide →](https://pj-casey.github.io/ledger-toolkit/agent-guide.html)**

![Version](https://img.shields.io/badge/version-4.0-BBB0FF?style=flat-square)
![React](https://img.shields.io/badge/React-18.3.1-61DAFB?style=flat-square&logo=react&logoColor=white)
![Single File](https://img.shields.io/badge/single_file-HTML-F57375?style=flat-square)
![Chains](https://img.shields.io/badge/chains-60+-7AC26C?style=flat-square)
![Errors](https://img.shields.io/badge/error_patterns-82-FFBD42?style=flat-square)

---

<!-- Replace with actual screenshot: load a log, take a screenshot of the Overview, save as screenshot.png in the repo root -->
![Toolkit Screenshot](screenshot.png)

---

## What it does

A CS agent receives a Ledger Wallet log export from a customer. They drop it into this tool. In under 5 seconds they see:

- **What device** — model, firmware, installed apps (what's outdated, what's missing)
- **What accounts** — live on-chain balances and USD values, fetched automatically from 40+ blockchains
- **What went wrong** — 82 error patterns diagnosed with severity, actions, and causes
- **Everything else** — full timeline, network requests, device communication, raw JSON search

No install. No build step. No API keys. One HTML file in a browser.

## Quick start

```
1. Open ledger-toolkit.html in any browser
2. Drag a customer's log file onto the drop zone
3. Read the diagnosis
```

Accepts `.json`, `.txt`, and `.log` files exported from Ledger Wallet (desktop or mobile).

## Features

### Diagnostic Mode

| Section | What it shows |
|---|---|
| **Overview** | Health pills, stat cards (Entries / Accounts / Portfolio / Issues), unified device card with app status chips, environment data, adaptive error grid, session info, quality score |
| **Issues** | Master-detail error list sorted by severity. Error timeline with clustering analysis. Affected account links. Copy Errors button |
| **Accounts** | Live blockchain balances + fiat USD. Health squares. Explorer links (60+ chains). Token badges. Xpub Scanner (BTC/LTC/DOGE/BCH) |
| **Timeline** | Every event, chronological. Density strip. Search + type filter. Expandable raw data |
| **Network** | HTTP requests with error summary |
| **APDU** | Device communication. Rejection codes decoded |
| **Raw JSON** | Full log tree with progressive search, expand/collapse, path copy |

### Customer View

Shows the customer's wallet as they see it. Load `app.json` for exact balances, transaction history, and account names. Encrypted-safe parsing for Ledger Sync users.

### Live On-Chain Balances

Fetches balances automatically on log load. 40+ chains: all EVM (Ethereum, Polygon, BSC, Arbitrum, Optimism, Base…), BTC, LTC, DOGE, BCH, SOL, XTZ, XRP, ADA, NEAR, TRX, TON, ATOM, ALGO, APT, SUI, STX, XLM, HBAR, EGLD, KAS, FIL. Fiat prices via CoinGecko. Portfolio total on Overview. Sidebar fiat hints.

### Mobile Log Support

Ledger Wallet Mobile (iOS/Android) logs fully supported. Automatic format normalization. MOBILE badge in header. No-sync amber visibility treatment prevents misreading "no data" as "empty."

### Copy & Export

| Format | Use case |
|---|---|
| **Quick Summary** | Ticket notes — device, top accounts, top errors |
| **Full Export** | Escalation — everything, all accounts, all errors |
| **Customer Summary** | Account-focused — balances and portfolio for the customer |
| **Copy Errors** | Error investigation — severity, actions, causes |

All formats include live balances when available.

### Keyboard Shortcuts

| Key | Section |
|---|---|
| `Ctrl+1` | Overview |
| `Ctrl+2` | Issues |
| `Ctrl+3` | Accounts |
| `Ctrl+4` | Timeline |
| `Ctrl+5` | Network |
| `Ctrl+6` | APDU |
| `Ctrl+7` | Raw JSON |

Mac: use `⌘` instead of `Ctrl`.

## Architecture

```
ledger-toolkit.html (~3,700 lines)
├── React 18.3.1 + Babel standalone (no build step)
├── bitcoinjs-lib + bs58 + buffer (CDN, for Xpub scanning)
├── Fixed viewport: 100vh, no page scrolling
├── Live network calls: public blockchain RPCs + CoinGecko
└── Data layer: 82 errors · 60 chains · 45+ tx explorers · 40+ balance APIs
```

## Files

| File | Purpose |
|---|---|
| `ledger-toolkit.html` | The tool |
| `agent-guide.html` | Investigation guide for CS agents (v4.0) |
| `toolkit-release-notes.md` | Full release history |
| `CLAUDE.md` | Technical reference for AI-assisted development |
| `SESSION_HANDOFF.md` | Session context for continuity between AI sessions |
| `ROADMAP_LIVE_BALANCES.md` | Live balance feature spec |

## Built with

This tool is built using an AI-assisted workflow:

- **[Claude](https://claude.ai)** — architecture, design review, prompt authoring
- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — implementation from `.md` spec prompts
- **Branch model:** `experimental` → `main` on release
- **Core rule:** *Change how things look, never how things work.* Data layer is frozen.
