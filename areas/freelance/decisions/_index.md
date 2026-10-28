---
type: index
owner: shared
created: 2026-09-09
updated: 2026-09-21
tags: [index, decision]
---

# Decisions

> [!NOTE] Summary
> Freelance Decision Records. Append-only — to reverse one, write a new record
> and set `superseded-by` on the old. Numbering is global across Mentor and
> Architect; take the next free number here.

| FDR | Status | Impact | Date | Owner | Summary |
|-----|--------|--------|------|-------|---------|
| [[fdr-001-no-hosting-on-personal-infra\|FDR-001]] | accepted | high | 2026-09-08 | mentor | Client sites are never hosted on Luis's netcup-vps or arch-thinkpad. Default: static hosting in the client's own account (Cloudflare Pages). Personal infra costs Luis money and makes him a permanent ops dependency |
| [[fdr-002-client-pays-third-party-costs-directly\|FDR-002]] | accepted | medium | 2026-09-08 | mentor | The client buys domain and running services with their own payment method from day one. Fronting €12 sets a "Luis pays, reimburse later" precedent and makes a gift conditional |
| [[fdr-003-separate-vault-from-infra-docs\|FDR-003]] | accepted | medium | 2026-09-09 | shared | The practice gets its own vault (`freelance-practice`), not a folder in `infra-docs` or `flashcard-tools-vault`. Conventions borrowed from both; client-fact source markers replace infra's provenance markers |
| [[fdr-004-business-identity-on-neurogenesis-dev\|FDR-004]] | accepted | medium | 2026-09-13 | mentor | No second domain or site: business section on neurogenesis.dev; business email `luiswg@neurogenesis.dev` via Cloudflare Email Routing → hub Gmail, sent via Gmail "Send mail as". Personal Gmails forward into one hub; Proton stays manual. Revisit at 3 case studies |
| [[fdr-005-no-public-prices-until-first-paid-quote\|FDR-005]] | accepted | low | 2026-09-14 | mentor | /services shows offers "auf Anfrage" + free 30-min call, no from-prices; page stays unlinked/noindex until employer consent and Gewerbe/Finanzamt registration. Impressum, Datenschutz, /contact publish now |
| [[fdr-006-gewerbe-einzelunternehmen-kleinunternehmer\|FDR-006]] | accepted | high | 2026-09-16 | mentor | Gewerbe as Einzelunternehmen, Kleinunternehmer (§19 UStG), EÜR. IHK Frankfurt confirmed in writing that web development is "rein gewerblich" — Freiberufler route closed. Unblocks the Gewerbeanmeldung and FDR-005 gate (b) |
| [[fdr-007-bilingual-site-with-language-switch\|FDR-007]] | accepted | medium | 2026-09-21 | mentor | neurogenesis.dev business pages (home, about, services, contact, legal) exist in EN and DE with a language switch; articles EN-only. EN at `/`, DE under `/de/`, no auto-redirect; outreach to German prospects links to `/de/services`. Replaces the "DE canonical, no i18n" split in web-presence; Architect to add a TDR addendum |

## When to write one

- A choice a future session could reasonably reverse (hosting, ownership, pricing model, tool choice for a client)
- A deviation from a standard for one client
- Anything whose *cost* will be felt later by someone who did not see the reasoning

Client facts are not decisions — they go in the client note with a source marker.
