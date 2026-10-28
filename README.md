# credit-card-reconciliation

An agent skill for Claude Code, Codex, Cursor and other agents that support `SKILL.md`. It reconciles your credit card statements against your email receipts.

## What it does

- Finds each card's statement email and unlocks the password-protected PDF, following the bank's own password rules.
- Lists every charge above a threshold you choose and checks that the extracted figures add up to the statement's printed totals.
- Matches each charge to the order, booking, invoice or receipt email that explains it, reading the full HTML of each email.
- Flags mismatches, duplicates, refunds that never arrived and fees the bank didn't reverse.
- Asks you to approve unmatched and direct payments with tick-box cards.
- Keeps a ledger, a per-card coverage file so no day is skipped or checked twice, and a monthly spending report.
- Runs once, or every month after your statements arrive.

## Safety model

- The mailbox is **read-only**. The agent never sends, deletes, archives or labels anything.
- All financial data stays in one local folder with owner-only permissions.
- Passwords and personal identifiers are collected through masked input or environment variables. They never appear in chat, files or logs.
- Nothing is marked "matched" without an email whose amount agrees.

## Install

Clone into your agent's skills directory:

```bash
# Claude Code
git clone https://github.com/subhash-arabhi/credit-card-reconciliation ~/.claude/skills/credit-card-reconciliation

# Codex
git clone https://github.com/subhash-arabhi/credit-card-reconciliation ~/.codex/skills/credit-card-reconciliation

# Cursor
git clone https://github.com/subhash-arabhi/credit-card-reconciliation ~/.cursor/skills/credit-card-reconciliation
```

Then ask your agent something like *"reconcile my credit card statements against my email receipts"*.

## Requirements

- A mailbox connector or tool your agent can use to read email (for example Gmail).
- Python with `pikepdf` or `pypdf`, and `pdftotext` (poppler). The skill installs whichever are missing.

## License

[MIT](LICENSE)
