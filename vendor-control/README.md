# Vendor Onboarding & Bank Change Control — n8n

The workflow that decides who is allowed to be paid, and whether their bank account may change.

It is the other half of the [payment run](../payment-run/). That workflow **holds** a payment when
a vendor's bank account changed recently. This one is the process that **releases** it — and the
only legitimate release is a person phoning the vendor on the number already on file.

Both read and write the same vendor master, `PAY Vendor Bank Details`. There is one master, not a
copy per workflow.

It runs on seeded demo data with no credentials, and every control fires on the first execution.

```
*Kontrol Vendor VLOG-2920* · 2026-09-16

Diproses   : 10 permintaan
Disetujui  : 2
Ditahan    : 6
Ditolak    : 2
Rata-rata kelengkapan data: 97%

Log tersimpan: 10 dari 10
Master vendor diperbarui: 2 baris

PERLU TELEPON VERIFIKASI (1):
- Cahaya Teknik Presisi (REQ-007) — telepon nomor di berkas, BUKAN 02155507890

- [DUPLICATE_NPWP]  REQ-002 PT Sinar Baja Elektrik Jaya → ap_supervisor
  NPWP 017234567890000 sudah terdaftar atas nama Sinar Baja Elektrik.
- [DUPLICATE_NAME]  REQ-003 Cahaya Teknik Presisi Utama → ap_supervisor
  Namanya 75% mirip dengan "Cahaya Teknik Presisi" yang sudah ada.
- [NAME_MISMATCH]   REQ-005 Mega Teknik Abadi → treasury
  Nama rekening "BUDI SANTOSO" tidak cocok dengan badan usaha "PT MEGA TEKNIK ABADI".
- [SOD_VIOLATION]   REQ-008 Berkah Jaya Mandiri → finance_manager
  Diminta oleh orang yang juga berwenang menyetujui perubahan rekening.
```

Then treasury makes the call — and the vendor denies ever asking:

```
*Hasil verifikasi telepon*

Diproses          : 2
Terverifikasi     : 1
Gagal / disangkal : 1
Master diperbarui : 1

PERCOBAAN PENIPUAN TERTANGKAP:
- Cahaya Teknik Presisi (REQ-007)
  Vendor MENYANGKAL pernah meminta ganti rekening. Email berasal dari
  alamat gmail, bukan domain perusahaan mereka. Perlakukan sebagai
  percobaan penipuan: jangan ubah master, simpan permintaan aslinya
  sebagai barang bukti, dan periksa pembayaran lain ke vendor ini.
```

The master row for Cahaya Teknik Presisi was verified untouched afterwards — same account number,
same `updatedAt` as the reset. The fraudulent account never reached the table the payment run reads,
so the 1,45 milyar payment stays held.

## How the two workflows fit together

```
3-Way Match     →  approves the invoice
Payment Run     →  holds it: vendor bank account changed 3 days ago, by email
Vendor Control  →  treasury phones the vendor on the OLD number
                →  vendor denies the request
                →  master unchanged, attempt recorded as evidence, payment stays held
```

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 35 nodes (31 functional + 4 sticky notes) |
| `reset.json` | rebuilds the 10 demo requests, 7 nodes |

## Running it yourself

No credentials needed.

1. Import `reset.json` and `workflow.json` — and the [payment run's](../payment-run/) `reset.json`,
   because the vendor master is shared.
2. Re-pick each Data table node from its dropdown; table IDs are per-instance.
3. Run **Reset Demo Pembayaran** first. It seeds the vendor master, including tax IDs.
4. Run **Reset Demo Vendor**, then **Proses Antrean Permintaan**.

To test the callback path, drive the `Konfirmasi Telepon (Webhook)` trigger in manual mode —
no need to activate the workflow or expose a public endpoint:

```json
POST /webhook/vendor-callback
{
  "confirmations": [
    { "request_key": "REQ-007", "vendor": "Cahaya Teknik Presisi",
      "result": "failed", "called_by": "rina.treasury@example.com",
      "number_used": "021-5550-9911 (nomor lama di master)",
      "note": "Vendor menyangkal pernah meminta ganti rekening." }
  ]
}
```

## The seven controls, in order

1. **Incomplete data** — you cannot assess what is not there. Scored, and the missing fields named.
2. **Invalid NPWP** — 15 digits. A malformed tax identity never enters the master.
3. **Sanctions** — a legal bar, before anything else.
4. **Duplicates, searched three ways** — similar name, same NPWP, or **same bank account**. Two
   different vendors sharing one account almost always means one of them is not a real vendor.
5. **Account name mismatch** — an individual's account receiving a company's invoices is the
   classic red flag. The demo catches `BUDI SANTOSO` against `PT MEGA TEKNIK ABADI`.
6. **Segregation of duties** — whoever requests a bank change may not approve it.
7. **Request channel** — a bank change arriving by anything other than the vendor portal requires
   a phone confirmation before it is applied.

### Why control 7 says "the OLD number"

A bank-change fraud email usually includes a phone number for "verification". Calling it reaches
the fraudster, who confirms everything. So the task text names the account and the old number
from the master side by side, and says explicitly not to use any contact in the request. A
control that says "verify by phone" without saying *which* phone is not a control.

## Only approved changes touch the master

A filter sits in front of every write to the vendor master. It looks redundant. It is the most
expensive node in the workflow.

Without it, a decision that is **computed but not enforced** still writes the new account to the
master — while the run stays green, the report reads correctly, and the fraudulent account is
now the one the payment run pays. That is the defect class this whole portfolio is built around.

A verified bank change also resets `changed_on`, so the payment run holds that vendor for another
14 days. That is deliberate: a recently changed account is still recently changed, even when the
change was genuine.

## Three defects worth recording

**The engine judged the wrong data, and stayed green.** It read `$input.all()`, but the loaders
were chained, so its input was the six master rows rather than the ten requests. It produced six
decisions about vendors instead of ten about requests. Nothing errored. It now reads the requests
node by name.

**The report disappeared on the run that needed it most.** It sat behind the approved-only filter.
With zero approvals the filter emitted zero items, n8n stopped the branch, and no report was
produced at all. It now runs from the log write, in parallel with the master update.

**A control that could never fire.** The duplicate-NPWP check compared against the vendor
`notes` field, because the master had no NPWP column. It looked implemented and could never
trigger. The column now exists and is seeded, and the check genuinely catches REQ-002.

That last one is the worst kind: a control that exists in the code but cannot fire is worse than
no control, because everyone downstream assumes it is covering them.
