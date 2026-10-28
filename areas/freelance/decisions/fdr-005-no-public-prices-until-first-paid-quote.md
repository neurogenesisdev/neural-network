---
type: decision
id: FDR-005
aliases: [FDR-005]
status: accepted
impact: low
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-14
updated: 2026-09-14
tags: [decision, pricing, marketing]
---

# FDR-005 — No public prices on neurogenesis.dev until the first paid quote; /services gated on registration

> [!NOTE] Summary
> /services lists three offers with "auf Anfrage" and a free 30-minute call,
> no from-prices. The page is built now but published only after employer
> consent and Gewerbe/Finanzamt registration.

## Context

Zero paid references; the catalog in `services/` is unvalidated; Kleinunternehmer
status (and its "no VAT" wording) is not yet confirmed by the Finanzamt.
Nothing is registered and the employer has not consented in Textform.

## Decision

On 2026-09-14 Luis decided (Mentor recommendation, accepted via the Architect handoff):

1. /services shows offers without prices — "Preis auf Anfrage, Festpreis nach einem kostenlosen 30-Minuten-Gespräch."
2. /services stays unlinked and noindex until employer consent (Textform) and the Gewerbeanmeldung/Fragebogen are done. Impressum, Datenschutz and /contact publish immediately.

## Alternatives considered

1. **From-prices ("ab 700 €")** — filters tire-kickers, but anchors low before the first real quote and would need correct VAT wording. Rejected for now.
2. **Publish /services now** — advertising paid services before registration is a Gewerbe/Finanzamt exposure and pre-empts the employer notification. Rejected.

## Consequences

- Positive: no wrong number to walk back; first quotes are made in conversation where scope can be set.
- Negative / cost: fewer self-qualified leads; every inquiry costs a 30-minute call.

## Revisit when

First paid quote sent, or the "after 3 case studies" column in `services/` becomes the active one.
