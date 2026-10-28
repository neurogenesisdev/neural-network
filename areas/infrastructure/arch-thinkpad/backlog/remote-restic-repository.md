---
type: backlog-item
status: resolved
priority: high
device: arch-thinkpad
projects: [backups]
tags: [restic, backups, remote, b2, hetzner, netcup]
created: 2026-08-21T00:00:00Z
updated: 2026-08-30T00:00:00Z
resolved: 2026-08-30
commit: pending
---

> [!SUCCESS] Resolved 2026-08-30 by [[../decisions/20260830-remote-repository-is-b2|ADR-0028]]
> **Backblaze B2**, via restic's S3 backend, with a bucket-scoped application key.
> Repository initialised and the first snapshot taken the same evening: 20.015 GiB
> processed, **12.809 GiB stored**, 249,561 files, 1h18m. Cost at that size is about
> $0.03/month — the first 10 GB are free.
> [[../decisions/20260818-backup-cadence-open|ADR-0012]] is decided with it (daily
> timer, option D), and [[../spec/backups|spec/backups]] is the chapter.
>
> **Two of this item's three options had stopped existing, and it did not know.** The
> Hetzner VPS was gone, which the item records. The Storage Box it calls "already
> owned … already reachable over SFTP" is **also gone** — established by the operator
> on 2026-08-30, nine days after this text was written. The `u531077` line below was
> `inherited-unverified` and read as fact for the whole of that window, while being
> the load-bearing input to the one decision this item blocked.
>
> **The Borg paragraph turned out to be the least useful thing here**, and the
> netcup products it never considered turned out to be the real comparison. ADR-0028
> covers both netcup options and rejects them on grounds — NFS-only and roaming
> clients for Storage Space, provider correlation for block storage — that this item
> had no reason to anticipate.
>
> **Still open, and named in ADR-0028's *Revisit when*:** no restore has been
> performed, and the timer has not been observed firing.

# Remote restic repository

> [!NOTE] Summary
> The only off-machine backup target is the Samsung T5, which is not always attached. Nothing survives the laptop and the drive being lost together. PRD §9.4 specified a remote repo on the Hetzner VPS; that VPS has been offline since July.

## Status

**Blocked** on the VPS, or on choosing a different target.

## Options

- **Backblaze B2** — already used for the hive project. Use a **separate bucket and a separate application key** scoped to that bucket, so a laptop compromise cannot reach the server's backups.
- **Hetzner Storage Box** — already owned (`u531077`, 1 TB, BX11), already reachable over SFTP. restic supports SFTP natively.
- **Resume the VPS** — restores the original plan.

## Note

Borg is an alternative *program*, not a remote for restic. Switching would abandon the existing repository, its snapshots and the exclusion tuning, and Borg has no native B2 support. No reason to migrate.

## Related

- [[20260818-backup-cadence-open|ADR-0012]] — cadence, undecided
- [[20260818-restic-exclusions|ADR-0008]] — depends on cadence
