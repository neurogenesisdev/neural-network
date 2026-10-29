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
- [x] `vault-snap`: a git undo log for `/srv/vault` with its repository outside the vault (`/var/lib/vault-git`, group `vault`) and a `vault-snap "<why>"` wrapper that any vault member can run without sudo, so the snapshot-before-edit rule from CLAUDE.md can be followed on the server. Done 2026-10-28, and extended: the server is the source of truth, pushing hourly to GitHub (ADR-0040 amended 2026-10-28, Decision 3). The laptop git is no longer canonical.
- [ ] On the laptop, replace these two lines in the root `CLAUDE.md` (agents may not edit it):
  ```diff
  - This vault's Git repository has no remote. Git is local history only; Obsidian Sync carries the files.
  - Git is commit-only while Sync watches. Before an agent job, make the approved snapshot commit.
  + The server's git (/var/lib/vault-git) is the source of truth and pushes hourly to a private GitHub repo; Obsidian Sync carries the files.
  + Never run git by hand in the vault. On the server, the snapshot is vault-snap "<why>"; make it before an agent job.
  ```
- [ ] Laptop git: stop committing to the laptop's own vault repository, or replace it with a clone of `neurogenesisdev/neural-network`.
- [ ] Laptop, infra-tools tidy-up (ADR-0040 §4: executables live in infra-tools). Bring the server-only vault git pieces into the repo: `/usr/local/bin/vault-snap`, `/usr/local/lib/vault-git-hooks/secret-scan` (with its `pre-commit` and `pre-push` links), `vault-autocommit.{service,timer}`, `/etc/gitconfig` (`safe.directory`), and the `/var/lib/vault-git` layout (reftable, root-owned top folder, `config`, `HEAD` and `info/`). Also commit the `ROOT_TREES` change to `vault/apply-tiers.sh`, which is uncommitted in `/opt/infra-tools`. That clone also has 1 commit not on GitHub (not from this session), so check it first.
- [ ] Laptop, infra-tools: bring the claude.ai connector into the repo. The 11 local commits on `~mcpread/obsidian-remote-mcp` branch `local/netcup` (as patches over `v1.5.0` plus an apply script) exist only in that clone and in restic. Also the units `mcpread-{gateway,auth,alive}` and `srv-vaults-code.mount`, `/usr/local/lib/mcpread/alive.sh`, `/etc/mcpread/vault-policy.json`, the slices and the nftables rules. List: [[mcp-endpoint]].
- [ ] Assign an ID to the connector ADR ([[20261029-claude-ai-vault-connector-read-write]], `ADR-TBD`, supersedes ADR-0037). On acceptance, give ADR-0037 `superseded_by` and a dated line pointing to it.
- [x] Stop `.claude` being renamed away: the vault's top folder is now `root:vault` 3770 (sticky), done 2026-10-28 (ADR-0040, third amendment that day).
- [ ] `vault-drift` is failing since 2026-10-28 12:59: `luis`'s council release `0.1.4` has 3 files changed in place (`agents/sysadmin.md`, `hooks/scope.py`, `hooks/tests/test_scope.py`) that no longer match its MANIFEST. Cut a new release or reinstall; it alerts every night until then.
- [x] Rewrite `areas/infrastructure/netcup-vps/_index.md`. It still describes Debian, Hermes and the auto-* timers. A good first job for `cc-sys`.
- [ ] neurogenesis.dev site sync: read `areas/writing/published/` (was `publish/blog/`).
- [ ] 15 unread freelance digests in `archive/freelance/digest/`: give the n8n digest workflow a reader, or switch it off.
- [ ] Add "`loginctl show-user luis -p Linger` = yes" to the checks in [[rebuild-2026-10]].
- [ ] `areas/finance/transactions/scu-all.csv` was deleted by a Sync client after publish (noticed 2026-10-22, restored from git). Check that it stays.
- [ ] Tune the 3 skill descriptions that missed in the trigger eval (`tech-decision`, `prd`, `venture-check`) after a few weeks of real use.
- [ ] After a week of normal use (≥ 2026-10-29): delete the old `infra-docs` and `freelance-practice` Sync remotes (Obsidian → Sync → manage remote vaults).
- [ ] After a month (≥ 2026-11-22): delete `~/dev/neural-network.pre-merge`, `~/dev/scratch/merge/`, `/srv/vault.infra-docs-2026-10`, `/srv/*.pre-merge`, and empty `.trash/`.
- [ ] Monthly: `/doctor prompt-audit` and `/skill-doctor` in a server terminal.
- [ ] Monthly: rebase obsidian-remote-mcp's `local/netcup` on the latest upstream release, test, review if the auth server or a write tool changed, then redeploy (steps in [[mcp-endpoint#Care]]). First due 2026-11-29.
- [x] Switch `sys` over to `claude-agent@sys` (stop `claude-rc-sys.service` once Luis has moved off SSH; see [[areas/infrastructure/netcup-vps/_index|netcup-vps]]).
- [ ] Fold the settings keys, the sid hook and the agent files into `infra-tools`, and extend `merge_settings` to cover them.
- [ ] Monthly: `claude auth login` for `dev` and `luis`. First due 2026-11-28.
- [ ] Daily-note ramble-then-sort automation.
- [ ] Carry the whole-vault sysadmin scope into `infra-tools`.

## Links

- Design: [[2026-10-20-master-vault-design-v4]] · [[2026-10-20-council-design]] · [[migration-log]]
- Tools: `infra-tools` (GitHub) — `claude/`, `vault/apply-tiers.sh`
