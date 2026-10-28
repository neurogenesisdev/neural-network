---
type: project
status: active
created: 2026-10-22
tags: [project, vault, merge]
---

# Vault merge 2026-10 — follow-ups

> [!NOTE] Summary
> The merge and the council are live as of 2026-10-22 ([[adr-0040-vault-merge-and-agent-council]]). This card holds what is left, in priority order. Close it when the last box is ticked.

## Next actions

- [x] First unit run completed 2026-10-22 14:20: snapshot `496f6bb2`, prune done, `last-success` written, stale alarm cleared (2 stale locks from the 10-20 runs removed with `restic unlock`).
- [ ] **Laptop B2 timer still skips every metered night.** The metered-link guard skips every night. Decide a fallback (for example, run on a metered link when the last success is older than 3 days) and verify `/var/lib/restic/last-success` gets written. Highest priority: data integrity.
- [ ] `vault-snap`: a git undo log for `/srv/vault` with its repository outside the vault (`/var/lib/vault-git`, group `vault`) and a `vault-snap "<why>"` wrapper that any vault member can run without sudo, so the snapshot-before-edit rule from CLAUDE.md can be followed on the server. The laptop git stays canonical.
- [x] Rewrite `areas/infrastructure/netcup-vps/_index.md`. It still describes Debian, Hermes and the auto-* timers. A good first job for `cc-sys`.
- [ ] neurogenesis.dev site sync: read `areas/writing/published/` (was `publish/blog/`).
- [ ] 15 unread freelance digests in `archive/freelance/digest/`: give the n8n digest workflow a reader, or switch it off.
- [ ] Add "`loginctl show-user luis -p Linger` = yes" to the checks in [[rebuild-2026-10]].
- [ ] `areas/finance/transactions/scu-all.csv` was deleted by a Sync client after publish (noticed 2026-10-22, restored from git). Check that it stays.
- [ ] Tune the 3 skill descriptions that missed in the trigger eval (`tech-decision`, `prd`, `venture-check`) after a few weeks of real use.
- [ ] After a week of normal use (≥ 2026-10-29): delete the old `infra-docs` and `freelance-practice` Sync remotes (Obsidian → Sync → manage remote vaults).
- [ ] After a month (≥ 2026-11-22): delete `~/dev/neural-network.pre-merge`, `~/dev/scratch/merge/`, `/srv/vault.infra-docs-2026-10`, `/srv/*.pre-merge`, and empty `.trash/`.
- [ ] Monthly: `/doctor prompt-audit` and `/skill-doctor` in a server terminal.
- [x] Switch `sys` over to `claude-agent@sys` (stop `claude-rc-sys.service` once Luis has moved off SSH; see [[areas/infrastructure/netcup-vps/_index|netcup-vps]]).
- [ ] Fold the settings keys, the sid hook and the agent files into `infra-tools`, and extend `merge_settings` to cover them.
- [ ] Monthly: `claude auth login` for `dev` and `luis`. First due 2026-11-28.
- [ ] Daily-note ramble-then-sort automation.
- [ ] Carry the whole-vault sysadmin scope into `infra-tools`.

## Links

- Design: [[2026-10-20-master-vault-design-v4]] · [[2026-10-20-council-design]] · [[migration-log]]
- Tools: `infra-tools` (GitHub) — `claude/`, `vault/apply-tiers.sh`
