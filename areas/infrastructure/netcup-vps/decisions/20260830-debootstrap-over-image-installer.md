---
title: Install by debootstrap from rescue, not the SCP image installer
type: decision
id: ADR-0023
aliases: [ADR-0023]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: high
created: 2026-08-30
updated: 2026-08-30
tags: [decision, install, debootstrap, rescue, netcup, reproducibility]
constrains: build/stage1, build/stage2
---

# ADR-0023 — Install by debootstrap from rescue, not the SCP image installer

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

The system was bootstrapped with `debootstrap` from the netcup rescue system
over SSH. The panel's *Official Images* installer cannot produce the required
disk layout, and the ISO-over-VNC route can only produce it badly.

## Context

The starting request was to reinstall away from the netcup Debian default while
choosing partitions and subvolumes at install time. That is not an option the
panel offers.

netcup's own documentation (read 2026-08-30) states the image installer provides
exactly two partition layouts — one large partition with the OS as root, or a
small OS partition leaving the remainder unallocated. There is no filesystem
selection, no subvolume definition, and no encryption option. **The choice of
image is irrelevant to this**: the Arch, Ubuntu and Rocky images in the same
picker are subject to the same installer.

The panel's list is also stale, which is a useful signal about how much to trust
it: it was still offering Debian 11 Bullseye and openSUSE Leap 15.6 as fresh
installs on 2026-08-30. Leap 15.6 reached end of life on 2026-04-30.

Three routes could produce the layout:

| Route | Subvolume control | Determinism | Console typing |
|---|---|---|---|
| Debian netinst via SCP *DVD Drive* | Partial — `partman-btrfs` makes a flat btrfs mounted at subvolid 5, no `@` | Low, GUI clicks | Heavy |
| **Rescue system + `debootstrap`** | Full | High — a shell script | None |
| Arch ISO via *DVD Drive* | Full | Medium | Heavy |

## Decision

Rescue system plus `debootstrap`, driven from arch-thinkpad by
`ssh root@<host> 'bash -s' < script.sh`.

## Rationale

The Debian installer's btrfs support is the disqualifier for the obvious route.
`partman-btrfs` creates a single filesystem and mounts the top level as root; it
does not create `@`/`@home`. Taking that route means installing, then booting
rescue anyway to restructure `/` into `@` — strictly more work than doing it
correctly once.

Between the two remaining, `debootstrap` wins on the property this estate is
organised around: **the install becomes a reviewable artifact rather than a
transcript of VNC clicks.** The scripts were written in nvim on arch-thinkpad,
read before execution, and piped in. They carry explicit preconditions and
postconditions, matching the shape `arch-thinkpad/spec/` already uses. A GUI
install produces no such artifact and cannot be re-run.

The secondary benefit turned out to matter more than expected: **no console
typing.** The SCP console is `en-us` while the operator's keyboard is German, and
the session independently demonstrated three separate ways that pasting into a
remote terminal corrupts the payload. Every one of those would have applied to a
VNC install.

## Alternatives Considered

1. **Debian netinst via DVD Drive** — rejected. Flat btrfs requires a
   post-install restructure from rescue, so the rescue system is in the critical
   path either way; better to use it for the whole job.
2. **Arch ISO via DVD Drive** — rejected. Full control and the operator already
   knows the procedure from arch-thinkpad, but a rolling release on an
   unattended remote host violates the requirement that things fail gracefully
   after weeks of neglect. See [[20260830-debian-over-rhel-family|ADR-0024]].
3. **Accept the panel's "small partition" layout and build btrfs in the free
   space** — rejected. Leaves the OS on a filesystem chosen by the installer,
   which is the thing being avoided.

## Consequences

### Positive

- The install is in [[../build/_index|build/]] and can be re-run against a wiped
  disk. Rebuilding this machine is reading two scripts, not remembering an
  evening.
- Preconditions caught real errors. `debootstrap` on a Grml host defaulting to
  the host's suite would have installed **sid**; the explicit suite argument plus
  a postcondition asserting `trixie` in the target's `os-release` made that
  loud instead of silent.
- Only what was asked for is installed. `--no-install-recommends` throughout,
  and the package list is visible in stage 2.

### Negative

- **The rescue system is now load-bearing** and its quirks are operational
  knowledge rather than trivia. Its root password is separate from the panel's
  root-password reset; it is Grml, tracking sid, with a kernel far newer than the
  target's; it has no `xterm-kitty` terminfo. All of that is in
  [[../runbooks/rescue-system-entry|the rescue runbook]] because it cost time on
  the first pass.
- **Minimalism has a cost that showed up immediately.** `debootstrap` plus
  `--no-install-recommends` omitted `dbus`, so `hostnamectl` and `systemctl
  --user` failed on the first boot — and `systemctl --user` is a prerequisite for
  the rootless podman quadlets this machine exists to run. Fixed the same day;
  the general lesson is that a `--no-install-recommends` base needs an explicit
  check that the *intended workload's* prerequisites are present, since nothing
  will pull them in.
- No `cloud-init`. Network configuration is a static file this vault owns rather
  than something regenerated by the provider. That is an improvement, but it
  means **the panel and the system can disagree** and only the system is
  authoritative.

## Revisit when

netcup's installer gains filesystem and subvolume selection, or a second netcup
machine makes a reusable image worth building.
