Review the current state of ledger-toolkit for issues across three areas:

1. **Security** — XSS risks from user-provided log data, unsafe innerHTML, eval usage, CDN integrity
2. **Performance** — large log file handling (10k+ entries), unnecessary re-renders, memory leaks in XpubView scanner
3. **Correctness** — chain explorer URL mappings, error diagnosis logic coverage, account ID parsing edge cases

For each finding, include:
- Severity (critical / warning / info)
- File and function/section affected
- Suggested fix

Combine everything into a single prioritized report.
