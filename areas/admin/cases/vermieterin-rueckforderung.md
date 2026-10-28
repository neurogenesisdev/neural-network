---
type: case
status: open
created: 2026-09-29
updated: 2026-09-30
counterparty: Lilja Reifegerste (former landlady; address to add)
aktenzeichen: 
amount: "claim 8,900 (10 x 890 after tenancy end) minus arrears 2,976 (luis, document) = 5,924 net; plus Kaution ~2,670 (luis) minus documented damages; plus 3 items"
direction: claim
deadline: 
tags: [case, claim, housing, open]
---

# Lilja Reifegerste — overpaid rent + retained items

> [!NOTE] Summary
> Mietvertrag ended July/August 2024. The Dauerauftrag "Miete" (890 €) kept running: Sparkasse
> statements show 14 transfers Feb 2024 – Dec 2025. After the tenancy ended: **10 transfers = 8,900 €**
> (Oct 24, Nov 24, Dec 24, Jan 25, May 25, Sep 25, Oct 25, Nov 25, Dec 25) plus Aug 2024 (890) if the
> contract had already ended by Aug 1. She also kept a rowing machine (>1,000 €), a guitar amp (~200 €)
> and JBL speakers (~200–300 €) "as Kaution". Her counterclaim: **2,976 € Mietrückstand** (luis — document
> photographed). The real Kaution (~3 months = ~2,670 €, luis) should have covered damages with money
> left. **Net position: 8,900 − 2,976 = 5,924 €** + Kaution balance + items − documented damages.

## Facts
- Landlady: Lilja Reifegerste (luis)
- Mietvertrag ended July or August 2024 (luis, approx.) — old address on Sparkasse statements: Waldschulstr. 132, 65933 Frankfurt (letter, Kontoauszug 9/2024) — confirm this was the rented flat
- Transfers "Lilija Reifegerste Miete Luis Gordon", Dauerauftrag, 890.00 each (letter: Sparkasse Kontoauszüge, `transactions/_landlady.csv`):
  2024-02-01 · 2024-03-01 · 2024-04-02 · 2024-06-03 · 2024-08-01 · 2024-10-01 · 2024-11-04 (Wert 01.11.) · 2024-12-02 · 2025-01-02 · 2025-05-02 · 2025-09-01 · 2025-10-01 · 2025-11-03 · 2025-12-01 — total 12,460 €; no transfer in 2026
- Missing months (no funds): May 24, Jul 24, Sep 24, Feb–Apr 25, Jun–Aug 25 (letter)
- Standing order continued after the tenancy ended, through ca. July 2024 – autumn 2025; ~10 payments of ~900 € went through, some months skipped for lack of funds (luis, approx.)
- No rent debit visible in September 2024 (letter, Kontoauszug 9/2024)
- **Is the Dauerauftrag still active?** Nothing since 2025-12-01, but that may be lack of funds, not cancellation. Verify in online banking and delete it (luis → do today)
- Landlady retained rowing machine (>1,000 €), guitar amplifier (~200 €), JBL speakers (~200–300 €) against damages Luis could not pay at the time (luis)
- Items, as far as certain (luis): Concept2 RowErg · Boss Katana amp · JBL Boombox (v1); possibly more
- Landlady's own statement of missed rent during the tenancy: **2,976 €** (luis; document page photographed → file in `correspondence/` once scanned; the letter's date and her signature matter)
- Mietkaution paid: ~3 months' rent (~2,670 €) (luis, approx. — find the Kaution receipt/transfer in 2023 statements if needed); damages very likely covered by it with money to spare (luis)
- No Kautionsabrechnung (deposit settlement) received (assumed — confirm). Landlord must account for the Kaution within a reasonable period after the tenancy ends, usually ≤ 6 months (assumed)
- Whether damages were ever itemized in writing: unknown — check paper records for an Übergabeprotokoll or damage letter (luis)
- Legal basis for the rent: payment without legal obligation → § 812 BGB claim; landlady may offset genuine counterclaims (damages, arrears, utilities) (assumed — confirm via Beratungshilfe)
- Items kept without a settled, itemized damage claim: weak position for her (assumed)
- Standard limitation period 3 years from year-end of each payment → first payments (2024) expire 2027-12-31; no urgency, but no reason to wait (assumed)
- She would ideally acknowledge the total herself in writing (luis)

## Next action
- [x] Run `system/scripts/parse_sparkasse.py` → `_landlady.csv` → exact list above (2026-09-30)
- [ ] **Check Sparkasse online banking: delete the Dauerauftrag if it still exists**
- [ ] Drag the rent-arrears document photo into chat → extract date, amount breakdown, any damage list → correspondence note
- [ ] Pin down the contract end date (Mietvertrag, Kündigung, Übergabeprotokoll) — decides whether Aug 2024 counts
- [ ] Search paper records / email for Übergabeprotokoll, damage list, Kaution receipt, Mietvertrag (end date!)
- [ ] Add address to `contacts.md`
- [ ] Beratungshilfe consultation with the numbers in hand → [[beratungshilfe]]
- [ ] Then: draft Rückforderungsschreiben (German, factual, itemized, 14-day deadline, no justification paragraph)

## Waiting for
- Luis: bank export

## Correspondence
- (none yet)

## Log
### 2026-09-30 (evening)
- Luis: arrears per landlady's document 2,976 €; Kaution ~3 months; items named. Net claim recalculated to ~5,924 € + Kaution balance + items. Letter draft rewritten to net the arrears openly and demand Kautionsabrechnung.
### 2026-09-30 (later)
- Statement data in: 14 × 890 €, 10 (or 11) after the tenancy ended = 8,900–9,790 €. Amount now `letter`-grade.
### 2026-09-30
- Corrected timeline: tenancy ended mid-2024, overpayments ran during the Notunterkunft period, not after autumn 2025. Name added.
### 2026-09-29
- Case opened from Luis's account. All amounts flagged unverified until the export is in.
