---
type: area
status: active
created: 2026-04-21
updated: 2026-09-30
tags: [recurring]
---

# Recurring — subscriptions, memberships, insurances, fixed bills

> [!NOTE] Summary
> Built 2026-09-30 from Sparkasse statements 2024-01 → 2026-08 (`transactions/_recurring-candidates.csv`),
> Google Play subscriptions (2026-09-30) and PayPal automatic-payment list. "Bounced" = Rücklastschrift in
> Jun–Aug 2026 → the merchant now holds an unpaid claim (Mahnung/Inkasso likely). Decide **keep / kill / settle**
> per row; record Kündigung date and confirmation in the last column.

## Active or unresolved (statement-verified unless marked)

| Name | What | € / month | Billed via | Last seen | Bounced Jun–Aug 26? | Decision | Cancelled / settled |
|------|------|-----------|-----------|-----------|--------------------|----------|---------------------|
| Telekom Mobilfunk (Kundenkonto 0055959706) | mobile contract, RG ~120–190/mo, **4,400 € over 25 months** | ~135 avg | Lastschrift | 2026-05 paid; 174.54 + 178.54 bounced Jun | **yes** | settle arrears, downgrade or cancel at end of term — check Mindestlaufzeit | [ ] |
| LAGERBOX Frankfurt (Einheit 3338, Vertrag 122100275) | storage unit; 279–588/mo in 2024–25, 82.95 since 2026 | 82.95 | Lastschrift; Aug 26 via Klarna 392 | 2026-08 (Klarna) | Jun only — arrears settled with Lagerbox, one manual payment Aug/Sep, debit resumes Oct (luis) | keep only if contents > cost | [ ] confirm Oct debit goes through |
| Hetzner Online (K0637893825) | server — 57.27/57.33 bounced Jun/Jul, 5.45 typical earlier | 5–57 | Lastschrift | 2026-08 | **yes** | reconcile with `infra-docs` (VPS supposedly retired) — cancel leftover | [ ] |
| Adobe (via PayPal) | Creative Cloud? 23.79 | 23.79 | PayPal | 2026-08 active | no | kill unless used for client work | [ ] |
| FitSevenEleven (Vertrag 26-5048) | gym, 23/2 weeks = 46/mo | 46 | Lastschrift | 23 ×3 bounced Jun–Jul | **yes** | cancel in writing; settle | [ ] |
| RSG Group (McFit/John Reed/Gold's, F0001-25140770) | gym "Sommer Special" 24.90 + fees 59.80 bounced Aug | 24.90 | Lastschrift | 2026-08 bounced | **yes** — escalating fees | cancel in writing; settle | [ ] |
| DB Vertrieb Abo 061012510 | Deutschlandticket 63 | 63 | Lastschrift | 63 bounced Jul | **yes** | Zahlungserinnerung seen Jul 13 — settle, decide keep vs. RMV | [ ] |
| DB BahnCard 7081411280147 | BahnCard 199/yr bounced Jul 14 | 16.58 | Lastschrift | 2026-07 bounced | **yes** | cancel (auto-renews yearly) + settle | [ ] |
| Amazon Prime | 8.99 | 8.99 | Lastschrift | bounced Jun | yes | kill or settle | [ ] |
| Amazon Prime Video Channels | 3.99 + 5.99 + 9.99 (two channels bounced Jun) | ~20 | Lastschrift | bounced Jun | yes | kill | [ ] |
| Audible | 9.95 | 9.95 | Lastschrift | 2026-05 | no | kill | [ ] |
| HP Instant Ink (Worldpay) | 4.99 | 4.99 | Lastschrift | bounced Jun/Jul; account locked (Gmail) | yes | kill | [ ] |
| EUROBILL TECH / centrohelp.com | adult-site premium subscription (luis) 14.99 | 14.99 | Lastschrift | 2026-05 paid, Jun bounced | yes | cancelled already (luis, to confirm) — check no Oct debit; the bounced 14.99 may still be dunned | [ ] |
| Proton (PayPal) | 12.99 | 12.99 | PayPal | 2026-05 | no | keep if primary mail stays Proton; else downgrade | [ ] |
| Spotify | 12.99 | 12.99 | Google Play | next 2026-10-10 | — | keep or kill | [ ] |
| SoundCloud Go | 5.99 | 5.99 | Google Play | next 2026-10-10 | — | kill? | [ ] |
| ChatGPT Plus | 21.99 | 21.99 | Google Play | next 2026-10-17 | — | keep one AI sub, not several | [ ] |
| Google One AI Pro (5 TB) | 5.49 (promo?) | 5.49 | Google Play | next 2026-10-25 | — | check price after promo | [ ] |
| Google Play Pass | 4.99 | 4.99 | Google Play | next 2026-10-29 | — | kill | [ ] |
| Unknown Play sub — €12.99 on Dec 11 2026 | cut off in export (page 2 top) | 12.99 | Google Play | — | — | identify | [ ] |
| Google Payment Ireland (via PayPal) | PayPal CSV 2026-09-30 resolves these: **137.50/mo Jan–Mar 2026 then 275.00 on 2026-04-02** (= Google AI Ultra, 50 % intro then full price; no charge since → cancelled?) · 29.99/mo Nov 25–Apr 26 · 33.99/mo Oct–Dec 25 · 21.99/mo Jun 25–Jan 26 (ChatGPT via Play, now on SCU card) · 4.19 weekly Mar–Apr 26 · 10.99 monthly | var. | PayPal | 2026-09-25 (5.49) | 10.99 bounced Jun | confirm AI Ultra is dead; identify 29.99 and 33.99 (Play order history) | [ ] |
| Crunchyroll | 6.99–8.99 | 8.99 | PayPal | 2026-05 | no | kill | [ ] |
| Medium | 4.44 | 4.44 | PayPal | 2026-05 | no | kill | [ ] |
| freeCodeCamp donation | 4.46 | 4.46 | PayPal | 2026-05 | no | your call | [ ] |
| G2A | 2.96/mo (G2A Plus?) | 2.96 | PayPal | 2026-06 | no | kill | [ ] |
| ImmobilienScout24 Plus | 19.99 | 19.99 | PayPal | 2026-05 | no | kill unless actively searching | [ ] |
| Ffm Art of Boxing | boxing gym 125/mo Dec 25–Jan 26; 290 + 105 gear (bounced Nov 25) | 125? | Lastschrift | 2026-01 | Nov 25 | still a member? cancel or confirm ended | [ ] |
| RMV Handyticket | 58 bounced Sep + Nov 2025 | — | Lastschrift | 2025-11 | 2025 | likely Mahnung outstanding | [ ] |
| Sparkasse Kontoführung ("siehe Anlage Nr. 1") | account fee ~21–45/mo?! (44.68 in Apr 26) | 21–45 | fee | 2026-08 | — | check Kontomodell; Basiskonto/online model is cheaper | [ ] |
| Klarna | 426 + 392 in Aug 26 (Kleinanzeigen + Lagerbox) — instalments? | ? | Lastschrift | 2026-08 | — | check open Klarna balance | [ ] |
| AOK | not visible in Sparkasse — paid via employer or other account? | ? | ? | — | — | confirm | [ ] |

Cancelled/ending (Play): Ground News ends 2026-11-10 · stats.fm ends 2026-12-04. Expired Play subs Apr–Aug 2026 (Claude Max, Perplexity, CamScanner, WEBTOON, one sec, AppBlock, Chess Platinum, Total Launcher, PDFelement) — dead, nothing to do.

**Monthly total if everything above kept:** ~450–520 € excl. Telekom arrears. **Bounced-debit arrears (Jun–Aug 26, principal only):** Telekom ~353 · FitSevenEleven 69 · RSG ~113 · DB 63 + 199 · Hetzner ~115 · Lagerbox 83 · Amazon ~25 · HP 10 · Eurobill 15 · PayPal 17.99 → **≈ 1,060 €** before Mahngebühren.

## Not subscriptions (PayPal "automatic payments" list = merchants with a billing agreement, not recurring charges)
Booking.com · Uber · Bolt/Voi/Lime (e-scooter commute — pay-per-ride, budget category "transport") · Flink · Takeaway/Lieferando · iHerb · Steam · Coinbase/MoonPay/Payward(Kraken) · Verifone/Paddle/2checkout (payment processors) · Athletic Greens (check — AG1 is normally a subscription; none seen in Sparkasse) · Dynalist (was 4.81/mo until 2026-03, ended).

Google Play subscriptions are charged to the **Service Credit Union card** (USD) *and* via PayPal (EUR) — both feeds needed. PayPal CSV (2024-01 → 2026-09) is at `~/Documents/PayPal/paypal.CSV`; nearly every line is bank-funded from Sparkasse, so it adds merchant names, not new money.

## Next action
- [ ] Identify: the €12.99 Dec-11 Play sub, the Google 10.99/17.99 items, Klarna balance
- [ ] Kill pass: mark Decision column; I draft Kündigungen in one batch (German, one paragraph each, Einschreiben where notice period matters)
- [ ] Settle pass: bounced-debit creditors → each becomes a `debts.md` row; check letter pile for their Mahnungen
- [ ] Ask Sparkasse for cheaper Kontomodell
