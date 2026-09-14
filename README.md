# n8n workflows

Finance-operations automation built in n8n. Each one runs end to end on seeded demo data
with **no credentials to set up**, so you can import it and press Execute rather than take
my word for it.

The demo data in each is deliberately broken in specific ways, so every branch fires on
every run instead of only the happy path.

| | what it does | the number |
|---|---|---|
| [**Invoice Processing & 3-Way Match**](ap-3way-match/) | Accounts payable. Validates each invoice, blocks duplicates, matches it against the purchase order *and* the goods receipt, applies a price tolerance, then routes by value to touchless posting, one approver, or an exception queue owned by a named team. | 9 invoices → 1 posted with no human touch, 2 to an approver, 6 held with the reason and the owning team attached |
| [**AR Collections & Dunning**](ar-collections/) | Accounts receivable. Ages the book, holds back anything disputed or under a live promise to pay, escalates broken promises, respects a contact cadence, and chases per **customer** rather than per invoice. | 12 open invoices → 4 emails, not 12. Run it twice and the second run correctly sends nothing |

## What these are meant to show

**The rules live in one place and in a deliberate order.** In both pieces a single Code node
holds every rule that decides whether money moves, applied top to bottom. You can read the
decision without tracing wires across a canvas.

**Every reason *not* to act comes before the reasons to act.** A disputed invoice is never
chased. A duplicate is caught before anyone wastes time matching it. Getting that order
wrong is how finance automation does more damage than doing nothing.

**Not acting is recorded too.** Restraint that leaves no trace is indistinguishable from a
broken workflow.

**They are idempotent.** Both write to state they later read, so both were run twice before
being believed. Two of the three real defects found during the AP build were invisible on
the first run; each README says what they were.

**The destination nodes are real nodes, switched off.** Xero, Slack and Gmail appear where
the output would actually land, wired as side branches so the data path is unchanged whether
they are on or off. No credential is needed to run the demo, and nothing is faked with an
HTTP node pretending to be an integration.

## One manual step on import

n8n Data Table IDs are per-instance. After importing, open the Data table nodes and re-pick
each table from the dropdown. Each folder's README lists the tables and columns it needs.
