---
title: Six subvolumes, for layout hygiene rather than rollback
type: decision
id: ADR-0026
aliases: [ADR-0026]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: medium
created: 2026-08-30
updated: 2026-08-30
tags: [decision, btrfs, subvolumes, snapshots, backups, zram]
constrains: build/stage1, build/stage2, spec/backups
---

# ADR-0026 — Six subvolumes, for layout hygiene rather than rollback

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

`@`, `@home`, `@log`, `@cache`, `@containers`, `@snapshots` on `/dev/vda3`.
The justification is **not** the one that motivates the same layout on
arch-thinkpad: netcup's hypervisor snapshots are the better rollback primitive
here, and these subvolumes exist for exclusion boundaries and space accounting.

## Context

arch-thinkpad uses subvolumes so that snapper can take pre/post-transaction
snapshots and roll back a bad upgrade. Copying that layout to the server without
re-deriving it would import the structure and lose the reason.

The reason does not transfer. netcup takes **offline snapshots at the
hypervisor**, covering the whole disk including `/boot`, exportable off-site and
importable into another netcup server. That is strictly better than an in-guest
snapper rollback for a machine nobody is sitting in front of: it survives a
broken bootloader, a corrupted root, and a filesystem that will not mount — none
of which snapper survives.

So the question became: with rollback already solved better elsewhere, what do
subvolumes still buy?

## Decision

Six subvolumes, mounted with `noatime,compress=zstd:1,discard=async`:

| Subvolume | Mountpoint | In-guest snapshots | In restic |
|---|---|---|---|
| `@` | `/` | Yes | Yes |
| `@home` | `/home` | Yes | Yes |
| `@log` | `/var/log` | No | Yes |
| `@cache` | `/var/cache` | No | No |
| `@containers` | `/var/lib/containers` | No | No — rebuildable from quadlets |
| `@snapshots` | `/.snapshots` | — | **No** |

## Rationale

Two things survive the loss of the rollback argument:

- **Exclusion boundaries.** `/var/lib/containers` and `/var/cache` are large,
  churning, and entirely reconstructible. Having them as separate subvolumes
  means an exclusion is a mount boundary rather than a path pattern that has to
  stay correct as the tree changes.
- **Space accounting.** `btrfs qgroup` can answer "what is consuming the disk"
  per subvolume. On a box checked monthly, that is the difference between a
  diagnosis and a guess.

`@snapshots` is excluded from restic for the recursion reason already established
on arch-thinkpad in
[[../../arch-thinkpad/decisions/20260818-restic-exclusions|ADR-0008]]: backing up
a snapshot directory means backing up copies of everything else.

**Fewer subvolumes than arch-thinkpad, deliberately.** Each one is a mount unit,
an fstab line, and a thing to reason about under pressure. `@srv` and `@var-tmp`
were considered and dropped — nothing on this machine uses `/srv`, and `/var/tmp`
is small and already excluded from backup by pattern.

## Swap: zram, no partition and no swapfile

Recorded here rather than as its own ADR because it is a consequence of the
layout decision.

8 GiB RAM, `zram-generator` with `zram-size = min(ram / 2, 4096)` and zstd.

A btrfs swapfile requires `nodatacow`, cannot be snapshotted, cannot be
compressed, and must not span subvolume boundaries — a set of exceptions carved
into the filesystem for a file that, on a 4-CPU 8 GiB VM running three small
services, will mostly be unused. A swap *partition* would have been a fourth
partition sized by guesswork at install time.

zram is RAM-backed, compresses at roughly 3:1 on typical workloads, needs no disk
at all, and can be resized by editing one config file and rebooting. The
accompanying `vm.swappiness = 180` is not a typo: it is the value appropriate
when swap is faster than the filesystem, which inverts the usual advice.

## Alternatives Considered

1. **Flat, no subvolumes** — rejected. Loses the exclusion boundaries, which are
   the surviving justification.
2. **Full arch-thinkpad parity, including snapper** — rejected. Imports a
   rollback mechanism that is worse than the one netcup already provides, plus a
   package, a timer, and a `.snapshots` directory that would need managing.
3. **Swap partition** — rejected. Sized by guesswork, unresizable without
   repartitioning, on disk for no benefit over zram.

## Consequences

### Positive

- Restic and snapshot exclusions are mount boundaries, not patterns.
- Per-subvolume space accounting.
- Adding a subvolume later is three commands and an fstab line, with no
  repartitioning — demonstrated during this very build, see below.

### Negative

- Six mounts to keep consistent between `fstab` and reality. `findmnt -R /`
  is the check.
- **In-guest rollback is not available and is not meant to be.** Anyone reading
  this layout and expecting snapper semantics will be wrong. `@snapshots` is
  mounted and empty; it exists so that adding snapper later does not require
  restructuring, not because snapshots are being taken.

## The `@cache` addendum, and why it is the useful part of this record

`@cache` was **not** in the original five. It was added after `debootstrap` had
already run, at the operator's prompting — the correct instinct, and the
resulting procedure is the thing worth remembering.

The trap: `debootstrap` had already written apt archives into `/mnt/var/cache`.
Mounting a fresh empty subvolume over that directory **hides** the content rather
than moving it, and the failure is silent — the mount succeeds, the directory
appears empty, and the original data is inaccessible but still consuming space
under the mountpoint.

The order that works:

```bash
# from rescue, with the target mounted at /mnt
mkdir -p /mnt2 && mount -o subvolid=5 /dev/vda3 /mnt2
btrfs subvolume create /mnt2/@cache
umount /mnt2 && rmdir /mnt2

mv /mnt/var/cache /mnt/var/cache.old
mkdir -p /mnt/var/cache
mount -o noatime,compress=zstd:1,discard=async,subvol=@cache /dev/vda3 /mnt/var/cache
cp -a /mnt/var/cache.old/. /mnt/var/cache/     # cp, not mv — crossing a mount boundary
rm -rf /mnt/var/cache.old
```

Two details that are easy to get wrong:

- **The scratch mountpoint must be outside `/mnt`.** Creating `/mnt/btr` and
  mounting the btrfs top level there nests the filesystem inside the system being
  built. `/mnt2` is correct; `/mnt/btr` is not.
- **`cp -a`, not `mv`.** The destination is a different filesystem from the
  operator's perspective — it is the mount just created — so `mv` would be a copy
  followed by a delete anyway, and `cp -a` makes the intent explicit and the
  failure recoverable.

The postcondition that catches a botched job is `ls /mnt/var/cache` returning
`adduser apt debconf ldconfig private` rather than nothing. An empty directory
there is the silent-failure signature.

**The general lesson:** any subvolume added over a path that already holds data
needs a move, not just a mount, and the check is that the content is visible
afterward. This is the vault's recurring defect class — a check that succeeds and
therefore reports nothing — in a new place.

## Revisit when

`btrfs qgroup` shows an unexpected consumer that would benefit from its own
subvolume, or services land that write large reconstructible data outside
`/var/lib/containers`.
