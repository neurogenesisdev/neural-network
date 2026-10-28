---
type: index
owner: shared
created: 2026-09-09
updated: 2026-09-27
tags: [index]
---

# Freelance — area index

> [!NOTE] Summary
> Index of Luis's freelance dev & automation practice (formerly the freelance-practice vault, merged into neural-network 2026-10-21). Start here, then
> drill into the folder index. The rule that matters: every client fact states
> its source — `client`, `luis`, or `assumed`. Assumed facts become discovery
> questions, never scope.

## Folders

| Folder | Owner | Description |
|--------|-------|-------------|
| [[profile/_index\|profile]] | Mentor | Founder context — who Luis is, constraints, goals. Read at session start |
| [[clients/_index\|clients]] | Mentor (+ Architect in `tech/`) | One folder per client: brief, scope, status, tech notes |
| [[standards/_index\|standards]] | Mentor | How Luis works — communication, delivery, pricing, scope. Imperative |
| [[services/_index\|services]] | Mentor | Service catalog and price ranges |
| [[areas/freelance/deliverables/_index\|templates]] | Mentor | Deliverable templates sent to clients or handed to the Architect |
| [[areas/freelance/decisions/_index\|decisions]] | shared | FDR records — what was chosen and why. Append-only |
| [[lessons/_index\|lessons]] | Mentor | Retros per project or incident, each tied to a standard it changed |
| [[business/_index\|business]] | Mentor | Gründung, tax, insurance, employment constraints |
| [[marketing/_index\|marketing]] | Mentor | Positioning, outreach, case studies, lead tracking |
| [[areas/freelance/runbooks/_index\|runbooks]] | Architect | Repeatable technical procedures for client work |
| [[areas/freelance/backlog/_index\|backlog]] | shared | **The one home for every deferred or wanted item.** An item lives there or it doesn't exist; other notes keep pointers only (since 2026-09-27) |
| [[_templates/_index\|_templates]] | shared | Templater skeletons for vault notes — not for clients |
| `_inbox/` | shared | Transient drafts and raw input. Not indexed; cleared within two sessions |
| `_attachments/` | shared | Exported PDFs, reference images. No client raw uploads |

## Active

| Item | Status | Where |
|------|--------|-------|
| Chantal Fuelber — portfolio website (pro bono) | discovery forms sent; waiting on her answers (her work starts after end of Sept 2026) | [[clients/chantal-fuelber/_index\|chantal-fuelber]] |
| Business registration | employer consent 2026-09-16; **Gewerbeanmeldung filed 2026-09-17**, Gewerbeschein pending; next: Fragebogen zur steuerlichen Erfassung by 2026-10-17 | [[business/gruendung\|gruendung]] |
| neurogenesis.dev | round one shipped; round two in build (/about live 2026-09-27, /services public — gate opened 2026-09-27) | [[business/web-presence\|web-presence]], [[business/site-redesign-round-two\|round two]] |

## Open questions

| Question | Where |
|----------|-------|
| Gewerbe vs. Freiberufler — answered 2026-09-09 (Gewerbe); confirm at IHK, then FDR | [[business/gruendung\|gruendung]] |
| Price ranges — filled 2026-09-09; validate against first 3 quotes | [[services/_index\|services]] |
| Payslip gross vs. 1,977.50 € (Krankenkasse presumption) | [[business/model\|model]] |

## Naming conventions

- Client folders: `clients/<firstname-lastname>/`, `_index.md` is the client note
- Decisions: `decisions/fdr-NNN-slug.md`
- Lessons: `lessons/YYYY-MM-DD-slug.md`
- Templates: `_templates/tpl-<type>.md` (vault), `templates/<deliverable>.md` (client-facing)
- All filenames kebab-case, no spaces, no umlauts

## Note structure

YAML frontmatter, flat, no nesting. Every note: `type`, `status`, `created`,
`updated` (YYYY-MM-DD), `tags`. First body element after the H1 is a
`> [!NOTE] Summary` callout. Client notes mark each fact `(client)`, `(luis)`,
or `(assumed)`. Decisions add `id`, `impact`, `owner`, `supersedes`, `superseded-by`.

## Tags

Controlled vocabulary. Add here before using.

- **Structural:** `index`, `client`, `standard`, `decision`, `lesson`, `template`, `runbook`, `profile`
- **Engagement:** `pro-bono`, `fixed-price`, `retainer`, `portfolio`
- **Domain:** `website`, `automation`, `booking`, `newsletter`, `shop`, `hosting`, `ownership`, `legal`
- **Business:** `gruendung`, `tax`, `insurance`, `pricing`, `marketing`, `communication`, `delivery`, `scope`
- **Client niche:** `yoga`, `creator`, `food`, `travel`, `art`
- **Vault:** `vault`, `tooling`, `backlog`

## When to update

- **A client said something that changes scope** — update the client note, mark `(client)`, date it
- **A decision was made** — write an FDR; link it from the client note or standard it affects
- **A project or incident ended** — write a lesson, then change the standard it points at
- **A price, template, or standard changed** — edit in place, bump `updated`

## Anti-patterns

- Don't build on an `assumed` fact — ask the client
- Don't write credentials or the client's contact data into a tracked note
- Don't put implementation detail in Mentor notes — the Architect owns `tech/` and `runbooks/`
- Don't rely on Dataview or Bases for agent navigation — keep the static tables current
- Don't nest deeper than `clients/<name>/tech/`
- Don't invent tags
