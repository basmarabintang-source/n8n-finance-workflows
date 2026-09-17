# Bank Reconciliation — n8n

The workflow that checks whether the money actually did what the books say it did.

Every other piece in this repo ends at a *claim*: an invoice approved, a payment sent, a bank reply
saying "settled". Reconciliation is where those claims meet the bank statement — and it reads from
two of the other workflows at once: payments sent from the [payment run](../payment-run/), and open
invoices and customers from [AR collections](../ar-collections/). It is the first piece that puts
the payables side and the receivables side in one view.

It runs on seeded demo data with no credentials. This is the report from the first run after the
reset, execution 2953, exactly as the workflow wrote it:

```
*Rekonsiliasi Bank STM-20260917* · 2026-09-17

*Integritas rekening koran*
Saldo awal 900.000.000 + kredit 328.000.000 − debit 503.179.000 = 724.821.000
Saldo akhir tercatat 724.821.000 — COCOK
Saldo berjalan meleset di 1 baris: baris 6 tercetak 621.915.800, seharusnya 621.905.800. Saldo akhirnya tetap benar, jadi ini salah cetak kolom, bukan transaksi hilang.

Cocok        : 6 · 699.094.200 IDR
Perlu tindak : 3 · 110.000.000 IDR
Kritis       : 3 · 64.984.800 IDR
Suspense     : 1 · 7.500.000 IDR
Log tersimpan 13 dari 13 · tagihan AR diperbarui 6

*KRITIS — posisi uang tidak pasti*
- [REJECTED_BUT_DEBITED] baris 8 · 6.000.000 · CV BERKAH JAYA MANDIRI
  Catatan pembayaran menyatakan transfer TRX-REC-003 DITOLAK bank (rekening tujuan sudah ditutup), tapi rekening koran menunjukkan 6.000.000 benar-benar keluar. Uangnya keluar dan tidak tercatat terbayar - vendor bisa dibayar lagi.
- [DUPLICATE_LINE] baris 11 · 8.584.800 · PT SINAR BAJA ELEKTRIK
  Referensi TRX-REC-001 sebesar 8.584.800 sudah muncul di baris 2 pada rekening koran yang sama. Periksa dengan bank apakah transaksinya memang terjadi dua kali.
- [MISSING_DEBIT] 50.400.000 · Mitra Logistik Nusantara
  Pembayaran TRX-REC-004 ke Mitra Logistik Nusantara sebesar 50.400.000 tercatat LUNAS pada 2026-09-14, tapi tidak ada debit apa pun di rekening koran. Vendor mungkin belum menerima uangnya, sementara buku kita sudah menganggapnya terbayar.

*Perlu tindak AR*
- [PARTIAL] INV-7006 · 10.000.000
  Dibayar 10.000.000 dari sisa 14.300.000. Masih kurang 4.300.000 - tagihan tetap terbuka dan penagihan AR akan mengejar sisanya.
- [OVERPAYMENT] INV-7005 · 5.000.000
  Dibayar 5.000.000 untuk sisa 4.750.000. Yang diterapkan ke faktur hanya 4.750.000. Kelebihan 250.000 belum diterapkan ke mana pun - AR yang memutuskan: jadikan kredit pelanggan atau kembalikan. Bukan pendapatan.
- [DISPUTED_PAID] INV-7007 · 95.000.000
  PT Global Niaga Persada membayar 95.000.000 untuk INV-7007 yang masih DISENGKETAKAN (Customer says 40 of 200 units arrived damaged; credit note requested, quality team still inspecting.). Dana diterapkan, tapi sengketanya harus ditutup - kalau nota kredit tetap diterbitkan, pelanggan jadi kelebihan bayar.

*Suspense*
- [UNKNOWN_PAYER] 7.500.000 dari PT MAKMUR SEJAHTERA (ref TRF-88213)

*Cocok*
- [MATCH_EXACT] 180.000.000 → INV-7002
- [MATCH_PAYMENT] 8.584.800 → Sinar Baja Elektrik INV-REC-01
- [BANK_FEE] 6.500 → akun biaya bank
- [MATCH_COMBINED] 30.500.000 → INV-7004 + INV-7003
- [MATCH_PAYMENT] 480.000.000 → Anugerah Metalindo INV-REC-02
- [BANK_FEE] 2.900 → akun biaya bank
```

## Two directions, not one

A reconciliation that only asks *"whose is this statement line?"* misses half the problem. This one
also asks the reverse: **for every payment we claim settled, is there a debit at the bank?**

Those two directions produce the two findings in the run above that matter most:

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

No credentials needed. Five Data Tables must exist first — two of this piece's own, and three it
shares:

| table | columns | owned by |
|---|---|---|
| `REC Bank Statement` | line_key, statement_id, account_code, line_no (number), value_date, description, reference, counterparty, debit (number), credit (number), running_balance (number), opening_balance (number), closing_balance (number) | this piece |
| `REC Match Log` | match_key, line_key, statement_id, value_date, direction, amount (number), counterparty, result, code, matched_to, applied_amount (number), reason, route_to, execution_id | this piece |
| `PAY Payment Lines` | see the [payment run](../payment-run/) | payment run |
| `AR Open Invoices` | see [AR collections](../ar-collections/) | AR collections |
| `AR Customers` | see [AR collections](../ar-collections/) | AR collections |

1. Import `reset.json` and `workflow.json`, plus [AR collections](../ar-collections/)' `reset.json` —
   reconciliation reads the AR invoices and writes cash back to them.
2. Re-pick each Data table node from its dropdown; table IDs are per-instance.
3. Run **Reset AR Demo**, then **Reset Demo Rekonsiliasi**, then **Rekonsiliasi Rekening Koran**.

The reset clears only its own payment rows (`run_key = REC-DEMO`), so it does not disturb the
payment run's demo — and the payment run's reset leaves these rows alone in return.

## What it checks

**Statement integrity, first.** Opening balance plus credits minus debits must equal the stated
closing balance. Separately, every line's running balance is recomputed: the demo statement has
one mistyped balance on line 6, which is flagged precisely as a column misprint — the closing
balance is still right, so no transaction is missing. The result is reported, not enforced: a
statement that does not foot is still processed, with the failure at the top of the report.

**Then each line, in order:**

1. **Duplicate line** — the same reference, amount and direction twice on one statement. Every
   other line is registered as seen here, including lines step 2 then skips.
2. **Already reconciled** in a previous run — skipped, never applied twice.
3. **Bank fee** — debits under 50.000 whose description reads as a charge (BIAYA, ADMIN, FEE,
   CHARGE, PROVISI), marked as bank charges for accounting to book.
4. **Outgoing** — matched to a sent payment by bank reference, then checked for status
   (`REJECTED_BUT_DEBITED`) and amount (`AMOUNT_DIFFERS`). Anything unmatched is
   `UNEXPLAINED_DEBIT`, the most urgent finding there is.
5. **Incoming with an invoice number** — `MATCH_EXACT`, `PARTIAL`, `OVERPAYMENT`,
   `DISPUTED_PAID`, or `UNKNOWN_INVOICE` when the number is not in AR.
6. **Incoming without one** — the sender is matched to a customer, then the workflow looks for a
   combination of that customer's open invoices that sums to the amount exactly.
7. **Everything else** — suspense.

On the demo data, eleven result codes fire on the first run and `ALREADY_RECONCILED` on the second.
`AMOUNT_DIFFERS`, `UNEXPLAINED_DEBIT`, `UNKNOWN_INVOICE`, `NO_MATCHING_COMBINATION` and
`AMBIGUOUS_COMBINATION` are implemented, but no seeded line reaches them.

### One transfer, several invoices

CV Anugerah Teknik sent 30.500.000 described only as `PELUNASAN TAGIHAN` — no invoice number. The
workflow finds the customer by name, then searches up to 12 of their undisputed open invoices,
oldest due first, and finds that `INV-7004 (22.000.000) + INV-7003 (8.500.000)` sums to exactly
30.500.000. It splits the cash across both. Customers pay like this constantly, and it is the case
most matching rules give up on.

The search does not stop at the first combination that fits. If a second one fits too, applying
either would be a guess — and a wrong guess settles an invoice that was not paid while the one that
was keeps being chased. So an ambiguous payment is not applied; it goes to suspense as
`AMBIGUOUS_COMBINATION`. That path was tested against a constructed case outside n8n; the demo
data has none.

### Overpayments are capped, not credited

Bengkel Karya Mandiri paid 5.000.000 against 4.750.000 outstanding. The invoice is credited
4.750.000 — exactly its remaining balance — so the 250.000 excess neither inflates the invoice nor
is booked as revenue. The line goes to the AR clerk as `OVERPAYMENT`, and the match log keeps the
5.000.000 received next to the 4.750.000 applied. The workflow does not create the customer credit
itself; AR decides whether the excess becomes a credit or is refunded.

### Suspense is not revenue

7.500.000 arrived from PT MAKMUR SEJAHTERA, which matches no customer. It is parked in suspense.
Booking unidentified money as income, then discovering it belonged to a customer, is a quick way to
make the accounts wrong in two places.

## Cash goes back to AR

Matched receipts are written to `AR Open Invoices` — the table [AR collections](../ar-collections/)
reads. A customer who paid this morning will not be chased tomorrow.

## Running it twice

A reconciliation is run every day against overlapping data; one that is only correct on its first
execution is not correct. Execution 2954 ran immediately after 2953:

```
*Rekonsiliasi Bank STM-20260917* · 2026-09-17

*Integritas rekening koran*
Saldo awal 900.000.000 + kredit 328.000.000 − debit 503.179.000 = 724.821.000
Saldo akhir tercatat 724.821.000 — COCOK
Saldo berjalan meleset di 1 baris: baris 6 tercetak 621.915.800, seharusnya 621.905.800. Saldo akhirnya tetap benar, jadi ini salah cetak kolom, bukan transaksi hilang.

Cocok        : 0 · 0 IDR
Perlu tindak : 0 · 0 IDR
Kritis       : 3 · 64.984.800 IDR
Suspense     : 1 · 7.500.000 IDR
Dilewati (sudah direkonsiliasi): 9
Log tersimpan 13 dari 13 · tagihan AR diperbarui 0

[critical and suspense sections omitted: the same four items as the first run]
```

The nine lines matched or handed to AR the first time were skipped, and nothing was applied to AR
twice. The three critical findings and the suspense line are reported again, deliberately: they
are not resolved until someone resolves them.

## The second entry point: resolving suspense

Once someone identifies the money in suspense, the assignment arrives on the webhook
`rekonsiliasi-suspense`. It is **checked, not applied blindly**, in this order:

1. There must be a named resolver.
2. The amount must be greater than zero.
3. The statement line must have been parked in suspense by a statement run.
4. The amount may not exceed that line's suspense amount minus what earlier resolutions already
   applied — earlier requests, and earlier assignments in the same request.
5. The invoice must exist and must not be in dispute.
6. The amount may not exceed what the invoice still owes.

Rule 4 counts only suspense resolutions. If a later statement run matches a line that was once in
suspense and applies its cash, the webhook does not know, and would accept a resolution for it
again.

These tests were run manually, with the request body pinned on the `Selesaikan Suspense (Webhook)`
trigger; the JSON shown after each is what `Balas Penyelesaian` returned. Execution 2955 sent three
assignments for the 7.500.000 on line 10:

```json
POST /webhook/rekonsiliasi-suspense
{
  "resolutions": [
    { "line_key": "STM-20260917-010", "invoice_number": "INV-7011", "amount": 7500000,
      "resolved_by": "rina.ar@example.com", "note": "Salah pilih faktur." },
    { "line_key": "STM-20260917-010", "invoice_number": "INV-7007", "amount": 7500000,
      "resolved_by": "rina.ar@example.com", "note": "Faktur sengketa." },
    { "line_key": "STM-20260917-010", "invoice_number": "INV-7013", "amount": 7500000,
      "resolved_by": "rina.ar@example.com",
      "note": "PT Makmur Sejahtera adalah perusahaan induk PT Karya Bersama Sentosa; bukti transfer dikirim pelanggan." }
  ]
}
```

and got back:

```json
{
  "diproses": 3,
  "diterapkan": 1,
  "nilai_diterapkan": 7500000,
  "ditolak": 2,
  "ar_diperbarui": 1,
  "log_tersimpan": 3,
  "daftar_diterapkan": [
    {
      "baris": "STM-20260917-010",
      "faktur": "INV-7013",
      "nilai": 7500000,
      "sisa_suspense": 0
    }
  ],
  "daftar_tolak": [
    {
      "faktur": "INV-7011",
      "alasan": "Jumlah 7.500.000 melebihi sisa tagihan 3.200.000 pada INV-7011."
    },
    {
      "faktur": "INV-7007",
      "alasan": "INV-7007 sedang disengketakan; selesaikan sengketanya dulu."
    }
  ]
}
```

Then the same assignment to INV-7013 was sent again, together with one for line 1, which was
matched and never in suspense (execution 2956):

```json
{
  "diproses": 2,
  "diterapkan": 0,
  "nilai_diterapkan": 0,
  "ditolak": 2,
  "ar_diperbarui": 0,
  "log_tersimpan": 2,
  "daftar_diterapkan": [],
  "daftar_tolak": [
    {
      "faktur": "INV-7013",
      "alasan": "Uang di baris STM-20260917-010 (7.500.000) sudah seluruhnya diterapkan. Menerapkannya lagi berarti membukukan uang yang sama dua kali."
    },
    {
      "faktur": "INV-7013",
      "alasan": "Baris STM-20260917-001 tidak pernah diparkir di suspense. Hanya uang yang ada di suspense yang bisa ditugaskan lewat jalur ini."
    }
  ]
}
```

Both were refused, the reply node still ran, and INV-7013 in AR still showed 7.500.000 paid, last
updated by the first request. On the next statement run (execution 2958), line 10 was skipped as
reconciled and the suspense section was empty.

The previous version checked only the resolver, the invoice, the dispute and the invoice balance —
never the statement line. It would have accepted both of these assignments: the same 7.500.000
again, and 1.000.000 from a line that was never in suspense. It had a second defect too: its reply
node ran only after the AR write, so a request in which every assignment was refused was never
answered.

Cash applied to the wrong invoice makes two customers wrong at once. Cash applied twice makes the
books wrong in a way no customer will ever complain about.

## Defects worth recording

Every one of these was silent: green runs doing the wrong thing.

**No cash ever reached AR.** The node that splits receipts per invoice read its input from the log
write, which returns only mapped columns — so the array it needed was gone, it emitted zero items,
and not one rupiah was applied. It now reads from the merge node directly.

**Last month's payments looked missing.** The reverse check had no date window, so a payment settled
on 27 August was flagged as absent from a September statement. That kind of false alarm teaches
people to skim the critical section. It is now bounded to the statement's own date range.

**A second run accepted a duplicate debit as a real payment.** Already-reconciled lines were skipped
*before* being registered as seen, so on re-running, the original line vanished from the duplicate
check and its copy was matched as a legitimate payment.

**The same second run raised a false 480.000.000 alert.** Skipped debit lines never recorded their
bank reference either, so the reverse check concluded that a payment matched the day before had
never left the bank.

Those two only appear on a **second consecutive run**, and were found by running it twice and
comparing the results line by line. The next three were found later, by checking this README
against the workflow claim by claim:

**Suspense cash could be applied twice.** Each assignment was checked against its invoice, never
against the money it came from.

**A request with nothing valid in it got no reply.** The summary ran after the AR update, which does
not run when every assignment is refused.

**An ambiguous payment was applied to whichever combination the search found first.** The README
said the combination was "the only" one; the code never checked.
