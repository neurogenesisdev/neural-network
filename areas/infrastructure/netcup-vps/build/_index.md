---
title: Build scripts — the install as an artifact
type: index
device: netcup-vps
provenance: measured
stale: false
last_verified: 2026-08-30
commit: pending
created: 2026-08-30
updated: 2026-09-13
tags: [index, build, debootstrap, install]
---

# Build

The three scripts that produced this machine on 2026-08-30. They are the reason
[[../decisions/20260830-debootstrap-over-image-installer|ADR-0023]] chose
`debootstrap` over a GUI install: the install is a reviewable, re-runnable
artifact rather than a memory of an evening.

| Script | Does | Destructive |
|---|---|---|
| [stage1-partition-bootstrap.sh](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage1-partition-bootstrap.sh) | Wipes `vda`, partitions, creates filesystems and subvolumes, `debootstrap trixie` | **Yes, entirely** |
| [stage2-configure-boot.sh](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage2-configure-boot.sh) | fstab, hostname, network, apt sources, kernel, GRUB, locale, user, SSH keys, sshd hardening | No — operates on an unbooted target |
| [stage3-baseline.sh](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage3-baseline.sh) | zram, `unattended-upgrades`, permissive nftables, Tailscale install | No |
| `hermes/`, `n8n/` (added 2026-09-13) | Quadlet units, configs and check scripts for the secretary and the n8n instance — deployed by `scp` + `install`, see the runbook | No |
| `auto/` (added 2026-09-13) | Versioned sources of everything under `/usr/local/lib/auto/` and the `auto-*` units — not build scripts; deployed by `scp` + `install`, see [[../runbooks/automations-stage-2\|runbook]] | No |

Stages 1 and 2 run from the rescue system —
[[../runbooks/rescue-system-entry|see the runbook]]. Stage 3 runs on the booted
machine.

## Site values are tokenised and must be substituted

Each script opens with a `# --- site values ---` block holding placeholders
rather than the real address, MAC or hostname. **The scripts as committed will
not run.** Substitute from `redact.map` before use.

This is a deliberate trade and it has a cost worth stating: a committed script
that cannot be executed as-is is one step further from reproducible than one that
can. The alternative — literal values — puts the public IPv4 and hostname in a
repository whose pre-commit hook would not object to them, since its pattern set
covers UUIDs, MACs, Tailscale ranges and machine-id shapes and a routable address
is none of those. Separating site values from logic is better engineering in any
case; the tokenisation is a side effect of doing that.

**Nothing enforces this.** A future edit can inline a real address and no gate
will fire.

## Two conventions these scripts follow

**Preconditions refuse rather than warn.** Stage 1 will not run unless `/` is an
overlay mount — so an accidental invocation on the installed system or on
arch-thinkpad aborts instead of wiping a disk — and unless `CONFIRM=WIPE` is set
in the environment, which is deliberately not defaultable. Stage 2 refuses if the
target is not `trixie`, if any of the six mounts is absent, or if the SSH key
placeholder is still in place.

**Postconditions assert against generated artifacts, not inputs.** The check that
`net.ifnames=0` survived reads `/boot/grub/grub.cfg`, not `/etc/default/grub` —
asserting on the input would pass while `update-grub` silently failed. Likewise
fstab is built from `blkid` at run time rather than from recorded UUIDs, per the
root `CLAUDE.md` rule against constants in postconditions.

Both were load-bearing. The suite assertion exists because the rescue system is
Grml tracking **sid**, and a `debootstrap` that inferred its suite from the host
would have installed sid onto the server silently.

## What is not here

**Stage 3b — the lockdown — was never written**, because it is gated on verifying
Tailscale reachability first and that verification has not happened. See
[[../backlog/stage-3b-lockdown|stage-3b-lockdown]]. Writing it in advance would
produce a script whose safe execution depends on a condition the script cannot
check.

**No `spec/` chapter.** These scripts are the executable form; promoting them to
step records in the schema
[[../../arch-thinkpad/spec/_schema|arch-thinkpad/spec/_schema]] defines is
deferred rather than rejected. The gain would be transclusion into the runbooks,
which currently duplicate a few commands.

## Known gaps in what was built

Two things the scripts should have done and did not, both fixed by hand
afterward and both belonging in a revision:

- **`dbus`, `dbus-user-session` and `libpam-systemd` are missing from stage 2's
  package list.** Without them `hostnamectl` fails and — more seriously —
  `systemctl --user` does not work, which is a prerequisite for the rootless
  podman quadlets this machine exists to run. Installed manually on 2026-08-30;
  a reboot was required for PAM to register the session.
- **`@cache` is created in stage 1 in the committed version, but was not in the
  version that ran.** It was added afterward, which required moving content
  `debootstrap` had already written rather than simply mounting over it. The
  procedure and its silent-failure mode are in
  [[../decisions/20260830-subvolume-layout#The `@cache` addendum, and why it is the useful part of this record|ADR-0026]].

**So the committed stage 1 is not byte-identical to what ran.** It is the
corrected version. That is the right choice for a script whose purpose is to
rebuild the machine, and the wrong one for a historical record — the history is
in [[../sessions/20260830-netcup-vps-rebuild|the session log]].
