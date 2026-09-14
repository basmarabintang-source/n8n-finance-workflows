# Employee Onboarding Coordinator — n8n

Getting a new starter ready involves HR, IT, facilities, a workshop manager and a hiring
manager, none of whom can see what the others have done. That coordination is the job.

This runs every morning: it works out what each new hire needs from a checklist, creates
whatever is missing, tracks who owes what, chases each team **once** with everything they
owe, and tells HR who will not be ready on day one.

It runs on seeded demo data with no credentials.

```
*Onboarding — 2026-09-14*
5 hires in flight. Ready 1 · on track 1 · at risk 2 · blocked 1.
41 checklist items: 20 done, 20 open, 1 blocked, 5 overdue.
8 item(s) were created automatically this morning from the checklist.

*Who needs a person today*
- Bayu Nugroho  (at_risk, started 4d ago) — Laptop imaged and issued (IT), Safety induction (Workshop)
- Agus Salim    (blocked, in 1d)          — Email account (IT), Laptop (IT), Desk and card (Facilities), ...
- Dewi Lestari  (at_risk, in 3d)          — Desk and card (Facilities), Safety induction (Workshop), PPE, ...

*By team*
IT           50% done   open 5   blocked 1   overdue 2
Workshop     25% done   open 3   blocked 0   overdue 1
Finance       0% done   open 1   blocked 0   overdue 1
HR           67% done   open 5   blocked 0   overdue 0
Facilities   43% done   open 4   blocked 0   overdue 0
Sales         0% done   open 2   blocked 0   overdue 0

4 team digest(s) went out, 2 held back as already chased or not yet due.
```

## The checklist is data, not code

This is the part worth stealing. The onboarding checklist lives in a table, one row per step:

| applies_to | task | owner | lead_days | mandatory | blocks_day_one |
|---|---|---|---|---|---|
| `ALL` | Signed contract returned | HR | 7 | yes | yes |
| `ALL` | Email account created | IT | 3 | yes | yes |
| `ROLE:Sales Executive` | CRM account and pipeline access | IT | 2 | yes | yes |
| `DEPT:Workshop` | Safety induction completed | Workshop | 1 | yes | yes |
| `TYPE:contract` | Contractor payment schedule set up | Finance | 3 | yes | no |

HR adds one row to give every future mechanic a new induction step, and nobody opens the
workflow. Twelve rows and six hires become the exact 41 tasks those people need, each with
a due date derived from its lead time against that person's start date.

A checklist buried inside a Code node is a checklist that goes stale the first time the
process changes.

## Four states, not two

| state | what it means | what happens |
|---|---|---|
| **ready** | every *mandatory* item done | confirmation to the manager — so "ready" and "forgotten" don't look identical |
| **on track** | outstanding items, but nothing due yet | **deliberately silent** |
| **at risk** | a day-one item still open inside 7 days of starting | manager is told, teams are chased |
| **blocked** | something needs a decision | escalated separately, with the reason |

**Blocked is kept apart from at-risk on purpose.** A dashboard shows late and blocked in the
same colour, and they need opposite responses: one wants a reminder, the other wants a human
to make a decision. In the demo, IT cannot create an account because the name on the contract
does not match the ID card. No number of reminders will fix that.

**On track is silent on purpose.** Someone three weeks out with an untouched checklist is not
late. Mailing about them teaches everyone to ignore the mail.

## Grouped twice, on two different axes

- **By hire, for the manager** — one message about their person.
- **By owning team, for the doers** — IT gets a single mail listing all six things IT owes
  across three different new starters, sorted by how late each one is.

A team already chased within the last two days is skipped entirely.

## Running it

1. Import both JSONs.
2. Create three Data Tables — `HR New Hires`, `HR Onboarding Checklist`, `HR Onboarding Tasks`
   — with the columns the nodes reference, then re-pick each table in the Data table nodes.
   Data table IDs are per-instance; this is the only manual step.
3. Run **Reset Onboarding Demo**. It seeds everything.
4. Run **Employee Onboarding Coordinator**.

Start dates are computed relative to today, so one hire is always starting tomorrow and one
has always already started. The reset also wipes the task table, which is what leaves one
hire with no tasks at all — that is how the demo shows tasks being derived from the checklist
rather than just read.

## A bug worth showing

The first working version computed `send_digest` for every team and then never acted on it.
All six teams reached the mail node while the report cheerfully claimed two had been held
back. The run was green; it was caught by counting the items leaving each node.

The fix is the `Only Teams Worth Chasing` filter. A flag that nothing enforces is a comment.

## Deliberately not here

- **No real approval round-trip.** Completing a task means editing the table; a production
  version would give each line a one-click link that resolves back into the workflow.
- **No working-day awareness.** Lead times are calendar days, so a contract due 7 days before
  a Monday start lands on a Sunday.
- **No provisioning.** It tracks that IT created the account; it does not create it. Wiring
  the Google Workspace or Entra node in is a day's work and a credential.
- **"6 minutes per task"** is a stated assumption, not a measurement.
