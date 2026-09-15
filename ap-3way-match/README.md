# Invoice Processing & 3-Way Match — n8n

An accounts-payable pipeline that takes a batch of supplier invoices and decides, for each
one, whether it gets paid without anyone looking at it, needs one named approver, or stops
dead in a queue owned by the team that can actually fix it.

It runs. The demo batch is deliberately full of broken invoices, so every branch fires on
every execution and you can read the real output instead of taking my word for it.

```
11 invoices worth 1.663.745.000 IDR came in.

Posted with no human touch:  1 (8.760.000 IDR)
Waiting on an approver:      2 (1.500.400.000 IDR)
Held in the exception queue: 8 (154.585.000 IDR)

Touchless rate 9.1% - 1 of 11 invoices were posted with nobody touching them, worth about 7
clerk-minutes. The other 10 still cost a person their attention.
Ledger: 11 of 11 decisions written.

Billed against receipts that were already fully invoiced: 2 (50.785.000 IDR). Every one of
these would have been a second payment for goods already paid for.

Held, and who owns it:
- [QTY_MISMATCH]     PT Kemasan Prima INV-03      -> warehouse     - Billed 5000 but only 4600 received on GRN-88103 (short-shipped). Short by 400.
- [NO_RECEIPT]       Sinar Baja Elektrik INV-04   -> warehouse     - Nothing booked in against PO-2026-4405 yet.
- [PO_NOT_FOUND]     Berkah Jaya Mandiri INV-05   -> procurement   - No purchase order PO-2026-9999 exists.
- [VALIDATION]       Cahaya Teknik Presisi        -> ap_clerk      - no invoice number; amount zero; currency USD; unreadable date.
- [DUPLICATE]        Cahaya Teknik Presisi 8899   -> ap_supervisor (also OVER_CONSUMPTION) - Already accepted 2026-08-29 as posted.
- [OVER_CONSUMPTION] Cahaya Teknik Presisi INV-08 -> ap_supervisor (also PRICE_VARIANCE) - Billed 600 but only 0 of PO-2026-4404 is still
                                                                    open. 600 matchable less 600 already invoiced on the posted ledger.
- [OVER_CONSUMPTION] Sinar Baja Elektrik INV-10   -> ap_supervisor - Billed 35 but only 30 of PO-2026-4401 is still open (10 already
                                                                    invoiced earlier in this same batch).
- [PRICE_VARIANCE]   PT Kemasan Prima INV-11      -> procurement   - Billed 9.900.000 vs expected 9.400.000. Over tolerance by 312.000.
```

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 25 nodes |
| `reset.json` | empties the ledger and restores the one seeded payment, so the demo can be re-run from clean |
| `data/ap-purchase-orders.csv` | 6 purchase orders |
| `data/ap-goods-receipts.csv` | 5 goods receipts — note PO-2026-4405 has none, on purpose |
| `data/ap-invoice-ledger-seed.csv` | one already-paid invoice, so the duplicate guard has something real to catch |

## Running it yourself
> **Run `reset.json` before each demo run.** The goods receipts are drawn down as invoices
> are accepted, so a second run on the same data legitimately finds the purchase orders
> already consumed and holds almost everything. That is the draw-down working, not a fault,
> and the run report says so in as many words — but it is not the picture you want to open
> cold in front of someone.


No credentials needed. It uses n8n **Data Tables**, not an external database.

1. Create three Data Tables named `AP Purchase Orders`, `AP Goods Receipts` and
   `AP Invoice Ledger`, with the columns in the CSV headers. Types are string except
   `qty_ordered`, `unit_price`, `qty_received` and `amount`, which are number.
2. Import the three CSVs.
3. Import both workflow JSONs.
4. Open the four Data table nodes and re-pick the table from the dropdown. Data table IDs
   are per-instance, so the saved ones will not resolve on yours. This is the only manual
   step.
5. Open `ap-3way-match` and hit **Execute workflow**.

## The rules, in the order they are applied

One Code node holds every rule that decides whether money leaves the company. Order matters
and is deliberate — a duplicate is caught before anyone wastes time matching it, and nothing
is matched at all until the fields are readable.

1. **Validation** — vendor, invoice number, PO reference, a plausible amount, IDR, a readable
   date, and the line total agreeing with quantity × unit price.
2. **Duplicate** — has this vendor + invoice number already been accepted?
3. **PO exists**
4. **Goods actually received** — is there a goods receipt at all?
5. **Quantity** — billed vs received
6. **Over-PO** — billed vs ordered
7. **SKU** — invoice line vs PO line
8. **How much of that receipt is still open** — the goods receipt is *drawn down*, not
   re-read in full for every invoice. Quantity already invoiced, both on the posted ledger
   and earlier in the same batch, is netted off first: `open = min(ordered, received) −
   already invoiced`. Without this a receipt for 600 units can back an unlimited number of
   invoices, which is the exact failure a three-way match exists to prevent. It is checked
   **before** price, because a receipt billed twice is a double payment, while a price is
   something procurement can still renegotiate.
9. **Price** — line total vs PO price, within 2% or 50.000 IDR, whichever is larger. The
   absolute floor exists so a 2% tolerance on a small line does not reject over pennies.

An invoice can fail more than one of these. `exception_code` is the first by priority and
decides who owns it, but every other rule it failed is carried alongside and printed, so an
invoice routed to AP as a duplicate still tells procurement its price is wrong.

What survives is routed by value: under 10M posts itself, under 100M needs a manager, above
that a director. Payment terms are parsed at the same time, so `2/10 NET 30` produces a
pay-by date that captures the discount rather than quietly missing it.

## Where the output goes

Four destination nodes are included and **switched off**, wired as side branches so the
invoice data reaches the ledger by the same path whether they are on or off:

- **Xero** — auto-approved invoices become an `ACCPAY` bill, status `AUTHORISED`. That status
  is only defensible because of where the node sits: this branch carries only invoices that
  cleared the full three-way match under the touchless ceiling. Xero's `plannedPaymentDate`
  is fed from the discount date, so the early-payment saving is actually captured.
- **Slack** — the one approver who has to look at it, with the match evidence already in the
  message so nobody has to go and look anything up.
- **Gmail** — each exception to the team that owns it (warehouse, procurement, AP), not to one
  shared inbox nobody reads; plus a single run report to the finance lead.

Turn one on by attaching a credential and picking the organisation, channel or address.
Nothing else changes.

## Notes on the build

Three things were wrong in the first working version and are worth writing down, because two
of them were invisible until it had been run twice.

**A rejected duplicate overwrote the payment it was checked against.** The ledger upserted on
`invoice_key`, so the DUPLICATE row landed on the same key as the original posted payment.
After one run the ledger no longer knew the August invoice had been paid.

**A rejected invoice would come back as a duplicate.** A price-variance rejection was written
under its plain key, so once procurement fixed the PO and the vendor resubmitted, it would be
refused as already-processed.

Both are fixed by two independent guards, either of which alone would be enough: exception
rows get their own ledger key (`invoice_key + ' #' + exception_code`), and the duplicate index
is built only from rows that were actually accepted.

**Everything after the branches ran three times.** The three artifact branches fed the ledger
write directly, so the run summary was rebuilt three times and the webhook was answered three
times. A Merge node fixed it. The only tell was `previousNodeRun: 0,1,2` in the execution
data — nothing about the canvas looked wrong.

The lesson in all three: a workflow that writes to the state it reads has to be run twice
before it is believed.

## Deliberately not here

- **No authentication on the webhook.** It is a demo endpoint; the workflow ships inactive.
  Anything real gets header auth and a rate limit.
- **No partial-invoice tracking.** A vendor billing 300 of 600 received passes. Correct for
  this demo, wrong for production — that needs a running invoiced-to-date per PO line.
- **The Xero contact is the vendor name, not a ContactID.** Xero wants a GUID; a real
  deployment needs a contact lookup in front, or a cached vendor-to-ContactID map.
- **The "clerk-minutes saved" figure** is 7 minutes per untouched invoice. It is an assumption
  stated in the code, not a measurement.
