# Vendor Onboarding & Bank Change Control — n8n

The workflow that decides who is allowed to be paid, and whether their bank account may change.

It is the other half of the [payment run](../payment-run/). That workflow **holds** every payment to a
bank account that changed and has not been verified since, and the hold does not expire. This
workflow is the only thing that marks an account verified: by approving a new vendor or a bank
change submitted through the vendor portal, or by a phone callback that passes its checks. Each of
those sets `verified_on` in the vendor master, and the payment run pays that account from its next
run.

Both use the same vendor master, `PAY Vendor Bank Details`: this workflow writes it, the payment run
only reads it, and the payment run's demo reset re-seeds it. There is one master, not a copy per
workflow.

It runs on seeded demo data with no credentials. This is the queue report from execution 2970,
exactly as the workflow wrote it:

```
*Kontrol Vendor VLOG-2970* · 2026-09-17

Diproses   : 10 permintaan
Disetujui  : 2
Ditahan    : 6
Ditolak    : 2
Rata-rata kelengkapan data: 97%

Log tersimpan: 10 dari 10
Master vendor diperbarui: 2 baris

PERLU TELEPON VERIFIKASI (1):
- Cahaya Teknik Presisi (REQ-007) — telepon nomor di berkas, BUKAN 02155507890

Semua keputusan:
- [DISETUJUI] REQ-001 Karya Presisi Nusantara (new) → ap_clerk
  Vendor baru lolos seluruh pemeriksaan. Kelengkapan 100%.
- [DISETUJUI] REQ-010 Mitra Logistik Nusantara (bank_change) → ap_clerk
  Ganti rekening lewat portal vendor, data lengkap, nama rekening cocok, peminta bukan penyetuju.
- [CALLBACK_REQUIRED] REQ-007 Cahaya Teknik Presisi (bank_change) → treasury
  Permohonan ganti rekening masuk lewat email, bukan portal vendor. Telepon Cahaya Teknik Presisi memakai nomor LAMA yang tersimpan di master - jangan memakai nomor mana pun yang tertulis di permintaan ini - lalu catat konfirmasinya.
- [NPWP_INVALID] REQ-004 Bumi Sentosa Logistik (new) → ap_clerk
  NPWP 14 digit, seharusnya 15. Identitas pajak yang salah bentuk tidak boleh masuk master.
- [SANCTIONS] REQ-006 PT Kemasan Prima (bank_change) → legal
  PT Kemasan Prima sedang ditahan: Masuk daftar tahan sejak sengketa pajak. Semua pembayaran wajib ditinjau legal. Legal harus melepas dulu.
- [DUPLICATE_NPWP] REQ-002 PT Sinar Baja Elektrik Jaya (new) → ap_supervisor
  NPWP 017234567890000 sudah terdaftar atas nama Sinar Baja Elektrik. Satu badan pajak, satu baris master.
- [DUPLICATE_NAME] REQ-003 Cahaya Teknik Presisi Utama (new) → ap_supervisor
  Namanya 75% mirip dengan "Cahaya Teknik Presisi" yang sudah ada. Vendor kembar memecah riwayat pembayaran dan membuka jalan bayar ganda.
- [NAME_MISMATCH] REQ-005 Mega Teknik Abadi (new) → treasury
  Nama rekening "BUDI SANTOSO" tidak cocok dengan badan usaha "PT MEGA TEKNIK ABADI". Rekening atas nama perorangan untuk tagihan badan usaha adalah bendera merah klasik.
- [SOD_VIOLATION] REQ-008 Berkah Jaya Mandiri (bank_change) → finance_manager
  Diminta oleh treasury.lead@example.com, yang juga berwenang menyetujui perubahan rekening. Peminta dan penyetuju harus dua orang berbeda.
- [DATA_INCOMPLETE] REQ-009 Sentra Kemas Indonesia (new) → ap_clerk
  Kelengkapan 71%. Belum ada: contact_email, contact_phone. Tidak bisa dinilai sebelum lengkap.
```

Treasury then makes the call to Cahaya Teknik Presisi — and the vendor denies ever asking. This
callback, execution 2965, followed an earlier queue run of the same demo (2964, with the same ten
decisions):

```
*Hasil verifikasi telepon*

Diproses          : 1
Terverifikasi     : 0
Gagal / disangkal : 1
Tidak diterima    : 0
Master diperbarui : 0

PERCOBAAN PENIPUAN TERTANGKAP:
- Cahaya Teknik Presisi (REQ-007)
  TELEPON GAGAL MEMVERIFIKASI. rina.treasury@example.com menelepon 021-5550-9911 (nomor lama di berkas vendor) pada 2026-09-17 dan vendor MENYANGKAL meminta perubahan ini. Vendor menyangkal pernah meminta ganti rekening. Email berasal dari alamat gmail, bukan domain perusahaan mereka. Perlakukan sebagai percobaan penipuan: jangan ubah master, simpan permintaan aslinya sebagai barang bukti, dan periksa pembayaran lain ke vendor ini.
```

The master row for Cahaya Teknik Presisi was untouched afterwards — same account number, same
`updatedAt` as the reset. A second callback marking the same request `verified` (execution 2966) was
refused: a denial is evidence, not a draft. The payment run that followed (execution 2967) still
held the 1,45 milyar invoice.

## How the two workflows fit together

```
3-Way Match     →  matches the invoice and routes it to the director
Payment Run     →  holds it: vendor bank account changed by email, never verified
Vendor Control  →  treasury phones the vendor on the number already on file
                →  vendor denies:   master unchanged, attempt recorded, payment stays held
                →  vendor confirms: requested account written, verified_on set, hold released
```

## Files

| file | what |
|---|---|
| `workflow.json` | the pipeline, 37 nodes (33 functional + 4 sticky notes) |
| `reset.json` | rebuilds the 10 demo requests, 7 nodes |

## Running it yourself

No credentials needed. This workflow uses three Data Tables. Step 3 below also runs the payment
run's reset, so the payment run's five tables must exist as well — their columns are in
[its README](../payment-run/).

| table | columns |
|---|---|
| `VENDOR Requests` | request_key, request_type, vendor, legal_name, npwp, bank_name, account_number, account_name, contact_email, contact_phone, requested_by, requested_on, channel, status, notes |
| `VENDOR Change Log` | log_key, request_key, vendor, request_type, outcome, risk_code, reason, route_to, requested_by, decided_on, callback_required (boolean), callback_by, callback_on, callback_number_used, quality_score (number), execution_id |
| `PAY Vendor Bank Details` | shared with the payment run — see its [README](../payment-run/) |

1. Import `reset.json` and `workflow.json` — and the [payment run's](../payment-run/) `reset.json`,
   because the vendor master is shared.
2. Re-pick each Data table node from its dropdown; table IDs are per-instance.
3. Run **Reset Demo Pembayaran** first. It seeds the vendor master, including tax IDs.
4. Run **Reset Demo Vendor**, then **Proses Antrean Permintaan**.

To test the callback path, pin a body on the `Konfirmasi Telepon (Webhook)` trigger and run it in
manual mode — no need to activate the workflow or expose a public endpoint. That is how every
callback test below was run:

```json
POST /webhook/vendor-callback
{
  "confirmations": [
    { "request_key": "REQ-007", "vendor": "Cahaya Teknik Presisi",
      "result": "failed", "called_by": "rina.treasury@example.com",
      "number_used": "021-5550-9911 (nomor lama di berkas vendor)",
      "note": "Vendor menyangkal pernah meminta ganti rekening." }
  ]
}
```

## The seven controls, in order

1. **Incomplete data** — you cannot assess what is not there. Scored, and the missing fields named.
2. **Invalid NPWP** — 15 digits. A malformed tax identity never enters the master.
3. **Sanctions** — a legal bar, checked right after the two data checks and before everything else.
4. **Duplicates, searched three ways** — similar name, same NPWP, or **same bank account**. Two
   different vendors sharing one account almost always means one of them is not a real vendor. The
   demo fires the name and NPWP searches; no seeded request shares an account. Before those, the
   request type must match the master: a "new vendor" request naming a vendor that already exists
   is refused as `VENDOR_EXISTS`, and a bank change for a vendor that does not exist as
   `VENDOR_NOT_FOUND`. See [the new-vendor route](#a-new-vendor-request-cannot-replace-an-existing-account).
5. **Account name mismatch** — an individual's account receiving a company's invoices is the
   classic red flag. The demo catches `BUDI SANTOSO` against `PT MEGA TEKNIK ABADI`.
6. **Segregation of duties** — whoever requests a bank change may not approve it. The demo knows one
   approver, `treasury.lead@example.com`; there is no approver directory behind it.
7. **Request channel** — a bank change arriving by anything other than the vendor portal requires a
   phone confirmation before it is applied.

### Why control 7 says "the number already on file"

A bank-change fraud email usually includes a phone number for "verification". Calling it reaches
the fraudster, who confirms everything. So the task says explicitly not to use any contact in the
request — the report above names the one to avoid — and the callback refuses a confirmation whose
number is the one written in the request. A control that says "verify by phone" without saying
*which* phone is not a control.

## The callback is checked, not trusted

Once a confirmed callback releases a payment, the callback itself is the control. Every
confirmation is checked before anything is written:

1. The request must exist and be for the same vendor.
2. It must not already have been decided by a callback — a denial cannot be overturned later, and
   one request cannot be decided twice in the same message.
3. It must actually be waiting for a callback (`CALLBACK_REQUIRED` in the change log).
4. The result must be `verified` or `failed` (case and surrounding spaces are ignored; anything
   else is refused).
5. The caller must be named, and must not be the person who made the request.
6. The number called must be recorded, and must not contain the number written in the request.
   Both are reduced to digits and a leading `62` is read as `0`, so `+62 21 5550 7890` is caught
   as `02155507890` (execution 2976).
7. If an account number is confirmed, it must match the request.

The account written to the master always comes from the **request**, never from the callback
message. Execution 2971 sent six confirmations at once:

```
*Hasil verifikasi telepon*

Diproses          : 6
Terverifikasi     : 1
Gagal / disangkal : 0
Tidak diterima    : 5
Master diperbarui : 1

REKENING DIVERIFIKASI - tahanan rekening di payment run lepas:
- Cahaya Teknik Presisi (REQ-007) · Bank Permata 9911223344999

KONFIRMASI TIDAK DITERIMA - master tidak diubah:
- Cahaya Teknik Presisi (REQ-007)
  KONFIRMASI TIDAK DITERIMA: ap.clerk@example.com adalah pengaju permintaan ini dan tidak boleh memverifikasinya sendiri.
- Cahaya Teknik Presisi (REQ-007)
  KONFIRMASI TIDAK DITERIMA: nomor yang ditelepon ((021) 5550-7890) adalah nomor yang tertulis di permintaan itu sendiri. Telepon harus ke nomor lama yang sudah tersimpan, bukan kontak dari pihak yang meminta perubahan.
- Cahaya Teknik Presisi (REQ-007)
  KONFIRMASI TIDAK DITERIMA: rekening yang dikonfirmasi (9911000000000) berbeda dari rekening di permintaan (9911223344999).
- Mitra Logistik Nusantara (REQ-010)
  KONFIRMASI TIDAK DITERIMA: REQ-010 tidak sedang menunggu konfirmasi telepon.
- Cahaya Teknik Presisi (REQ-007)
  KONFIRMASI TIDAK DITERIMA: REQ-007 sudah diputuskan oleh konfirmasi sebelumnya dalam muatan yang sama.
```

The one that passed carried `"bank_name": "Bank Palsu"` in its message; the master received the
requested Bank Permata account, with `verified_on` and `changed_on` both set to that day and
`change_channel` set to `telepon_terverifikasi`. The payment run that followed (execution 2972)
no longer held the 1,45 milyar invoice for the bank change — it moved on to the cash-position
control and deferred it there.

## Only approved changes touch the master

A filter sits in front of every write to the vendor master. It looks redundant. It is the most
expensive node in the workflow.

Without it, a decision that is **computed but not enforced** still writes the new account to the
master — while the run stays green, the report reads correctly, and the fraudulent account is now
the one the payment run pays. That is the defect class this whole portfolio is built around.

A change applied here sets `verified_on` and `changed_on` to the same day. The payment run holds an
account whenever `changed_on` is set and `verified_on` is missing or on an earlier date, so an
approved change is payable at once, and a change recorded with a later `changed_on` and no new
`verified_on` is held. Bank changes keep the vendor's existing NPWP, payment terms and sanctions
flag; only a new vendor's NPWP comes from its request.

### A new-vendor request cannot replace an existing account

The channel rule and the callback apply to bank changes. New vendors are approved on the other
controls alone, from any channel, and the master is written by vendor name. So until this check
existed, an email asking to onboard "Cahaya Teknik Presisi" as a *new* vendor, with a different
account, passed every control and overwrote the real vendor's account — marked verified, with no
call made. The same shape of email that the callback exists to stop simply walked in by the other
door.

Execution 2975 ran the queue with two extra test requests added to the seeded ten: that email, and
a portal bank change for a vendor that does not exist. The first was refused as `VENDOR_EXISTS`,
the second as `VENDOR_NOT_FOUND`, and Cahaya's master row was unchanged. The seeded ten got the
same decisions as in execution 2970.

## What it does not do yet

- **The master has no phone column.** The callback can refuse the number written in the request,
  but it cannot confirm that the number used really was the one on file.
- **Requests are never closed.** Nothing updates `VENDOR Requests.status`, so every queue run decides
  every request again and logs it again. A request already decided by a callback still cannot be
  decided a second time.
- **Anyone who can edit the vendor master can bypass this.** The payment run trusts the master's
  dates. An account number edited directly, without moving `changed_on`, is paid with no hold at all.
- **Dates, not times.** `changed_on` and `verified_on` are calendar dates, so a change recorded
  later on the same day as a verification is not held.
- **Matching is by vendor name.** The master is written, and the payment run looks vendors up, by
  name; there is no vendor ID.

## Defects worth recording

**The engine judged the wrong data, and stayed green.** It read `$input.all()`, but the loaders
were chained, so its input was the six master rows rather than the ten requests. It produced six
decisions about vendors instead of ten about requests. Nothing errored. It now reads the requests
node by name.

**The report disappeared on the run that needed it most.** It sat behind the approved-only filter.
With zero approvals the filter emitted zero items, n8n stopped the branch, and no report was
produced at all. It now runs from the log write, in parallel with the master update.

**A control that could never fire.** The duplicate-NPWP check compared against the vendor `notes`
field, because the master had no NPWP column. It looked implemented and could never trigger. The
column now exists and is seeded, and the check genuinely catches REQ-002.

The next five were found while tying the payment run's hold to verification, the last one by
checking this README against the workflow afterwards:

**Approving a request wrote an empty bank account into the master.** The master update read its
data from the change-log write, which returns only the log's columns — so the bank name, account
number and channel arrived empty, and the vendor's payment terms were overwritten with `NET 30`.
While the payment run's hold was a 14-day timer, a vendor changed that day was held for two
weeks, so the empty account never reached a bank file in a demo run. Once verification released the
hold, execution 2967 put two Mitra Logistik invoices into the bank file with the account `****` and
no bank. The update now takes the request's data by `request_key`, keeps the existing terms and
sanctions flag, and stores a new vendor's NPWP without overwriting an existing vendor's.

**The callback wrote whatever account the message carried.** It never looked at the request, so a
callback could verify an account nobody had asked for.

**A callback could be approved by the person who requested the change, using the number the
request supplied** — which is exactly how the fraud this workflow exists to catch would get through.

**A callback where every call failed got no reply.** The summary ran after the master update, which
does not run when nothing is verified — so the most important outcome, a vendor denying the
request, produced no report at all.

**A new-vendor request could take over an existing vendor.** Described
[above](#a-new-vendor-request-cannot-replace-an-existing-account): it bypassed the callback entirely.

The callback tests quoted in this README (2965, 2966, 2971) and the queue run 2970 ran before the
`VENDOR_EXISTS` / `VENDOR_NOT_FOUND` rules and the number normalisation were added; neither changes
any of their output.

The worst of these share one shape: a control that exists in the code but cannot fire, or fires and says
nothing, is worse than no control, because everyone downstream assumes it is covering them.
