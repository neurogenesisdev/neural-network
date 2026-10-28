---
type: decision
id: FDR-003
aliases: [FDR-003]
status: superseded
impact: medium
owner: shared
client:
supersedes:
superseded_by: "[[adr-0040-vault-merge-and-agent-council]]"
created: 2026-09-09
updated: 2026-09-09
tags: [decision, vault, tooling]
---

# Separate vault from infra-docs and flashcard-tools

> [!NOTE] Summary
> The freelance practice lives in its own vault and repo, `freelance-practice`.
> Structure borrows from `infra-docs` and `flashcard-tools-vault`; the
> personal PARA vault (`Neural Network`) was not a fit.

## Context

`infra-docs` documents Luis's machines: collector state, ADRs about disks and
firewalls, a redaction gate for hardware identifiers. `flashcard-tools-vault`
documents the SaaS: codemap, specs, sessions, 236-then-83 tags. The freelance
vault holds client facts, agreements, prices, and a third party's contact data
— a different lifecycle, sensitivity, and set of writers.

## Decision

On 2026-09-09 Luis directed the Mentor to build the vault at
`~/dev/freelance-practice`. Borrowed from infra-docs: static `_index.md`
tables, flat frontmatter, append-only dated decisions, `CLAUDE.md` = `AGENTS.md`,
a root index that names the rule that matters most. Borrowed from
flashcard-tools: `> [!NOTE] Summary` callout as first body element, controlled
tag vocabulary (kept in `_index.md` rather than a separate note while it is
small), `_inbox/` for transient drafts, `profile/` for founder context,
`created`/`updated` fields. Replaced: infra's `measured / inferred /
inherited-unverified` becomes `client / luis / assumed`; `redact.map` becomes a
gitignored `contact.md`.

## Alternatives considered

1. **Folder inside infra-docs** — rejected: mixes client PII with system state under one remote; infra's rules do not apply.
2. **Folder inside flashcard-tools-vault** — rejected: the SaaS vault is product documentation with its own automation (vault-manager plugin, Mem0 ingestion); client agreements do not belong in a vector store scoped to the product.
3. **PARA layout as in Neural Network** — rejected: the setup doc's folders already are the areas; PARA adds a layer without adding navigation, and "archive" hides closed clients that should stay findable.

## Consequences

- Positive: short, specific `CLAUDE.md`; own remote and access rules; both Claude projects share one source of truth.
- Negative / cost: a third vault to mount in Claude Desktop; cross-vault references are by name only.
