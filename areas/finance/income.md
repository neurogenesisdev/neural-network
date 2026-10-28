---
type: area
status: active
created: 2026-09-29
updated: 2026-09-29
tags: [income]
---

# Income

> [!NOTE] Summary
> What comes in, from whom, and where it sits relative to the
> Pfändungsfreigrenze (the garnishment-exempt threshold). Needed for
> Ratenzahlung offers, Beratungshilfe and Insolvenzgeld applications.

## Sources

| Source | Type | Net / month | Paid on | Account | Since | Proof |
|--------|------|-------------|---------|---------|-------|-------|
| DDK Detektei | employment, part-time (luis) | ? | ? | ? | 2026 | payslips — collect last 3 |
| Gewerbe (web/automation) | self-employed, Gewerbeanmeldung 2026-09-17 (see `freelance-practice/business`) | 0 so far (luis) | — | — | 2026-09 | — |
| Bürgergeld / Wohngeld | none currently? (assumed — confirm) | | | | | |

## Thresholds

- Pfändungsfreigrenze (single, no dependants): look up current figure on bmj.de "Pfändungsfreigrenzen" — changes every July (assumed). Income below it → no wage garnishment possible.
- Krankenkasse Nebentätigkeit boundary: ≤ 20 h/week self-employed keeps employee status (tracked in `freelance-practice/business/model.md`)

## Next action
- [ ] Collect last 3 DDK payslips → `_attachments/letters/` → fill row
- [ ] Look up current Pfändungsfreigrenze, record with date
- [ ] Check whether Wohngeld or aufstockendes Bürgergeld applies at current income (assumed — ask Schuldnerberatung)
