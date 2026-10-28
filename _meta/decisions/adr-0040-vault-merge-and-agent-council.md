---
type: decision
id: ADR-0040
aliases: [ADR-0040]
status: accepted
impact: high
provenance: measured
device: shared
domain: vault
constrains: vault layout, server permissions, agent council, decision IDs
stale: false
created: 2026-10-22
last_verified: 2026-10-22
commit: pending
supersedes: "[[fdr-003-separate-vault-from-infra-docs]]"
amends: "[[20261018-single-operator-rebuild]] (ADR-0039)"
superseded_by:
tags: [decision, vault, merge, claude-code, council, permissions]
---

# ADR-0040 — One master vault and an agent council

## Summary

`infra-docs` and `freelance-practice` are merged into `neural-network` as areas. The server enforces folder permissions in three tiers, and the 21 claude.ai Projects are replaced by a Claude Code council installed from `infra-tools`. Accepted 2026-10-22 after every migration gate and acceptance test passed.

## Context

Three vaults meant three instruction sets, three Sync remotes, cross-vault links that didn't resolve, and agents that could only see one slice of Luis's life at a time. The rebuild in ADR-0039 made the netcup server a phone-first operator host. That host needed one vault to work in, and a safe way to let agents write to it.

## Decision

1. **One vault.** `neural-network` is the only vault. Infrastructure lives in `areas/infrastructure/<device>/`, the business in `areas/freelance/`. Layout per [[2026-10-20-master-vault-design-v4]], plus addenda A1 (dailies in `journal/daily/YYYY/MM/`) and A2 (custom explorer order), recorded in [[migration-log]].
2. **No folder restructuring before 2027-10** without a new ADR.
3. **The vault's git history stays local.** No remote. Off-site copies are restic (B2, encrypted on the client) and Obsidian Sync. The old `infra-docs` and `freelance-practice` GitHub repos stay frozen until 2027-10.
4. **Executables live in `infra-tools`** (GitHub, private): build scripts, collector, redaction hook, the server permission script, the council.
5. **Server permission tiers** (v4 §3b): skeleton folders (dev can't add, remove or rename children), leaf folders (dev reads and writes), protected trees (`_generated/`, `state/`, `.claude/`, `.obsidian/`, `_meta/legacy-rules/`, `_meta/scratch/`), and contract files at 0640. Applied by `/usr/local/lib/vault-tools/apply-tiers.sh`, checked every night by `vault-drift.timer`.
6. **Synced rule files are editable content, with residual risk accepted.** A `CLAUDE.md` changed on the laptop arrives on the server as `luis`. A new `CLAUDE.md` in a leaf folder is detected by the nightly check, not prevented. The guard and scope hooks plus the managed deny rules stop agents from editing them.
7. **One inbox.** n8n writes only to `_inbox/n8n/`, bind-mounted into its container.
8. **Decision IDs are assigned by Luis** in `_meta/decisions/_index.md`. Agents draft with `ADR-TBD`. FDR-001 to FDR-007 keep their IDs.
9. **The council:** 5 boundary agents (`sysadmin`, `case-manager`, `solutions-architect`, `socratic-tutor`, `writing-mentor`) with empty prompts, 2 workers (`researcher`, `reviewer`), 2 rule-only personas (freelance and health, through area `CLAUDE.md`), 11 voice-invocable skills. Music and Japanese stay in claude.ai. Per [[2026-10-20-council-design]] r4. Installed at user scope for `luiswg`, `dev` and `luis` from `infra-tools/claude/` by `install.sh`.
10. **Rules that must hold are managed settings** (`/etc/claude-code/managed-settings.json`): no delegation to boundary agents; no agent edits to `~/.claude/{agents,skills,hooks,council,settings.json}`, to any vault `CLAUDE.md`/`AGENTS.md`, or to any vault `.claude/`.
11. **Phone access** goes through seven Remote Control hosts: `vault`, `case`, `biz`, `health`, `write`, `dev` as `dev`, and `sys` as `luis`. The agent is set by the folder's `.claude/settings.json`, which exists on the server only. `luis` has linger enabled.

## Alternatives Considered

1. **Keep three vaults, link across them.** Rejected: links don't resolve across vaults, and that means three rule sets and three remotes.
2. **Infrastructure as a separate vault next to the master.** Rejected: a second Sync account, `--add-dir` doesn't load the other vault's `CLAUDE.md`, and siloing was the original problem.
3. **Port all 21 Projects as sub-agents.** Rejected: delegation degrades past roughly 8–10 overlapping agents, a non-empty `--agent` prompt replaces Claude Code's system prompt, and most Projects had no tool or isolation boundary to justify an agent.
4. **Protect files by mode alone (v3).** Rejected after measurement: a 0640 file in a group-writable folder can still be deleted or replaced.

## Consequences

- Everything Luis keeps is in one place for every agent; letters, finance, health and infrastructure can be read together where a task needs it.
- New top-level or area folders, and any new `CLAUDE.md`, need the `luis` lane plus an entry in `apply-tiers.sh`.
- `sysadmin` hands every `sudo` command to Luis; nothing runs as root unattended.
- Rollback within the first weeks: restic `107b0aba` (pre-publish), `~/dev/neural-network.pre-merge`, the server's `*.pre-merge` folders, git tag `pre-merge-2026-10-20` in each source repo, and the old Sync remotes until they are deleted.

## Verification (2026-10-20 … 22)

- Baseline and restore: restic `d9c9c425`, restored byte-identical; restic `107b0aba` before publish.
- Links: 0 regressions over 1,504 resolved baseline edges, after every move step.
- Server tiers: 18/18 real-host tests as `dev` and `n8n`; `apply-tiers --check` clean.
- Council: A1–A9 on the laptop; A10 (seven phone sessions) passed 2026-10-22. Skill triggers 101/110, with 0 false triggers.
- Managed settings: agent edits to `~/.claude/agents/` denied for `dev`.

## Amendments

- Supersedes [[fdr-003-separate-vault-from-infra-docs]]. Its rationale (different writers, client data, lifecycle) is handled by area rules, the scope hooks and the server tiers.
- Amends ADR-0039 (the rebuild): the vault root default ACL is replaced by the tier model; the operator's Remote Control hosts are the seven council hosts; `luis` runs Claude Code as the `sysadmin` lane.
- Amended 2026-10-25: Decision 5 reduced to one tier at Luis's request — `dev` may edit the entire vault, including the root, skeleton folders, `.claude/`, `.obsidian/`, `_generated/`, `state/` and the contract files. This also removes ADR-0039 §3's kernel-enforced fence on `CLAUDE.md`, `.claude/` and `.obsidian/`. `n8n` stays confined to `_inbox/n8n/`. `apply-tiers.sh` (infra-tools `95580be`) now sets and checks 2770/0660 everywhere, and still reports any `CLAUDE.md`/`AGENTS.md` not on its list. The managed deny rules and the `scope.py` hook are unchanged, so agents are still stopped from editing instruction files and protected trees; only the OS boundary is gone.
- Amended 2026-10-28: Decision 11 changes. There are three permanent Remote Control hosts: `vault` (`/srv/vault`) and `dev` (`~dev/code`) as `dev`, and `sys` as `luis`. Each is an interactive `claude --remote-control` session in tmux (one `claude-agent@<inst>` user unit and one tmux server per agent), replacing the server-mode `claude-rc-*` units. Each agent is one continuous conversation, visible live on phone, browser and SSH, and resumed after a restart. Everything else starts on demand with `cc-new <base> <topic> [dir]` and is removed with `cc-rm`, e.g. `cc-new dev site …` for a repo or `cc-new vault case /srv/vault/areas/admin` for a one-off case-manager session; the area `.claude/settings.json` agent assignments stay in place for this. The per-area hosts (`case`, `biz`, `health`, `write`, `site`) are retired; the daily workflow of rambling into the daily note and sorting it at night replaces them, as a future automation. `dev` and `luis` each run `claude auth login` monthly. Details: [[areas/infrastructure/netcup-vps/_index|netcup-vps]].
