---
type: decision
id: FDR-007
aliases: [FDR-007]
status: accepted
impact: medium
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-21
updated: 2026-09-21
tags: [decision]
---

# FDR-007 — neurogenesis.dev is bilingual (EN/DE) with a language switch

> [!NOTE] Summary
> All business pages exist in English and German with a visible switch;
> articles stay English-only. Replaces the "DE canonical, no i18n" language
> split in [[../business/web-presence]].

## Context

[[../business/web-presence]] (2026-09-14) set German as canonical for
/services, /contact and the legal pages, English for the blog, and "no
i18n/hreflang at this size". During the redesign discovery
([[../business/site-design-brief]]) Luis rejected a German-first site: English
is his stronger written voice, the language of his field, and he does not want
to limit the client base to Germany. A non-German-speaking prospect could not
read the offer, the contact page or the privacy policy. The mentor's
counter-point stands too: German small-business clients need the offer in
German at the moment they decide whether to trust him.

## Decision

On 2026-09-21 Luis decided that every page not meant for one language only is
bilingual, with a language switch: home, /about, /services, /contact,
Impressum, Datenschutz/Privacy. Articles are English-only.

Mentor's shape, for the Architect to confirm technically: English at `/`,
German under `/de/`; no browser-language auto-redirect (the URL decides);
outreach to German prospects links to `/de/services`; German legal texts are
binding, the English versions carry a "convenience translation" line;
hreflang tags now make sense. Wording rules from web-presence apply in both
languages (no claimed experience, no plural "clients"/"Kunden", no
"agency"/"Agentur"); German keeps Sie. The FDR-005 gate covers /services in
both languages.

## Alternatives considered

1. **DE business pages, EN everything else, no switch** (previous state) — shuts out non-German clients; rejected by Luis.
2. **English only** — adds friction for the most likely first paying clients (Frankfurt small businesses, instructors); rejected by mentor.
3. **Add German later** — routing and components are expensive to retrofit; copy volume is small (5–6 short pages).

## Consequences

- Positive: one site serves local and international prospects; each visitor type can be sent to a page in their language.
- Negative / cost: every copy change is made twice; estimated +3–4 h in redesign round one; Architect needs a Tech Decision Record addendum for Astro i18n routing; legal generator output exists only in German, English legal pages are translations to keep in sync.

## Revisit when

Maintaining two languages visibly slows copy changes, or after 6 months without a single non-German enquiry.
