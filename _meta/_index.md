---
type: index
status: active
created: 2026-04-21
updated: '2026-04-21T12:38:15Z'
tags: [index, system]
---

# _meta — index

Vault infrastructure: documentation, schema, tag vocabulary, decision allocator, migration records and the PKM reference. Formerly `system/`; merged here in the vault merge (2026-10-21).

## Files in this directory

| Name | Type | Status | What it is |
|---|---|---|---|
| [[documentation]] | reference | active | how the vault is organised and used |
| [[_meta/schema\|schema]] | reference | active | frontmatter rules for new notes |
| [[_meta/vocabulary\|vocabulary]] | reference | active | controlled and grandfathered tags |
| [[dashboard-lib]] | reference | active | shared code for the [[home]] dashboard (daily note paths) |
| [[link-allowlist]] | schema | active | approved keyword → wikilink mappings |
| [[migration-log]] | reference | archive | the 2026 Notion migration log |
| [[pkm-reference]] | reference | active | PKM methods reference |
| `migration-map.csv` | data | active | the vault-merge file map (session UUIDs masked; exact copy in the merge workspace) |
| `projects.yaml` | data | active | code-project manifest (v4 §6) |

## Subdirectories

- [[_meta/decisions/_index|decisions/]] — vault-wide ADRs and the ID allocator (Luis-maintained)
- `legacy-rules/` — the 9 retired instruction files from the three source vaults (inert; read-only)
- `scratch/` — design documents and reviews from the vault merge (read-only)
