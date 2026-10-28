---
type: backlog-item
status: resolved
priority: high
device: netcup-vps
projects: [backups]
tags: [backlog, backups, restic, storage-box, credentials]
created: 2026-08-30
updated: 2026-09-11
commit: pending
---

# restic-client

> [!NOTE] Summary
> **Resolved 2026-09-11.** Daily restic → B2 (`netcup-vps-restic-k7q2`, S3 endpoint), `LoadCredential=` secrets, monthly `check`, daily staleness check with ntfy `OnFailure=`, restore verified from the laptop, retention from the laptop. The VPS key can hide but not destroy; 30-day prior versions are the recovery window. Procedure: [[../runbooks/restic-b2-backup|restic-b2-backup]].
>
> Nothing on this machine is backed up. netcup's hypervisor snapshots cover the
> whole disk and are the rollback mechanism
> ([[../decisions/20260830-subvolume-layout|ADR-0026]]), but they live at the same
> provider as the machine and one exportable slot is not a backup strategy. This
> is the second restic client against the Hetzner Storage Box.

## Context

The Storage Box (BX11) work was already scoped for arch-thinkpad — a dedicated
sub-account over SFTP, with Backblaze B2 as an optional second remote for provider
diversity. This machine is a second client of the same target, with its own
sub-account and its own repository.

Right now the only thing standing between a netcup-side failure and total loss is
one exportable snapshot slot at netcup.

## Steps

1. Create a Storage Box sub-account for this host, separate from the laptop's.
2. Generate an SSH key on the VPS for the Storage Box **only**. This is a
   downward credential and is permitted under
   [[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]] regardless
   of how that record resolves.
3. Initialise the repository over SFTP.
4. Write the exclusion set. The subvolume layout was chosen partly to make this a
   mount-boundary question rather than a pattern-matching one:
   - `/var/lib/containers` — rebuildable from quadlets
   - `/var/cache` — rebuildable
   - `/.snapshots` — recursion, per
     [[../../arch-thinkpad/decisions/20260818-restic-exclusions|ADR-0008]]
   - `/var/log` — **included**, unlike the two above
5. systemd timer, and a `podman secret` or `LoadCredential=` for the repository
   password — **not** an environment file.
6. Restore test. An untested backup is a hypothesis.

## Blocked on

Not blocked. Reasonable to do before the services exist, so that the machinery is
proven on a system where losing the test data costs nothing.

## Note

**The repository password is the single most important secret on this machine**,
and [[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]] means it is
sitting on an unencrypted disk. A plaintext `RESTIC_PASSWORD` in a systemd unit
file or an environment file is exactly the pattern the decommissioned hive used
for `VAULTWARDEN_ADMIN_TOKEN`, and exactly what ADR-0021's negative consequences
forbid. Use `LoadCredential=` with a root-owned file, or `podman secret` if the
job is containerised.

**A backup is not a rollback and the two are being covered by different
mechanisms here.** netcup snapshots give whole-disk rollback at the same
provider; restic gives off-provider recovery of data. Neither substitutes for the
other, and the ADR-0026 reasoning depends on both existing. Right now only the
first does, which means **ADR-0026's rationale is currently only half true** —
worth noting in that record if this item stays open long.

**Restore must be tested from something other than this machine.** A restore
procedure that requires the host being restored is not a procedure.

## Related

- [[../../arch-thinkpad/backlog/remote-restic-repository|remote-restic-repository]] — the laptop's half, and the Storage Box decision
- [[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]] — why the repository password needs care
- [[../decisions/20260830-subvolume-layout|ADR-0026]] — the exclusion boundaries
