---
type: index
device: netcup-vps
updated: 2026-09-13
tags: [index]
---

# Decisions

Architecture Decision Records for netcup-vps. Same rules as
[[../../arch-thinkpad/decisions/_index|arch-thinkpad's]]: append-only, reverse by
writing a new record and setting `superseded-by` on the old one.

Numbering continues the global sequence from arch-thinkpad's ADR-0020 — see
[[../_index#ADR numbering is global, not per-device|the device index]] for why.

**Six of the first seven were settled before a single destructive command ran.**
That was the point: the disk layout, the boot mode and the encryption question are
all expensive to reverse once `debootstrap` has run, and cheap to argue
beforehand. ADR-0029 is the first record here written *after* the fact it
describes was measured.

## Decision Index

| ADR | Status | Impact | Date | Summary |
|-----|--------|--------|------|---------|
| [[20260830-no-luks-on-always-online-vps\|ADR-0021]] | accepted | high | 2026-08-30 | No FDE. **The operator's argument, not the advisor's** — a server that is never at rest never gets the protection at-rest encryption offers. Cost is repaid by application-layer encryption; the residual obligation is that no long-lived secret reaches disk in plaintext |
| [[20260830-bios-boot-retained\|ADR-0022]] | accepted | medium | 2026-08-30 | BIOS/GPT, not UEFI. The stock image shipped a dead 244 MiB ESP while the firmware booted legacy. Secure Boot is theatre on a host that can read guest RAM, so the ESP was dropped rather than repaired |
| [[20260830-debootstrap-over-image-installer\|ADR-0023]] | accepted | high | 2026-08-30 | The SCP image installer offers two fixed partition layouts and no filesystem choice, so it cannot produce this disk. Rescue + `debootstrap` over ISO-via-VNC: scriptable, reviewable, no console typing |
| [[20260830-debian-over-rhel-family\|ADR-0024]] | accepted | medium | 2026-08-30 | Debian 13 over Rocky/Alma 10, whose case was the strongest rival — podman is Red Hat's project and SELinux confines containers better. Rejected on **estate-wide cognitive load**, not on technical merit |
| [[20260830-boot-filesystem-is-ext4\|ADR-0025]] | accepted | medium | 2026-08-30 | `/boot` is a separate ext4 partition. GRUB can read btrfs; GRUB reading btrfs *with subvolumes and a default subvolume* is a known source of unbootable systems, and this one is debugged over VNC or not at all |
| [[20260830-subvolume-layout\|ADR-0026]] | accepted | medium | 2026-08-30 | Six subvolumes, and **the reason is layout hygiene, not rollback** — netcup's hypervisor snapshots are the better rollback primitive here. `@cache` was added after `debootstrap` had already written into `/var/cache`, which is the interesting part |
| [[20260830-vps-holds-no-credentials-upward\|ADR-0027]] | accepted | high | 2026-08-30, accepted 2026-08-31 | Trust flows inward: this box may hold credentials to things below it, never to arch-thinkpad or pixel-graphene. **What unblocked it was a misreading being cleared, not a trade-off being accepted** — the git case never needed an upward credential, because GitHub sits *below* this host in the trust order |
| [[20260831-sshd-binds-wildcard-nftables-is-the-control\|ADR-0029]] | accepted | medium | 2026-08-31 | sshd keeps listening on the wildcard; nftables is the control. **Declines defence in depth on purpose** — binding sshd to the tailnet address adds a boot-order dependency on `tailscaled`, and a machine that comes back *closed and unreachable* is worse than one that comes back *reachable and exposed* when the only fallback is a VNC console |
| [[20260911-persistent-credentials-on-netcup-vps\|ADR-0034]] | accepted 2026-09-12 | high | 2026-09-11 | **Amends ADR-0021's residual.** "No long-lived secret in plaintext" is unenforceable on a host that runs coding agents and timers; replaced by five properties (downward, revocable, least-privilege, one owner, listed) and a named credential table. Accept once stage 0/1 of the coding-rig plan matches the table |
| [[20260912-netcup-vps-is-the-coding-host\|ADR-0035]] | accepted | high | 2026-09-12 | **This host is the coding workstation.** Three harnesses, three boundaries: Claude Code host + OS sandbox, Codex host + permission profiles, OpenCode in a container. Agents commit on branches, Codex reviews, humans merge/push/deploy. Two named openings for the review plugin (`~/.codex` read/write, plugin state symlinked out of `~/.claude/plugins`). Day one the reviewer found two real defects the coder's own checks could not |
| [[20260913-contained-secretary-hermes\|ADR-0036]] | accepted | medium | 2026-09-13 | **A contained chat secretary (Hermes) as principal `hermes`.** Amends plan D7: admitted only inside a rootless container whose mounts are the boundary (vault ro, inbox ro + `secretary/` rw, no host home, no git key, no port), file tools only, no shell tool in the model's list. Measured the same day: laptop unreachable from inside, refusals without approval prompts, kill switch + linger restart, both alert paths. Two traps: `DropCapability=ALL` kills the image's init; `debugging` in `disabled_toolsets` silently removes `file` |

## Open

Nothing. ADR-0034 accepted 2026-09-12 once stage 1 matched its table.

**One consequence is still unimplemented**, and it is mechanical rather than
undecided: the deploy key exists on the host but its registration in the
repository's settings is unverified, and this record should move to `shared/` — it
constrains every device, not this one. Both in
[[../backlog/vps-trust-direction|the backlog item]].

## Not recorded here

**Whether to enable Tailscale SSH** is still not an ADR — it is a question in
[[../backlog/tailscale-ssh-decision|the backlog]]. Its premise is now settled, but
ADR-0029 changed the question rather than answering it: that record declined a
boot-order dependency on `tailscaled`, and Tailscale SSH deepens exactly that
dependency by moving *authorisation* onto the control plane too.

**The netcup panel firewall's default posture** is a measured fact rather than a
choice: rules evaluate top-to-bottom and the implicit system rules accept all
inbound and all outbound, so only the SMTP block and the ICMP allows are live. That
belongs in `state/` — which does not exist — and is recorded in
[[../sessions/20260830-netcup-vps-rebuild|the session log]] until it does.
