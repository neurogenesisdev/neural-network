---
type: area
status: active
created: 2026-09-29
updated: 2026-09-29
tags: [budget]
---

# Budget

> [!NOTE] Summary
> Category caps and a fixed savings set-aside. Empty until the first
> statement-based month is in `transactions/`; do not fill from memory.

## Method
- Source of truth: bank statements in `transactions/` (Sparkasse CSV; Service Credit Union PDF → parsed), not manual daily logging.
- Monthly: one summary note `transactions/YYYY-MM-summary.md` with actuals per category vs. cap.
- Categories: housing · food · transport · telecom · insurance · subscriptions · debt-repayment · savings · business · other.

## Caps

| Category | Cap / month | Basis |
|----------|-------------|-------|
| savings set-aside | ? | fixed first, before anything else |
| debt-repayment (Raten total) | ? | sum of agreed Ratenpläne |
| subscriptions | ? | after kill pass in [[recurring]] |
| food | ? | 2 months of actuals |
| transport | ? | |
| other | ? | |

## Next action
- [ ] Sparkasse export → first two months categorized → set caps
- [ ] Service Credit Union: check statement download format (PDF?) → decide parsing route
