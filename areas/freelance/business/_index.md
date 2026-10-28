---
type: index
owner: mentor
created: 2026-09-09
updated: 2026-09-21
tags: [index, gruendung, tax, insurance, pricing]
employer: DDK Detektei
---

# Business

> [!NOTE] Summary
> Gründung, tax, insurance, employment constraints. Luis is not registered yet.
> None of this is legal or tax advice; each line names who verifies it.

## Status

| Item | Status | Trigger / verify with |
|------|--------|-----------------------|
| Employment contract — Nebentätigkeit clause | **Erledigt.** Anzeige sent 2026-09-15, Zustimmung in Textform erhalten 2026-09-16 (Lisa Lorber, Assistenz der Geschäftsführung). Conditions: Vorrang der Haupttätigkeit, keine Beeinträchtigung von Leistung/Verfügbarkeit/Zuverlässigkeit, Arbeits- und Ruhezeiten. Revocable per §14(3) | PDF filed next to the Arbeitsvertrag. Keep `hours.csv` current — it is the evidence if the Ruhezeiten condition is ever questioned |
| Gewerbeanmeldung vs. Freiberufler | **Entschieden 2026-09-16: Gewerbe** (Einzelunternehmen, Kleinunternehmer, EÜR) — IHK Frankfurt confirmed in writing that web development is "rein gewerblich". [[../decisions/fdr-006-gewerbe-einzelunternehmen-kleinunternehmer\|FDR-006]]; reasoning and checklist in [[gruendung]] | **Gewerbeanmeldung submitted** (by 2026-09-19): form filled, fee paid, sent with proof of payment by email to the Ordnungsamt/Gewerberegister; automated receipt only so far. Expected: Gewerbeschein by email in 1–3 weeks; chase after 3 weeks. ELSTER account created and authenticated 2026-09-19 (Privatperson, username luiswg, personal Gmail, login via AusweisApp + Personalausweis, plus ElsterSecure app on the phone as second method (biometrics) — no certificate file, no KeePass entry needed). Then: Fragebogen zur steuerlichen Erfassung (ELSTER, within 1 month of start) → Berufsgenossenschaft (within 1 week of start) → IHK membership letter arrives on its own |
| Kleinunternehmerregelung (§19 UStG) | expected. Verified 2026-09-09: ≤ 25,000 € previous year, ≤ 100,000 € current year (net); founding-year rule to confirm | Re-verify each January; IHK / Finanzamt |
| Health insurance — nebenberuflich boundary | not relevant yet. Rule of thumb: self-employed ≤ 20 h/week and profit below salary. Employment presumed main if > 20 h/week **and** pay > 1,977.50 €/month gross (half Bezugsgröße 2026) — check payslip | Krankenkasse; details in [[model]] |
| Invoicing (Pflichtangaben) | not needed yet | Template before first paid client |
| Scheinselbständigkeit | not relevant yet | Flag if one client exceeds ~5/6 of revenue |

## Employment contract §14 Nebentätigkeit (read 2026-09-09, transcribed from photo)

1. Der Arbeitnehmer verpflichtet sich, jede entgeltliche oder das Arbeitsverhältnis beeinträchtigende Nebenbeschäftigung vor ihrer Aufnahme dem Arbeitgeber gegenüber in Textform anzuzeigen. Sie ist nur mit Zustimmung des Arbeitgebers zulässig.
2. Der Arbeitgeber erteilt die Einwilligung, wenn die Wahrnehmung der dienstlichen Aufgaben durch die Nebenbeschäftigung nicht behindert und sonstige berechtigte Interessen des Arbeitgebers nicht beeinträchtigt werden.
3. Der Arbeitgeber kann seine Einwilligung jederzeit widerrufen, wenn sein betriebliches Interesse dies auch unter Berücksichtigung der Belange des Arbeitnehmers erfordert.
4. Der Arbeitnehmer hat bei der Wahrnehmung seiner Nebentätigkeit die gesetzlichen Bestimmungen zum Arbeitszeitgesetz zu beachten und einzuhalten. Der Arbeitnehmer stimmt seine Arbeitszeiten bei dem Nebenarbeitgeber selbstständig ab und überwacht diese im Rahmen des Arbeitszeitgesetzes.

What it means in practice (luis + mentor, 2026-09-09; not legal advice):

- Self-employment is covered: the clause says "Nebenbeschäftigung", not "Arbeitsverhältnis", and §4 names self-organised hours. Consent is the gate, not the legal form.
- Notify in Textform (email is enough) **before** starting paid work, and get the consent in Textform too. A verbal okay at the interview is not consent under this clause.
- Consent is revocable at any time (§3). Standard: never sign a client commitment that a revocation would leave stranded — one project at a time until the practice has a track record; retainers need a termination clause of ≤ 1 month.
- ArbZG applies contractually to the combined hours (job + freelance): max 8 h/day on average, 10 h peak, 11 h rest before the next shift, 48 h/week average. Luis's stance (2026-09-09): record freelance hours; allocate them so no legal problem arises; no hard "11 h before a shift" rule — he treats the freelance work as recreational. Mentor's note: the enforcement risk is the employer (§3 revocation), not the state; the hour log is the protection.
- Employer: DDK Detektei (luis, 2026-09-09).
- Notification email drafted 2026-09-09 (Mentor chat). Checked 2026-09-09 against the Gewerbe recommendation: no change needed, send as is.

## Notes

| Note | Description |
|------|-------------|
| [[nebentaetigkeit-anzeige]] | Employer notification email per §14 — draft, not sent (2026-09-09) |
| [[gruendung]] | Freiberufler vs. Gewerbe comparison, §19 thresholds, recommendation, Gründung checklist with dates (2026-09-09) |
| [[model]] | Business model: revenue mix at 10–20 h/week, Krankenkasse boundary, price-raise triggers (2026-09-09) |
| [[site-design-brief]] | neurogenesis.dev redesign brief: purpose ranking, audiences, visual direction, content inventory, deferred backlog (logo, headshot, testimonials, articles), 15 h cap, done criteria — **active**, handed to the Architect (2026-09-21) |
| [[site-redesign-round-one]] | Architect's plan for the redesign: design inventory vs. brief, 3 accent blues to pick, Claude Code task list T0–T10 (~14.5 h, shippable at every cut), demo-loop plan (demo Gmail + fabricated inbox); routing decisions in infra-docs ADR-0038 (2026-09-21) |
| [[web-presence]] | neurogenesis.dev business pages: Impressum facts, publication gate, /services copy, contact (Cal.com), newsletter, Datenschutz processors — handed to the Architect (2026-09-14) |
