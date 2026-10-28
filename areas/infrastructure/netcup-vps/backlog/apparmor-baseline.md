---
type: backlog-item
status: active
priority: medium
device: netcup-vps
projects: [netcup-build]
tags: [backlog, security, apparmor, containers, podman]
created: 2026-08-30
updated: 2026-08-30
commit: pending
---

# apparmor-baseline

> [!NOTE] Summary
> Debian ships AppArmor and nothing on this machine enforces a profile. The RHEL
> alternative rejected in [[../decisions/20260830-debian-over-rhel-family|ADR-0024]]
> would have arrived with SELinux enforcing container labels by default, so
> choosing Debian took on a confinement debt that is currently unpaid.

## Context

ADR-0024 rejected Rocky/Alma partly on the grounds that "the marginal confinement
gain doesn't pay the ecosystem tax" for three self-controlled services behind
Tailscale. That argument is defensible and it is **not** an argument for zero
confinement — it is an argument that AppArmor's level is sufficient. AppArmor at
zero profiles is a different proposition from AppArmor configured.

The gap matters most for the containers this machine exists to run. Podman applies
a default AppArmor profile to containers when the framework is active; whether it
is active here has not been checked.

## Steps

1. Establish the baseline. **`inferred` until run:**

   ```bash
   sudo aa-status
   systemctl is-enabled apparmor
   podman info --format '{{.Host.Security.AppArmorEnabled}}'
   ```

2. If AppArmor is not enabled, decide whether to enable it. On Debian this is
   ordinarily active by default; a `debootstrap` minimal base may not have pulled
   in `apparmor` at all.
3. Confirm podman applies `containers-default-*` to rootless containers, and that
   the quadlets do not disable it.
4. Consider profiles for the host-side services with network exposure. Not
   `apparmor.d` — see the note.

## Blocked on

Not blocked, but most meaningful after the services exist. Confining a machine
that runs sshd and nothing else is a small win.

## Note

**Do not install `apparmor.d` here.** That is the arch-thinkpad arrangement, and
it carries a known defect recorded in that device's notes: package updates
silently revert enforced profiles, requiring a post-upgrade re-enforce script.
Importing a mechanism whose failure mode is *silent reversion* onto a machine
checked monthly is worse than having no profiles, because it produces a
documentation claim that will drift from reality with nothing to notice.

Debian's own packaged profiles plus podman's container default are the right
scope. **Whatever is enabled must be checkable by the collector**
([[netcup-collector]]) — a confinement claim in this vault with nothing measuring
it is exactly the failure the root `CLAUDE.md` exists to prevent.

**The honest framing:** this is a debt taken on deliberately by ADR-0024, not an
oversight. It should be either paid or explicitly written off with an amendment to
that record. Leaving it as an open backlog item indefinitely means ADR-0024's
consequences section describes a mitigation nobody performed.

## Related

- [[../decisions/20260830-debian-over-rhel-family|ADR-0024]] — where the debt was taken on
- [[netcup-collector]] — nothing can currently verify a profile is enforced
- [[../../arch-thinkpad/backlog/apparmor-enforce-stage-2|arch-thinkpad/apparmor-enforce-stage-2]] — the same gap on the laptop, at larger scale
