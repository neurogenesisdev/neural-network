---
type: index
device: shared
updated: 2026-10-06
tags: [index, shared]
---

# shared — cross-device records

Things that belong to no single device: domains, DNS, email, cross-device
decisions once settled (ADR-0027 is the candidate).

## Runbooks

| Runbook | Covers | Status |
|---|---|---|
| [[runbooks/neurogenesis-dev-dns-and-email\|neurogenesis.dev DNS + business email]] | zone state, Cloudflare Email Routing → hub Gmail, Gmail Send-as, DMARC weakness + relay backlog | measured 2026-09-14 |
| [[runbooks/neurogenesis-dev-site\|neurogenesis.dev site operations]] | repo `neurogenesis-dev`, build gate, GitHub Actions deploy to Cloudflare Pages, legal/business pages, /services gate, site backlog | measured 2026-09-15 |
| [[runbooks/keepassxc-database\|KeePassXC database]] | title and secret conventions, folder system, rules for scripted changes, open cleanup items | measured 2026-10-06 |

## Decisions

| ADR | Status | Summary |
|---|---|---|
| [[decisions/20260921-neurogenesis-dev-i18n-routing\|ADR-0038]] | proposed 2026-09-21 | neurogenesis.dev bilingual routing (TDR addendum to freelance-practice FDR-007): Astro built-in i18n, EN at `/`, DE at `/de/`, hreflang + x-default, rule-based switch, post URLs unchanged, EN legal = convenience translation, Brevo in privacy pages, Inter + JetBrains Mono under OFL. Four flags for Luis |
