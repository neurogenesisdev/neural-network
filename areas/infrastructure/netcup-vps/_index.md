---
type: index
device: netcup-vps
updated: 2026-09-12
tags: [index]
---

# netcup-vps

netcup RS 1000 G12, Nuremberg. The phone-first operator host: Claude Code runs here as `dev`, the vault lives at `/srv/vault` through Obsidian Sync, and n8n runs as its own user. Rebuilt 2026-10-18 under [[20261018-single-operator-rebuild|ADR-0039]] (checklist: [[rebuild-2026-10]]); the vault layout, permission tiers and agent council come from [[adr-0040-vault-merge-and-agent-council|ADR-0040]].

**Provenance:** everything below is `measured` on 2026-10-25 from a `luis` session with sudo, unless marked otherwise. This host has no collector, so these are dated observations, not continuous verification (see [[netcup-collector]]).

> [!IMPORTANT] The only route in is the tailnet, plus the netcup VNC console.
> nftables input policy is `drop`; only UDP 41641 and `tailscale0` are accepted. `nftables.service` staying enabled is load-bearing and only re-checked by hand — [[20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]].

## Host

| | |
|---|---|
| OS | Ubuntu 26.04.1 LTS (`resolute`), kernel 7.0.0-38-generic |
| Ubuntu Pro | Attached (free personal). `esm-apps`, `esm-infra` and `livepatch` enabled; Livepatch running. snapd is installed only for `canonical-livepatch` |
| Updates | `unattended-upgrades` with the release, security and both ESM security origins |
| sudo | sudo-rs 0.2.13 |
| Hardware | 4 vCPU, 7.7 GiB RAM, 256 GB virtio disk, 3.9 GiB zram swap |
| Disk | BIOS/GPT. ext4 `/boot` on `vda2`; btrfs on `vda3` with seven subvolumes: `@`, `@home`, `@srv`, `@containers`, `@log`, `@cache`, `@snapshots`. **No full-disk encryption, by design** — [[20260830-no-luks-on-always-online-vps\|ADR-0021]] |
| Network | systemd-networkd, static dual-stack; `systemd-resolved` with the stub `resolv.conf` |
| Tailscale | 1.104.1, node `netcup-vps` = `<TS_IPV4>`. Not version-held. `tailscale serve` exposes `https://netcup-vps.<tailnet>.ts.net:8443` → `127.0.0.1:5678` (n8n), tailnet only. No Funnel |
| SSH | Port 22 on the wildcard (ADR-0029), keys only, `PermitRootLogin no`, `AllowUsers luis dev`; config in `/etc/ssh/sshd_config.d/10-hardening.conf`. Host key fingerprints: [[ssh-lockout-recovery]] |
| Outer ring | netcup panel firewall policy `tailnet-only` — `inherited-unverified`, not rechecked since the rebuild ([[areas/infrastructure/netcup-vps/runbooks/panel-firewall-outer-ring|panel firewall runbook]]) |

## Principals

| User | uid | Groups | Login | What it is for |
|---|---|---|---|---|
| `luis` | 1000 | `sudo`, `vault` | SSH (password set) | Admin. Owns `/srv/vault`, runs Obsidian Sync and the `sys` Remote Control host, owns `/opt/infra-tools`. Node 22 via mise |
| `dev` | 1001 | `vault` | SSH (password locked) | The coder. Two permanent Remote Control agents (`vault`, `dev`) plus on-demand ones (since 2026-10-28), Codex, repos under `~dev/code`. No general sudo. Node 22 via mise; mise shims on `PATH` for non-interactive shells from `~/.zshenv` (added 2026-10-23) |
| `n8n` | 1002 | — (not in `vault`) | nologin | n8n and its Postgres as rootless podman Quadlets |
| `root` | 0 | | locked | No SSH. Console access goes through `luis` |

All three users linger. Subuid/subgid ranges: `luis` 100000, `dev` 165536, `n8n` 231072 (65536 each).

### sudo rules (`/etc/sudoers.d/`)

| File | Rule |
|---|---|
| `dev-n8nctl` | `dev ALL=(root) NOPASSWD: /usr/local/sbin/n8nctl` — a root-owned wrapper accepting `status`, `restart`, `stop`, `start`, `logs [n]`, which runs `systemctl --user -M n8n@` / `journalctl --user -M n8n@` on `n8n.service` |
| `luis-nopasswd` | `luis ALL=(ALL:ALL) NOPASSWD: ALL` — added 2026-10-23 |
| `README` | Package default, no rules |

Plus the stock `%sudo ALL=(ALL:ALL) ALL` in `/etc/sudoers`.

> [!WARNING] `luis-nopasswd` contradicts ADR-0039 §2
> ADR-0039 gives `luis` **password** sudo and rules out NOPASSWD, because `luis` runs the long-lived `ob sync` process. Since 2026-10-23 every process running as `luis` — `ob sync`, the `claude-rc-sys` sessions — can become root without a prompt. Either amend ADR-0039 or remove the file.

## systemd units

### System (custom, all enabled)

| Timer | Schedule | Runs |
|---|---|---|
| `restic-b2.timer` | daily, ±1 h | `/usr/local/bin/restic-b2 backup … / /boot /home /srv /var/log`, credentials via `LoadCredential=` from `/etc/credstore` — [[restic-b2-backup]]. Last success 2026-10-25 00:53 |
| `restic-b2-check.timer` | monthly | `restic check --read-data-subset=5%` |
| `restic-b2-stale.timer` | 09:00 | `/usr/local/bin/restic-b2-stale-check` |
| `vault-drift.timer` | 03:30 | `/usr/local/lib/vault-tools/drift-check.sh` — see [[#Vault]] |
| `n8n-db-dump.timer` | 03:30 | `User=n8n`, nightly Postgres dump with a healthchecks ping |
| `n8n-alive.timer` | hourly | `User=n8n`, alive check with a healthchecks ping |

Both n8n jobs run `/usr/local/lib/n8n/*.sh` through the healthchecks wrapper `/usr/local/lib/auto/hc-run.sh`. Every unit has `OnFailure=ntfy-alert@%n`, the system template that posts to ntfy (topic from `/etc/credstore`). Other enabled services are the distribution's own plus `nftables`, `tailscaled`, `ssh`, `unattended-upgrades` and the Livepatch snap. No failed units.

### User units

| User | Unit | Working directory / does |
|---|---|---|
| `luis` | `obsidian-sync@vault.service` | `mise exec -- ob sync --continuous --path /srv/vault`, `Restart=always` |
| `luis` | `sync-watchdog.timer` (hourly) | `~/.local/bin/sync-watchdog`: unit active, and a heartbeat written to `_generated/sync-heartbeat.md` must show `Upload complete` within 60 s; otherwise restart Sync |
| `luis` | `claude-agent@sys.service` | `/srv/vault`, permission mode `default` — the admin lane. Installed 2026-10-28; `claude-rc-sys.service` (server mode, capacity 2) stays running until Luis switches over from SSH |
| `dev` | `claude-agent@vault.service` | `/srv/vault`, `acceptEdits` |
| `dev` | `claude-agent@dev.service` | `~dev/code`, `acceptEdits` |
| `dev` | `claude-agent@<base>-<topic>.service` | On demand: `cc-new <base> <topic> [dir]`, removed with `cc-rm` (e.g. `cc-new dev site ~/code/neurogenesis-dev`, `cc-new vault case /srv/vault/areas/admin`) |
| `n8n` | `n8n-db`, `n8n`, `n8n-network` | Generated from the Quadlets below |

Since 2026-10-28 (`measured`) the Remote Control hosts are **interactive** `claude --remote-control` sessions in tmux, one conversation per agent, visible live on phone, browser and SSH (`a <inst>` attaches). `claude-agent@.service` is `Type=forking` and runs `tmux -L <inst> new-session -d … ~/.local/bin/claude-agent-run <inst>`, one tmux server per agent, with `Restart=always`, `StartLimitBurst=5` per 600 s, `OnFailure=ntfy-alert@%n` (the per-user ntfy template), `MemoryHigh=800M`, `MemoryMax=1200M`. The runner sources `~/.config/claude-agents/<inst>.env` (`DIR`, `NAME`, `MODE`) and resumes the session in `~/.local/state/claude-agents/<inst>.sid`, which a SessionStart hook (`~/.local/bin/claude-agent-sid-hook`, in `~/.claude/settings.json`) rewrites on every start and `/clear`. tmux 3.6 moves each pane into its own `tmux-spawn-*.scope` outside the unit's cgroup, so the runner copies the unit's memory limits onto that scope. Shell helpers (`a`, `cc-ls`, `cc-new`, `cc-rm`, `cc-rc`) are in `~/.config/zsh/claude-agents.zsh`, sourced from `~/.zshrc`. The seven `claude-rc-*` units under `dev` are disabled; their files stay for rollback (`systemctl --user disable --now claude-agent@X && systemctl --user enable --now claude-rc-X`). Area `.claude/settings.json` agent assignments are unchanged and apply when an on-demand agent starts in that folder. Claude Code is 2.1.312 for `luis`, 2.1.310 for `dev` (2026-10-28). There is no `claude-update.timer`; Claude Code updates itself (`inferred`). No user timers under `dev`.

## n8n

Quadlets in `~n8n/.config/containers/systemd/`, podman 5.7.0, started by their `[Install]` sections:

- `n8n.network` — network `n8n`.
- `n8n-db.container` — `postgres:16-alpine` (tag, not digest), data in `~n8n/pgdata`, health check `pg_isready`, `DropCapability=ALL` with five added back.
- `n8n.container` — `ghcr.io/n8n-io/n8n`, pinned by digest. Data in `~n8n/data`, env from `~n8n/n8n.env`, `PublishPort=127.0.0.1:5678:5678`, `DropCapability=ALL`, `NoNewPrivileges`, `PidsLimit=512`. The only vault mount is `/srv/vault/_inbox/n8n` → `/vault/inbox`.

Secrets are podman secrets `n8n-db-password` and `n8n-encryption-key` (both also in KeePassXC). Rules for agents working on workflows: [[areas/infrastructure/netcup-vps/n8n/CLAUDE|n8n/CLAUDE.md]].

## Vault

`/srv/vault` is the `neural-network` vault on its own subvolume `@srv`, synced by `ob sync` as `luis`. Ownership is `luis:vault` throughout, with no `other` bits.

**Permissions** ([[adr-0040-vault-merge-and-agent-council|ADR-0040]] §5, amended 2026-10-25): one tier. Every folder is `2770` and every file `0660`, with a default ACL `g::rwx`, so `dev` can create, edit, rename and delete anything in the vault — including the root, `.claude/`, `.obsidian/`, `_generated/` and every `CLAUDE.md`. Set by the tier script `/usr/local/lib/vault-tools/apply-tiers.sh`, identical to `vault/apply-tiers.sh` in infra-tools. The skeleton, protected and contract tiers were retired 2026-10-25.

What still stops agents from editing instruction files and protected trees is Claude Code, not the kernel: the deny rules in `/etc/claude-code/managed-settings.json` and the council's `scope.py` hook.

`n8n` gets traverse-only ACLs on the root and `_inbox/`, and `rwX` (plus default) on `_inbox/n8n/` only. `vault-drift.service` runs `apply-tiers.sh --check` nightly: every folder `2770`, every file `0660`, every owning-group ACL entry `rw`, and no `CLAUDE.md`/`AGENTS.md` outside its list. Then it runs the infra-tools council installer's `--check` as `luis` and `dev`, and confirms `managed-settings.json` exists. Clean on 2026-10-25 after the change.

> [!WARNING] No git repository in `/srv/vault`
> ADR-0040 §3 and the vault contract assume local, commit-only git history. On this host `/srv/vault` has no `.git`; the only server-side history is `/srv/vault.infra-docs-2026-10/.git`, the pre-merge copy. Whether the laptop's copy of the vault has a repository was not checked. Until one exists here, "make the snapshot commit before an agent job" cannot be done on the server; the undo paths are Obsidian Sync version history and restic.

Pre-merge copies kept until ≥ 2026-11-22 ([[vault-merge-2026-10]]): `/srv/vault.infra-docs-2026-10`, `/srv/neural-network.pre-merge`, `/srv/freelance-practice.pre-merge`.

## /opt/infra-tools

A clone of the private `neurogenesisdev/infra-tools` repository, owned by `luis`, on `main` and level with `origin/main`. It is the home of everything executable ([[adr-0040-vault-merge-and-agent-council|ADR-0040]] §4):

- `claude/` — the agent council (agents, skills, hooks, `install.sh`, evals, tests), version 0.1.4. Installed and checked clean for `luis` and `dev`.
- `vault/apply-tiers.sh` — the tier script above.
- `netcup-vps/build/` — `ubuntu-2604/` stage scripts and `n8n/` Quadlet sources. `netcup-vps/archive/` keeps the retired builds. `netcup-vps/collect/` holds only `gen-redact-map.sh`; there is still no collector.
- `arch-thinkpad/`, `finance/` — the laptop's collector and devenv, and the bank-statement parsers.
- `.githooks/` — the pre-commit redaction gate.

## Repositories and deploy keys

| Where | Repository | Key |
|---|---|---|
| `~dev/code/neurogenesis-dev` | `neurogenesisdev/neurogenesis-dev`, branch `main` | `site-deploy`, via SSH host alias `github-site` |
| `/opt/infra-tools` (as `luis`) | `neurogenesisdev/infra-tools` | `infra-tools-deploy`, via SSH host alias `github-infra-tools` |

`~dev/code` holds only that one repository. Whether each key has write access on GitHub was not checked from here. Codex 0.163.0 runs as `dev` with the AppArmor profile `/etc/apparmor.d/codex-bwrap`; Playwright's Chromium is under `~dev/.cache/ms-playwright`.

## Drift from the rebuild record

Things the runbook or ADR-0039 say that the host does not match, as of 2026-10-25:

- **sudo for `luis`** is passwordless (see the warning above).
- **No memory slices.** Runbook step 3.4 set `MemoryHigh`/`MemoryMax` for `dev` (6G/8G) and `n8n` (1.5G/2G); no `user-*.slice.d` drop-ins exist and both slices show `infinity`. The runbook also assumes 16 GB RAM; the host has 7.7 GiB.
- **No `claude-update.timer`** (runbook 5.3).
- **systemd-resolved is in use**; stage 2 planned a static `resolv.conf`.
- **Postgres is pinned by tag**, not digest.
- **`/srv/vault` has no git repository** (see the warning above).
- Small: `/etc/systemd/system/chronyd.service` is a dangling symlink (chrony removed, config files left); `claude-rc-site.service` has `Description=… (dev)`.

## Contents

| Folder | Contents |
|---|---|
| [[areas/infrastructure/netcup-vps/decisions/_index\|decisions/]] | Device ADRs, ADR-0021 through ADR-0039. The global allocator is [[_meta/decisions/_index\|_meta/decisions]] |
| [[areas/infrastructure/netcup-vps/runbooks/_index\|runbooks/]] | Procedures — the 2026-10 rebuild, rescue entry, SSH lockout, restic → B2, panel firewall |
| [[areas/infrastructure/netcup-vps/backlog/_index\|backlog/]] | Open and resolved items |
| [[areas/infrastructure/netcup-vps/sessions/_index\|sessions/]] | Working narrative, including the [[20261018-rebuild\|2026-10-18 rebuild]] |
| [[areas/infrastructure/netcup-vps/build/_index\|build/]] | Points at the scripts in infra-tools |
| `n8n/` | The `CLAUDE.md` contract for n8n work |
| state/ | **Does not exist.** No collector for this device |

## Redaction

Identifiers use the vault's placeholder convention; the mapping lives in a gitignored `redact.map`, generated by `infra-tools/netcup-vps/collect/gen-redact-map.sh` (where the map itself is kept was not rechecked: `inherited-unverified`).

| Token | Covers |
|---|---|
| `<SERVER_ID>` | netcup's `v2202…` server identifier and its `ultrasrv.de` hostname |
| `<NETCUP_IPV4>` / `<NETCUP_GW4>` | Public IPv4 and its gateway |
| `<NETCUP_IPV6>` / `<NETCUP_PREFIX6>` | Public IPv6 address and the routed /64 |
| `<PERMADDR_eth0>` | virtio NIC MAC |
| `<FSUUID_vda2>` / `<FSUUID_vda3>` | `/boot` and root filesystem UUIDs |
| `<MACHINE_ID>` | `/etc/machine-id` |
| `<TS_IPV4>` / `<TS_IPV6>` | This host's Tailscale addresses — enforced by the pre-commit hook |
| `<TS_IPV4_arch-thinkpad>` | Another device's Tailscale address, referenced from this folder |

The pre-commit hook does not catch the public IPv4 or hostname; tokenising them is discipline, not enforcement. SSH host key fingerprints are public by construction and stay unredacted in [[ssh-lockout-recovery]].
