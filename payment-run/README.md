# Payment Run & Bank Disbursement — n8n

The workflow that decides whether money actually leaves the company.

It reads the same **AP Invoice Ledger** that the [3-way match](../ap-3way-match/) writes, so these
two pieces connect: an invoice clears matching over there, and the cash goes out here. Approving
an invoice and paying it are different decisions, and the second one is the irreversible one.

It runs on seeded demo data with no credentials, and every control fires on the first execution.

```
*Run Pembayaran RUN-2900* · 2026-09-16
Rekening sumber OPS-IDR-01 · boleh dipakai 750.000.000 IDR

Dibayar   : 7 baris · 607.469.600 IDR
Tertahan  : 5 baris · 1.949.000.000 IDR
Sisa kas  : 142.530.400 IDR
Diskon bayar-awal diambil: 350.400 IDR
Butuh dua tanda tangan: 1 baris

Berkas bank: 607.469.600 IDR, checksum 1020568414
Baris tersimpan: 12 dari 12

Tertahan, dan siapa pemiliknya:
- [BANK_CHANGED] Cahaya Teknik Presisi INV-R2874-09 · 1.450.000.000 → treasury
  Nomor rekening berubah 3 hari lalu lewat email oleh finance.cahaya@gmail.com.
  Telepon vendor memakai nomor LAMA yang tersimpan, bukan kontak apa pun di pesan
  yang meminta perubahan.
- [CASH_SHORT]   Berkah Jaya Mandiri PAY-1008 · 310.000.000 → treasury
  Ditunda, bukan ditolak.
- [DUPLICATE]    Sinar Baja Elektrik PAY-1007 · 22.000.000 → ap_supervisor
  Sudah dibayar pada 2026-08-27 (ref TRX-SEED-0001).
- [SANCTIONS]    PT Kemasan Prima PAY-1004 · 47.000.000 → legal
```

That 1.45 billion hold is the whole point. The AP workflow had **approved** that invoice — the
match was clean. What stopped it was the vendor's bank account changing three days earlier, by
email, from a gmail address.

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 38 nodes (33 functional + 5 sticky notes) |
| `reset.json` | rebuilds all demo data, 16 nodes |

No CSVs here — unlike the AP piece, the reset workflow seeds every table itself, and computes
all dates relative to today so the demo never goes stale.

## Running it yourself

No credentials needed. It uses n8n **Data Tables**, not an external database.

1. Import `reset.json` and `workflow.json`.
2. Open the Data table nodes and re-pick each table from the dropdown — data table IDs are
   per-instance, so they will not match yours.
3. Run **Reset Demo Pembayaran** first. It creates 6 vendors, 1 bank account, 1 prior payment,
   and 9 payable invoices.
4. Run **Jalankan Proposal Pembayaran**.

> Running the [3-way match](../ap-3way-match/) demo first is optional but worth doing — its
> approved invoices flow into this run alongside the seeded ones, which is the integration
> working rather than a diagram claiming it does.

## The six controls, in the order they are applied

Order is deliberate, and the reasoning matters more than the code:

1. **No bank details** — you cannot pay into nothing.
2. **Sanctions / blocklist** — a legal bar, checked before any money question.
3. **Duplicate payment** — checked against payments that actually *settled*, because a transfer
   cannot be clawed back. Counting what was *intended* rather than what the table returned is
   the easiest way to pay twice.
4. **Bank account changed recently** — payment redirection fraud. See below.
5. **Below the minimum** — transfer cost exceeds the benefit; batched into the next run.
6. **Cash position** — last, because running out of cash is a *scheduling* problem, not a
   rejection. Those lines say "deferred, not refused" and carry into the next run.

### Why number 4 is the reason this workflow exists

Successful payment fraud almost always looks the same. An email arrives saying *"our bank
details have changed"*, someone updates the vendor master, and one large transfer disappears.
The invoice is real, the vendor is real, the approval is real. Nothing in a 3-way match catches
it, because nothing about the *invoice* is wrong.

The defence is not technology. It is holding the payment and phoning the vendor **on the number
you already had** — never a number in the message that asked for the change. So the hold text
says exactly that, and the remittance advice tells vendors the same thing in reverse: if the
account looks wrong, call us on a number you already know, do not reply to the email.

The window is 14 days by default, in the constant `HARI_REKENING_BARU`.

### Early-payment discounts are checked against the cost of cash

`2/10 NET 30` means 2% off for paying 20 days early — roughly a 36% annualised return, against
a 12% cost of cash, so it is worth taking. The workflow compares the two rather than grabbing
any discount it sees. In the run above that is 350,400 IDR captured on two invoices, and
skipped everywhere the arithmetic did not justify it.

## The bank file, and why it has a control total

The file carries a header, one line per payment, and a trailer with the line count, the total
value, and a checksum anyone can recompute from the contents.

Banks reject files whose totals do not agree. That is what stops a line being dropped, altered,
or added between our system and theirs. A file without a control total merely *looks* like a file.

## The second entry point: the bank's reply

This is the half most payment automations leave out. Sending a file is not the same as the money
arriving — accounts close, names mismatch, funds fall short.

The webhook `payment-run-ack` takes the bank's response, splits settled from rejected, updates
the affected lines, and reconciles the run. Without it our records would claim everything was
paid, and the error would surface a month later as a vendor phone call.

```json
POST /webhook/payment-run-ack
{
  "run_key": "RUN-2900",
  "results": [
    { "vendor": "Mitra Logistik Nusantara", "invoice_number": "PAY-1002",
      "amount": 50400000, "status": "settled", "bank_ref": "TRX-99120" },
    { "vendor": "Berkah Jaya Mandiri", "invoice_number": "PAY-1005",
      "amount": 6000000, "status": "rejected", "reason": "rekening tutup" }
  ]
}
```

## Where it actually goes

Slack, Gmail and the HTTP call to the bank are **real nodes, switched off**, carrying no
credentials, so the demo runs for anyone who opens it with nothing to set up. Message bodies are
composed in Set nodes beforehand, so switching one on changes no logic.

The HTTP node is the only thing here that moves money. It should stay off until the approval path
is real and the checksum is verified on the bank's side.

## Three defects worth recording

All three were **silent** — the run stayed green while doing the wrong thing.

**The reset seeded nothing, and reported success.** A delete against an already-empty table emits
zero items, and n8n stops a branch when a node outputs nothing — so all four seeding steps never
executed. No error anywhere. `alwaysOutputData` on the delete nodes fixes it. Counting items per
node was the only thing that caught it.

**`autoMapInputData` tried to write arrays into scalar columns.** The summary node emits both
table fields and report arrays; auto-mapping attempted all of them and killed the run. Columns
are now listed explicitly.

**The report printed empty sections.** It sat downstream of the table write, which returns only
the 16 mapped columns — so the hold list, the payment list and the written-row count vanished.
Headings rendered with nothing under them and nothing errored. It now reads from the summary node
directly.

The common thread: a number that describes an *intention* rather than an *event* is a bug. The
run summary counts rows the data table actually handed back, never rows the engine meant to write.
