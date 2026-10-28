# neural-network — rules

Luis's personal vault: PARA + MOC hybrid (see `system/documentation.md`).
Claude projects that write here: **Finance & Legal Case Manager**
(`areas/admin/`, `areas/finance/`, `projects/altlasten-2026.md`). Other
projects read; they write only to `_inbox/`.

## Navigation

Read `maps/home.md`, then the folder `_index.md` you need. Never read the
whole vault. Indexes are static tables kept by hand — agents cannot see
Dataview or Datacore results. Every note in `areas/admin/` and
`areas/finance/` opens with a `> [!NOTE] Summary` callout; read that before
deciding to read the rest.

## Read before you write

Check the folder index before creating a note. One subject, one note. Update
in place; never create `-v2` or `-new` files. Ask before deleting, renaming,
moving, or adding a top-level folder. After writing, update the folder's
`_index.md` row and the note's `updated` date. Drafts and unprocessed input
go to `_inbox/` and leave within two sessions.

## State the basis for every fact

Every fact in `areas/admin/` and `areas/finance/` carries its source, in prose
or frontmatter:

- `letter` — a document says so (name the file in `correspondence/`)
- `luis` — Luis stated or decided it
- `assumed` — reasoned, not confirmed. **Assumed facts become questions or
  lookups; never write a letter or pay money on one.**

Amounts, dates, file numbers and deadlines are `letter` or `luis` only. If
Luis recalls a figure ("about 900 €"), write it as `luis, approx.` and add a
task to verify against a statement.

## Cases are the unit

Anything with a counterparty (Behörde, Gericht, Gläubiger, Vermieter,
Insolvenzverwalter) is a case in `areas/admin/cases/<slug>.md` from
`_templates/tpl-case.md`. A case has: counterparty, Aktenzeichen, status,
next action, deadline, log. Money owed or claimed appears once more in
`areas/finance/debts.md` as a row linking the case. Never duplicate the log.

## Correspondence is one folder

Every letter, Bescheid, email or form — any topic — gets one note in
`areas/admin/correspondence/YYYY-MM-DD-<sender>-<slug>.md` from
`_templates/tpl-correspondence.md`, with the scan/photo in
`_attachments/letters/`. The `case:` field routes it. A letter without a case
creates the case.

## Sensitive data

Aktenzeichen, IBAN, Kundennummer, birthdate belong in the case or
correspondence note only — never in an `_index.md`, `maps/`, or a summary
callout. No passwords or tokens anywhere; reference the KeePassXC entry by
name. This vault is synced to Luis's own devices only. Before adding it to
any remote or the netcup MCP, exclude `areas/admin/correspondence/` and
`_attachments/letters/` or confirm the host is encrypted at rest.

## Language

Notes are in English with German terms as the documents use them (Bescheid,
Mahnung, Insolvenzgeld, Pfändungsfreigrenze). Letters and emails drafted for
Luis to send are in German, in his voice: short, direct, factual, no
"erfahrungsgemäß", no phrasing that reads as generated. Never claim facts or
experience he does not have.

## Advice boundary

The Case Manager explains process and drafts documents. It is not a lawyer,
Steuerberater, or therapist and says so when a question needs one; it names
the free service that fits (Beratungshilfe, Schuldnerberatung, Rechtsantragstelle)
and turns that into a case.

## Tags

Controlled vocabulary for `areas/admin/` and `areas/finance/`. Add here first.

- **Structural:** `index`, `case`, `correspondence`, `template`
- **Case type:** `debt`, `claim`, `application`, `insolvency`, `housing`,
  `vehicle`, `jobcenter`, `insurance`, `tax`, `accounts`
- **Status:** `open`, `waiting`, `blocked`, `closed`
- **Finance:** `income`, `budget`, `recurring`, `transactions`

## Boundaries

Freelance business lives in `freelance-practice`; machines and server in
`infra-docs`. Link by name, do not duplicate. Personal Gewerbe *obligations*
(Finanzamt, IHK, Berufsgenossenschaft letters) are cases here; the business
model and clients are not.
