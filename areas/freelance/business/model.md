---
type: decision-basis
status: active
created: 2026-09-09
updated: 2026-09-15
tags: [gruendung, pricing, insurance]
---

# Business model — replacing the part-time salary

> [!NOTE] Summary
> At 10–20 h/week the practice replaces a part-time security salary only when
> (a) recurring revenue covers ~40 % of the target and (b) the effective rate
> is ≥ 60 €/h. Timeline: 18–30 months from first paid client. The Krankenkasse
> boundary is crossed on purpose, not by drift, at the "flip" point below.
> Salary and tax figures are assumptions — replace with the payslip.

## Target

| Item | Figure | Source |
|------|--------|--------|
| Part-time security pay, 28 h/week | ~1,700–2,000 €/month gross, ~1,350–1,550 € net | assumed — insert payslip |
| Self-employed profit needed for the same net | **~2,300–2,700 €/month = 28–32 k€/year** | Mentor estimate: ESt at that level ~10–15 %, plus full GKV+PV on profit ~20 % once hauptberuflich (no employer half), no RV obligation |
| Buffer to *quit* rather than *match* | +3 months expenses in the bank, pipeline ≥ 2 months | standard |

Until hauptberuflich, business profit is taxed as extra income on top of salary
(ESt only, no KV on it) — so early on each euro of profit is worth *more* net
than it will be after the flip.

## Capacity

| Hours/week for business | Hours/month | Billable at 60 % | Revenue at 60 €/h effective |
|---|---|---|---|
| 10 | 43 | 26 | 1,560 € |
| 15 | 65 | 39 | 2,340 € |
| 20 | 86 | 52 | 3,120 € |

60 % billable is optimistic for year 1 (discovery calls, marketing, admin,
learning eat the rest). At 10 h/week the target is not reachable; 15–20 h/week
at ≥ 60 €/h effective is the corridor.

## Revenue mix that works

Recurring revenue is what makes 15 h/week survivable — it needs 2–4 h/week
of work and pays every month whether a new client shows up or not.

| Stage | When | One-off | Recurring | Monthly total | Hours/week |
|---|---|---|---|---|---|
| 0 — portfolio | now → Q1 2027 | Chantal pro bono; 1–2 paid one-pagers at 700–1,200 € | 1–2 care plans | 0–600 € | 10–12 |
| 1 — first paying practice | 2027 | 1 site every 6–8 weeks (1,500–2,800 €) + 1 automation/month (400–1,200 €) | 5–8 care plans, 2 automation retainers | 1,200–2,200 € | 12–15 |
| 2 — stable side business | 2028 | 1 site/month at raised prices (2,800–4,500 €) | 12–15 care plans, 4–5 automation retainers ≈ 1,300–1,800 € | 3,000–4,500 € | 15–20 |
| 3 — flip | when stage 2 holds 3 months in a row | | recurring ≥ 1,500 €/month | ≥ 3,000 € | decide: cut job hours or quit |

Rule of thumb: **recurring ≥ 40 % of monthly revenue** before relying on it;
**no client > 40 % of revenue** (Scheinselbständigkeit is assessed on
dependence; 5/6 of revenue from one client is the Rentenversicherung's
red line, 40 % is the practical one).

## Where the Krankenkasse boundary bites

The Krankenkasse decides hauptberuflich vs. nebenberuflich (GKV-Spitzenverband
guidelines; verify with own Kasse). 2026 Bezugsgröße = 3,955 €/month → half =
1,977.50 €, three quarters = 2,966.25 €.

| Situation | Presumption |
|---|---|
| Employed > 20 h/week **and** gross pay > 1,977.50 €/month | Employment is the main job; self-employment is nebenberuflich even with good profit — unless self-employment "clearly predominates" in hours and income |
| Employed ≤ 20 h/week **or** pay ≤ 1,977.50 € | Presumption flips: self-employment is hauptberuflich unless he shows it is smaller in hours and income than the job |
| Self-employed > 20 h/week (some Kassen: > 30 h) | Hauptberuflich regardless |
| Business profit > salary | Hauptberuflich |

Consequences of hauptberuflich: no Pflichtversicherung via the employer;
freiwillige GKV on *all* income including profit (~15–17 % + PV ~4 %),
minimum contribution ≈ 250–300 €/month even at low profit (verify current
Mindestbemessung), employer's half of KV on the salary is lost.

**Where it bites for Luis:** with 28 h/week his pay is *near* the 1,977.50 €
line. If the payslip is below it, the Kasse may presume hauptberuflich as soon
as the business is visible — he would have to show hours (log!) and profit
below the job. Actions: (1) check the gross on the payslip now; (2) keep the
freelance hour log — it is the evidence; (3) stay ≤ 20 h/week and profit <
salary until the flip is intended; (4) a move to Personenschutz/Chauffeur
raises pay above the line and makes this easier, not harder.

## Kleinunternehmer ceiling

Revenue > 25,000 € in a calendar year → VAT from the next year: +19 % on price
for private clients, or –16 % margin if absorbed. Stage 2 hits this. Plan the
price raise for that year to include VAT for private clients and quote net +
VAT to businesses (no change for them).

## When to raise prices

| Trigger | Action |
|---|---|
| 3 case studies with testimonials published | Move to the "after 3 case studies" column in [[../services/_index|services]] (+50–80 %) |
| 2 of the last 3 quotes accepted without negotiation | +15–20 % on the next quote |
| Booked more than 4 weeks out | +15–20 % |
| Kleinunternehmer status ending next year | Re-price to include VAT for private clients |
| Every 12 months regardless | +5–10 % on new quotes; existing retainers with 2 months' notice |

## Cost triggers tied to revenue

| Trigger | Action |
|---|---|
| Revenue > 5,000 €/year | datenschutz-generator.de free licence ends — buy the premium licence for neurogenesis.dev (price: verify on the site) and budget it as a fixed cost. Same rule applies per client site that uses the generator: the client's revenue, the client's licence (FDR-002) |
| First paid client | Business phone: VoIP number forwarding to the mobile (sipgate-class, a few €/month). Not before — the Impressum stays phone-free regardless |
| First client conversation by email | Domain-signed sending (Resend/Brevo relay, ~30 min, Architect backlog) so business mail passes DMARC at strict receivers |
| Never | Lower prices to win a client — cut scope instead |

## Review

Update this note when: payslip figures are in, `business/hours.csv` (manual `hours` command, summed weekly by the triage automation) shows the first full month — replace the capacity table's assumptions with logged hours, first paid client is invoiced,
recurring revenue passes 500 €/month, or hours pass 15/week for a month.

Sources: [gulp/Randstad — Bezugsgröße 2026 and hauptberuflich rules](https://www.gulp.de/freelancing/wissen/versicherungen/hauptberuflich-nebenberuflich-selbststaendig-krankenversicherung),
[haufe — GKV presumptions](https://www.haufe.de/personal/entgelt/keine-kv-pflicht-hauptberuflich-selbststaendig-erwerbstaetiger_78_222294.html),
[DAK — nebenberuflich](https://www.dak.de/dak/leistungen/nebenberuflich-selbststaendig-krankenversicherung_64014). Verify with own Krankenkasse before the flip.
