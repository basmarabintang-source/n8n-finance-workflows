# AR Collections & Dunning — n8n

Chasing overdue invoices is the job companies solve by hiring a room of people. This runs
every morning, decides for each customer whether to chase, hold off, or escalate, and sends
**one message per customer** rather than one per invoice.

It runs on seeded demo data with no credentials. Twelve open invoices become **four emails,
not twelve**.

```
*Accounts Receivable — 2026-09-14*
IDR 425.650.000 outstanding across 11 invoices. 54.9% of it is overdue, weighted average 13 days late.

*Aging*
current   2 inv   IDR 192.000.000   expect IDR 188.160.000
1-30      6 inv   IDR 204.700.000   expect IDR 188.324.000
31-60     2 inv   IDR  14.650.000   expect IDR  10.987.500
61-90     1 inv   IDR  14.300.000   expect IDR   7.865.000

Expected to collect: IDR 395.336.500

*Went out today: 4 message(s) covering 7 invoices*
- PT Sumber Rejeki Abadi (courtesy) 1 inv, IDR 180.000.000, oldest  0d [AM pinged]
- CV Anugerah Teknik     (firm)     2 inv, IDR  30.500.000, oldest 15d
- UD Sinar Terang        (legal)    2 inv, IDR  17.500.000, oldest 72d [AM pinged]
- Bengkel Karya Mandiri  (legal)    2 inv, IDR  14.650.000, oldest 51d [AM pinged]

*Held back on purpose: 2*
- PT Global Niaga Persada: Contacted 1 day(s) ago and the reminder cadence is one contact
  every 5 days. Chasing again today would read as harassment.
- Toko Maju Jaya: INV-7001 not due for another 5 days | INV-7008 customer promised to pay
  by 2026-09-18, which has not passed yet.

In dispute: 1 (IDR 95.000.000) · promised: 1 (IDR 31.000.000) · broken promises: 1
Over credit limit: UD Sinar Terang
```

## Running it

1. Import both JSONs.
2. Create three Data Tables — `AR Customers`, `AR Open Invoices`, `AR Collection Log` — with
   the columns the nodes reference, then open the Data table nodes and re-pick each table
   from the dropdown. Data table IDs are per-instance, so the saved ones will not resolve on
   yours. This is the only manual step.
3. Run **Reset AR Demo** first. It seeds everything; there are no CSVs to import.
4. Run **AR Collections and Dunning**.

The reset workflow computes every due date **relative to today** rather than storing fixed
dates. Without that, a demo seeded once slowly rots until every invoice is 200 days overdue
and the whole dunning ladder collapses into a single "legal" bucket.

## Every reason NOT to chase comes first

That ordering is the part that matters. An automated chaser that gets it wrong does more
damage than no chaser at all.

| check | what happens |
|---|---|
| settled | nothing to collect, never chased |
| in dispute | never dunned, whatever the age — chasing a disputed invoice loses you the customer *and* the argument |
| promise to pay, not yet expired | held; chasing before the date you agreed breaks trust for nothing |
| promise to pay, expired | chased one rung **harder** — the trust was already spent |
| not yet due | ignored, unless it is large and within 3 days, which earns a courtesy note |
| contacted too recently | held by the cadence guard |

Only then does the ladder apply — 1–7 days a reminder, 8–21 firm, 22–45 final, 46+
collections — and the customer, not the invoice, is the unit.

## The two things that make it not a toy

**One message per customer.** A customer with four overdue invoices gets one email listing
four lines, pitched at the hardest stage any single one has reached. The naive version sends
four separate emails and makes the company look like a robot.

**It remembers.** The collection log is read at the start of every run and written at the end,
keyed on customer + date. That is what stops a daily scheduler mailing the same person every
morning until they pay. Proof, from two consecutive runs of the demo:

```
run 1: messages_sent=4  covering 7 invoices  held=2
run 2: messages_sent=0  covering 0 invoices  held=6   <- every one suppressed, with its reason
```

Run 2 sending nothing is the feature, not a failure. Run the reset workflow to go again.

## Where the output goes

Four destination nodes, all **switched off** and carrying no credential, wired as side
branches so the log write and the report behave identically whether they are on:

- **Gmail** — the reminder, and the final notice with credit control copied in
- **Slack** — the account manager, by name, when an account reaches final notice or crosses
  the value threshold
- **Gmail** — one morning report to the finance lead

## Deliberately not here

- **No payment-gateway integration.** Marking invoices paid is assumed to come from the
  accounting system; this only decides who to chase.
- **No part-payment negotiation.** A promise to pay is a date and nothing more; real
  collections tracks agreed instalment schedules.
- **The collection probabilities** (98% current, 92% at 1–30, down to 30% past 90) are
  industry rules of thumb, not fitted to any real ledger. Same for "9 minutes per manual
  chase". Both are stated in the code as assumptions rather than dressed up as measurements.
- **No working-day awareness.** A 5-day cadence counts calendar days, so a Friday chase can
  land again on a Wednesday.
