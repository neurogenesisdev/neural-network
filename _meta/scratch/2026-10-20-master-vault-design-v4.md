---
title: "Master vault design v4 — final consolidation target for neural-network"
type: design
status: final        # supersedes v3 (2026-10-19). Codex on-disk review applied (§12); Luis's six decisions of 2026-10-19 23:44 applied
provenance: inferred  # design; measured figures cite the Codex review of the real mirrors
created: 2026-10-20
updated: 2026-10-20
supersedes: "[[2026-10-19-master-vault-design-v2]]"
reviewers: [research report, Luis review, ChatGPT review, Codex gpt-6.1-astra on-disk review 2026-10-19 23:25, Luis decisions 2026-10-19 23:44]
tags: [vault, pkm, design, merge, claude-code]
---

# Master vault design v4 — final

> [!NOTE] Summary
> One vault, `neural-network`, absorbs `infra-docs` and `freelance-practice` as areas. PARA + `journal/` + `notes/` + `maps/` + `_meta/` at the top. The server enforces a three-tier folder model (skeleton / leaf / protected) because file modes alone protect nothing; machine output lives in one protected `_generated/` root. Synced `CLAUDE.md` files are editable content with an accepted, stated residual risk. ADR-0038 is a duplicate and gets an erratum; the merge is ADR-0040. Migration runs in a disconnected staging vault built from laptop originals, gated by a total file mapping and resolved-edge comparison — nothing uploads until it verifies. This is the last restructure for 12 months (no folder changes before 2027-10 without an ADR).

## 0. What changed from v3 — the decisions

| # | v3 said | v4 (decided 2026-10-19 23:44) | Why |
|---|---|---|---|
| 1 | `journal/YYYY/MM/` for all dailies; `reviews/` flat with interval in the name | **`journal/daily/` stays flat.** Reviews in `journal/weekly/ monthly/ quarterly/ yearly/`, folders created on first use | 196 files is no measured navigation problem; the month split costs a date-path helper in `maps/home.md` and 5 cross-month edges. Flat reviews sort wrongly (`2026-10` < `2026-Q4` < `2026-W42`) |
| 2 | "Kernel-enforced" 0640 files + nightly chmod | **Skeleton / leaf / protected model** (§3b), `_generated/` root, `setfacl -b/-k` before setting ACLs | A 0640 file in a group-writable folder can be deleted or renamed-over; protection must cover every ancestor |
| 3 | Synced `CLAUDE.md`, nightly chmod as the guard | **Synced rules are editable content with accepted residual risk**, stated in the ADR. Server copies 0640 under protected ancestors; laptop guard hook refuses `CLAUDE.md` writes | Server ownership cannot authenticate an edit that arrives through Sync as luis; say so instead of claiming enforcement |
| 4 | Merge = ADR-0039; "find the 43-vs-0038 discrepancy" | **ADR-0038 is duplicated** (website i18n routing 2026-09-21 and the rebuild). Rebuild → **ADR-0039** by dated erratum, legacy id kept for search; merge = **ADR-0040**. **FDR-001…007 keep their IDs** (seven, not eight) | Measured |
| 5 | `_inbox/` = all capture; service `inbox/` undiscussed | **One inbox.** n8n's landing zone becomes `_inbox/n8n/`, bind-mounted into the container in place of `/srv/vault/inbox` | Two top-level inboxes is a filing error waiting to happen; a subfolder keeps n8n's reach identical |
| 6 | `notes/` added, existing permanent notes unmentioned | **`notes/` is the only permanent-note home; the five notes in `resources/concepts/` move there** with their MOC links; the habit action's destination changes | Two homes for one note type |
| — | `_inbox/whisper/` deleted | Deleted **after** confirming the 7 transcripts exist elsewhere (check in §10 step 5); the `.webm` audio is already absent | Transcripts are text, not trash, until proven duplicated |
| — | Weekly/monthly/yearly skeletons by dev timer + ntfy | **No review timers at launch.** Weekly review is an opt-in pilot; reminders may open the note, nothing generates content | Zero reviews exist today; 15 unread digests show what unattended output becomes. Templater templates cannot be rendered by a timer anyway |
| — | `xps-windows/` → archive | **Dropped** — the subtree does not exist in infra-docs | Measured |
| — | Project cards for paid engagements | Cards for **any bounded delivery commitment, pro-bono included** | The two real clients are pro-bono and prospect |
| — | `projects/_status.md`, `maps/decisions.md` generated in place | Both move to **`_generated/`** | Their parents are writable |

## 1. Top level

```
neural-network/
├── CLAUDE.md          vault contract ≤ 60 lines, SYNCED, 0640 on the server
├── AGENTS.md          3 lines: "read CLAUDE.md" (Codex and other harnesses), 0640
├── .claude/           server-only: agents/ hooks/ skills/ rules/ settings.json — PROTECTED
├── _inbox/            all capture (phone, dev sessions, fleeting notes) — LEAF
│   └── n8n/           service ingress, mounted into the n8n container — LEAF, u:n8n:rwX
├── _templates/        Templater skeletons; one merged _index.md — LEAF
├── _attachments/      letters/ scans images PDFs — LEAF
├── _generated/        machine output only: projects-status.md, decisions.md, inbox-age.md — PROTECTED
├── _meta/             SKELETON: schema.md vocabulary.md documentation.md migration-log.md (0640)
│   ├── decisions/     vault-wide ADRs — LEAF
│   ├── legacy-rules/  the 8 retired instruction files, renamed inert — PROTECTED
│   └── scratch/       design docs and reviews from this merge, read-only after ADR-0040 — PROTECTED
├── journal/           SKELETON
│   ├── daily/         YYYY-MM-DD.md, flat — LEAF
│   └── weekly/ monthly/ quarterly/ yearly/   created on first use — LEAF
├── notes/             Zettelkasten permanent notes, flat — LEAF
├── projects/          one card per project — LEAF
├── areas/             SKELETON (§2)
├── resources/         reference by topic — LEAF
├── archive/           — LEAF
└── maps/              home.md, MOCs — LEAF
```

Skeleton / leaf / protected are server ACL tiers (§3b). On the laptop and phone every folder is an ordinary folder.

Every file in the three source vaults gets a destination in `_meta/migration-map.csv` (source vault, source path, destination path, action: move | rename | archive | delete | merge-into). Measured roots that v3 forgot and are now mapped: 7 infra `_scratch/` notes → `_meta/scratch/`; 4 JPEGs at neural-network's root → `_attachments/`; neural-network's root copy of the design doc → delete (older, also titled v3); `freelance-practice/_inbox/digest/` (15 notes, 3–5 weeks unread) → `archive/freelance/digest/`, and the n8n digest workflow either gets a named reader or is switched off (backlog item, not a merge step).

The alternatives (ACE, Johnny Decimal, type-first root) and why they lost are unchanged from v3 §1 and are not repeated.

## 2. Areas

```
areas/
├── admin/            cases/ correspondence/ — the case is the unit (unchanged)
├── finance/          notes + transactions/ (bank CSVs stay on the server)
├── health/
├── career/
├── self/             ← areas/personal/ (actual source path): typology, goals, philosophy, habits-list.md, learning/ folded in
│   └── experiments/  ← areas/self-experiments/; tagged by domain, linked from health etc.
├── writing/          ← publish/: drafts/ and published/, the two existing posts classified by their real state
├── infrastructure/   ← infra-docs (§3)
└── freelance/        ← freelance-practice (§4)
```

Consumers that hard-code the old paths and must change in the same step (Codex §3, all in `maps/home.md`, `_templates/daily-note.md`, `areas/personal/habits-list.md`): the habit registry path (3 places), the `areas/self-experiments` query, the areas label map, `publish/_index` and `system/_index` navigation, the blog habit action (`publish/blog` → `areas/writing/drafts`), the permanent-note habit action (`resources/concepts` → `notes`). `music/` stays un-created; the threshold is "an ongoing responsibility to manage", not "a file mentions music".

## 3. `areas/infrastructure/` — by device

```
areas/infrastructure/
├── _index.md                         (0640, admin-maintained)
├── CLAUDE.md                         scoped contract (0640)
├── arch-thinkpad/   decisions/ runbooks/ sessions/ backlog/ spec/ devenv/ _source/   — LEAVES
│                    state/            collector output — PROTECTED
│                    state-notes.md    ← state/notes.md, the one curated commentary file, moved out
├── netcup-vps/      decisions/ runbooks/ sessions/ backlog/ — LEAVES (build/ → repo)
├── pixel-graphene/
└── shared/          decisions/ runbooks/
```

- Depth 5 accepted (device → type is how Luis thinks about machines; indexes supply the type view). Device names stay — they are Tailscale hostnames and ssh aliases.
- `spec/boot.md` and `state/boot.md` are different artefacts (specification vs measured) and stay as they are; links to `boot` get qualified.
- **Executables leave the vault** → repo `~/code/infra-tools` (laptop + server clone), moved from laptop originals — the mirrors hold Markdown descriptions, not the scripts. The rescue-overlay check, `CONFIRM=WIPE`, the stage-3 accept-policy gate and redaction parity checks travel with them into that repo's own `CLAUDE.md`.
- `state/_index.md` becomes collector output (generated), so `state/` can be protected whole.
- Templates `tpl-decisions.md` / `tpl-backlog-items.md`: device chooser lists `hive` and omits `netcup-vps` — fixed at merge.
- The four nested infra `CLAUDE.md` files (`arch-thinkpad/state/`, `arch-thinkpad/decisions/`, `netcup-vps/build/`, `netcup-vps/decisions/`) are **retired** with the roots (§8); their live rules fold into `areas/infrastructure/CLAUDE.md` (Codex draft, 40 lines, is the starting text).

### 3b. Server permission model — skeleton / leaf / protected

Why: Linux grants delete and rename-over to anyone with write on the *directory*, regardless of the file's mode. Protection is therefore a property of the whole path from the vault root down, not of a file.

| Tier | Mode / ACL | dev can | Applies to |
|---|---|---|---|
| **Skeleton** | `2750 luis:vault`, `setfacl -b -k`, default `u::rwx,g::r-x,o::---` | read, traverse; **not** add/remove/rename children | vault root, `areas/`, `areas/infrastructure/` and each device folder, `areas/freelance/`, `journal/`, `_meta/` |
| **Leaf** | `2770 luis:vault`, default `g::rwx` | read, write, create, delete inside | every content folder: `journal/daily/`, `areas/admin/cases/`, `notes/`, `_inbox/`, device `runbooks/` … |
| **Protected** | tree-wide `2750`/`0640`, no group write | read only | `_generated/`, `arch-thinkpad/state/`, `.claude/`, `.obsidian/`, `_meta/legacy-rules/`, `_meta/scratch/` |
| **Contract files** | `0640` under a skeleton parent | read only | root `CLAUDE.md`, `AGENTS.md`, `areas/infrastructure/CLAUDE.md`, `areas/freelance/CLAUDE.md`, skeleton `_index.md` files, `_meta/*.md` |

n8n keeps traverse-only (`u:n8n:--x`) on the root and `_inbox/`, `u:n8n:rwX` on `_inbox/n8n/` only. Nothing else changes for the service.

Cost, accepted: a new top-level folder, area or device folder needs the luis lane (which the 12-month freeze wants anyway); skeleton `_index.md` files are admin-maintained — dev proposes edits as a diff in the session, Luis applies. Ordinary notes everywhere else stay atomically writable by dev.

What this still does not stop, stated in ADR-0040 as accepted residual risk: dev creating a *new* `CLAUDE.md` inside a writable leaf (mitigated by the guard hook + nightly drift report, not prevented); a laptop-side edit of a synced contract arriving as luis. The nightly job compares against a manifest and **reports** drift to ntfy; it never deletes.

Real-host acceptance tests, run as `dev` and as `n8n` (not inferred from `ls`), repeated immediately after a phone-created file syncs in: direct write, temp+rename-over, unlink, parent rename, and creating a `CLAUDE.md` in a protected root all **denied**; atomic replacement of a phone-owned note in a leaf **allowed**; n8n can write `_inbox/n8n/` and read nothing else; luis can still Sync a dev-created note; protections survive generator replacement, service restart and reboot. The ACL script is Codex §6 (syntax-checked, never run), parameterised by the final layout.

## 4. `areas/freelance/`

Kept as is: `profile/ clients/<name>/ standards/ services/ deliverables/ (← templates/) decisions/ lessons/ business/ marketing/ runbooks/ backlog/` (the existing folder, not a `backlog.md`). Its `_inbox/`, `_attachments/`, `_templates/` merge into the root ones; the four Templater skeletons get a `freelance-` prefix only if a basename collides (none does today — optional policy). 12 explicit relative links need rewriting (Codex §2 table: template index, discovery inventory, headshot, backlog). `clients/<name>/contact.md` stays git-ignored and still syncs — accepted.

**FDR-003 ("separate vault from infra-docs") is explicitly superseded by ADR-0040**; its rationale stays on file. Project cards in `projects/` for every bounded engagement, pro-bono included. Standards and lessons stay local to the area; the root contract does not absorb business rules.

## 5. Decision records

- **Erratum first.** `netcup-vps/decisions/20261018-single-operator-rebuild.md` gets `id: ADR-0039`, `legacy_id: ADR-0038`, and a dated `Amended:` line explaining the collision. `shared/decisions/20260921-neurogenesis-dev-i18n-routing.md` keeps ADR-0038. Mentions of "ADR-0038" are updated **by resolved record**, never by global replace — some refer to the website decision, some to the rebuild, some are ambiguous and get a note.
- Merge decision = **ADR-0040** in `_meta/decisions/`, superseding FDR-003 and amending the credential wording of the VPS nested rule.
- Convention from now on: new files `adr-NNNN-slug.md`, frontmatter `id`, `domain`, `status`, `supersedes`, `superseded_by` (one spelling; the 20 `superseded-by` fields are normalised on decision records only). Existing dated filenames stay. FDR-001…007 keep IDs and files; `_generated/decisions.md` lists both series.
- **Allocation is serialised through one human.** Agents draft with `id: ADR-TBD`; Luis assigns the next number when accepting, in `_meta/decisions/_index.md` (0640). Four concurrent sessions cannot race a file only one person edits. (The duplicate happened under a prose rule saying "global"; prose is not an allocator.)
- Life domains do not get ADRs; admin/finance decisions live inside cases.

## 6. Code projects — tracked, not merged

Unchanged in substance: repos own `STATUS.md` (`status`, `next_action`, `blockers`, `updated`), written by a `/wrap` skill; `projects/<name>.md` cards own purpose and links; an explicit manifest `_meta/projects.yaml` lists slug → remote → per-host checkout → card; absent checkout shows `unavailable`, old `STATUS.md` shows `stale` (display policy, configurable, not backfilled).

Changed: the aggregate is **`_generated/projects-status.md`**; the generator and its dependencies are installed in an admin-owned location (`/usr/local/lib/vault-tools/`, source in `infra-tools`, installed by luis after review) — a luis-run script that imports from dev's writable checkout would be a privilege bridge. `STATUS.md` is parsed as bounded data (size and schema limits), never sourced. **Pilot with one active repo** end-to-end before extending the manifest; the merge does not depend on every repo adopting `/wrap`.

The project-vault vs master-folder rule (v3 §6) stands: infra-docs merges; `flashcard-tools`, `neurogenesis-dev`, `vast-vllm`, `devbox`, `kp-audit` stay out with cards.

## 7. Journal

- `journal/daily/YYYY-MM-DD.md`, flat, unchanged paths — `maps/home.md`'s `DAILY` constant, the Periodic Notes folder and the five cross-month links all stay valid.
- Reviews: `journal/weekly/2026-W42.md`, `journal/monthly/2026-10.md`, `journal/quarterly/2026-Q4.md`, `journal/yearly/2026.md`. Folders appear when the first one is written (Periodic Notes creates them). Explorer order is alphabetical (daily, monthly, quarterly, weekly, yearly) — accepted; periodic notes are opened via Calendar/commands and the dashboard, not by scrolling.
- Two basename collisions after merge are renamed **before** trusting bare date links: `_inbox/2026-10-04.md` → `_inbox/2026-10-04-capture.md`; `freelance _inbox/digest/2026-09-28.md` → archived with a `digest-` prefix.
- **Cadence:** weekly review as an opt-in pilot for four weeks, written by Luis in Obsidian from the existing template; `/weekly` drafts only on request. Monthly/quarterly/yearly templates stay available, nothing scheduled. Reminder automation (Tasker `obsidian://daily`, laptop timer) stays a backlog item.
- Historical journal frontmatter (187 legacy metric headers, two generations of habit markup, 40 files with mixed line endings) is **preserved as is**. New dailies: `type: journal`, tags optional.

## 7b. Zettelkasten pipeline

Fleeting (`_inbox/`, template exists) → permanent (`notes/`, template exists, own words, ≥1 meaningful link — no placeholder links) → MOCs (`maps/`, curated, ~6 notes as a heuristic) → `areas/writing/drafts/` → `published/`. Literature notes (`resources/<topic>/`, `type: literature`) are written when Luis summarises a source; the two existing web clippings stay clippings.

Measured correction: the dashboard's "capture" button creates a **task**, not a fleeting note. One explicit fleeting-capture action is added to `maps/home.md` and tested (creates `_inbox/<timestamp>-<slug>.md` from the fleeting template). `/capture` from the phone writes the same file. The five existing permanent notes move to `notes/` with backlinks intact; the one with an empty MOC placeholder gets a real parent.

## 8. Agent layer

- **Synced contracts:** root `CLAUDE.md` (≤60 lines; Codex's 49-line draft is the base, trimmed), `AGENTS.md` (three lines pointing at it, so Codex loads the same contract), `areas/infrastructure/CLAUDE.md`, `areas/freelance/CLAUDE.md`. `journal/CLAUDE.md` only if a rule turns out to be journal-specific. Trust model: **editable content, residual risk accepted and stated** (§3b). Contradictions resolved, not concatenated (Codex §5): no automatic `updated` bumps; curated `state-notes.md` exception; remote sensitive-data wording retired, no-credentials rule kept; later credential policy wins over the old VPS nested rule.
- **All eight imported instruction files retire** to `_meta/legacy-rules/<vault>-<path>-CLAUDE.md` (inert basenames) at staging time, before any agent opens the merged tree.
- `.claude/` (server-only, protected): `settings.json` with `permissions.deny` for `.env`, `secrets/`, `_generated/**`, `state/**`; `hooks/` — `PreToolUse` guard (deny writes to protected paths, any `CLAUDE.md`/`AGENTS.md`, secret patterns, moves > N files), `Stop` reminder for `/wrap`; `rules/` for server-operator specifics only (`n8nctl`, lanes); `agents/` and `skills/` per the separate council design (projects-analysis + ChatGPT roster) — **not** part of this document's scope beyond the three launch skills `/wrap`, `/capture`, `/file-note`.
- Rules load must be **tested** on the installed Claude Code and Codex before the operator units restart (RB Phase 5.6 is still deferred — install and smoke-test in step 9, not after publishing). `/context` in a Remote Control session; a broad infra question must show `areas/infrastructure/CLAUDE.md` loaded.
- Evidence-not-authority rule and explicit-go for external actions: unchanged.
- Indexes: skeleton `_index.md` files are hand-written, admin-maintained; the only generated navigation is `_generated/decisions.md`, `_generated/projects-status.md`, `_generated/inbox-age.md` (oldest untriaged item + count, linked from `maps/home.md`). Bases skill optional (zero `.base` files exist).
- `/doctor prompt-audit` monthly.

## 9. Frontmatter — future-facing, no backfill

**Measured reality:** 521 valid headers, 76 keys, 476 tags; `provenance` 84 (70 measured, 13 inferred), `last_verified` 75, `created/updated` ~490, `source` holds URLs in 2 clippings, cases carry `amount`/`direction`, `status` means different things per type, `published` is a Boolean on posts and a date on clippings. Two malformed headers (`areas/admin/cases/jobcenter-rueckforderung.md:6`, `journal/daily/2025-06-17.md:7`).

**Rules:**
- Core for **new** notes: `type` required, `tags` optional (controlled via `_meta/vocabulary.md`, union of the three vaults, grandfathered tags documented). Per-type optional fields in `_meta/schema.md`, enforced by the Templater skeleton. Journal: `type: journal` only.
- `origin` / `evidence` split is adopted for **new** notes. Migration maps `provenance: measured|inferred` → `evidence:` only where the meaning is identical; `source:` URLs stay URLs; nothing invents `origin:`, `reviewed_at`, `next_action` or `domain` on old notes.
- Case notes keep `amount`, `direction`; schema adds them officially. `status` validated per type; no universal enum.
- No bulk YAML load-and-dump (YAML 1.1 loaders reinterpret times, dates, quoting). Procedure: fix the two invalid headers → change templates → patch only approved semantic mappings with a before/after ledger → byte-compare bodies and unmapped scalars → test property parsing in Obsidian.
- No automatic `updated` bumps; existing `created`/`updated` untouched.

## 10. Migration order — staging vault, gated

Principle: all link-affecting moves happen in Obsidian on the complete merged tree in a **disconnected staging vault built from laptop originals** (the server mirrors lack `.obsidian` config and 15 media files). Before upload, rollback = delete staging. After upload, rollback is a coordinated multi-client operation — that transition is the release gate.

| Step | Work | Gate to pass |
|---|---|---|
| 0 | **Freeze + baseline.** All devices synced; stop server RC units, `obsidian-sync@*`, generators; pause Sync on phone. Tag the three laptop repos `pre-merge`. Export per vault: file list + SHA-256, Obsidian's resolved link graph (source, target identity, heading/block, embed type), active plugins and their settings files, ignored files. restic snapshot + restore test. | Three complete inventories; restore diff clean |
| 1 | **Decide on paper** — done in this document. Write `_meta/migration-map.csv` (every source file → destination/action) and `_meta/projects.yaml`. | Map is total: zero unmapped files |
| 2 | **Executables out** → `~/code/infra-tools` from a working clone of infra-docs; guards transferred to its `CLAUDE.md`; redaction hook re-tested there. | Hook passes parity test in the new repo |
| 3 | **Build staging** at `~/dev/scratch/vault-staging/` (no Sync): `git filter-repo --to-subdirectory-filter` for infra (`areas/infrastructure`) and freelance (`areas/freelance`), `--allow-unrelated-histories` into a neural-network clone; **retire all 8 instruction files** to `_meta/legacy-rules/` immediately; drop imported `.obsidian/`, `.gitignore`, `.githooks/`. History audit for secrets before combining. | No file named `CLAUDE.md`/`AGENTS.md` outside the four intended paths |
| 4 | **Erratum + identities:** ADR-0039 erratum; rename the two date collisions; resolve the three `_templates/_index.md` into one; delete neural-network's root design copy; fix device choosers in the two infra templates. | `grep -rl 'ADR-0038'` reviewed by hand, each hit classified |
| 5 | **Moves, in Obsidian only:** `personal`→`self`, `self-experiments`→`self/experiments`, `publish`→`areas/writing/{drafts,published}` (classified), `system`→`_meta`, `templates`→`deliverables`, `_inbox`/`_attachments`/`_templates` unions, the five permanent notes → `notes/`, `state/notes.md` → `state-notes.md`, scratch/JPEGs/digests per map. Whisper: `grep` transcript text outside `whisper/`; delete the folder if found, else archive the 7 notes. Then the residual pass: YAML links, code strings, Markdown paths, the 12 freelance relative links, the 10 bare `_index` links qualified with their headings, the 63 imported-path candidates re-resolved. | Resolved-edge comparison: every edge either identical or in the approved-change list; already-broken links classified separately |
| 6 | **Consumers + schema:** every row of Codex §3 (`maps/home.md` constants, queries, label map, habit actions; daily/weekly/monthly/yearly templates; startup template). New fleeting-capture action. Fix the two malformed headers; apply approved frontmatter mappings with ledger; merged `.gitignore` (contact files, `.trash`, `workspace.json`); `_meta/vocabulary.md`, `schema.md`. | Acceptance list below passes offline |
| 7 | **Publish.** Laptop's live `neural-network` tree fast-forwarded to the merged branch; Obsidian open; laptop the only Sync writer until idle. `infra-docs` and `freelance-practice` remotes untouched for 12 months. | Sync idle, zero conflict files on the laptop |
| 8 | **Server.** Stop everything; `/srv/vault` → `/srv/vault.infra-docs-2026-10`; `/srv/neural-network` and `/srv/freelance-practice` mirrors removed (`ob` device entries revoked); recreate `/srv/vault` empty with skeleton ownership; `ob sync-setup --vault neural-network --path /srv/vault`; one `ob sync`; **apply the full tier script** (Codex §6 parameterised); re-point the n8n Quadlet `Volume=` to `/srv/vault/_inbox/n8n`; fresh git baseline. | Real-host ACL tests (§3b) pass as dev and n8n, then again after a phone-created note syncs in |
| 9 | **Rules + agents:** install `.claude/` (settings, hooks, rules, launch skills); smoke-test loading with `/context` for Claude Code and a Codex `exec` reading `AGENTS.md`; ten path-cited retrieval probes (Codex §9 list); only then re-enable `obsidian-sync@vault`, the watchdog, generators, RC units. | All ten probes cite the intended file; protected paths denied in a live dev session |
| 10 | **Phone.** Check for unsynced edits; delete the local vault and re-download clean; test daily note, capture, habit toggle. | Laptop ↔ server ↔ phone converged, zero conflicts |
| 11 | **Sign-off.** Write ADR-0040 in `_meta/decisions/` (supersedes FDR-003, amends VPS nested rule, states the residual-risk acceptance); `_meta/migration-log.md`; rollback rehearsal documented; weekly-review pilot starts. Recurring automation (reviews, status, digests) is enabled only with a named reader. | ADR-0040 accepted; `/doctor prompt-audit` clean |

**Offline acceptance (step 6, repeated on the server at step 9):** inventory + attachment hashes match the map; resolved edges match except approved changes; no active imported instruction file; templates, periodic notes, startup template, habit toggle, no-daily task fallback, ISO-week boundary, fleeting capture and `obsidian://` capture work; root contract ≤60 lines; both harnesses load their contracts.

## 11. Resolved questions (cumulative)

1. `_index.md` stays; new links use vault-relative paths; the ten bare index links are qualified (not auto-pointed at a root index).
2. Depth 5 under `infrastructure/<device>/<type>/` accepted.
3. `origin`/`evidence` split — new notes only.
4. FDR IDs kept; new freelance decisions `adr-`; third decisions location `_meta/decisions/` for vault-wide only.
5. `.claude/rules/*.md` with `paths:` load late; mitigated by the synced area contracts and by naming the rule files in the root contract; tested in step 9.
6. `/wrap` authoritative for `STATUS.md`; `Stop` hook reminder only.
7. Concurrency: last-writer-wins at capacity 4 for notes (ADR-0038→0039 accepted); single-writer for generated files via `_generated/`; ADR IDs serialised through Luis.
8. Device names stay. Journal stays flat. One inbox.

## 12. Codex review disposition (2026-10-19 23:25, gpt-6.1-astra, on-disk)

| # | Finding | Decision |
|---|---|---|
| P0-1 | File modes do not protect against rename-over/unlink through writable parents; nightly chmod leaves a window; chmod leaves named ACL grants dormant | **Accepted** → §3b tiers, `_generated/`, `setfacl -b -k`, real-host tests |
| P0-2 | Mirrors lack `.obsidian` config and 15 media files; staging from them would ship a broken capture/habit flow | **Accepted** → staging from laptop originals, settings/media baseline in step 0 |
| P1-3 | ADR-0038 duplicated; a new 0039 would hide it | **Accepted** → erratum, 0039/0040, human allocator |
| P1-4 | Four nested infra `CLAUDE.md` survive a root-only retirement | **Accepted** → all eight retired |
| P1-5 | Daily basenames are not unique after merge; bare boot/index links ambiguous | **Accepted** → renames + qualified links + edge comparison |
| P1-6 | Whisper notes hold transcript text | **Accepted with Luis's correction**: delete after confirming the text exists elsewhere |
| P1-7 | Universal schema coercion destroys data; YAML round-trips are not cosmetic | **Accepted** → §9 |
| P1-8 | Merging `inbox/` into `_inbox/` widens n8n's reach | **Accepted in effect, different shape**: one inbox, n8n gets a mounted subfolder |
| P2-9 | Dashboard capture creates tasks; dev-timer templates would be unrendered | **Accepted** → explicit capture action; no review timers |
| P2-10 | Paid-only cards; empty `notes/` beside five existing permanent notes; unread digests | **Accepted** → pro-bono cards, notes moved, digests archived + reader decision |
| P2-11 | Residual files unmapped | **Accepted** → total map, §1 |
| §7 | Keep `journal/daily/` flat; month split is a preference, not a correction | **Accepted** (Luis, 23:44) |
| §7 | Flat `reviews/` | **Rejected by Luis** — sorts wrongly → interval subfolders |
| §8 | Choose and state the instruction trust model | **Accepted**: editable + residual risk, stated in ADR-0040 |
| §3 | Drop xps-windows migration | **Accepted** — nothing to migrate |
| §6 | Generator must not execute dev-controlled code as luis | **Accepted** → admin-owned install |
| §9 | "Every note needs tags" contradicts "journal: type only" | **Accepted** → type required, tags optional |

## 13. What this document does not decide

The agent/skill roster (which personas, which workers, which skills) — that is the council design, reviewed separately (`_scratch/2026-10-19-projects-analysis.md` + ChatGPT roster). This document only reserves `.claude/` as protected, names the three launch skills, and requires rule-loading to be tested before the operator restarts.
