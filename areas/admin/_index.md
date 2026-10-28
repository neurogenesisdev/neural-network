---
type: index
status: active
created: 2026-09-29
updated: 2026-09-30
tags: [index, admin]
---

# admin — index

> [!NOTE] Summary
> Bureaucracy and legal: every Vorgang with a counterparty, plus all incoming
> official correspondence. Money view of the same items lives in
> [[../finance/debts|finance/debts]]. Static tables — keep them current by hand.

## Cases

| Case | Counterparty | Direction | Amount (status) | Status | Next action | Deadline |
|------|-------------|-----------|-----------------|--------|-------------|----------|
| [[cases/fm-safe-protect-insolvenz\|FM Safe & Protect Insolvenz]] | Insolvenzverwalter / Agentur für Arbeit | claim | ~1,800–2,300 net (statements) | open | coworker messaged 09-30; look up Insolvenzverwalter; Insolvenzgeld application | ASAP — Nachfrist argument weakens |
| [[cases/abschleppung-hd-oo-661\|Abschleppung HD-OO-661]] | Stadt Frankfurt | owe | unknown | waiting | wait for reply; ask Amt 21 re Pfändung 761.83; confirm Abmeldung | chase 2026-10-20 |
| [[cases/vermieterin-rueckforderung\|Reifegerste Rückforderung]] | Lilja Reifegerste | claim | 8,900–9,790 + items (statements) | open | **delete Dauerauftrag if still active**; find Mietvertrag end date; Beratungshilfe | — |
| [[cases/jobcenter-rueckforderung\|Jobcenter]] | Jobcenter / Inkasso-Service BA | owe | several k; ≤ ~2.8k overlap Aug–Dec 25 (assumed) | open | letter backlog intake | — |
| [[cases/beratungshilfe\|Beratungshilfe + Schuldnerberatung]] | AG Frankfurt; Schuldnerberatung | none | — | open | gather documents; apply | — |

## Other files

| File | Purpose |
|------|---------|
| [[accounts]] | Which email address holds which service; consolidation plan |
| [[contacts]] | Behörden, Sachbearbeiter, phone numbers, hours |
| `correspondence/` | One note per letter/email, any topic; see [[correspondence/_index]] |

## Intake

Paper → photo → `_inbox/letters/` → correspondence note from `tpl-correspondence` → linked case → row here.
Email → n8n class `amt-finanzen` → same note format (automation: see [[../../resources/automation/finance-triage]] once built).

## Open questions

| Question | Where |
|----------|-------|
| Who is the Insolvenzverwalter for 810 IN 249/26 F? | [[cases/fm-safe-protect-insolvenz]] |
| Is the 890 € Dauerauftrag to Reifegerste still active? | [[cases/vermieterin-rueckforderung]] |
| Were damages ever itemized in writing by the landlady? | [[cases/vermieterin-rueckforderung]] |
| What did Amt 21 Pfändung AZ FFM.2026.033673.9 (761.83, May 26) enforce? | [[cases/abschleppung-hd-oo-661]] |
| Is HD-OO-661 abgemeldet (Kfz-Steuer)? | [[cases/abschleppung-hd-oo-661]] |
| Which unknown creditors are in the unread letter pile? Bounced debits give the shortlist: [[../finance/debts]] | letter backlog intake |
| What is EUROBILL TECH / centrohelp.com (14.99/mo)? | [[../finance/recurring]] |
