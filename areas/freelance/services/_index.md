---
type: index
owner: mentor
created: 2026-09-09
updated: 2026-09-16
tags: [index, pricing]
---

# Services & pricing

> [!NOTE] Summary
> Catalog with market-anchored ranges (checked 2026-09-09). Two columns:
> what the German market charges, and what Luis quotes now as an unproven
> freelancer with 0 paid references. All prices are final prices (Kleinunternehmer,
> no VAT — see [[../business/gruendung|gruendung]]). Every retainer: monthly,
> cancellable with ≤ 1 month notice — employer consent is revocable, so no
> commitment Luis cannot exit fast.

## Market anchors (Germany, 2026)

| Anchor | Figure | Source |
|--------|--------|--------|
| Freelancer hourly rate, all fields, DACH average | 103 €/h | [freelancermap Kompass 2026](https://www.freelancermap.de/freelancer-kompass/einkommen) |
| IT freelancer median; software & web dev; age 20–30 | 95 / 90 / 80 €/h | [freelancermap IT rates](https://www.freelancermap.de/blog/stundensatz-it-freelancer/) |
| Web designer hourly: entry / experienced freelancer / agency | 30–50 / 60–80 / 100–250 €/h | [kopfundstift](https://kopfundstift.de/webdesign-kosten/) |
| One-pager / 5-page site / 10+ pages, freelancer-to-small-agency | 1,000–2,500 / 2,500–5,000 / 5,000–12,000 € | kopfundstift |
| Freelancer website total (broad range) | 800–5,000 € | [rheinspace](https://rheinspace.de/insights/website-erstellen-lassen-kosten/) |
| Website maintenance | 25–85 €/month freelancer; "ab 50 €" typical; agencies 60–250 € | kopfundstift, rheinspace |
| n8n freelancer: hourly / simple workflow / medium project | ab 120 €/h / ab 500 € / 1,500–3,000 € | [justinkeirath.com](https://www.justinkeirath.com/n8n-freelancer/) |

Caveat: the freelancermap figures are project-freelancer rates (B2B, enterprise
clients). For creators and solo professionals the realistic ceiling is the
kopfundstift/rheinspace bracket. Uncertainty ± 20 %.

## Catalog

| Service | Format | Market range | Luis now (0–2 paid refs) | After 3 case studies | Reasoning |
|---------|--------|--------------|--------------------------|----------------------|-----------|
| One-pager / landing page (1 page, contact form, legal pages) | fixed | 1,000–2,500 € | **700–1,200 €** | 1,200–2,000 € | Entry offer for creators; ~12–18 h of work → 50–70 €/h effective. Never below 700 € — that is where DIY builders sit |
| Portfolio / brochure website (up to 5 pages, CMS the client can edit, legal pages, basic SEO) | fixed | 2,500–5,000 € | **1,500–2,800 €** | 2,800–4,500 € | Core product. 25–40 h with AI-assisted build → 50–70 €/h. Beginner discount is the missing references, not lower quality — say so |
| Website care plan (see below) | monthly retainer, ≤ 1 month notice | 25–85 € | **Basic 39 € / Standard 79 €** | 49 / 99 € | Recurring floor. Priced so a client says yes without thinking; the real value is the relationship and upsell |
| Automation setup — 1 workflow (n8n, script, or agent; e.g. booking → calendar → confirmation, form → CRM, content repurposing) | fixed | 500–3,000 € | **400–1,200 €** | 900–2,500 € | 6–15 h incl. discovery and testing. Scope by trigger + steps + 1 integration; each extra integration +150–300 € |
| Automation retainer (monitoring, error fixes, small changes, up to 2 h/month) | monthly, ≤ 1 month notice | 100–300 € | **89–149 €** | 149–249 € | Workflows break when APIs change; clients pay for "it keeps working". Cap included hours, bill the rest hourly |
| Automatisierungs-Check (60–90 min call + 1-page written plan) | fixed | 150–400 € | **free as lead magnet → 149 €** | 249 € | Free while it feeds the pipeline; charge once 2 in 3 checks convert to paid work |
| Hourly fallback (undefined work, extra revisions, training) | hourly | 60–90 € | **55–65 €/h** | 75–90 €/h | Below the 80 €/h junior IT rate by design; only for work that cannot be scoped. Minimum billing unit 30 min |

Pro-bono / portfolio terms: see [[../standards/pricing|pricing]] — written trade
(testimonial + case study + referral), still 2 revision rounds, still a scope.

## Care plan (recommended retainer)

| | Basic — 39 €/month | Standard — 79 €/month |
|---|---|---|
| Updates (CMS, plugins, dependencies), security patches | monthly | monthly |
| Backup check + restore test | monthly | monthly |
| Uptime monitoring + SSL/domain expiry watch | yes | yes |
| Content/small changes included | 30 min/month | 90 min/month |
| Response time for "site is down" | next business day | same day (within Luis's off-shift hours; state them) |
| Monthly 5-line status mail | no | yes |
| Extra work | 55 €/h | 55 €/h |
| Term | monthly, either side cancels with 1 month notice to month end | same |

Rules: unused minutes do not roll over; hosting/domain paid by the client
directly ([[../decisions/fdr-002-client-pays-third-party-costs-directly|FDR-002]]);
a care plan needs an AVV if the site stores customer data (forms, bookings).
Offer the care plan in every website proposal as the default "after launch"
option; first month free for launch clients.

## Pricing rules for quotes

- Quote a fixed price from an hour estimate × 55–70 €/h, then round up to the range. Never show the hours.
- Payment: 50 % on order, 50 % at handover (websites); 100 % on order below 500 €.
- 2 revision rounds included; extra rounds at hourly.
- Raise prices per the triggers in [[../business/model|model]].

## AI services — how they are sold (2026-09-16)

Luis's strongest relative advantage is practical AI tooling knowledge; most
creators and small-business owners are at zero. It is real and monetisable, but
it is **not** sold as expertise:

- "KI-Beratung" / "Ich kenne mich mit AI aus" is the most crowded and least
  trusted label in German small business right now — a buyer cannot tell it from
  a LinkedIn prompt-pack seller. Never a headline.
- Sell productised outcomes at a fixed price instead: "Content-Pipeline: aus
  einem Video werden Reels, Captions und Posts — automatisch, in deinem Stil";
  "KI-Assistent auf deine Inhalte trainiert"; "Video-Produktion mit KI-Tools".
  These sit under the existing *Automatisierung* catalog line, priced the same way.
- Keep AI items off /services until one has been delivered and written up as a
  case study. The second prospect (`clients/twitch-friend/`) is the candidate.
- Hourly knowledge transfer is the fallback rate only, never the offer.

The Gewerbe wording covers all of this ([[../business/gruendung|gruendung]] step 3).

## Expansion candidates (not offered yet)

Booking and payment setup for instructors and coaches · Google Business Profile
and basic local SEO · newsletter setup · AI assistant trained on a client's
content · migration off expensive no-code tools · content repurposing pipelines.
