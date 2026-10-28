---
type: project
status: active
created: 2026-09-29
updated: 2026-09-29
area: "[[../areas/admin/_index]]"
due: 
tags: [project]
---

# Altlasten 2026 — clear the backlog

> [!NOTE] Summary
> Work through every open financial/legal/bureaucratic item until the case
> table has no row without a Ratenplan, a filed claim, or a closed status —
> and the letter pile is empty. Managed with the Claude project
> *Finance & Legal Case Manager*. Weekly triage generates the "do this week" list.

## Goal
Every counterparty known, every amount verified, every case in one of: paid / Ratenplan running / claim filed / closed. Income, recurring costs and budget tracked from statements.

## Success criteria
- [ ] Letter backlog fully intaken (0 unread letters, every one a correspondence note)
- [ ] All cases in `areas/admin/_index.md` have verified amounts
- [ ] Insolvenzgeld application submitted; Forderungsanmeldung filed
- [ ] Landlady claim sent (or consciously dropped after legal advice)
- [ ] Ratenplan agreed with Jobcenter and Stadt Frankfurt
- [ ] Recurring table verified; kill pass done
- [ ] Email accounts consolidated; triage automation running weekly
- [ ] 2 consecutive months with a statement-based budget summary

## Tickets

| # | Ticket | Case / file | Priority | Status | This week's step |
|---|--------|-------------|----------|--------|------------------|
| 1 | Insolvenzgeld + Forderungsanmeldung | [[../areas/admin/cases/fm-safe-protect-insolvenz]] | **P0** | open | message coworker; look up Insolvenzverwalter; find payslips |
| 2 | Letter backlog intake | all | **P0** | open | photograph every unread letter into `_inbox/letters/` |
| 3 | Sparkasse export | [[../areas/finance/_index]] | P1 | open | export all months since 2024-01 |
| 4 | Beratungshilfe + Schuldnerberatung | [[../areas/admin/cases/beratungshilfe]] | P1 | open | gather documents; go to Rechtsantragstelle |
| 5 | Landlady Rückforderung | [[../areas/admin/cases/vermieterin-rueckforderung]] | P1 | blocked by 3 | itemize transfers after export |
| 6 | Abschleppung | [[../areas/admin/cases/abschleppung-hd-oo-661]] | P2 | waiting | chase 2026-10-20 if silent |
| 7 | Jobcenter Ratenplan | [[../areas/admin/cases/jobcenter-rueckforderung]] | P2 | blocked by 2 | — |
| 8 | Email account consolidation | [[../areas/admin/accounts]] | P2 | open | forwarding from 2 Gmails + Proton |
| 9 | Recurring kill pass | [[../areas/finance/recurring]] | P3 | blocked by 3 | — |
| 10 | Triage automation (n8n class + weekly prompt) | `freelance-automation-inbox` pattern | P3 | open | after 2 and 3 exist |

## Waiting for
- Stadt Frankfurt reply (since 2026-09-29)
- Coworker reply (once messaged)

## Resources
- [[CLAUDE|vault rules]]
- `freelance-automation-inbox/triage/prompt.md` — weekly triage prompt to clone

## Log
### 2026-09-29
- Project created. Discovered FM Safe & Protect insolvency (AG Frankfurt, 810 IN 249/26 F, opened 2026-03-10) — reprioritized ticket 1 to P0.
- Sent status email to Stadt Frankfurt re: HD-OO-661.
- Vault: added `CLAUDE.md`, `areas/admin/`, restructured `areas/finance/`, two templates.
