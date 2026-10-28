# freelance-practice — rules

Source of truth for Luis's freelance dev & automation practice. Two Claude
projects write here: **Freelance Business Mentor** (clients, scope, standards,
pricing, marketing, business) and **Freelance Solutions Architect**
(`clients/<name>/tech/`, `runbooks/`, technical decisions). Claude Code
implements from the Architect's briefs.

## Navigation

Read `_index.md`, then the folder `_index.md` you need. Never read the whole
vault. Indexes are static tables, kept by hand — agents cannot see Dataview or
Bases results. Every note opens with a `> [!NOTE] Summary` callout; read that
before deciding to read the rest.

## Read before you write

Check the folder index before creating a note. One subject, one note. Update
in place; do not create `-v2` or `-new` files. Ask before deleting, renaming,
moving, or adding a top-level folder. After writing, update the folder's
`_index.md` row and the note's `updated` date. Drafts and half-processed input
(raw call notes, a client's message to review) go in `_inbox/` and leave within
two sessions.

## State the basis for a client fact

Every fact about a client carries its source, in prose or frontmatter:

- `client` — the client said or sent it (say when)
- `luis` — Luis decided or observed it
- `assumed` — reasoned, not confirmed. **Assumed facts are the ones that break
  projects.** Turn them into discovery questions; never build on them silently.

A scope document contains only `client` and `luis` facts.

## Never write credentials or contact data into tracked notes

No passwords, API keys, tokens, recovery codes. Reference the password-manager
entry by name (`KeePassXC: chantal-cloudflare`). Client email, phone, and
postal address go in `clients/<name>/contact.md`, which is gitignored — the
vault is pushed to a private remote; a third party's contact data should not
be. Backup of the whole vault, ignored files included, is restic → B2 (see
`infra-docs`); git is history and a second copy, not the backup.

## Decisions are append-only

A choice with a rationale that a future session could reverse gets a record in
`decisions/` from `_templates/tpl-decision.md`, named `fdr-NNN-slug.md`
(Freelance Decision Record). To reverse one, write a new record and set
`superseded-by` on the old. Numbering is global across both Claude projects;
take the next free number from `decisions/_index.md`.

## Standards are imperative, lessons are dated

`standards/*.md` say what to do, present tense, one line each. `lessons/` holds
what happened and which standard or template it changed. A lesson that changes
no standard is a diary entry — extract the rule or drop the note.

## Templates come in two kinds

- `_templates/` — Templater skeletons for *vault notes*. Never send these to anyone.
- `templates/` — *deliverable* templates Luis sends to clients or hands to the
  Architect: discovery inventory, scope document, handoff brief, case study.
  Ready-to-copy markdown, no Templater syntax.

## Tags are a controlled vocabulary

Use only tags listed in `_index.md` → Tags. Add a tag there first, then use it.
The flashcard vault reached 236 ad-hoc tags before it was fixed; this one starts
controlled.

## Legal defaults are not optional

Every client website scope includes Impressum, Datenschutzerklärung, cookie
consent if there are embeds or tracking, and an AVV if Luis will maintain
systems holding the client's customer data. A scope note without them is
incomplete, not minimal.

## Boundaries

Luis's own machines (arch-thinkpad, netcup-vps) are documented in `infra-docs`.
His SaaS is documented in `flashcard-tools-vault`. Do not duplicate either
here; link by name.
