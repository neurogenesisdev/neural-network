---
title: "Master vault design v3 — consolidation target for neural-network"
type: design
status: reviewed   # v3 = v2 + ChatGPT critical review (04:13) + Luis's night notes (05:04) applied; see §12–§13
provenance: inferred   # design intent; research report + Luis's review of 2026-10-19 03:36 incorporated
created: 2026-10-19
updated: 2026-10-19
reviewers: [research report 2026-10-19, Luis review 03:36, ChatGPT critical review 04:13 (conditional approval)]
tags: [vault, pkm, design, merge, claude-code]
---

# Master vault design v3

> [!NOTE] Summary
> One Obsidian vault (`neural-network`) becomes the memory and working surface for Luis and for Claude Code (operator on netcup-vps, cwd = vault). `infra-docs` and `freelance-practice` merge in as **areas**. Top level stays PARA + `journal/` + `maps/` + `_meta/`. Agent rules move out of synced notes into admin-owned `.claude/`. Code projects stay in their repos and report status *into* the vault through one generated file. v3 applies the second reviewer's findings (§12): staging-vault migration, no synced instruction files, kernel-enforced generated files, `origin`/`evidence` split, minimal schema, explicit git scope.

## 0. What changed from the research recommendation (Luis's review)

| Topic | Research said | v2 decision |
|---|---|---|
| Infra layout | one area flattened by type, `device:` field | **per-device subfolders** `areas/infrastructure/<device>/<type>/`; depth accepted, because a human browsing should see the device first, not click into every note's frontmatter |
| `publish/` | left at top level (oversight) | → `areas/writing/` with `drafts/` and `published/` |
| `system/` | keep at top | → `_meta/` (utility folder, underscore like `_inbox`); scripts leave the vault |
| `personal/` | unchanged | → `areas/self/` (it holds typology, goals, philosophy, habits — "personal" said nothing) |
| Journal | year folders for new notes only, delete weekly/monthly/yearly | **`journal/YYYY/MM/`** for all notes (moved inside Obsidian); weekly/monthly/yearly become `journal/reviews/`, periodic-boundary templates stay |
| `_inbox/whisper/` | keep | delete (Whisper plugin no longer used; Wispr Flow on the phone, laptop dictation is a backlog item) |
| Decision records | freeze ADR-NNNN + FDR-NNN | **one convention vault-wide**: `adr-NNNN-slug.md`, FDRs renumbered into the global sequence with `domain: freelance` |
| Skills (kepano/obsidian-skills) | install | optional; install `obsidian-markdown` + `obsidian-bases` only; Bases are in use and wanted |
| Correspondence / letters / bank CSVs on the server | exclude | **stay** (Luis's call; they hold no credentials) |

## 1. Top level

```
neural-network/
├── CLAUDE.md          # the vault contract (≤ 60 lines), SYNCED — the laptop's Claude Code needs it too; luis-owned 0640 on the server
├── .claude/           # server-only (dot-folders don't sync): agents/, hooks/, skills/, rules/ — admin-owned
├── _inbox/            # all capture; the agent's default write target; cleared within two sessions
├── _templates/        # Templater skeletons only, prefixed by domain where they collide
├── _attachments/      # letters/, scans, images, PDFs
├── _meta/             # vocabulary.md (tags), schema.md, documentation.md, migration-log.md, decisions/ (vault-wide ADRs), legacy-rules/ (retired CLAUDE.md files, inert)
├── journal/           # YYYY/MM/YYYY-MM-DD.md + reviews/ (flat, interval in the name)
├── notes/             # Zettelkasten: permanent atomic notes, flat, never nested in PARA (§7b)
├── projects/          # one card per active project; in-vault projects keep a folder; generated _status.md
├── areas/             # see §2
├── resources/         # reference by topic — unchanged for now
├── archive/           # finished projects, retired devices/clients, archived repo cards
└── maps/              # home.md (entry), MOCs, generated decisions.md
```

Why these and not the alternatives — short versions of what Luis asked about:

- **ACE (Atlas / Calendar / Efforts)** is Nick Milo's LYT scheme: *Atlas* = knowledge and maps (≈ PARA resources + maps), *Calendar* = everything time-stamped (daily notes, logs, sessions), *Efforts* = things you push on, graded by intensity (on / ongoing / simmering / sleeping) instead of PARA's done/not-done projects. Its one real contribution is making **time a first-class axis**. PARA has none. That is why `journal/` stays top-level rather than under an area: daily notes are linked from every domain, and a time axis that lives inside one area is wrong for the other six.
- **Zettelkasten** (Luhmann's slip-box): atomic notes, one idea each, written in your own words, each linked to at least one existing note; structure emerges from links, not folders; "fleeting → literature → permanent" processing. It suits people whose output is *writing from accumulated ideas*. v2 left it out as unearned; Luis then stated the practice he wants (§7b), the templates already exist, so v3 adds a top-level `notes/`. Rule kept from the research: the slip-box is **parallel** to PARA, never inside it.
- **Johnny Decimal**: every folder gets a number `AC.ID` (`13.02 Investments`), max two levels, an index file is the law. Strength: two clicks to anything, stable across tools and file systems. Weakness here: numbers are a mass rename now and on every reshuffle, which breaks links; and it is a filing system for files, not a linking system for notes. Rejected.
- **Type-based top level** (decisions/ runbooks/ …): good for agent globs, bad for a human on a phone — a case spreads over five folders. Used *inside* areas only.

## 2. Areas

```
areas/
├── admin/            cases/, correspondence/          — cases are the unit (unchanged)
├── finance/          notes + transactions/ (bank CSVs; stay on the server)
├── health/
├── career/
├── self/             ← was personal/: typology, goals, philosophy, habits (+ learning/ folded in) + experiments/ (self-experiments: health, productivity, … — tagged by domain, linked from the area they touch)
├── writing/          drafts/, published/  ← was publish/; blog drafts for neurogenesis.dev
├── infrastructure/   ← infra-docs, per device (§3)
└── freelance/        ← freelance-practice (§4)
```

`music/` is **not** created until there is content (Milo: structure must be earned). `learning/` (1 note) folds into `self/`.

## 3. `areas/infrastructure/` — option A, by device

```
areas/infrastructure/
├── _index.md
├── arch-thinkpad/    decisions/ runbooks/ sessions/ backlog/ spec/ state/ devenv/ _source/
├── netcup-vps/       decisions/ runbooks/ sessions/ backlog/ (build/ → repo)
├── pixel-graphene/
├── shared/           decisions/ runbooks/
└── xps-windows/      status: retired → moves to archive/infrastructure/xps-windows/ at merge time
```

- Files sit at depth 5 (`areas/infrastructure/netcup-vps/decisions/adr-0038-….md`). Accepted: the human path is device → type, which matches how Luis thinks about the machines.
- **Executables leave the vault**: `build/`, `collect/`, `scripts/`, the redaction hook's code → new repo `~/code/infra-tools` (laptop + server clone). Runbooks link to scripts by repo path. Reason: Obsidian Sync resolves non-Markdown conflicts by last-modified-wins, and scripts are code (project-vault rule, §6). The vault keeps Markdown plus the generated `state/*.md` output. This is the good half of "keep infra-docs separate" without the two-vault cost (second Sync account, two instruction sets, `--add-dir` not loading `CLAUDE.md`).
- `state/` stays machine-generated; **enforced by the kernel**: `state/` directories and every generated file are luis-owned `0750`/`0640` with no group write (the same pattern as `.obsidian/`), so the operator uid cannot write them. A `PreToolUse` hook denying writes under `areas/infrastructure/*/state/**` is the second layer, not the boundary. Generators run as luis (user timer).
- The infra `CLAUDE.md` rules (provenance markers, look-it-up order, redaction placeholders, assert-against-live-state) become `.claude/rules/infrastructure.md` with `paths: ["areas/infrastructure/**"]`.
- The redaction pre-commit hook stays on the vault repo.

## 4. `areas/freelance/`

Kept as is, trimmed: `profile/ clients/<name>/ standards/ services/ deliverables/ (was templates/) decisions/ lessons/ business/ marketing/ runbooks/ backlog.md`. Its `_inbox/`, `_attachments/`, `_templates/` merge into the root ones (templates prefixed `freelance-`). Rules → `.claude/rules/freelance.md` (`paths: ["areas/freelance/**"]`). Each paid engagement with an end date gets a card in `projects/` linking back to the client folder.

`clients/<name>/contact.md` is gitignored but **Obsidian Sync ignores `.gitignore`** — the files reach the server and phone. Accepted, consistent with the correspondence decision.

## 5. Decision records — one convention

- File name `adr-NNNN-slug.md`, frontmatter `id: ADR-NNNN`, `domain:`, `status:`, `supersedes:`, `superseded_by:`. One global sequence for the whole vault **from now on**. Vault-wide decisions (this merge = ADR-0039) live in `_meta/decisions/`; device/freelance decisions stay per area.
- **MADR** ("Markdown Architectural Decision Records", v4.0.0) is just a *template standard*: a fixed section order (Context → Decision drivers → Options → Decision outcome → Consequences) and the `NNNN-title.md` naming with numbers never reused. Luis's ADRs already follow that shape; "MADR vs ADR-NNNN" is not a choice between two systems — the files are MADR-style, the prefix is the local flavour. Keep the prefix, keep the sections.
- Existing `FDR-001…008`: **IDs are durable identifiers, not display order** (reviewer, accepted). They keep their IDs and files; `maps/decisions.md` lists both series. New freelance decisions use `adr-NNNN` with `domain: freelance`. *Luis's consistency preference noted — renumbering is reversible later at the same cost (8 files), so defer rather than decide now.*
- Records live **per area** (`areas/infrastructure/<device>/decisions/`, `areas/freelance/decisions/`), so folder-scoped rules apply; a generated `maps/decisions.md` lists all of them across domains. Life domains do not get ADRs — admin/finance decisions live inside cases.
- Same `NNNN-slug` discipline for any future sequenced note type; dated types (sessions, lessons, correspondence) keep `YYYY-MM-DD-slug`.
- Pre-merge check: the inventory shows 43 ADR files but numbering reaches 0038 — find the duplicates or non-ADR files in `decisions/` first.

## 6. Code projects — tracked, not merged

Terms, in plain words: (a) a *card* = one note per project in `projects/` with the stable facts; (c) *symlink* = a filesystem shortcut from the vault into the repo (rejected: Sync would try to sync the repo); (d) a *generated status file* = a script reads each repo's `STATUS.md` and writes one table into the vault.

Design: **(a) + (d), single writer per field.**
- Each repo owns `STATUS.md` (frontmatter `status`, `next_action`, `blockers`, `updated`). Only the code session writes it — via a `/wrap` skill the session runs before Luis leaves (a Stop hook is unreliable for Remote Control sessions that never end cleanly; verify which hook event fires).
- `projects/<name>.md` cards hold what the human owns: why, area, repo path, `review_after`, links.
- `projects/_status.md` is generated from an explicit **manifest** `_meta/projects.yaml` (slug → repo remote, per-host checkout path, vault card), not from a glob of `~/code/*`. A repo not present on the server shows `unavailable`, a `STATUS.md` older than N days shows `stale`. The generator runs as luis, writes atomically, only when the content changed, and the output file is luis-owned `0640` (dev cannot edit it; kernel-enforced).
- `STATUS.md` v1 fields: `status`, `next_action`, `blockers`, `updated`. Written by `/wrap` (authoritative); a lightweight `Stop` hook only warns about unsaved work (`Stop` fires after normal turns, not after interruption; `SessionEnd` is a different event). A timer flags stale files.

**Rule — project vault vs. master-vault folder.** It is a project vault (lives in or beside its repo) if any of: (1) its primary artefact is code/config/a build that ships somewhere; (2) its docs must travel with the repo (clones, CI, agents with cwd = repo); (3) it has its own lifecycle and can be archived as a unit with the code. Otherwise it is a folder in the master vault. Docs about Luis's own life, machines and business operations always go to master, even if technical. Hence infra-docs merges; `flashcard-tools`, `neurogenesis-dev`, `vast-vllm`, `devbox`, `kp-audit` stay out. Retired/archived ones (`xps-windows`, `freelance-automation-inbox`) get an `archive/` card; content stays in the repos.

## 7. Journal

- `journal/YYYY/MM/YYYY-MM-DD.md` for **all** daily notes, existing ones moved inside Obsidian (basenames are unique, so `[[2026-03-01]]` links keep resolving). Daily Notes / Periodic Notes format `YYYY/MM/YYYY-MM-DD`.
- `journal/reviews/` is **flat; the interval is in the filename**: `2026-W42.md`, `2026-10.md`, `2026-Q4.md`, `2026.md` (≈ 70 files a year). The existing weekly/monthly/yearly Templater templates are pointed at this folder.
- **Trigger: alongside the daily note, never instead of it.** A `dev` user timer creates the skeleton from the template — Sunday 18:00 for the week (pre-filled with links to the seven daily notes), last day of the month, 31 Dec — and pings ntfy; the daily template on those days carries one line "review due → `[[2026-W42]]`". Luis writes the content; the agent drafts only on request (`/weekly`). Agent-written summaries nobody reads are the part practitioners abandon.
- Reminder automation (backlog, after the merge): phone — Tasker at a fixed time opens `obsidian://daily?vault=neural-network`; laptop — a systemd user timer with `notify-send` and `xdg-open obsidian://daily?vault=neural-network`. Laptop dictation outside Obsidian (whisper.cpp / nerd-dictation hotkey) is its own backlog item.

## 7b. Zettelkasten pipeline (added v3)

Stated practice: Wikipedia deep dives, journal ramblings and voice rambles from a phone Code-tab session → fleeting notes → fleshed out into permanent notes → interlinked from MOCs (biohacking, philosophy, AI, …) → blog post or newsletter. The gap it bridges is "non-writer → writer".

- **Fleeting** = `_inbox/` (type `fleeting`, existing template). Capture from the daily note, the home dashboard (Datacore capture, exists) and `/capture` from the phone.
- **Permanent** = `notes/<slug>.md`, flat, one idea each, own words, ≥ 1 link, existing template. `/triage` turns a fleeting note into one (or discards it); the agent proposes, Luis approves the wording.
- **Literature** = `resources/<topic>/<slug>.md` with `type: literature` (existing template) — a source summary stays near its topic.
- **MOCs** = `maps/` (hand-curated; new ones when a cluster of ≈ 6 notes appears, not before).
- **Output** = `areas/writing/drafts/` → `published/`.
- Rule: `notes/` is parallel to PARA. A permanent note is linked *from* projects and areas, never moved into them.

## 8. Agent layer

- **Root `CLAUDE.md` stays, synced** (reversal of v3-draft, see §12 P0-2). Reason: `.claude/` does not sync, so a Claude Code session in the vault *on the laptop* would otherwise have no rules. The root file is the vault contract (≤ 60 lines; vendor ceiling 200; context files make the agent do *more* of whatever they mention, so say only what it cannot discover). **Nested `CLAUDE.md`** per area where rules differ (`areas/infrastructure/`, `areas/freelance/`, `journal/`), also synced. The injection path is not Luis editing them — it is a laptop Claude Code session being talked into editing one, Sync carrying it, the server operator obeying. Mitigations: on the server all `CLAUDE.md` files are luis-owned `0640` (root one already is; nested ones get a nightly admin-lane `chmod`, because the folder's default ACL would otherwise make them dev-writable); the laptop's guard hook refuses writes to any `CLAUDE.md`; `.claude/rules/*.md` on the server duplicate nothing — they hold only server-operator specifics.
- **Imported instruction files are retired at import**: `infra-docs/CLAUDE.md`, `AGENTS.md`, `freelance-practice/CLAUDE.md` move to `_meta/legacy-rules/<vault>-CLAUDE.md` (inert names) so Claude Code cannot load them as nested memory under `areas/infrastructure/` or `areas/freelance/`. Their content is folded into `.claude/rules/*.md` by hand.
- Vault content is **evidence, not authority**: correspondence, imported documents, clippings and research can contain text addressed to an AI. The rules state it, and consequential external actions (sending, paying, deleting, pushing) require Luis's explicit go in the session.
- `.claude/` (server-only, admin-owned): `agents/`, `hooks/`, `skills/`, and `rules/` for server-specific operator rules (paths, `n8nctl`, what the dev lane may not touch). Domain *knowledge* rules live in the synced nested `CLAUDE.md` files so laptop and server agree.
- Enforcement by `PreToolUse` hooks: deny writes to `generated: true` paths (`state/**`, `*-status.md`, generated indexes); deny secret patterns; deny moves of more than N files.
- `.claude/agents/`: `case-manager` (admin, finance), `freelance-mentor`, `freelance-architect` (freelance), `sysadmin` (infrastructure), from the retired Project instructions.
- `.claude/skills/`: `obsidian-markdown`, `obsidian-bases` from kepano/obsidian-skills; `/wrap`, `/weekly`, `/triage`.
- Indexes: static (agents cannot see Bases/Dataview) but **generated** from frontmatter by a `_meta/` script; Bases for the human views.
- `.claude/` is server-local; Sync does not carry dot-folders other than `.obsidian` (verify on this setup).
- `/doctor prompt-audit` monthly for contradicting rules.

## 9. Frontmatter schema — small core, type-specific rest

**Core (every note):** `type`, `tags` (controlled). That is all. `domain` is implied by the folder.

**Per type** (defined in `_meta/schema.md`, enforced by the Templater skeleton, not by prose):
- `decision`: `id`, `status: proposed|accepted|superseded|rejected`, `date`, `supersedes`, `superseded_by`, `domain`.
- `case`: `status: open|waiting|blocked|closed`, `counterparty`, `aktenzeichen`, `next_action`, `deadline`.
- `runbook`/`spec`: `device`, `evidence`, `reviewed_at`.
- `state`: `generated: true`, `device`, `collected_at`.
- `client`, `standard`, `lesson`, `project`, `device`, `moc`: their existing fields.
- `journal`: **none beyond `type`** (the date is the filename). `review`: `period`.

**Provenance is two fields, not one** (reviewer, accepted — origin and confidence are independent):
```yaml
origin:   letter | client | luis | system | web     # where it came from
evidence: measured | reported | inferred | assumed | unverified   # how sure
[source_ref: "[[correspondence/2026-…]]"]  [verified_at: YYYY-MM-DD]
```
Inline markers `(client)`, `(luis)`, `(assumed)`, `(measured)` stay for claim-level status inside a note.

**No automatic `updated:` bumps** (Sync churn, meaningless). `reviewed_at` is set only by a deliberate check; ordinary edits are visible in git/Sync history. Existing `created`/`updated` fields are left alone, not backfilled.

## 10. Migration order — staging vault, verify before publish

Principle (reviewer, accepted): **all link-affecting moves happen inside Obsidian on the complete merged tree in a disposable staging vault; nothing is uploaded until it verifies.** Before the first upload, rollback is `rm -rf` of the staging copy; after it, rollback is a distributed Sync operation — that transition is the release gate. Source vaults are never modified while still connected to their Sync remotes.

0. **Inventory + freeze.** All devices synced; stop the server's RC units, `obsidian-sync`, generators; pause Sync on phone and server. Commit + tag the three laptop repos `pre-merge`. Record: file list with hashes, every wikilink with its *resolved* target (script: `[[link]]` → path), active plugins, tracked vs ignored files.
1. **Backup + restore test.** restic snapshot; restore into a scratch dir and diff against the working trees, ignored files included.
2. **Settle identities on paper** (this document): final paths, basename collisions (`_index.md` stays; link format = absolute path; existing bare index links rewritten), the ADR 43-vs-0038 discrepancy resolved, `ADR-0039` reserved, `_meta/projects.yaml` manifest written.
3. **Executables out.** In a working clone of infra-docs: `build/ collect/ scripts/` + the redaction hook's code → new repo `~/code/infra-tools`; runbook links updated; hook re-tested there.
4. **Build the staging vault** (laptop, `~/dev/scratch/vault-staging/`, not connected to Sync): clone of `neural-network` repo + `git filter-repo --to-subdirectory-filter areas/infrastructure` (infra clone) and `areas/freelance` (freelance clone) merged with `--allow-unrelated-histories` for history; then **remove from the working tree** the imported `.obsidian/`, `CLAUDE.md`, `AGENTS.md`, `.gitignore`, `.githooks/` (moved to `_meta/legacy-rules/` or dropped).
5. **Normalise in Obsidian, in the staging vault only:** `personal/`→`self/`, `publish/`→`areas/writing/`, `system/`→`_meta/`, journal → `YYYY/MM/`, `templates/`→`deliverables/`, `xps-windows`→`archive/`, `_inbox`/`_attachments`/`_templates` unions, `_inbox/whisper/` deleted. All as Obsidian moves. Then the script rewrites every remaining link **against the real destination tree** and compares resolved edges with the step-0 mapping (not just "unresolved ≤ baseline"; two files with one basename can silently change a link's meaning). Embeds, attachments, heading and block links checked separately.
6. **Configure + verify offline:** master `.obsidian` settings; plugins/hotkeys; Templater includes and hardcoded paths; Daily/Periodic Notes paths; Bases queries; `_meta/vocabulary.md` union; explicit `.gitignore` for the merged repo (contact files, `.trash`, `workspace.json`). Full acceptance list below, run here, before any upload.
7. **Publish.** Replace the laptop's live `neural-network` working tree with the verified staging tree (git: fast-forward to the merged branch), Obsidian open, laptop the **only** Sync writer; watch Sync activity until idle. `neural-network` remains the surviving remote (keeps its history). The `infra-docs` and `freelance-practice` remotes are left untouched for 12 months (only Sync history for those files).
8. **Server.** Stop `obsidian-sync`; move `/srv/vault` aside (`/srv/vault.infra-docs-2026-10`); recreate `/srv/vault` with the Phase-3 ownership/ACLs; `ob sync-setup --vault neural-network --path /srv/vault` into the **empty** directory (connecting a populated tree triggers merge behaviour); `ob sync` once; apply kernel protections (`.obsidian`, `state/`, generated files, `_meta/decisions`); fresh git baseline on the server with the explicit `.gitignore` (server git = snapshot/undo only; full history lives in the laptop repo); start `obsidian-sync` + watchdog.
9. **Phone.** Check for unsynced local edits first; prefer deleting the local vault and re-downloading clean over merging stale files into the new layout. Test capture and daily note.
10. **Agent layer + sign-off** (admin lane): `.claude/CLAUDE.md`, `rules/`, `agents/`, `hooks/`, `skills/`; generators; smoke-test rule loading with `/context` in a Remote Control session; the ten probes; restart RC units; write `ADR-0039` in `_meta/decisions/`; record rollback.

**Acceptance (step 6, repeated at step 10 on the server):** file inventory and attachment hashes match the expected mapping; no link changed target except as mapped; no active imported `CLAUDE.md`/`AGENTS.md`; templates, periodic notes, Bases and `obsidian://` capture work; operator uid denied writes to `.claude/`, `state/`, generated files, `_meta/decisions/`, and allowed in content folders (tested as `dev`, not inferred from `ls`); ten path-cited agent retrievals across admin, freelance and infrastructure; Sync converged on laptop, server, phone with zero conflict files.

## 11. Resolved by the second review

1. `_index.md` stays; new-link format = absolute path; existing bare `[[_index]]` links rewritten in step 5; lint in the hook. (Renaming ~50 files buys nothing.)
2. Depth 5 under `infrastructure/<device>/<type>/` is fine for the agent (globs, recursive grep); human-navigation concern only, and that is why it was chosen. No compulsory empty subfolders.
3. Provenance split into `origin` + `evidence` (§9).
4. FDR IDs kept as historical identifiers; new freelance decisions `adr-`. Open for Luis (§12 #4).
5. `.claude/rules/*.md` with `paths:` load on first Read/Write/Edit of a matching file — a broad question about infrastructure may be answered before the rules load; mitigated by `.claude/CLAUDE.md` naming the rule files and by the `sysadmin` agent reading its rule file first. Smoke-test with `/context` on the installed 2.1.x.
6. `/wrap` authoritative; `Stop` hook as a reminder only; staleness timer.
7. At 2,000 notes the risks are duplicate basenames, stale frontmatter, generated-vs-curated navigation drift and concurrent edits — all addressed by lifecycle rules, none by more hierarchy.

## 12. Review disposition (ChatGPT, 2026-10-19 04:13)

| # | Finding | Decision |
|---|---|---|
| P0-1 | Link rewrite before merge is the wrong order | **Accepted** → staging vault, §10 |
| P0-2 | `.claude/` protection incomplete; synced root `CLAUDE.md` is an injection surface | **Accepted in part, then revised.** Root is already `2750` (not group-writable) so entries cannot be replaced by the operator — verified on the live host 2026-10-19. The synced-instruction risk is real but the fix "no root `CLAUDE.md`" was wrong: the laptop's Claude Code needs the synced file. Kept synced; kernel-protected on the server (`0640` luis, nightly chmod for nested); laptop guard hook refuses `CLAUDE.md` writes (§8) |
| P0-3 | Generated-file protection by hook/flag is weak | **Accepted** → luis-owned `0640`/`0750`, generators run as luis, hooks second layer (§3, §6) |
| P0-4 | Imported `CLAUDE.md` resurrect as nested memory | **Accepted** → retired to `_meta/legacy-rules/` at import (§8, §10.4) |
| P1-5 | `source:` conflates origin and confidence | **Accepted** → `origin` + `evidence` (§9) |
| P1-6 | Git scope ≠ Sync scope; `.gitignore` not synced; history may hold secrets | **Accepted** → explicit merged `.gitignore`; server git = fresh baseline, history only on the laptop; history audit before step 4 (§10) |
| P1-7 | Status aggregation assumes repos exist on the server | **Accepted** → manifest + `unavailable`/`stale` (§6) |
| P1-8 | Universal schema will drift; auto-`updated` churns | **Accepted** → small core, type-specific, no auto bumps (§9) |
| P1-9 | `unresolved ≤ baseline` can pass with wrong targets | **Accepted** → resolved-edge mapping (§10) |
| P2-10 | Generated navigation becomes a second system | **Accepted** → generate mechanical lists only (`maps/decisions.md`, `projects/_status.md`); `maps/` stays hand-curated |
| P2-11 | `_inbox/` SLA unenforced | **Accepted** → generator reports oldest untriaged item and count into `maps/home.md`; nothing auto-moves |
| P2-12 | Lower-trust content read beside instructions | **Accepted** → "evidence, not authority" rule + explicit go for external actions (§8) |
| Lock | Rename devices to role names, OS in metadata | **Rejected.** `arch-thinkpad`, `netcup-vps`, `pixel-graphene` are the Tailscale hostnames, ssh aliases and the names in every runbook; renaming ripples far beyond the vault. OS already is metadata (`state/`). |
| Lock | RC vault lane capacity 1 until concurrency is deliberate | **Rejected.** ADR-0038 accepted last-writer-wins at capacity 4 for notes; the single-writer requirement applies to generated files, now kernel-enforced. Revisit on the first lost update. |
| Lock | ADR-0039 in `_meta/decisions/` not under the VPS | **Accepted** (§5) |
| Defer | Universal schema backfill, auto summaries, more agents/indexes, vector search | **Accepted** — launch with: 4 agents, 2 generated files, 3 skills, 1 hook set. Add on observed friction only. |
| Precedents | None matching all five domains; closest: Donnelly (dev second brain, months), Nick/createwith (7 months), Yu (merged two vaults after a month because of siloing), Larson (6 months, mobile) | Noted; consistent with the one-vault decision and with "fewer automations at launch". The 4-reorganisations-in-a-year counterexample is the reason this is the **last** restructure for 12 months: no folder changes before 2027-10 without an ADR. |

**Open for Luis:** #4 (keep `FDR-` IDs vs renumber — reviewer and I say keep; your call), and whether `_meta/decisions/` is acceptable as the third decisions location (vault-wide only).

## 13. Luis's night notes (2026-10-19 05:04) → v3

| Note | Applied |
|---|---|
| Root `CLAUDE.md` should exist with general rules; nested ones per area | Yes — §8, reversal of the draft; laptop sessions were the decisive argument |
| How is `journal/reviews/` divided; what triggers them | Flat, interval in the filename; dev timer creates the skeleton alongside the daily note on boundary days + ntfy; Luis writes it — §7 |
| Self-experiments are not only health (productivity too) | `areas/self/experiments/`, tagged by domain — §2 |
| Zettelkasten wanted: Wikipedia dives, journal and voice rambles → fleeting → permanent → MOCs → blog/newsletter; templates and Datacore capture exist | `notes/` top level, pipeline in §7b; existing templates reused; daily-note capture points at `_inbox/` |
