---
type: backlog-item
status: active
priority: medium
device: netcup-vps
projects: [vault]
tags: [backlog, vault, collector, state, redaction]
created: 2026-08-30
updated: 2026-08-30
commit: pending
---

# netcup-collector

> [!NOTE] Summary
> There is no `collect.sh` for this device, so `netcup-vps/state/` does not
> exist. Every factual claim in this folder is a point-in-time measurement from
> 2026-08-30 that nothing will re-verify or notice drifting. The vault's central
> rule — ground claims in generated state — currently cannot be followed here.

## Context

`arch-thinkpad/collect/collect.sh` is Arch-specific throughout: `pacman -Qqe`,
`sbctl`, `snapper`, `usbguard`, `mkinitcpio`. None of it runs on Debian, and
several of its subsystems have no analogue on this machine — there is no Secure
Boot ([[../decisions/20260830-bios-boot-retained|ADR-0022]]), no LUKS
([[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]]), and no snapper
([[../decisions/20260830-subvolume-layout|ADR-0026]]).

So this is not a port. It is a second collector sharing conventions and redaction
machinery with the first.

## Steps

1. Write `netcup-vps/collect/collect.sh` emitting the sections that apply:
   - `packages.md` — `dpkg-query -W -f`, plus `apt-mark showmanual` for the
     explicit set. **Note the arch-thinkpad blind spot applies here too**:
     manual-only lists hide dependencies, and absence is not evidence of absence.
   - `services.md` — enabled system and user units. The user-unit half matters
     more here than on the laptop, since rootless podman quadlets live there.
   - `storage.md` — `lsblk`, `blkid`, `btrfs subvolume list`, `findmnt`, `fstab`,
     `swapon --show`, and the btrfs `qgroup` figures ADR-0026 exists to make
     available.
   - `network.md` — interfaces, `nft list ruleset`, listening sockets,
     `/etc/network/interfaces`, Tailscale status.
   - `boot.md` — `/proc/cmdline`, `GRUB_CMDLINE_LINUX`, installed kernels.
     **`net.ifnames=0` should be an asserted invariant here**, not just recorded.
   - `containers.md` — quadlet units, images with digests, `podman secret ls`
     (names only).
2. Reuse `redact.sh` and extend `gen-redact-map.sh` for this host's identifiers —
   the tokens are already listed in [[../_index#Redaction|the device index]].
3. Decide where it runs from. The laptop's collector runs locally with sudo; this
   one either runs on the server and its output is fetched, or runs over SSH from
   arch-thinkpad. The second keeps the vault on one machine and needs care around
   the `bash -s`/`sudo` stdin collision recorded in
   [[../sessions/20260830-netcup-vps-rebuild|the session log]].
4. Add a `state/_index.md` and delete the "no `state/`" section from
   [[../_index|the device index]].

## Blocked on

Not blocked, but low-value until the machine stops changing. A collector run
against a host that is about to gain a firewall, a tailnet address and three
services records a state that is obsolete within the week.

**Reasonable order: after [[stage-3b-lockdown|stage 3b]], before the services.**

## Note

**The gap is not uniform, and the dangerous part is the runbooks.** ADRs record
decisions and age gracefully; a decision made on 2026-08-30 is still the decision
that was made. The runbooks contain *current-state* claims — the six-subvolume
mount sequence, the host key fingerprints, `passwd -S root` returning `L` — and
those are read under pressure, when being wrong is most expensive.

Until the collector exists, treat anything load-bearing in
[[../runbooks/_index|runbooks/]] as needing verification against the host rather
than against the file.

**Do not hand-write `state/` in the meantime.** That is precisely the failure the
root `CLAUDE.md` forbids: an inference recorded as a measurement. Facts with
nowhere else to live go in the session log, marked as measured on a date, which is
where the 2026-08-30 panel readings currently are.

## Related

- [[../_index#No `state/`, and what that costs|Device index — the cost]]
- [[../../arch-thinkpad/backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]] — the same class of gap, already found once
- [[../../arch-thinkpad/backlog/collector-stale-file-detection|collector-stale-file-detection]] — a defect to avoid reproducing
