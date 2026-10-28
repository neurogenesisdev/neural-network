---
type: index
status: active
created: 2026-09-29
updated: 2026-09-29
tags: [index, finance]
---

# finance — index

> [!NOTE] Summary
> Money state: income, recurring costs, debts/claims, budget, statements.
> Process and correspondence for anything owed or claimed lives in
> [[../admin/_index|admin]].

| File | Purpose | State |
|------|---------|-------|
| [[income]] | sources, net, Pfändungsfreigrenze | skeleton — payslips missing |
| [[recurring]] | subscriptions, insurances, fixed bills, kill flags | historical list; rebuild from export |
| [[debts]] | every owed/claimed amount, one row each, linked to cases | drafted, amounts approx. |
| [[budget]] | category caps, savings set-aside | empty until statements are in |
| `transactions/` | `YYYY-MM-<bank>.csv` + `YYYY-MM-summary.md` | empty |
| [[lagerraum-versicherung]] | storage-unit insurance note (pre-existing) | check if still relevant |
| [[subscriptions]] | redirect stub → recurring | delete when unlinked |

## Banks

| Bank | Account use | Export | Notes |
|------|-------------|--------|-------|
| Sparkasse Heidelberg | main (luis) | CSV per month via online banking | export 2024-01 → today first |
| Service Credit Union | second account (luis) | no CSV (luis, unconfirmed) | check PDF statements |

## Next action
- [ ] Sparkasse export (all months since 2024-01) → `transactions/`
- [ ] Payslips → [[income]]
- [ ] Repeating debits → [[recurring]]
