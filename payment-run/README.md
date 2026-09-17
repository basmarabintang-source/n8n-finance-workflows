# Payment Run & Bank Disbursement — n8n

The workflow that decides whether money actually leaves the company.

It reads the same **AP Invoice Ledger** that the [3-way match](../ap-3way-match/) writes, so these
two pieces connect: an invoice clears matching over there, and the cash goes out here. Matching an
invoice, approving it, and paying it are three different decisions, and only the last one cannot
be undone.

It runs on seeded demo data with no credentials. On the first execution after the reset, all seven
controls fire. This is the report from that run, execution 2960, exactly as the workflow wrote it:

```
*Run Pembayaran RUN-2960* · 2026-09-17
Rekening sumber OPS-IDR-01 · boleh dipakai 740.200.000 IDR

Dibayar   : 5 baris · 548.484.800 IDR
Tertahan  : 9 baris · 2.121.385.000 IDR
Sisa kas  : 191.715.200 IDR
Diskon bayar-awal diambil: 175.200 IDR
Butuh dua tanda tangan: 1 baris

Berkas bank: 548.484.800 IDR, checksum 498084355
Baris tersimpan: 14 dari 14

Tertahan, dan siapa pemiliknya:
- [BANK_CHANGED] Cahaya Teknik Presisi PAY-1003 · 1.450.000.000 → treasury
  Nomor rekening Cahaya Teknik Presisi diubah 3 hari lalu lewat email oleh finance.cahaya@gmail.com, dan perubahan itu belum diverifikasi (verifikasi terakhir: 2026-02-03). Tahanan ini tidak lepas dengan lewatnya waktu. Yang melepasnya hanya konfirmasi telepon terverifikasi di kontrol vendor, ke nomor LAMA yang tersimpan - bukan kontak apa pun di pesan yang meminta perubahan.
- [CASH_SHORT] Berkah Jaya Mandiri PAY-1008 · 310.000.000 → treasury
  Kas yang boleh dipakai 740.200.000 (saldo 900.000.000 dikurangi cadangan 150.000.000, dikurangi 9.800.000 yang sudah di berkas bank tapi belum dibalas). Run ini sudah mengalokasikan 488.584.800, sedangkan tagihan ini 310.000.000. Ditunda, bukan ditolak.
- [DUPLICATE] Sinar Baja Elektrik PAY-1007 · 22.000.000 → ap_supervisor
  Faktur ini sudah dibayar pada 2026-08-28 (ref TRX-SEED-0001). Membayarnya lagi tidak bisa ditarik kembali.
- [TOO_SMALL] Sinar Baja Elektrik PAY-1013 · 85.000 → ap_clerk
  Nilainya 85.000, di bawah ambang 100.000 - biaya transfer melebihi manfaatnya. Tidak ada yang menggabungkannya otomatis: AP clerk yang memutuskan kapan dibayar.
- [SANCTIONS] PT Kemasan Prima PAY-1004 · 47.000.000 → legal
  PT Kemasan Prima sedang ditahan: Masuk daftar tahan sejak sengketa pajak. Semua pembayaran wajib ditinjau legal. Legal harus melepas dulu.
- [IN_FLIGHT] Berkah Jaya Mandiri PAY-1014 · 9.800.000 → treasury
  Faktur ini sudah ada di berkas bank RUN-SEED-2 (2026-09-16) dan balasan banknya belum masuk. Mengajukannya lagi sekarang berarti membayar dua kali bila berkas pertama lunas. Kalau bank menolaknya, faktur ini diajukan lagi dengan sendirinya di run berikutnya.
- [AWAITING_APPROVAL] Mitra Logistik Nusantara PAY-1011 · 180.000.000 → director
  Disetujui rudi.manager@example.com sebagai manager, padahal faktur ini butuh persetujuan director. Persetujuan dari kewenangan yang kurang tidak dihitung.
- [AWAITING_APPROVAL] Anugerah Metalindo PAY-1010 · 95.000.000 → manager
  Belum disetujui. Pencocokan tiga arahnya bersih, tapi faktur ini masih menunggu persetujuan manager.
- [NO_BANK_DETAILS] CV Sumber Makmur Teknik PAY-1012 · 7.500.000 → ap_clerk
  Tidak ada rekening terverifikasi untuk CV Sumber Makmur Teknik. Tidak ada tujuan transfer.

Dibayar:
- Sinar Baja Elektrik PAY-1001 · bersih 8.584.800 (diskon 175.200) → accounting_auto
- Anugerah Metalindo PAY-1006 · bersih 480.000.000 → dual_signature
- Mitra Logistik Nusantara PAY-1002 · bersih 50.400.000 → manager
- Berkah Jaya Mandiri PAY-1005 · bersih 6.000.000 → accounting_auto
- Mitra Logistik Nusantara PAY-1009 · bersih 3.500.000 → accounting_auto
```

That 1,45 milyar hold is the whole point. The invoice matched cleanly and the director had
approved it; nothing about the invoice was wrong. What stopped it was the vendor's bank account
changing three days earlier, by email, from a gmail address.

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 40 nodes (35 functional + 5 sticky notes) |
| `reset.json` | rebuilds all demo data, 16 nodes |

No CSVs here — unlike the AP piece, the reset workflow seeds every table itself, and computes all
dates relative to today so the demo never goes stale.

## Running it yourself

No credentials needed. It uses n8n **Data Tables**, not an external database. Create these five
first:

| table | columns |
|---|---|
| `AP Invoice Ledger` | invoice_key, vendor, invoice_number, po_number, amount (number), currency, decision, route_to, exception_code, note, processed_at, execution_id, qty_billed (number), approved_by, approved_role, approved_on |
| `PAY Vendor Bank Details` | vendor, bank_name, account_number, account_name, currency, verified_on, changed_on, changed_by, change_channel, sanctions_flag (boolean), payment_terms, notes, npwp |
| `PAY Bank Accounts` | account_code, bank_name, account_number, currency, balance (number), min_buffer (number), cost_of_cash_pct (number), as_of |
| `PAY Payment Lines` | line_key, run_key, vendor, invoice_number, amount (number), currency, status, route_to, hold_code, hold_reason, account_number_masked, pay_on, discount_taken (number), bank_ref, settled_on, reject_reason, execution_id |
| `PAY Payment Runs` | run_key, run_date, status, lines_total, lines_paid, lines_held, value_paid, value_held, control_total (all number), file_checksum, account_code, balance_before, balance_after, settled_count, rejected_count (all number), execution_id |

1. Import `reset.json` and `workflow.json`.
2. Open the Data table nodes and re-pick each table from the dropdown — data table IDs are
   per-instance, so they will not match yours.
3. Run **Reset Demo Pembayaran**. It creates 6 vendor accounts, 1 bank account, 2 earlier
   payments, and 14 invoices.
4. Run **Jalankan Proposal Pembayaran**.

The [3-way match](../ap-3way-match/) demo can feed this run too. Run it **before** step 3, not
after: its reset empties the whole AP Invoice Ledger, including the invoices this reset seeds. Its
invoices then go through the same seven controls. The reset here deletes only its own invoices
(`po_number = PAY-RUN`) and leaves the bank reconciliation's payment rows (`run_key = REC-DEMO`)
alone.

## The seven controls, in the order they are applied

Order is deliberate, and the reasoning matters more than the code:

1. **No bank details** — no vendor master row, or a row with no account number. You cannot pay
   into nothing.
2. **Sanctions / blocklist** — a legal bar, checked before any money question.
3. **Already paid, or already being paid** — `DUPLICATE` when the invoice settled in an earlier
   run; `IN_FLIGHT` when it is already in a bank file the bank has not answered yet. See
   [running it twice](#running-it-twice).
4. **Bank account changed and not verified** — payment redirection fraud. The hold does not
   expire. See below.
5. **Not approved** — or approved by someone without enough authority. See below.
6. **Below the minimum** (100.000) — transfer cost exceeds the benefit, so it is held for the AP
   clerk. Nothing combines small invoices automatically; the same invoice is held again on every
   run until someone deals with it.
7. **Cash position** — last, because running out of cash is a *scheduling* problem, not a
   rejection. Those lines say "deferred, not refused" and are proposed again next run.

Number 4 comes before number 5 on purpose: an invoice still waiting for a signature is flagged when
the vendor's account has changed and not been verified, so the approver finds out **before**
signing, not after.

### Why number 4 is the reason this workflow exists

Successful payment fraud almost always looks the same. An email arrives saying *"our bank details
have changed"*, someone updates the vendor master, and one large transfer disappears. The invoice
is real, the vendor is real, the approval is real. Nothing in a 3-way match catches it, because
nothing about the *invoice* is wrong.

The defence is not technology. It is holding the payment and phoning the vendor **on the number
you already had** — never a number in the message that asked for the change. So the hold text says
exactly that, and the remittance advice tells vendors the same thing in reverse: if the account
looks wrong, call us on a number you already know, do not reply to the email.

### A bank-change hold ends with verification, not with time

The vendor master records two dates: `changed_on`, when the account last changed, and
`verified_on`, when it was last verified. An account is held whenever `changed_on` is set and
`verified_on` is missing or on an earlier date, for as long as that stays true. Apart from this
demo's reset, only the [vendor control](../vendor-control/) workflow writes `verified_on`: when it
approves a new vendor or a bank change submitted through the vendor portal, or when a phone callback
passes its checks. A new-vendor request naming an existing vendor is refused there, so that route
cannot replace an account either.

This used to be a 14-day timer: from day 15 the vendor was paid whether or not anyone had called,
and a genuinely verified change restarted the clock. Now:

- **Time alone releases nothing.** With Cahaya Teknik Presisi's change moved back 30 days in the
  master, execution 2961 still held the invoice.
- **A denial keeps it held.** After the vendor denied the change on the callback (vendor control
  execution 2965) and a later attempt to mark it verified was refused (2966), execution 2967 still
  held it.
- **A verified callback releases it on the next run.** Once the callback verified the change and
  wrote the requested account (2971), execution 2972 no longer held the invoice for the bank
  change. It moved on to control 7 and deferred it there, because 1,45 milyar is more than the cash
  available.

### A clean match is not an approval

The 3-way match decides whether an invoice is *correct*. Whether the company *agrees to pay it* is
a separate decision, and a clean match above the touchless ceiling only means it is ready for a
person to look at.

So an invoice is payable when either the 3-way match marked it `auto_approve` — the payment run
trusts that decision and does not re-check the amount — or an approver is recorded in
`approved_by` with an `approved_role` senior enough for the route it was sent to. A
director-level invoice approved by a manager does not count. (`approved_on` is shown in the
report but not required.) In the run above, PAY-1010 has no approver at all, and PAY-1011 was
approved by a manager when it needed a director. Both are held as `AWAITING_APPROVAL`.

### Early-payment discounts are checked against the cost of cash

`2/10 NET 30` means 2% off for paying 20 days early — roughly a 36% annualised return, against a
12% cost of cash, so it is worth taking. The workflow compares the two rather than grabbing any
discount it sees. In the run above that is 175.200 IDR on PAY-1001, the paid invoice from the one
vendor with discount terms. The demo has no case where the cost of cash outweighs the discount, so
that comparison is in the code but not exercised.

### Cash is what is actually free

The balance in `PAY Bank Accounts` is a snapshot taken on `as_of`. Two things have not reached it
yet, and both are deducted: money already in a bank file the bank has not answered, and payments
settled since the snapshot. In the run above, 9.800.000 was already in flight from an earlier
file, so only 740.200.000 of the 750.000.000 above the buffer was usable.

## The bank file, and why it has a control total

The file carries a header, one line per payment, and a trailer with the line count, the total
value, and a checksum anyone can recompute from the contents. This is the file execution 2960
built:

```
HDR,2960,2026-09-17,5,548484800
0001,Sinar Baja Elektrik,******7766,Bank Mandiri,8584800,IDR,PAY-1001
0002,Anugerah Metalindo,******5544,Bank Mandiri,480000000,IDR,PAY-1006
0003,Mitra Logistik Nusantara,******1004,Bank Central Asia,50400000,IDR,PAY-1002
0004,Berkah Jaya Mandiri,******7889,Bank Rakyat Indonesia,6000000,IDR,PAY-1005
0005,Mitra Logistik Nusantara,******1004,Bank Central Asia,3500000,IDR,PAY-1009
TRL,5,548484800,498084355
```

Banks reject files whose totals do not agree. That is what stops a line being dropped, altered, or
added between our system and theirs. A file without a control total merely *looks* like a file.

Only lines that were actually **stored** in `PAY Payment Lines` go into the file. A payment that is
sent but not recorded is invisible to control number 3 next time — and gets paid again. Account
numbers are masked in the demo, so the file shows the format, not something a bank could execute.

## Running it twice

Executions 2946 to 2950 below ran before the bank-change hold was tied to verification. Their treasury
report would differ today only in the reason under PAY-1003's hold; the fraud alert for that hold
has also been reworded.

A payment run that is only correct on its first execution is not correct. Execution 2947 was run
immediately after 2946, before any bank reply:

```
*Run Pembayaran RUN-2947* · 2026-09-17
Rekening sumber OPS-IDR-01 · boleh dipakai 191.715.200 IDR

Dibayar   : 0 baris · 0 IDR
Tertahan  : 14 baris · 2.670.045.000 IDR
Sisa kas  : 191.715.200 IDR

Berkas bank: 0 IDR, checksum 0
Baris tersimpan: 14 dari 14

[hold list omitted: the same 14 invoices, six of them IN_FLIGHT: the five from RUN-2946 plus PAY-1014 from RUN-SEED-2]
```

Nothing was paid. The five invoices already in the first file were held as `IN_FLIGHT`, and the
558.284.800 sitting in unanswered bank files was deducted from usable cash, so the 310 million
invoice stayed deferred.

The previous version counted only settled payments and had no approval control. Replayed against
the same table state 2947 loaded, its engine proposes seven lines for 738.284.800: the five again,
plus PAY-1014, already in flight, and PAY-1011, approved only by a manager. That is a replay of the
old code, not an execution.

## The second entry point: the bank's reply

This is the half most payment automations leave out. Sending a file is not the same as the money
arriving — accounts close, names mismatch, funds fall short.

The webhook `payment-run-ack` takes the bank's response and checks every line of it against the
lines that were **actually in that run's file**. Settled and rejected lines are written back.
A reply for a line that was never in the file — held, or simply unknown — is reported and not
written: writing it would create a "settled" payment out of nothing, which control number 3 would
then treat as proof the invoice was paid.

The test reply, execution 2948, was run manually with this body pinned on the
`Balasan Bank (Webhook)` trigger, so the workflow did not need to be activated. It covered each
case at once: three settled lines (one for a
different amount than was sent), one rejected, one not answered at all, one for the held
1,45 milyar invoice, and one for an invoice that does not exist.

```json
POST /webhook/payment-run-ack
{
  "run_key": "RUN-2946",
  "results": [
    { "vendor": "Sinar Baja Elektrik", "invoice_number": "PAY-1001", "amount": 8584800, "status": "settled", "bank_ref": "TRX-40101" },
    { "vendor": "Anugerah Metalindo", "invoice_number": "PAY-1006", "amount": 480000000, "status": "settled", "bank_ref": "TRX-40102" },
    { "vendor": "Mitra Logistik Nusantara", "invoice_number": "PAY-1002", "amount": 50000000, "status": "settled", "bank_ref": "TRX-40103" },
    { "vendor": "Berkah Jaya Mandiri", "invoice_number": "PAY-1005", "amount": 6000000, "status": "rejected", "reason": "rekening tujuan sudah ditutup" },
    { "vendor": "Cahaya Teknik Presisi", "invoice_number": "PAY-1003", "amount": 1450000000, "status": "settled", "bank_ref": "TRX-40104" },
    { "vendor": "PT Nusantara Fiktif", "invoice_number": "PAY-9999", "amount": 75000000, "status": "settled", "bank_ref": "TRX-40105" }
  ]
}
```

```
*Rekonsiliasi run RUN-2946*

Di berkas     : 5 baris
Dibalas bank  : 4 baris
Lunas         : 3 · 538.584.800 IDR
Ditolak       : 1 · 6.000.000 IDR
Belum dibalas : 1 · 3.500.000 IDR
Baris diperbarui di tabel: 4 dari 4

Ditolak bank, perlu tindakan:
- Berkah Jaya Mandiri PAY-1005 · 6.000.000 — rekening tujuan sudah ditutup

Belum dibalas bank, jangan dianggap lunas:
- Mitra Logistik Nusantara PAY-1009 · 3.500.000

Balasan untuk baris yang tidak ada di berkas, TIDAK ditulis:
- Cahaya Teknik Presisi PAY-1003 · 1.450.000.000 — Baris ini ditahan dan tidak pernah masuk berkas bank.
- PT Nusantara Fiktif PAY-9999 · 75.000.000 — Tidak ada baris PAY-9999 untuk PT Nusantara Fiktif di RUN-2946.

Lunas dengan jumlah berbeda dari yang dikirim:
- Mitra Logistik Nusantara PAY-1002 · dikirim 50.400.000, dibalas 50.000.000
```

`PAY Payment Lines` had the same number of rows before and after, and PAY-1003 stayed `held`. The
next run, execution 2950, paid the rejected PAY-1005 again, held the three settled invoices as
`DUPLICATE`, and kept the unanswered PAY-1009 `IN_FLIGHT`. An empty reply (execution 2949, also
pinned) wrote nothing and still reached `Balas ke Bank`.

## Where it actually goes

Slack, Gmail and the HTTP call to the bank are **real nodes, switched off**, carrying no
credentials, so the demo runs for anyone who opens it with nothing to set up. The Slack alert and
the treasury emails take bodies already composed in Set nodes, so switching those on changes no
logic.

Two are not ready to switch on as they stand. The vendor remittance email is addressed to the
vendor's *name*, because the vendor table has no email column yet. And the HTTP node would send a
file even when there is nothing to pay — execution 2947 built an empty one — so it needs a guard in
front of it first. It is the only node here that moves money, and it should stay off until the
approval path is real and the checksum is verified on the bank's side.

## What it does not do yet

- **Verification is read from the vendor master, not proven.** The run cannot see an account change
  by itself: it trusts both dates in the master. An account number edited directly without moving
  `changed_on` is paid with no hold, and anyone who can edit the table can also set `verified_on`.
  The dates are calendar dates, so a change recorded later on the same day as a verification is not
  held.
- **A rejected payment is proposed again as it is.** PAY-1005 was rejected because the account was
  closed, and execution 2950 put it in the next file to the same account. Nothing requires the
  vendor's bank details to be checked first.
- **The run header is not updated from the bank reply.** `settled_count` and `rejected_count` in
  `PAY Payment Runs` stay at 0; the line statuses in `PAY Payment Lines` are the record of what
  settled.
- **Approvals are data, not a workflow.** The demo seeds who approved what. Nothing here asks an
  approver or records the approval as it happens.
- **Same-day settlements may be deducted twice from cash.** A payment settled on the day the
  balance was recorded may already be in that balance. The workflow deducts it anyway: erring that
  way delays a payment, erring the other way overdraws the account.

## Defects worth recording

All but one of these were **silent** — the run stayed green while doing the wrong thing. The
exception is the `autoMapInputData` defect, which failed the run outright.

**The reset seeded nothing, and reported success.** A delete against an already-empty table emits
zero items, and n8n stops a branch when a node outputs nothing — so every seeding step after it
never executed. `alwaysOutputData` on the delete nodes fixes it. Counting items per node was the
only thing that caught it.

**`autoMapInputData` tried to write arrays into scalar columns.** The summary node emits both table
fields and report arrays; auto-mapping attempted all of them and killed the run. Columns are now
listed explicitly.

**The report printed empty sections.** It sat downstream of the table write, which returns only the
16 mapped columns — so the hold list, the payment list and the written-row count vanished. It now
reads from the summary node directly.

The next four were found later, by checking this README against the workflow, claim by claim:

**Invoices still waiting on an approver were paid.** A clean 3-way match was treated as approval.
In the run this README used to show, three of the seven paid lines had never been approved.

**Running the proposal twice paid the same invoices twice.** Only settled payments counted as paid,
so everything in an unanswered bank file was proposed again. Once that was fixed, the same test
exposed the next one: money in that first file still counted as available cash.

**A bank reply could create a payment out of nothing.** The reply was written with an upsert, so a
line that was never sent became a new "settled" row — which the duplicate check then trusted.

**A payment that failed to save still went into the bank file.** The bank file was built from the engine's
decisions, not from the rows the table returned.

The last two were found while tying the hold to verification:

**The bank-change hold was a timer.** It lasted 14 days from the change and then paid, whether or
not anyone had called the vendor — and a genuine verification started the clock again. It now
lasts until vendor control marks the change verified.

**A vendor with an empty account number could be paid.** Control 1 only checked that a master row
existed. When vendor control briefly wrote empty accounts into the master (a defect on its side,
[now fixed](../vendor-control/#defects-worth-recording)), execution 2967 put two invoices into the
bank file with the account `****`. A row with no account number is now held as `NO_BANK_DETAILS`.

The common thread: a number that describes an *intention* rather than an *event* is a bug. A clean
match is not an approval, a file sent is not a payment settled, and a decision is not a row stored.
