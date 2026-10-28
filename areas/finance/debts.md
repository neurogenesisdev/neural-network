---
type: area
status: active
created: 2026-09-29
updated: 2026-09-30
tags: [budget]
---

# Debts and claims — money view

> [!NOTE] Summary
> One row per counterparty: what I owe, what I'm owed. Process, letters and log live in the linked
> case. Amounts are `letter` (statement or document), `luis` or `assumed`. The bounced-debit block
> is new (2026-09-30): every Rücklastschrift Jun–Aug 2026 is a creditor who now holds a claim.

## I owe

| Counterparty | Amount | Basis | Ratenplan | Case / note |
|--------------|--------|-------|-----------|-------------|
| Jobcenter / Inkasso-Service BA | several k (luis, approx.); ≤ ~2,815 for Aug–Dec 25 overlap (assumed) | Bürgergeld overlapping FM wages | none | [[../admin/cases/jobcenter-rueckforderung]] |
| Stadt Frankfurt (Abschleppung) | unknown | Gebührenbescheid expected | none | [[../admin/cases/abschleppung-hd-oo-661]] |
| Telekom Deutschland | ≥ 353 (174.54 + 178.54 bounced Jun 26) + fees (letter) | mobile contract arrears | none | → case when Mahnung found; [[recurring]] |
| DB Vertrieb | 63 (Abo Jul 26) + 199 (BahnCard Jul 26) bounced (letter) | Deutschlandticket + BahnCard | none | Zahlungserinnerung 2026-07-13 in Gmail |
| RSG Group (McFit) | 24.90 + 28.40 + 59.80 bounced Jul–Aug 26 (letter) — fees escalating | gym | none | [[recurring]] |
| FitSevenEleven | 3 × 23 bounced Jun–Jul 26 (letter) | gym | none | [[recurring]] |
| Hetzner Online | 57.27 + 57.33 bounced Jun–Jul 26 (letter) | server | none | reconcile with `infra-docs` |
| LAGERBOX Frankfurt | 82.95 bounced Jun 26 (letter); Aug paid via Klarna 392 | storage unit | Klarna? | **contents at risk if unpaid** |
| Amazon (Prime, Video Channels, Kindle) | 8.99 + 5.99 + 9.99 (Jun 26) + 11.75 (Nov 25) bounced (letter) | subscriptions | none | [[recurring]] |
| HP Instant Ink / Worldpay | 2 × 4.99 bounced (letter); account locked | subscription | none | [[recurring]] |
| EUROBILL TECH / centrohelp.com | 14.99 bounced Jun 26 (letter) | **unidentified** | none | identify first |
| PayPal | 17.99 bounced Jun 26 (letter) → PayPal balance likely negative | — | none | check PayPal balance |
| Klarna | 426.34 (Kleinanzeigen) + 392 (Lagerbox) Aug 26 (letter) | instalments? | ? | check open Klarna account |
| RMV | 2 × 58 bounced Sep/Nov 25 (letter) | Handyticket | none | probably Mahnung outstanding |
| Ffm Art of Boxing | 105 bounced Nov 25 (letter) | gear | none | check membership status |
| Inkasso / Ämter / Anwälte (unknown) | unknown | letter pile | none | → cases after intake |
| Deutsche Bahn Zahlungserinnerung (Jul 13) | ? | Gmail | — | probably = DB row above |

**Bounced-debit arrears, principal only:** ≈ 1,060 € (Jun–Aug 26) + 2025 items ≈ 220 €. Mahn- and Inkassogebühren on top.
**Total owed (known, excl. Jobcenter + city):** ≈ 1,300–1,500 € · **incl. Jobcenter (assumed):** ≈ 4–5k

## Paid enforcement (closed, for the record)
| Counterparty | Amount | Date | Reference |
|---|---|---|---|
| Hauptzollamt Gießen (Pfändung 18.09.2024) | 129.86 | 2024-10-21 | AZ 163001-2024-3420, Pf.Nr. 112564 |
| Hauptzollamt Gießen (Pfändung 02.08.2025) | 53.39 | 2025-09-03 | AZ 173425-2025-3420, Pf.Nr. 131210 |
| Stadtverwaltung Frankfurt Amt 21 (Pfändung 01.04.2026) | 761.83 | 2026-05-05 | AZ FFM.2026.033673.9, Pf.Nr. 146347 — purpose unknown |

## I am owed

| Counterparty | Amount | Basis | Case |
|--------------|--------|-------|------|
| FM Safe & Protect (Insolvenz) / Insolvenzgeld | ~1,800–2,300 net (letter: prior wages) | Dec 2025 wages | [[../admin/cases/fm-safe-protect-insolvenz]] |
| Lilja Reifegerste | 8,900–9,790 (letter: 10–11 × 890) + items ~1,500 (luis) | overpaid rent after tenancy ended | [[../admin/cases/vermieterin-rueckforderung]] |

**Total claimable:** ≈ 12–13.5k before any offsets

## Rules
- A row appears here only if a case exists or a `recurring.md` row points here.
- Update the amount here the same session a Bescheid is filed in `correspondence/`.
