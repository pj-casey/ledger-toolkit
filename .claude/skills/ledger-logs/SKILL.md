---
name: ledger-log-format
description: When parsing, analyzing, or debugging Ledger Wallet export logs
---

# Ledger Wallet Log Format

Logs are exported via Ledger Live: Settings → Help → Export Logs (or Ctrl+E).

## Entry structure
Every entry has: type, message, timestamp, level, data (payload object, varies by type)

Common types: analytics, bridge, network, action, persistence, walletsync, countervalues, error, live-dmk-logger

## exportLogsMeta entry
Special entry with message="exportLogsMeta" containing:
- release: app version
- git_commit: build commit
- userAnonymousId: hashed user ID
- accountsIds: array of all account ID strings

## Account ID format
`js:2:{currencyId}:{address}:{derivationMode}`

For UTXO chains (bitcoin, litecoin, dogecoin), the address field is an xpub/ypub/zpub.
For account-model chains (ethereum, solana, etc.), it's the actual address.

## Sync success events
- type=analytics, message contains "SyncSuccess" (not "AllAccounts")
- data.currencyName, data.operationsLength, data.duration, data.tokens[]

## Device info extraction
Found across multiple analytics entries in data fields:
- modelId: nanoS | nanoSP | nanoX | stax | europa | apex
- deviceVersion: firmware version
- osType + osVersion: host OS
- totalAccounts, totalOperations, accountsWithFunds, tokenWithFunds

## Common error signatures
| Match string | Category | Severity | Meaning |
|---|---|---|---|
| TransportStatusError | hardware | high | Device communication failed |
| DisconnectedDevice | hardware | high | USB connection lost |
| ManagerDeviceLockedError | hardware | medium | PIN not entered |
| CantOpenDevice | hardware | high | Another app using device |
| GenuineCheckFailed | hardware | high | Escalate if persistent |
| SyncError | software | medium | Retry sync |
| LedgerAPI4xx | software | low | Price API, safe to ignore |
| FETCH_ERROR | server | low | Background request, safe to ignore |
