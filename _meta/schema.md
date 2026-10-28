---
type: reference
created: 2026-10-21
tags: [index]
---

# Frontmatter schema

> [!NOTE] Summary
> The rules for frontmatter on **new** notes, from master-vault design v4 §9. Nothing is backfilled. Existing headers stay exactly as they are, including their `created`/`updated` values and the fields this file does not mention.

## What exists (measured before the merge)

- 521 valid headers, 76 distinct keys and 476 tags (Codex census; this vault's own count is 479 distinct tags, see [[vocabulary]]).
- `provenance` on 84 notes (70 `measured`, 13 `inferred`); `last_verified` on 75; `created`/`updated` on about 490.
- `source` holds URLs in 2 web clippings. Cases carry `amount` and `direction`.
- `status` means different things for different note types. `published` is a Boolean on posts and a date on clippings.
- Two headers are malformed: `areas/admin/cases/jobcenter-rueckforderung.md` line 6 and `journal/daily/2025/06/2025-06-17.md` line 7. They are fixed in merge step 6, not by any agent.

## Core fields for a new note

| Field | Rule |
|---|---|
| `type` | **Required.** One of the types below. |
| `tags` | Optional. Only tags listed in [[vocabulary]]; register a new tag there first. |
| `created` | Set once by the template. Never changed afterwards. |
| `updated` | Optional. Never bumped automatically; change it only when Luis asks for it. |
| `origin` | Optional. Where the content came from (a person, document, export or session). Does not exist on old notes and is never invented for them. |
| `evidence` | Optional. How well it is supported: `measured`, `inferred` or `inherited-unverified`. |

`origin`/`evidence` replace `provenance` for new notes only. A migration maps an old `provenance: measured|inferred` to `evidence:` only where the meaning is identical. `source:` URLs stay URLs. Nothing adds `origin`, `reviewed_at`, `next_action` or `domain` to an old note.

## Types and their optional fields

The Templater skeleton in `_templates/` sets these fields. `status` is checked per type; there is no vault-wide status list. The values below are the ones each template offers.

| Type | Template | Optional fields | `status` values |
|---|---|---|---|
| `journal` | daily-note, weekly-, monthly-, quarterly-, yearly-note | none; `type: journal` only, tags optional | — |
| `fleeting` | fleeting-note | — | active |
| `permanent` | permanent-note | `aliases`, `up` (a real parent MOC, not a placeholder) | active |
| `literature` | literature-note | `author`, `source` (URL) | unread (template default) |
| `moc` | moc | `up` | active |
| `project` | project-note | `area`, `due` | active (template default) |
| `blog-post` | blog-post | `title`, `date`, `excerpt`, `published` (Boolean) | draft, published |
| `case` | tpl-case | `counterparty`, `aktenzeichen`, `amount`, `direction` (owe, claim, none), `deadline` | open, waiting, blocked, closed |
| `correspondence` | tpl-correspondence | `received`, `doc_date`, `sender`, `channel`, `kind`, `amount`, `deadline`, `aktenzeichen`, `case` | open, answered, filed |
| `decision` | tpl-decisions, tpl-decision | see *Decision records* | proposed, accepted, superseded, deprecated |
| `backlog-item` | tpl-backlog-items | `priority`, `device`, `projects`, `commit` | active, blocked, done, archived |
| `client` | tpl-client | `engagement`, `language`, `target_launch` | lead, discovery, scoping, building, review, launched, closed |
| `lesson` | tpl-lesson | `severity`, `client`, `changed` | — |
| `standard` | tpl-standard | — | active |
| `index` | — | — | active |
| `status` | — | `origin`, `evidence`; a hand-kept current-state page such as `areas/health/_status.md` | — |
| `profile` | — | `origin`, `evidence`; a person's background or learner profile | — |
| `reference` | — | vault reference pages such as this file | — |

Case notes keep `amount` and `direction`; they are official fields now.

## Decision records

- Frontmatter: `id`, `domain`, `status`, `supersedes`, `superseded_by`. `superseded_by` is the one spelling. The 20 existing `superseded-by` fields are normalised on decision records only, with a ledger.
- New records are named `adr-NNNN-slug.md`. Existing dated filenames stay.
- Agents write `id: ADR-TBD`. Luis assigns the number in [[_meta/decisions/_index|the allocator]] when he accepts the record.
- FDR-001 to FDR-007 keep their IDs and files.

## How headers are changed

No bulk YAML load-and-dump: YAML 1.1 loaders reinterpret times, dates and quoting. The procedure is:

1. Fix the two invalid headers.
2. Change the templates.
3. Patch only approved semantic mappings, with a before/after ledger.
4. Byte-compare the bodies and the unmapped scalars.
5. Test property parsing in Obsidian.
