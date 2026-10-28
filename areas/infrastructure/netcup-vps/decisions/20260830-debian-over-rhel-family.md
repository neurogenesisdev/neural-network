---
title: Debian 13 over the RHEL family for the container host
type: decision
id: ADR-0024
aliases: [ADR-0024]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: medium
created: 2026-08-30
updated: 2026-08-30
tags: [decision, distro, debian, podman, selinux, apparmor]
constrains: build/stage2, build/stage3
---

# ADR-0024 — Debian 13 over the RHEL family for the container host

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

Debian 13 (trixie). Rocky/Alma 10 had the better technical case for a rootless
podman host and was rejected on estate-wide cognitive load.

## Context

Since [[20260830-debootstrap-over-image-installer|ADR-0023]] discards the panel
installer, the distribution was genuinely open rather than constrained by what
netcup offers. The operator asked directly why Debian and not something else,
which is the right question and deserved more than inertia as an answer.

The workload is fixed and narrow: rootless podman with quadlets, nftables,
Tailscale, a restic client, and eventually Vaultwarden and Radicale. Neglected
for weeks at a time. Maintained by one person who also runs Arch on the desktop.

| Candidate | For | Against |
|---|---|---|
| **Debian 13** | podman 5.4.x, systemd 257 — quadlets fully supported. ~5yr with LTS. No vendor layer. `debootstrap` is native, so ADR-0023's route is the well-trodden one | Package versions visibly age by year three |
| **Rocky / Alma 10** | podman is Red Hat's project; quadlets, netavark and crun land there first. SELinux confines containers properly. 10-year lifecycle, no `dist-upgrade` ever | SELinux instead of AppArmor. `dnf --installroot` bootstrap is far less trodden |
| **Ubuntu 26.04 LTS** | Newer podman and kernel | snapd, Pro nags, telemetry. First hour spent removing things Debian never installs. No upside for a container host |
| **openSUSE Leap** | btrfs + snapper natively, which is literally the requested layout | 15.6 is EOL (2026-04-30) and still offered by netcup; Leap 16 point releases carry 24 months. A third packaging ecosystem for a box touched monthly |
| **Arch** | Toolchain parity with arch-thinkpad | Rolling release on an unattended remote host. `linux`, `systemd` and `podman` all moving between visits |
| **FreeBSD** | — | No podman |

## Decision

Debian 13.

## Rationale

The real fork was Debian versus Rocky/Alma, and **the RHEL case is stronger than
it is usually given credit for.** Podman is Red Hat's; the features this machine
will lean on are tested there first. SELinux with proper container labelling is a
materially better confinement story than AppArmor for containers. A decade of
support means never performing a release upgrade on this host.

It was rejected on two grounds specific to this operator:

1. **Cognitive load across the estate.** One person maintains an Arch laptop and
   this server. AppArmor knowledge transfers to Debian and evaporates on Rocky —
   including hard-won specifics like `apparmor.d` silently reverting enforced
   profiles after an upgrade. Debugging SELinux denials at 11pm after six weeks
   away, on the only RHEL machine in the estate, is precisely the failure mode
   the stated priority ordering exists to prevent. Operational simplicity ranks
   above elegance.
2. **The bootstrap path.** `debootstrap` into a prepared btrfs subvolume tree is
   well documented. `dnf --installroot` works but would have been improvised —
   inside a rescue system, on a box with no OS, which is where install procedures
   fail.

The SELinux advantage is real but this host runs three services the operator
controls, rootless, behind Tailscale, with nothing publicly exposed. The marginal
confinement gain does not pay the ecosystem tax.

## Alternatives Considered

Covered in the table above. The two worth restating as genuine rejections rather
than dismissals:

1. **Rocky/Alma 10** — see Rationale. This would be the right answer for an
   operator already running RHEL-family elsewhere, or for a host running
   workloads the operator does not fully trust.
2. **Arch** — attractive for parity and rejected on release model. Worth noting
   that the *rescue* system is Grml (sid-based) and its kernel was five major
   versions ahead of the target's, which is a small live demonstration of what
   rolling would mean here.

## Consequences

### Positive

- One security-update mechanism, one packaging ecosystem, one MAC framework
  across the estate.
- ~5 years before a release upgrade is forced.
- Debian's podman ships **no unqualified search registries by default**, so image
  references must be fully qualified. Mildly annoying in practice and a security
  property in principle — it aligns with the digest-pinning discipline in
  [[../../arch-thinkpad/decisions/20260825-devbox-pinning-four-layers|ADR-0016]]
  rather than fighting it.

### Negative

- Package versions age. By 2028 podman here will be well behind upstream, and
  quadlet features documented on podman's site may not exist. The mitigation is
  backports, which reintroduces a version-mixing question this record does not
  answer.
- **AppArmor is not configured on this machine at all.** Debian ships the
  framework; nothing here enforces a profile. The RHEL alternative would have
  arrived with SELinux enforcing by default. That is a real security delta and it
  is currently unpaid — filed in
  [[../backlog/apparmor-baseline|the backlog]] rather than pretended away.
- A future consolidation onto RHEL-family would mean rebuilding this host.

## Revisit when

The estate consolidates on one non-Debian family, or this machine is asked to run
workloads the operator does not control — at which point SELinux's confinement
advantage starts paying for its learning cost.
