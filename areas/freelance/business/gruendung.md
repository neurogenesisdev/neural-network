---
type: decision-basis
status: active
created: 2026-09-09
updated: 2026-09-16
tags: [gruendung, tax, legal]
---

# Gründung — Freiberufler vs. Gewerbe, and the checklist

> [!NOTE] Summary
> Recommendation (2026-09-09): **register a Gewerbe** (Einzelunternehmen,
> Kleingewerbe, Kleinunternehmer). Reason: no degree means the Freiberufler
> route needs a Wissensprüfung, the service mix is mostly gewerblich anyway,
> and the money difference at this scale is roughly zero. Not legal or tax
> advice — every line names who verifies it.

## 1. What the Finanzamt looks at

**The test.** Income is freiberuflich (§18 EStG) only if the activity is a
Katalogberuf (listed profession: Ingenieur, Architekt, Arzt, Journalist,
Künstler …) or an "ähnlicher Beruf" — a job that matches a listed one in both
*what you do* and *what you were trained in*. Everything else self-employed is
Gewerbe (§15 EStG). Verify: Finanzamt via the Fragebogen zur steuerlichen
Erfassung; IHK Frankfurt Gründungsberatung (free).

**Software development.** Counts as "ingenieurähnlich" when it is real
engineering-level work (designing and building qualified software; the old
Systemsoftware-vs-Anwendersoftware split was dropped by the BFH in 2004).
The catch is the training half of the test: an **Autodidakt** must prove
knowledge comparable in breadth and depth to a Diplom-Informatiker (FH) —
either through own, documented, complex projects or a **Wissensprüfung**
(an exam by a court-appointed expert, at his own cost, typically four figures;
BFH IV R 8/10, 2011). Two websites plus n8n flows will not carry that.

**Web design.** Freiberuflich only when it is *artistic* design work by someone
with a design qualification (FG ruling 2008; existenzgruendungsportal.de).
Building sites from templates/components with a technical focus is
gewerblich. Verify: existenzgruendungsportal.de, IHK.

**Automation setup (n8n, scripts).** Configuring tools for a client is
"EDV-Dienstleistung" — gewerblich unless it is engineering-level and the
training test is passed (same Autodidakt problem).

**Consulting.** "Beratender Betriebswirt" is a Katalogberuf but requires an
economics degree or equivalent. Without it: gewerblich.

**Mixed activities, Einzelunternehmer.** No Abfärbung (the "one gewerblich
partner colours the whole firm" rule) — that hits partnerships only. For a
sole proprietor: *trennbar* (separable) activities are taxed separately, each
by its own nature, with separate bookkeeping; *untrennbar* (one contract,
one price, design + build + automation) is classified as a whole by its
dominant element. A "website + booking automation" package is untrennbar and
gewerblich. Running two ledgers to keep a freiberuflich sliver is not worth it
at this size.

**Bottom line for this mix:** Finanzamt would classify websites-for-creators +
automation + consulting, delivered by an autodidact, as gewerblich with high
probability (mentor estimate ≥ 80 %). Verify: IHK Gründungsberatung, or a
Steuerberater for 1 h.

## 2. Consequences side by side

| | Gewerbe | Freiberufler |
|---|---|---|
| Registration | Gewerbeanmeldung at Ordnungsamt/Gewerbemeldestelle Frankfurt (fee ≈ 20–40 €, verify on frankfurt.de; online via Hessen service portal). Gewerbeamt informs Finanzamt and IHK | Fragebogen zur steuerlichen Erfassung only |
| Fragebogen zur steuerlichen Erfassung | Via ELSTER within 1 month of start (§138 AO). Get Steuernummer for the business | Same |
| IHK | Compulsory membership. Existenzgründer with Gewinn < 25,000 €/yr pay no Grundbeitrag in the first two years; Kleingewerbe with Gewinn ≤ 5,200 €/yr stays beitragsfrei (verify: IHK Frankfurt Beitragsordnung) | None |
| Gewerbesteuer | Freibetrag 24,500 € Gewerbeertrag (profit) — below that: 0 €. Above: Frankfurt Hebesatz 460 % (IHK Frankfurt, 2025/2026). §35 EStG credits 4.0 × Messbetrag against Einkommensteuer, i.e. up to a 400 % Hebesatz. Residual in Frankfurt ≈ 0.2 % of profit above the Freibetrag — at 30,000 € profit ≈ 115 €/yr | None |
| Bookkeeping | EÜR (simple income-minus-expenses statement) until revenue > 800,000 € or profit > 80,000 € | EÜR always |
| Umsatzsteuer | Identical (see §19 below) | Identical |
| Rentenversicherung | None (not a Pflichtversicherter category) | None for developers (Lehrer/Erzieher would be) |
| Kammer / Berufsrecht | IHK only | None |
| Switching later | Possible: deregister Gewerbe if the Finanzamt accepts freiberuflich status later (e.g. after a degree) | Reclassification = back to Gewerbe |

**Reclassification risk.** Registering as Freiberufler and being reclassified
(often years later, at a Betriebsprüfung): Gewerbesteuer back-assessed (0 € below
the Freibetrag, so cheap), IHK contributions back-billed for up to ~4 years,
a possible fine for the missed Gewerbeanmeldung (§146 GewO, in practice rare
and small), and the hassle. Registering a Gewerbe that was actually
freiberuflich: no penalty, small IHK cost, switch on request. The asymmetry
favours Gewerbe. Verify: Steuerberater.

## 3. Kleinunternehmerregelung §19 UStG (verified 2026-09-09)

- Previous calendar year total revenue ≤ **25,000 €** and current year ≤ **100,000 €** (net figures). Crossing 100,000 € mid-year: VAT applies from that sale onward, not retroactively.
- Founding year: no previous year, so the 25,000 € limit is tested against the founding year itself (not annualised under the post-2025 rule — verify with Finanzamt/IHK; sources disagree).
- Invoice line: "Gemäß § 19 UStG wird keine Umsatzsteuer berechnet." No VAT shown, no Vorsteuer reclaimed.
- Elect it in the Fragebogen. Opting out binds for 5 years.
- Meaning for pricing: quoted prices are final prices. That is a ~19 % edge for clients who cannot deduct VAT (private persons, other Kleinunternehmer — many solo instructors and creators); no edge for VAT-registered businesses.
- Sources: [taxtify](https://taxtify.de/steuer-lexikon/kleinunternehmerregelung/), [IHK Stuttgart](https://www.ihk.de/stuttgart/fuer-unternehmen/recht-und-steuern/steuerrecht/umsatzsteuer-national/kleinunternehmerregelung-in-der-umsatzsteuer-1843632). Re-verify each January.

## 4. Recommendation

**Register a Gewerbe, as Einzelunternehmen, Kleinunternehmer, EÜR.** Reasons, in
order: (1) the training test is the wall — an autodidact with two portfolio
sites cannot credibly claim ingenieurähnlich status, and a Wissensprüfung
costs more than years of IHK dues; (2) the actual service mix (templated
sites, tool configuration, packages) is gewerblich even for a graduate;
(3) financial difference below 24,500 € profit is exactly 0 €, above it
≈ 0.2 % of the excess; (4) the Gewerbe path has no downside if the Finanzamt
later disagrees, the Freiberufler path does. Revisit only if he completes a
recognised IT qualification or the work shifts to genuine bespoke software
engineering.

Decided 2026-09-09 (Luis + Mentor). **Confirmed 2026-09-16 by the IHK Frankfurt**
(Christoph Ewald, Team Unternehmensförderung, by email): "Die Tätigkeit als
Webentwickler ist rein gewerblich." Logged as
[[../decisions/fdr-006-gewerbe-einzelunternehmen-kleinunternehmer|FDR-006]].
Only *Unternehmensberatung* can be freiberuflich in individual cases — IHK
points to the Institut für freie Berufe (freie-berufe.com) for that; not worth
pursuing for this mix.

## 5. Gründung checklist

| # | Step | When | Who verifies / where |
|---|------|------|---------------------|
| 1 | Send the §14 notification email ([[nebentaetigkeit-anzeige]]). File the Textform consent | **Done** — sent 2026-09-15, Zustimmung erhalten 2026-09-16 | PDF next to the Arbeitsvertrag |
| 2 | IHK Gründungsberatung | **Done without an appointment** — requested 2026-09-15, answered by email 2026-09-16 (see §7). All four questions answered; no meeting needed | IHK Frankfurt, unternehmensfoerderung@frankfurt-main.ihk.de, +49 69 2197-2010 |
| 3 | **Gewerbeanmeldung** — there is **no online portal**: fill the PDF form **GewA 1** (Frankfurt version 32.2-11/2019), sign it, and email it to gewerberegister@stadt-frankfurt.de (CC gewerberegister.amt32@stadt-frankfurt.de) or post it to Stadt Frankfurt am Main, Der Magistrat – Amt 32, Gewerberegister, 60275 Frankfurt. **Fee 36,00 € incl. Empfangsbescheinigung (Gewerbeschein)**, or 28,00 € without — take the 36 €, the Gewerbeschein is what Finanzamt, bank and clients ask for. **Pay first** — Empfänger **Stadt Frankfurt am Main** (the city's own Bankverbindungen sheet lists "Kontoinhaber: Stadt Frankfurt am Main", not Ordnungsamt/Stadtkasse; since Oct 2025 banks verify payee name against IBAN), IBAN DE95 5001 0060 0007 1496 02, BIC PBNKDEFF, 36,00 €, Verwendungszweck "0320.51000000GewAnzeige Luis Gordon" — the clerk matches the payment by the name on the form, so use the personal name, not the Geschäftsbezeichnung and attach the payment receipt to the same mail — without it they issue a separate chargeable payment demand. Also attach a copy of the Personalausweis (front/back). Full field-by-field values: §8 below. Form fields: **Tätigkeit** "Erstellung, Betreuung und Wartung von Websites und Automatisierungslösungen; Einrichtung von und Beratung zu KI-gestützten Anwendungen und Arbeitsabläufen; Erstellung digitaler Medieninhalte; IT-Dienstleistungen" (revised 2026-09-16 to cover maintenance, AI services and media production. Deliberately omits "Schulungen/Unterricht": unterrichtende Tätigkeit is freiberuflich under §18 EStG and listing it invites a split into two income categories with separate bookkeeping — the mess [[../decisions/fdr-006-gewerbe-einzelunternehmen-kleinunternehmer|FDR-006]] avoids. Teaching inside an engagement is fine; it is delivered as Beratung); **Beginn** = filing date, not backdated, not future; **im Nebenerwerb** ticked; **Betriebsstätte** Am Poloplatz 8 (same as the Impressum) | **ERLEDIGT 2026-09-17** — GewA 1 unterschrieben per E-Mail an gewerberegister@stadt-frankfurt.de (CC amt32) gesendet, zusammen mit dem Zahlungsbeleg; 36,00 € am selben Tag überwiesen (Empfänger "Ordnungsamt Stadt Frankfurt am Main", Sparkasse, Standardüberweisung). Beginn der Tätigkeit: 17.09.2026. Ausweiskopie im Anschreiben angeboten, nicht mitgeschickt. **Warten auf die Empfangsbescheinigung (Gewerbeschein).** | frankfurt.de, Gewerberegister Ordnungsamt |
| 4 | **NÄCHSTER SCHRITT — fällig bis 17.10.2026.** Fragebogen zur steuerlichen Erfassung via ELSTER: Einzelunternehmen, Gewinnermittlung EÜR, Kleinunternehmer ja, expected revenue year 1 < 25,000 €, Abgabe USt-Voranmeldung nein. Get Steuernummer | Within 1 month of step 3 | Finanzamt Frankfurt (ELSTER) |
| 5 | Business bank account or separate sub-account (not legally required for Einzelunternehmen, but keeps EÜR clean) | With step 3 | — |
| 6 | Invoice template with Pflichtangaben (§14 UStG): full name + address, client, Steuernummer, invoice number (sequential), date, service description + period, net amount, §19 sentence, payment terms. E-Rechnung: must be able to *receive* XRechnung/ZUGFeRD from business clients; issuing plain PDF is fine for Kleinunternehmer (verify current status) | Before first paid invoice | templates/, Steuerberater or IHK |
| 7 | Krankenkasse: optional written notice "nebenberuflich selbständig, < 20 h/week, Gewinn below salary" and ask for written confirmation of nebenberuflich status. Protects against later re-rating | After step 3 | Own Krankenkasse |
| 7b | **Berufsgenossenschaft** (statutory accident insurance): the Gewerbeamt forwards the registration, but membership is mandatory and must be notified within one week of starting — likely VBG for IT services. Contribution for a sole proprietor with no employees is small (Mindestbeitrag), but it is not optional | Within 1 week of step 3 | VBG / the BG named in the letter that follows the Gewerbeanmeldung |
| 8 | Insurance: Berufshaftpflicht/IT-Haftpflicht (covers data loss, deadline damage) — optional at pro-bono stage, take it before the first paid client with customer data; typical 15–30 €/month for this size | Before first paid client | exali, Hiscox, or broker |
| 9 | Steuerberater: 1 h consult before the first Einkommensteuererklärung with business income (2027 for tax year 2026) or when revenue passes ~10,000 € | Q1 2027 | Steuerberater |
| 10 | Annual: re-verify §19 thresholds, IHK notice, Gewerbesteuer if profit > 24,500 €, KV boundary ([[model]]) | Every January | — |

## 6. Employer email check

The drafted email says "selbstständige Tätigkeit als Webentwickler und
Automatisierungsberater" and does not name a legal form. A Gewerbe needs no
change: the clause is about Nebenbeschäftigung, and the employer has no
interest in whether the Finanzamt calls it Gewerbe or freier Beruf. Send as is.

## 8. GewA 1 — field values (prepared 2026-09-17)

| Feld | Wert |
|---|---|
| 1, 2 | leer (kein Handelsregistereintrag) |
| 3 Geschäftsbezeichnung | `neurogenesis.dev` — optional, matches [[../decisions/fdr-004-business-identity-on-neurogenesis-dev\|FDR-004]] and the Impressum. No Rechtsformzusatz, kein "Agentur". A designation is **not** a trademark and grants no name protection |
| 4 / 5 / 6 | Gordon / Luis / Männl. |
| 7 | leer |
| 8 / 9 / 10 | Geburtsdatum / Geburtsort und -land / deutsch |
| 11 Wohnung | Am Poloplatz 8, 60528 Frankfurt am Main; Tel. +49 152 33552973; freiwillig: luiswg@neurogenesis.dev / neurogenesis.dev |
| 12, 14 | leer |
| 13 Öffentliche Hand | Nein |
| 15 Betriebsstätte | identisch mit Feld 11 |
| 16, 17 | leer |
| 18 Tätigkeit | "Erstellung, Betreuung und Wartung von Websites und Automatisierungslösungen; Einrichtung von und Beratung zu KI-gestützten Anwendungen und Arbeitsabläufen; Erstellung digitaler Medieninhalte; IT-Dienstleistungen" — Schwerpunkt "Erstellung … von Websites und Automatisierungslösungen" unterstreichen |
| 19 Nebenerwerb | **ja** |
| 20 Beginn | filing date, not backdated |
| 21 Art des Betriebes | **Sonstiges** (Dienstleistung) |
| 22 Personen | Keine |
| 23 | eine Hauptniederlassung |
| 24 | leer (kein Reisegewerbe) |
| 25 Grund | Neugründung → Neuerrichtung |
| 26, 27 | leer |
| 28–31 | Nein / leer |
| 32, 33 | Datum, Unterschrift |

## 7. IHK answers (2026-09-16, Christoph Ewald, Team Unternehmensförderung)

1. **Classification:** "Die Tätigkeit als Webentwickler ist rein gewerblich." Unternehmensberatung can be freiberuflich in individual cases — deeper advice via the Institut für freie Berufe, Universität Erlangen (https://www.freie-berufe.com/). Not pursued.
2. **Sequence confirmed as planned:** Gewerbeanmeldung at the Gewerbeamt → Fragebogen zur steuerlichen Erfassung incl. Kleinunternehmerregelung. The Gewerbeamt forwards the data to the Kammer and the **Berufsgenossenschaft**; letters from both follow.
3. **Krankenversicherung:** status depends on weekly hours and income; if self-employment income exceeds the employment income, status flips to hauptberuflich selbständig. IHK brochure "Existenzgründung im Nebenerwerb": https://www.frankfurt-main.ihk.de/blueprint/servlet/resource/blob/5267564/be821aaf6dd060add9edcb74b2190652/hihk-existenzgruendung-im-nebenerwerb-data.pdf — matches [[model]].
4. **Rechnungen / Impressum:** IHK pages on Angaben auf Geschäftsbriefen and Impressum; named contacts Frau Molino Ortega and Frau Hennig. Free web-law advice also via **BIEG Hessen** (https://www.bieg-hessen.de/beratung/) — useful before client sites go live.

Contact for follow-ups: unternehmensfoerderung@frankfurt-main.ihk.de, +49 69 2197-2010.

## Appendix — IHK Gründungsberatung request (2026-09-15)

Booked via the form / address on the IHK Frankfurt "Existenzgründung" page (do not guess an address).

```
Betreff: Terminanfrage Gründungsberatung – Nebenerwerb, Webentwicklung/Automatisierung

Sehr geehrte Damen und Herren,

ich plane die Aufnahme einer selbstständigen Tätigkeit im Nebenerwerb (neben einer Teilzeitanstellung) als Webentwickler und Automatisierungsberater für Selbstständige und kleine Unternehmen in Frankfurt am Main und bitte um einen Termin für eine Gründungsberatung.

Meine Fragen:
1. Einordnung als Gewerbe oder freiberufliche Tätigkeit (Websites, Automatisierungen, Beratung; keine formale IT-Ausbildung)
2. Ablauf und Reihenfolge: Gewerbeanmeldung, Fragebogen zur steuerlichen Erfassung, Kleinunternehmerregelung (§ 19 UStG)
3. Nebenerwerb neben einer Teilzeitanstellung: Krankenversicherung und Meldepflichten
4. Pflichtangaben auf Rechnungen und Website (Impressum) als Einzelunternehmer

Ich bin zeitlich flexibel; ein Termin vor Ort oder per Video ist beides möglich.

Mit freundlichen Grüßen
Luis Gordon
Am Poloplatz 8, 60528 Frankfurt am Main
luiswg@neurogenesis.dev
```

## Sources checked 2026-09-09

- [existenzgruendungsportal.de — Webdesigner freiberuflich oder gewerblich](https://www.existenzgruendungsportal.de/Redaktion/DE/BMWK-Infopool/Antworten/Gruendungsplanung/Freie-Berufe/kuenstl-publ-Taetig/Webdesigner-freiberufliche-oder-gewerbliche-Taetigkeit)
- [e-recht24 — Webdesigner Freiberufler](https://www.e-recht24.de/unternehmensgruendung/11066-webdesigner-freiberufler.html)
- [lexware — §18 EStG](https://www.lexware.de/wissen/unternehmerlexikon/paragraph-18-estg/)
- [IHK Frankfurt — Realsteuerhebesätze 2025/2026](https://www.frankfurt-main.ihk.de/recht/uebersicht-alle-rechtsthemen/steuerrecht/gewerbe-und-grundsteuer/realsteuerhebesaetze-5195618)
- BFH IV R 8/10 (Autodidakt) and the 2004 software ruling — from memory, verify with IHK/Steuerberater
