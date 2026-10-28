---
type: backlog-item
status: resolved
priority: low
device: arch-thinkpad
projects: [storage]
tags: [ntfs, seagate, filesystem, archive]
created: 2026-08-21T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# Move the 3.6 TB Seagate off NTFS

> [!NOTE] Summary
> The external Seagate holds an older partial archive on NTFS. NTFS cannot be fully repaired from Linux — `ntfsfix` clears flags and journal state only; `chkdsk` needs Windows, which no longer exists on this machine. Mounting currently requires `-o force` after a dirty flag.

> [!INFO] Made accessible 2026-09-02 — `measured`; the migration itself is still open
> **The barrier was not NTFS.** The drive (Seagate Expansion, `0bc2:231a`, 3.6 TB,
> label "Hive Colony") was **blocked by usbguard** — never plugged in since the
> policy was generated — so it did not appear as a block device at all. Allowed
> permanently (`usbguard allow-device -p`), rule now in `rules.conf`. After that:
> the volume was dirty and udisks refused it (`volume is dirty and "force" flag is
> not set`); `ntfsfix -d /dev/sdb2` cleared the flag, and `udisksctl mount` as the
> operator mounts it with `ntfs3`, `uid=1000`, read-write — reads and writes in
> `Media/` verified. Dolphin will do the same on the next plug-in. Contents: 1.3 TB
> `Media`, 194 GB in `$RECYCLE.BIN`, 108 GB `Proton Drive`, 62 GB `Backup`; 1.7 of
> 3.7 TB used. The dirty flag will return after any unclean unplug; `ntfsfix -d`
> is the reset and it repairs nothing — the case for moving off NTFS stands.

> [!SUCCESS] Closed 2026-09-03 — decision: stay on NTFS
> The drive works read-write under the kernel `ntfs3` driver, there is no Windows
> left to share it with, and reformatting 1.7 TB buys only the ability to run
> `fsck` on it. The residual is the dirty flag after an unclean unplug —
> `ntfsfix -d` resets it, and ejecting first avoids it. The 194 GB in
> `$RECYCLE.BIN` is the operator's to empty.

## Steps

1. Verify the archive contents and what is unique to this drive.
2. Copy elsewhere (T5 or Storage Box) before reformatting.
3. Reformat as ext4 or btrfs.
4. Copy back; verify.

## Note

`ntfsprogs` provides `ntfsfix`; `ntfs-3g` alone does not. Both are installed.
