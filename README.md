# n8n workflows

Finance-operations automation built in n8n. Each one runs end to end on seeded demo data
with **no credentials to set up**, so you can import it and press Execute rather than take
my word for it.

The demo data in each is deliberately broken in specific ways, so the exception paths fire
alongside the happy path instead of only the happy path.

| | what it does | the number |
|---|---|---|
| [**Vendor Onboarding & Bank Change Control**](vendor-control/) | Vendor master data. Screens new vendors and bank-change requests through seven controls — incomplete data, invalid NPWP, sanctions, duplicates searched by name, tax ID **and bank account**, account-name mismatch, segregation of duties, and request channel — then takes a phone verification back in before any bank change is applied. The other half of the payment run: that one holds payments to a recently changed account, this one decides whether the change reaches the vendor master at all. | 10 requests → 2 approved, 6 held, 2 refused. The callback then catches a forged bank change: treasury phones the vendor on the old number, the vendor denies asking, and the master stays untouched |
| [**Invoice Processing & 3-Way Match**](ap-3way-match/) | Accounts payable. Validates each invoice, blocks duplicates, matches it against the purchase order *and* the goods receipt, applies a price tolerance, then routes by value to touchless posting, one approver, or an exception queue owned by a named team. | 11 invoices → 1 posted with no human touch, 2 to an approver, 8 held with the reason and the owning team attached — including 2 caught billing a goods receipt that was already fully invoiced |
| [**Payment Run & Bank Disbursement**](payment-run/) | Treasury. Turns approved invoices into a bank payment file behind seven controls: missing bank details, sanctions, already paid or already in an unanswered file, **recently changed bank account**, missing or insufficient approval, below-minimum, and cash position. Builds the file with a control total from stored lines only, then checks the bank reply against what was actually sent. | 14 invoices → 5 paid (548.484.800 IDR), 9 held, every control firing — including a 1,45 milyar invoice the director had approved, whose vendor bank account had changed 3 days earlier, by email. Run again before the bank replies, it pays nothing |
| [**Bank Reconciliation**](bank-reconciliation/) | Treasury and receivables. Checks the bank statement against the books **in both directions** — every statement line must be explained, and every payment claimed as settled must appear as a debit. Verifies the statement foots, finds one transfer paying several invoices, caps overpayments, parks unknown money in suspense instead of revenue, writes cash back to AR, and refuses a suspense resolution that would apply the same money twice. | 12 statement lines and 4 sent payments → 6 matched, 3 AR follow-ups, 1 suspense, 3 critical — including a payment the books called rejected that the bank had taken, and one called settled that never left |
| [**AR Collections & Dunning**](ar-collections/) | Accounts receivable. Ages the book, holds back anything disputed or under a live promise to pay, escalates broken promises, respects a contact cadence, and chases per **customer** rather than per invoice. | 13 open invoices → 4 emails, not 13. Every assessed invoice is accounted for in the log, chased or explicitly held, and the report proves it from the rows the table returned |
| [**Employee Onboarding Coordinator**](hr-onboarding/) | People ops. Derives each new hire's tasks from a checklist **held as data**, creates what is missing, and chases each owning team once with everything it owes — while telling the manager who will not be ready on day one. | 6 hires × a 13-row checklist → the exact 42 tasks those people need, 8 of them created automatically; 5 team digests drafted, 2 held back |

## What these are meant to show

**The rules live in one place and in a deliberate order.** In the pieces that decide whether
money moves, one Code node holds the rules for the main run, applied top to bottom; the bank
reconciliation's suspense webhook has its own. You can read the decision without tracing wires
across a canvas.

**Every reason *not* to act comes before the reasons to act.** A disputed invoice is never
chased. A duplicate is caught before anyone wastes time matching it. Getting that order
wrong is how finance automation does more damage than doing nothing.

**Not acting is recorded too.** Restraint that leaves no trace is indistinguishable from a
broken workflow.

**They are built to be run again.** Most of them write to state they later read, so running
them twice back to back is part of testing them. Several of the real defects found while
building were invisible on the first run, and the runs were green when they were found — they
turned up by counting the items leaving each node. Each README says what they were.

**Configuration is data where it should be.** The onboarding checklist is a table, not a Code
node, so the people who own the process can change it without touching a workflow.

**The destination nodes are real nodes, switched off.** Xero, Slack and Gmail appear where
the output would actually land, wired as side branches so the data path is unchanged whether
they are on or off — except in onboarding, where the team email sits inline on purpose so the
chase log records whether a message really went out. No credential is needed to run the demo, and nothing is faked with an
HTTP node pretending to be an integration.

## Manual steps on import

1. Create the Data Tables each piece uses — importing a workflow does not create them. The
   payment run README lists all of its tables with their columns; for the others, the Data
   table nodes show which table and columns each step reads or writes.
2. n8n Data Table IDs are per-instance. After importing, open the Data table nodes and re-pick
   each table from the dropdown.
