---
type: index
status: active
owner: shared
created: 2026-09-27
updated: 2026-09-27
tags: [index, backlog]
---

# Backlog

> [!NOTE] Summary
> The one home for everything deferred, wanted, or not-yet-scheduled across the
> practice. Created 2026-09-27 because backlog items were scattered across
> notes and could not be found without reading everything. Rule: **an item
> lives here or it doesn't exist.** Other notes may keep a one-line pointer to
> this table; they no longer keep their own backlog tables. Items are grouped
> by area, ordered by priority inside the group. `Trigger` says what has to be
> true before the item is worked on. Done items move to the bottom with a date.

## How to use

- Add a row when a want or a deferral comes up in any session; link the note that has the context.
- Start an item: set status `in progress`, name the owner (Luis / Mentor / Architect).
- Finish an item: move the row to **Done** with the date. Don't delete.
- Weekly triage (Architect's report) reads this file, not the other notes.

## Vault & process

| # | Item | Trigger / when | Owner | Context | Status |
|---|------|----------------|-------|---------|--------|
| V1 | **Consistency and recency pass through the whole vault**: stale `updated` fields, `Active` tables, status lines that contradict later notes (e.g. root `_index.md` still says "register before first paid quote" though the Gewerbe was filed 2026-09-17; web-presence gate closed only 2026-09-27) | next quiet session, before round three of the site | Mentor + Architect | [[../_index]], [[../business/web-presence]] | open |
| V2 | **Update `CLAUDE.md` / `AGENTS.md` with read-and-write consistency rules**: when a fact changes, update every note that states it (root index Active table, folder index, the subject note) in the same turn; bump `updated`; on reading, prefer the newest dated line and flag contradictions instead of repeating the older one | with V1 | Mentor | [[areas/freelance/CLAUDE]] | open |
| V3 | Migrate the scattered backlog tables into this file and replace them with pointers: [[../business/site-design-brief]] "Deferred to later rounds", [[../business/web-presence]] "Round two backlog", [[../business/gruendung]] checklist open steps, [[../services/_index]] expansion candidates | with V1 | Mentor | — | open |
| V4 | Templates still "to write": client-brief, scope-document, technical-handoff-brief, handover-note, testimonial-request, case-study | each before its first use on Chantal's project | Mentor | [[areas/freelance/deliverables/_index]] | open |

## Gründung & business

| # | Item | Trigger / when | Owner | Context | Status |
|---|------|----------------|-------|---------|--------|
| B1 | Fragebogen zur steuerlichen Erfassung via ELSTER (Kleinunternehmer, EÜR) | **by 2026-10-17** | Luis | [[../business/gruendung]] step 4 | open |
| B2 | Chase the Gewerbeschein if not received | ~3 weeks after 2026-09-17 → from 2026-10-08 | Luis | [[../business/gruendung]] | open |
| B3 | Berufsgenossenschaft (VBG) registration | letter follows the Gewerbeanmeldung | Luis | [[../business/gruendung]] step 7b | open |
| B4 | Krankenkasse: written notice "nebenberuflich selbständig", ask for confirmation | after B1 | Luis | [[../business/gruendung]] step 7, [[../business/model]] | open |
| B5 | Berufshaftpflicht / IT-Haftpflicht (~€10–30/month, verify) — also the one "legit" signal clients read | before first paid client with customer data | Luis | [[../business/gruendung]] step 8 | open |
| B6 | Business bank account / sub-account | with B1 | Luis | [[../business/gruendung]] step 5 | open |
| B7 | Invoice template with Pflichtangaben, §19 sentence | before first paid invoice | Mentor | [[../business/gruendung]] step 6 | open |
| B8 | Contract / terms template for the first paid project | before first paid quote | Mentor | [[../business/site-design-brief]] | open |
| B9 | Steuerberater 1 h consult | Q1 2027 or revenue > ~€10k | Luis | [[../business/gruendung]] step 9 | open |
| B10 | Certificates: none for client trust; free badges (freeCodeCamp RWD, n8n Level 1/2, Anthropic Academy) only in a slow week | never before Chantal's site ships | Luis | [[../business/web-presence]] /about decisions | parked |

## neurogenesis.dev — round two (in build) and later

| # | Item | Trigger / when | Owner | Context | Status |
|---|------|----------------|-------|---------|--------|
| S1 | /about copy EN+DE, /now /uses /changelog → /about, articles to drafts + "coming soon" | — | Architect | [[../business/web-presence]], [[../business/site-redesign-round-two]] | **done 2026-09-27** (R1) |
| S2 | /services public + header link, Newsletter in header, Huberman-style footer, centred headings | — | Architect | [[../business/site-redesign-round-two]] | in progress (R2) |
| S3 | Site-wide design rules + copy (bands, accent, page anatomy, footer, intro lines, newsletter, /about sixth interest, DE home strings, FAQ page, contact form) | Architect handoff 2026-09-27 15:40 | Mentor + Luis | [[../business/web-presence]] "Design rules + copy for the site-wide sweep" | **done 2026-09-27** → Architect: design system + one-pass sweep |
| S15 | Newsletter philosophy: what it is for once it does more than article/business heads-ups; bilingual notification mails | when ≥ 3 articles exist | Mentor + Luis | [[../business/web-presence]] | parked |
| S16 | FAQ: replace/extend the six starter questions with real ones | after 3 real enquiries | Mentor | [[../business/web-presence]] | parked |
| S4 | Profile audit: LinkedIn, Instagram, Threads, X, Facebook — bios, photo, links, settings | — | Luis + Mentor | [[../marketing/profiles]] | **done 2026-09-28** (open: LinkedIn About + DE profile, GitHub after S5, banners) |
| S17 | Banners for LinkedIn / X / Facebook: wordmark on dark gradient, one export per platform size | any time | Architect | [[../marketing/profiles]] | open |
| S5 | GitHub cleanup: 3–5 repos public (site, sanitised n8n exports, dotfiles; Memogenesis only if defensible), names/descriptions/READMEs, **secret-scan full history first** | after S2 | Architect | [[../business/web-presence]] | open |
| S6 | Writing pipeline + authoring guide (frontmatter, callouts, Mermaid, series, auto reading time/ToC); fix vs rebuild depends on what failed | after Luis writes one article by hand | Architect | [[../business/web-presence]] | open |
| S7 | Hero gradient and hero→white transition (dark media, halo, `surface-1` band) — parked from T4/T8 | round two/three | Architect | [[../business/site-redesign-round-one]] | open |
| S8 | Demo loop recording on the fabricated-inbox Gmail account + n8n canvas screenshot | Luis's own recording work | Luis | [[../business/site-design-brief]] | open |
| S9 | Logo (N in half circle, vesicle/D) — text wordmark until then | after round two | Luis + Architect | [[../business/site-design-brief]] | open |
| S10 | Professional headshot (Snellenberg pose brief); interim photo in use since 2026-09-27 | before first paid outreach wave | Luis | [[../business/site-design-brief]], [[_attachments/Headshot]] | open |
| S11 | Full-screen portrait hero, multilingual hello intro, interactive hero piece, colour-coded bands, personal tagline, testimonial strip, client work as dark media tiles, Telegram assistant demo, writing strip on home, articles index by category, Memogenesis on /work, contact form | see triggers in the brief's deferred table (to migrate here in V3) | — | [[../business/site-design-brief]] | open |
| S12 | Music recording for /about (sing + self-accompaniment) | once Luis has his own space | Luis | [[../business/web-presence]] /about decisions | parked |
| S13 | YouTube / TikTok links on /about | when content exists | Luis | [[../business/web-presence]] | parked |
| S14 | Google OAuth consent-screen privacy URL → `/de/datenschutz` (301 covers it meanwhile) | any time | Architect | [[../business/site-redesign-round-one]] | open |

## Services & offer

| # | Item | Trigger / when | Owner | Context | Status |
|---|------|----------------|-------|---------|--------|
| O1 | Full service catalogue for use in calls (cards on /services are summaries) | before first paid call | Mentor | [[../services/_index]] | open |
| O2 | Price ranges validated against first 3 quotes; public prices revisited | after first paid quote (FDR-005) | Mentor | [[../services/_index]] | open |
| O3 | Not offered yet: shop, redesign/migration; invoices/documents automation on request only | more experience | — | [[../business/web-presence]] | parked |
| O4 | Expansion candidates: booking/payment setup, Google Business Profile + local SEO, AI assistant on client content, migration off no-code tools | after first paid projects | Mentor | [[../services/_index]] | parked |

## Clients

| # | Item | Trigger / when | Owner | Context | Status |
|---|------|----------------|-------|---------|--------|
| C1 | Chantal: discovery forms answered → TDR → scope → build | her work start (after end of Sept 2026) | Luis / Architect | [[../clients/chantal-fuelber/_index]] | waiting on client |
| C2 | Chantal: case-study + testimonial consent in writing; also okay for naming her field on the site | at launch | Luis | [[../business/web-presence]] Case studies | open |
| C3 | First automation client: discounted or pro bono for proof (Chantal model) | when one appears | Mentor | [[../business/web-presence]] | parked |

## Done

| # | Item | Done |
|---|------|------|
| — | (moved here from the tables above with a date) | |
