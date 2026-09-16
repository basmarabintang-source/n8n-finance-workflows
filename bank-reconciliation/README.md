# Bank Reconciliation — n8n

The workflow that checks whether the money actually did what the books say it did.

Every other piece in this repo ends at a *claim*: an invoice approved, a payment sent, a bank reply
saying "settled". Reconciliation is where those claims meet the bank statement — and it reads from
three of the other workflows at once: payments sent from the [payment run](../payment-run/), and
open invoices and customers from [AR collections](../ar-collections/). It is the first piece that
puts the payables side and the receivables side in one view.

It runs on seeded demo data with no credentials, and every check fires on the first execution.

> **Updated 16 September 2026. The sample output and the sections below still describe the
> previous version, and will be rewritten from a fresh run.** Checking this README against the
> workflow found real defects, now fixed in `workflow.json`:
>
> - **The suspense webhook could apply the same cash twice.** Re-posting a resolution applied it
>   again, and several assignments in one request could together apply more than the transfer was
>   worth. Each assignment is now checked against the money on its own statement line, including
>   what earlier requests already applied, and a line that was never in suspense is refused.
> - **A request in which every assignment was refused got no reply at all.** The summary now runs
>   on every request.
> - **A payment that matched more than one combination of invoices was applied to the first one
>   found.** It now goes to suspense as `AMBIGUOUS_COMBINATION` instead of being guessed.
>
> An overpayment's excess is no longer described as "recorded as customer credit": nothing records
> it, so the reason text now says the excess is unapplied and AR decides what happens to it.

```
*Rekonsiliasi Bank STM-20260916* · 2026-09-16

*Integritas rekening koran*
Saldo awal 900.000.000 + kredit 328.000.000 − debit 503.179.000 = 724.821.000
Saldo akhir tercatat 724.821.000 — COCOK
Saldo berjalan meleset di 1 baris: baris 6 tercetak 621.915.800, seharusnya 621.905.800.
Saldo akhirnya tetap benar, jadi ini salah cetak kolom, bukan transaksi hilang.

Cocok        : 6 · 699.094.200 IDR
Perlu tindak : 3 · 110.000.000 IDR
Kritis       : 3 · 64.984.800 IDR
Suspense     : 1 · 7.500.000 IDR

*KRITIS — posisi uang tidak pasti*
- [REJECTED_BUT_DEBITED] baris 8 · 6.000.000 · CV BERKAH JAYA MANDIRI
  Catatan pembayaran menyatakan transfer TRX-REC-003 DITOLAK bank, tapi rekening
  koran menunjukkan 6.000.000 benar-benar keluar. Vendor bisa dibayar lagi.
- [DUPLICATE_LINE]       baris 11 · 8.584.800 · PT SINAR BAJA ELEKTRIK
- [MISSING_DEBIT]        50.400.000 · Mitra Logistik Nusantara
  Tercatat LUNAS, tapi tidak ada debit apa pun di rekening koran.

*Cocok*
- [MATCH_COMBINED] 30.500.000 → INV-7004 + INV-7003
```

## Two directions, not one

A reconciliation that only asks *"whose is this statement line?"* misses half the problem. This one
also asks the reverse: **for every payment we claim settled, is there a debit at the bank?**

Those two directions produce the two most expensive findings in the run above:

- **`REJECTED_BUT_DEBITED`** — our records say the transfer failed, the bank took the money anyway.
  The vendor now looks unpaid and will be paid again.
- **`MISSING_DEBIT`** — our records say paid, and no money ever left. The vendor is still waiting
  while the books call the invoice settled.

Neither is visible from either side alone. You only find them by holding the ledger against the
statement in both directions.

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 39 nodes (35 functional + 4 sticky notes) |
| `reset.json` | rebuilds the 12-line statement and 4 sent payments, 10 nodes |

## Running it yourself

No credentials needed.

1. Import `reset.json` and `workflow.json`, plus [AR collections](../ar-collections/)' `reset.json` —
   reconciliation reads the AR invoices and writes cash back to them.
2. Re-pick each Data table node from its dropdown; table IDs are per-instance.
3. Run **Reset AR Demo**, then **Reset Demo Rekonsiliasi**, then **Rekonsiliasi Rekening Koran**.

The reset clears only its own payment rows (`run_key = REC-DEMO`), so it does not disturb the
payment run's demo.

## What it checks

**Statement integrity, first.** Opening balance plus credits minus debits must equal the stated
closing balance. If it does not, nothing below can be trusted. Separately, every line's running
balance is recomputed: the demo statement has one mistyped balance on line 6, which is flagged
precisely as a column misprint — the closing balance is still right, so no transaction is missing.

**Then each line, in order:**

1. **Already reconciled** in a previous run — skipped, never applied twice.
2. **Duplicate line** — the same reference and amount appearing twice on one statement.
3. **Bank fee** — small debits described as charges, posted straight to bank charges.
4. **Outgoing** — matched to a sent payment by bank reference, then checked for status
   (`REJECTED_BUT_DEBITED`) and amount (`AMOUNT_DIFFERS`). Anything unmatched is
   `UNEXPLAINED_DEBIT`, the most urgent finding there is.
5. **Incoming with an invoice number** — exact, `PARTIAL`, `OVERPAYMENT`, or `DISPUTED_PAID`.
6. **Incoming without one** — the sender is matched to a customer, then the workflow searches that
   customer's open invoices for a combination that sums to the amount exactly.
7. **Everything else** — suspense.

### One transfer, several invoices

CV Anugerah Teknik sent 30,500,000 with the reference `PELUNASAN TAGIHAN` — no invoice number. The
workflow finds the customer by name, then finds that `INV-7003 (8,500,000) + INV-7004 (22,000,000)`
is the only combination of their open invoices that sums to exactly 30,500,000, and splits the cash
across both. Customers pay like this constantly, and it is the case most matching rules give up on.

### Overpayments are capped, not credited

Bengkel Karya Mandiri paid 5,000,000 against 4,750,000 outstanding. The invoice is credited 4,750,000
— exactly its remaining balance — and the 250,000 excess is recorded as customer credit rather than
silently inflating the invoice or being booked as revenue.

### Suspense is not revenue

7,500,000 arrived from PT MAKMUR SEJAHTERA, which matches no customer. It is parked in suspense.
Booking unidentified money as income, then discovering it belonged to a customer, is a quick way to
make the accounts wrong in two places.

## Cash goes back to AR

Matched receipts are written to `AR Open Invoices` — the table [AR collections](../ar-collections/)
reads. A customer who paid this morning will not be chased tomorrow.

## The second entry point: resolving suspense

Once someone identifies the money in suspense, the assignment arrives on the webhook
`rekonsiliasi-suspense`. It is **checked, not applied blindly**: there must be a named resolver, the
invoice must exist, it must not be in dispute, and the amount may not exceed what is still owed.

```json
POST /webhook/rekonsiliasi-suspense
{
  "resolutions": [
    { "line_key": "STM-20260916-010", "invoice_number": "INV-7013", "amount": 7500000,
      "resolved_by": "rina.ar@example.com",
      "note": "PT Makmur Sejahtera adalah perusahaan induk PT Karya Bersama Sentosa." }
  ]
}
```

In testing, three assignments went in. One was applied. The other two were refused — one because
7,500,000 exceeded the 3,200,000 left on the invoice, one because the invoice was disputed — and the
AR table confirmed neither refused invoice was touched. Cash applied to the wrong invoice makes two
customers wrong at once.

## Four defects worth recording

All four were silent: green runs doing the wrong thing.

**No cash ever reached AR.** The node that splits receipts per invoice read its input from the
log write, which returns only mapped columns — so the array it needed was gone, it emitted zero
items, and not one rupiah was applied. It now reads from the merge node directly.

**Last month's payments looked missing.** The reverse check had no date window, so a payment settled
on 27 August was flagged as absent from a September statement. That kind of false alarm teaches
people to skim the critical section. It is now bounded to the statement's own date range.

**A second run accepted a duplicate debit as a real payment.** Already-reconciled lines were skipped
*before* being registered as seen, so on re-running, the original line vanished from the duplicate
check and its copy was matched as a legitimate payment.

**The same second run raised a false 480,000,000 alert.** Skipped debit lines never recorded their
bank reference either, so the reverse check concluded that a payment matched the day before had
never left the bank.

The last two only appear on a **second consecutive run**. A reconciliation is run every day against
overlapping data; one that is only correct on its first execution is not correct. Both were found by
running it twice and comparing the results line by line.
