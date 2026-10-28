---
type: index
status: active
created: 2026-04-21
updated: '2026-04-21T12:38:15Z'
tags: [index, _templates]
---

# _templates — index

Templater skeletons for new vault notes. They require the Templater plugin. These are internal only: client-facing deliverable templates live in [[areas/freelance/deliverables/_index|deliverables]].

This index merges the three pre-merge indexes (neural-network, infra-docs and freelance-practice) on 2026-10-21. It lists all 20 templates, each once (quarterly-note added in step 6).

## Notes and journal

| Template | Note type | Goes in |
|---|---|---|
| [[blog-post]] | blog-post | `areas/writing/drafts/` |
| [[daily-note]] | journal | `journal/daily/YYYY/MM/YYYY-MM-DD.md` |
| [[weekly-note]] | journal | `journal/weekly/` |
| [[monthly-note]] | journal | `journal/monthly/` |
| [[quarterly-note]] | journal | `journal/quarterly/` |
| [[yearly-note]] | journal | `journal/yearly/` |
| [[fleeting-note]] | fleeting | `_inbox/` |
| [[literature-note]] | literature | `resources/<topic>/` |
| [[permanent-note]] | permanent | `notes/` |
| [[moc]] | moc | `maps/` |
| [[project-note]] | project | `projects/` |
| [[startup-open-home]] | — | startup template (opens `maps/home.md`) |

## Admin

| Template | Note type | Goes in |
|---|---|---|
| [[tpl-case]] | case | `areas/admin/cases/<slug>.md` |
| [[tpl-correspondence]] | correspondence | `areas/admin/correspondence/YYYY-MM-DD-sender-slug.md` |

## Infrastructure

| Template | Note type | Goes in |
|---|---|---|
| [[tpl-decisions]] | decision | `areas/infrastructure/<device>/decisions/` |
| [[tpl-backlog-items]] | backlog-item | `areas/infrastructure/<device>/backlog/` |

## Freelance

| Template | Note type | Goes in |
|---|---|---|
| [[tpl-client]] | client | `areas/freelance/clients/<name>/_index.md` |
| [[tpl-decision]] | decision | `areas/freelance/decisions/` (new records: `adr-NNNN-slug.md`, `id: ADR-TBD`) |
| [[tpl-lesson]] | lesson | `areas/freelance/lessons/YYYY-MM-DD-slug.md` |
| [[tpl-standard]] | standard | `areas/freelance/standards/<topic>.md` |
