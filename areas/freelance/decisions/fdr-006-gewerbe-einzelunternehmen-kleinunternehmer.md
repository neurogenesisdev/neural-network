---
type: decision
id: FDR-006
aliases: [FDR-006]
status: accepted
impact: high
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-16
updated: 2026-09-16
tags: [decision, business, gruendung]
---

# FDR-006 — Gewerbe as Einzelunternehmen, Kleinunternehmer, EÜR

> [!NOTE] Summary
> The practice is registered as a Gewerbe (Einzelunternehmen), taxed under the
> Kleinunternehmerregelung (§ 19 UStG), with EÜR bookkeeping. The Freiberufler
> route is closed: the IHK Frankfurt confirmed in writing on 2026-09-16 that
> web development is "rein gewerblich".

## Context

The legal form had to be settled before the Gewerbeanmeldung, which in turn
gates the public /services page ([[fdr-005-no-public-prices-until-first-paid-quote|FDR-005]])
and any paid quote. The Mentor's analysis of 2026-09-09 ([[../business/gruendung|gruendung]])
recommended Gewerbe on three grounds: an autodidact cannot pass the training
half of the "ingenieurähnlich" test without an expensive Wissensprüfung; the
actual service mix (templated sites, tool configuration, packages) is gewerblich
even for a graduate; and below 24,500 € profit the financial difference is zero.
Luis had asked for confirmation from a competent body before committing.

## Decision

On 2026-09-16, after the IHK Frankfurt answered by email, Luis decided:

1. **Gewerbe**, as **Einzelunternehmen** under his own name. No Freiberufler
   application, no Wissensprüfung, no Institut für freie Berufe referral.
2. **Kleinunternehmerregelung (§ 19 UStG)**, elected in the Fragebogen zur
   steuerlichen Erfassung. Quoted prices are final prices.
3. **EÜR** (Einnahmen-Überschuss-Rechnung) as the accounting method.

Source of the confirmation: Christoph Ewald, Team Unternehmensförderung, IHK
Frankfurt am Main — "Die Tätigkeit als Webentwickler ist rein gewerblich."
Email of 2026-09-16, filed with the mail archive.

## Alternatives considered

1. **Freiberufler (§ 18 EStG)** — no Gewerbesteuer, no IHK membership, no
   Gewerbeanmeldung. Rejected: the IHK states the activity is gewerblich; an
   autodidact would need a Wissensprüfung (four figures, own cost) to argue
   otherwise, and reclassification years later is the more expensive failure mode.
2. **Splitting the activities** to keep a freiberuflich sliver (consulting)
   alongside the gewerblich work — legally possible for a sole proprietor when
   the activities are separable, but requires two ledgers and the packages Luis
   sells are untrennbar. Rejected as not worth the overhead at this size.
3. **Pursuing the Unternehmensberatung angle** via the Institut für freie Berufe
   (IHK's pointer). Rejected: "beratender Betriebswirt" needs an economics
   degree or equivalent; no degree, and the mix is dominated by build work.

## Consequences

- Positive: the classification question is closed with a written statement from
  the IHK; the Gewerbeanmeldung can proceed immediately; no risk of a later
  Betriebsprüfung reclassifying the business and back-billing IHK dues.
- Cost: IHK membership (Existenzgründer exemption applies while profit stays
  under 25,000 €; Kleingewerbe under 5,200 € profit stays beitragsfrei —
  verify against the current Beitragsordnung), Gewerbeanmeldung fee ≈ 20–40 €,
  and mandatory **Berufsgenossenschaft** membership (likely VBG), which the
  Gewerbeamt triggers automatically and which must be notified within one week
  of starting.
- Gewerbesteuer is 0 € below 24,500 € Gewerbeertrag and ≈ 0.2 % of the excess
  in Frankfurt after the § 35 EStG credit — negligible at the planned scale.
- Unblocks: [[fdr-005-no-public-prices-until-first-paid-quote|FDR-005]] gate (b).
  Once registered, /services can be linked and indexed.

## Revisit when

Luis completes a recognised IT qualification, the work shifts to genuine bespoke
software engineering, or revenue approaches the § 19 ceiling (then the pricing,
not the legal form, is what changes).
