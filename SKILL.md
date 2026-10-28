---
name: credit-card-reconciliation
description: Reconcile credit card statements against email receipts. Finds and unlocks password-protected statement PDFs in the mailbox, lists every charge above a threshold, matches each one to the order, booking, invoice or receipt email that explains it, flags mismatches, duplicates and unreversed fees, collects the cardholder's approval for anything unmatched, and maintains a ledger plus a monthly spending report. Use this whenever someone wants to check, audit, verify or reconcile credit card statements or charges, find unknown or suspicious card transactions, match card spend to receipts or orders, or set up a recurring monthly card spending review, even if they never say "reconciliation". Works as a one-off run or on a monthly schedule.
---

# Credit card statement reconciliation

## Purpose

For every card the person holds: verify each charge on the statement against the email that explains it, flag anything wrong, get the person's approval for whatever can't be matched, and keep a monthly spending report they can come back to.

The output is only valuable if the person can trust it. A wrong "matched" label is worse than no label. So the skill rests on three things:

- **Evidence:** a charge counts as matched only when you've opened an email whose amount agrees with it.
- **Continuity:** no day gets checked twice, and no day gets skipped.
- **Privacy:** statements and secrets never leave the person's machine.

## Workflow at a glance

Add these steps to your task list and work through them in order:

1. **Load state.** If the private folder already exists, read `playbook.md`, `config.json` and `coverage.json`. If `config.json` exists, skip Setup.
2. **Setup** (first run only). See §2.
3. **Fetch and unlock** each card's statement. See §3.
4. **Set the date range** for each card. See §4.
5. **Pick the transactions** to verify. See §5.
6. **Match** each one to an email. See §6.
7. **Get approvals** for anything unmatched or paid directly. See §7.
8. **Write the outputs:** ledger, CSVs, coverage file, report and chat summary. See §8.
9. **Run the pre-send checklist.** See §9.
10. **Add lessons** to the playbook. See §10.

## 1. Hard rules (apply to every step)

This skill works with someone's bank statements and mailbox. One sent email, leaked password or made-up match costs far more than any time it saves, so these rules always win.

- **The mailbox is read-only.** Never send, reply to, forward, delete, archive, label, move or mark anything. If a message to a bank or merchant would help (for example a dispute), draft it in chat or in a local file for the person to review and send themselves.
- **Never invent anything.** Don't make up a transaction, an amount or a match. "Matched" means you found an email and its amount agrees. If you didn't find one, say so plainly. Never guess.
- **Financial data stays local.** Keep everything in one private folder with owner-only permissions (`chmod 700` on the folder, `600` on the files). The default is `~/card-reconciliation-data/`; tell the person where it is. Don't paste statement contents into third-party tools, web services or online converters.
- **Secrets never appear anywhere.** Passwords, date of birth, tax IDs (PAN, SSN and similar), full card numbers and similar identifiers must never show up in chat, files, logs, scripts, commit messages or command output. Collect them through masked input (see "Collecting secrets" below) and read them from environment variables at runtime. Record the password *format*, never its value.

## 2. Setup (first run): ask before doing anything

Ask one question at a time. If your environment has a structured multiple-choice tool (choice cards, an ask-question tool), use it. If not, give numbered options in chat. Save the answers to `config.json` in the private folder. Never put secrets in it.

1. **Mailbox.** Which mailbox has the statements and receipts, and which mailboxes must NOT be touched? Confirm you have a working connector or tool for it. If you don't, help the person connect one before going further.
2. **Cards.** Which cards do they hold (bank, card name, last 4 digits)? Offer to discover them by searching the mailbox for statement emails, then confirm the list with them.
3. **Threshold.** What's the smallest charge worth verifying (for example 1,000 or 1,500 in their currency)?
4. **Schedule.** One-off, or monthly once all statements are out? If monthly, find the latest usual arrival day across all cards and schedule the run for the day after it.
5. **PDF passwords.** Don't guess, and don't ask yet. First read the statement emails (§3); they explain what the password is made of. Then ask only for those pieces, through secure input. Never ask in plain chat.
6. **Approval style.** Tick boxes (multi-select, up to about 6 items per card) or typed replies? Default to tick boxes.
7. **Report destination.** A page or dashboard updated each month (the default), or a file.

**Check your tools.** You need a PDF decryption library (for example `pikepdf` or `pypdf`) and a text extractor that preserves layout (for example `pdftotext` from poppler). Install whatever is missing. If a PDF "won't open", first rule out a missing or broken tool before you assume the password is wrong.

### Collecting secrets

- **Best option:** a masked secret-input tool from your environment.
- **If there isn't one:** ask the person to put each password piece in an environment variable themselves, in the shell where your commands run (or in their OS keychain), and to tell you only the variable names. Suggested names: `CC_NAME`, `CC_DOB`, `CC_PAN`.
- **Build the password inside the script**, following the format recorded in `notes.md`. Never echo or print it. Never pass it as a literal command-line argument, because that leaks into shell history and the process list. Example:

```python
import os, sys, pikepdf
# Format for this card (from notes.md): first 4 letters of name, uppercase + DDMM of DOB
pw = os.environ["CC_NAME"][:4].upper() + os.environ["CC_DOB"][:4]
with pikepdf.open(sys.argv[1], password=pw) as pdf:
    pdf.save(sys.argv[2])
```

## 3. Get and unlock the statements

- **Find each card's statement email** by searching for the bank's sender plus "statement". For each card, record in `notes.md`: the sender, the usual arrival day, the attachment name pattern and the password format. Record formats only, never values.
- **Find the password rule.** It's in the statement email itself, or in a bank FAQ the email links to. Read the full email, including its HTML part, and follow it literally:
  - which parts of the name to use
  - which letter case
  - which date format
  - the name as printed on the card
- **If unlocking fails**, work through these checks in order:
  1. Check that the tool opens an unencrypted PDF.
  2. Re-read the instructions in that exact email.
  3. Try the name exactly as it appears on the card, in the letter case the bank's example uses.
  4. Only then ask the person, and show them the bank's own wording.
- **Extract the text with layout preserved** (`pdftotext -layout`). Confirm the card's last 4 digits and the statement period from the PDF itself, not from the email.
- **Check every statement's totals.** The debits and credits you extracted must add up to the totals printed on the statement. If they don't, the extraction is wrong; fix it before going further, because every later step depends on it. Some banks print decoy figures in plain digits while the real amounts sit in an obfuscated font. If the totals don't add up, look for a second amount column or a font mapping that does.

## 4. Set the date range (continuity)

Keep `coverage.json` with the last verified date for each card, for example:

```json
{ "Example Bank Platinum 0000": { "verified_through": "2026-09-15" } }
```

- On each run, check each card from the day after its `verified_through` date up to the end date of its newest statement. Never skip days, and never check a day twice.
- If a card's new statement hasn't arrived yet, leave its range open, say so, and pick it up on the next run.
- Every report opens with a table showing, per card: checked from, checked to, and where the next check starts. Then add one line with the overall range.

## 5. Pick the transactions

- Take every debit at or above the threshold within the date range.
- **Refunded in full:** drop charges that were debited and later fully credited back (net zero). Mention them briefly in the notes only.
- **Return-linked debits** (for example an EMI or instalment pre-closure after a product return) aren't new spending. For each one:
  - pair it with the merchant's refund credit and the original order email
  - show the net amount
  - keep it out of spend totals by default
  - flag any fees, interest or tax the bank didn't reverse
- **Direct merchant payments** (local shops, individuals, in-store card terminals, anything that won't send an order email) go in their own section for approval, not under "unmatched".

## 6. Match each charge to an email

- Work out the brand from the merchant descriptor. Then search that brand's emails within about ±5 days of the charge: orders, bookings, invoices, payment confirmations and refunds.
- **Open every candidate email in full and read its HTML part,** not just the plain text or the search snippet. Many merchants put placeholder text in the plain-text part and keep the order details and amount paid only in the HTML. Don't stop after the first few results. This is the most common reason a charge that could be matched gets wrongly marked unmatched.
- Travel booked through aggregators often bills under the aggregator's descriptor. Match on the "amount paid" in the booking email.
- Match on the exact amount first. Small differences you can explain (forex markup, a convenience fee) are allowed, but label them `mismatch` and give the reason.
- Check alert emails from the person's other accounts for the same amount on the same day. Note any coincidence, but still ask the person; don't draw a conclusion.

Give each charge exactly one status:

| Status | Meaning |
|---|---|
| `matched` | Email found and the amount agrees. Record the email id and the item. |
| `mismatch` | The amount, date or merchant differs, or it's a duplicate, a refund that wasn't credited, or a charge after cancellation. Give the reason. |
| `unmatched` | Nothing found after a full search, including HTML bodies. |
| `direct` | A direct merchant payment, waiting for approval. |
| `disputed` | Raised with the bank. Check for the outcome on every run. |

## 7. Get approvals

- **What to ask about:** first the unmatched online charges, then the direct payments.
- **How to ask:** multi-select tick cards, up to about 6 items per card, one card at a time. Label each item "Merchant, amount, card, date" and include a free-text option. If you have no multi-select tool, list the items with numbers and ask the person to reply with the numbers they approve.
- **What the answers mean:** a ticked item is approved. Unticked items stay pending. Save any free text to `user_note`.
- **Record every answer.** If the person says a charge was disputed, mark it `disputed`. On later runs, look for a reversal or resolution email instead of asking again.
- **Carry forward** anything still pending and ask about it again on the next run.

## 8. Outputs

All files live in the private folder:

| File | Contents |
|---|---|
| `config.json` | Setup answers. No secrets. |
| `notes.md` | Per card: sender, usual arrival day, attachment name pattern, password format. Formats only. |
| `coverage.json` | Each card's verified-through date. Update it at the end of every run. |
| `ledger.csv` | Append-only master ledger (columns below). |
| `statements/<card>/<YYYY-MM>.csv` | One CSV per card per statement month. |
| `playbook.md` | Lessons learned (§10). |

**`ledger.csv` columns:** `statement_month, card, txn_date, amount, merchant_descriptor, category, status, matched_email_id, matched_item, notes, user_note`.

When appending, skip any row already in the ledger from an earlier run. If two identical charges really do appear on one statement, keep both and flag them; they may be a duplicate charge.

**Monthly report.** Edit the same page in place each month. Sections, in order:

1. Table of the period verified, per card (from §4)
2. Overview
3. All transactions, largest first, with a status filter
4. Totals by category, card and merchant, with charts
5. Matched
6. Unmatched online charges
7. Direct payments
8. Return-linked pairs (excluded from totals unless toggled on)
9. Discrepancies
10. Notes

**Chat summary,** in this order:

1. The date-range table
2. Totals and discrepancies, in a few lines
3. The report link
4. The tick cards

## 9. Pre-send checklist

Don't send the summary until every item is true:

- [ ] Every card has a statement for this run, or an explicit "not arrived yet".
- [ ] Every statement's extracted totals add up to its printed totals.
- [ ] Every charge from a known brand was searched, with full HTML bodies read.
- [ ] Every `matched` row has a `matched_email_id`.
- [ ] Fully refunded pairs are dropped, return-linked debits are paired, and fees the bank didn't reverse are flagged.
- [ ] Direct payments are in their own section.
- [ ] Each card's range starts exactly where the last coverage file left off.
- [ ] No secrets in any file, log, script or message.

## 10. Keep learning

Keep `playbook.md` next to the ledger and read it at the start of every run. After each run, add anything new you learned: a bank quirk, a merchant whose emails hide the details, a tool that failed and what fixed it. That way the next run doesn't repeat the same mistakes.

## 11. Monthly automation

If the person wants this monthly, schedule a run for the day after the last statement usually arrives. Use whatever scheduler your environment offers (agent scheduled tasks or automations, cron, launchd). If there isn't one, tell the person which date to rerun this.

Each scheduled run:

1. Reads `playbook.md` and `coverage.json`.
2. Runs §3 to §9.
3. Updates the report in place.
4. Sends the chat summary and the tick cards.
5. Adds new lessons to the playbook.

If no new statements have arrived, say so in one line and stop.
