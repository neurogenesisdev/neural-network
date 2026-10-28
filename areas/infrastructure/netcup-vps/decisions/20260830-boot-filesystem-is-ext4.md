---
title: /boot is a separate ext4 partition, not a btrfs subvolume
type: decision
id: ADR-0025
aliases: [ADR-0025]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: medium
created: 2026-08-30
updated: 2026-08-30
tags: [decision, boot, btrfs, ext4, grub, recovery]
constrains: build/stage1, build/stage2
---

# ADR-0025 — `/boot` is a separate ext4 partition, not a btrfs subvolume

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

`/dev/vda2`, 1 GiB, ext4, mounted at `/boot`. The kernel, initramfs and
`grub.cfg` live outside btrfs entirely.

## Context

GRUB can read btrfs, and a single-partition layout with `/boot` as a directory
inside `@` is common and works. Choosing otherwise costs a partition and 1 GiB,
so it needs a reason.

The reason is the interaction, not the filesystem. GRUB's btrfs driver has to
resolve the subvolume layout to find the kernel, and that resolution involves
the filesystem's *default subvolume* — a value that can be changed by
`btrfs subvolume set-default`, is changed by `snapper rollback`, and is not
visible in `fstab`. arch-thinkpad has already been bitten by the adjacent
version of this: `snapper rollback` requires `--ambit classic` and an explicitly
set default subvolume, because the stock Arch layout boots `@` directly and
overrides the default — so a rollback silently succeeds and changes nothing.

That class of failure produces a machine that does not boot, with a GRUB rescue
prompt as the only interface. On arch-thinkpad that is an inconvenience with a
recovery ISO and a keyboard. Here it is a VNC console at `en-us` layout on a
German keyboard, over a browser, at whatever hour it happens.

## Decision

A separate 1 GiB ext4 partition for `/boot`. GRUB reads a filesystem with no
subvolumes, no default-subvolume indirection, and no compression.

## Rationale

**One partition of complexity buys a large amount of recoverability.** The
components GRUB must read to reach a kernel are on the simplest, most widely
supported filesystem available, whose driver has no interaction with the
snapshot layer.

It also decouples two things that otherwise move together: the root filesystem
can be restructured, rolled back, or have its default subvolume changed, and none
of it can make the machine unbootable. Given that
[[20260830-subvolume-layout|ADR-0026]] leaves the door open to adding subvolumes
later — and one was in fact added mid-build — that independence is worth having.

**1 GiB rather than the more common 512 MiB** because Debian keeps the previous
kernel and initramfs, `unattended-upgrades` is configured with
`Remove-Unused-Kernel-Packages`, and a full `/boot` breaks an `initramfs` update
in a way that is discovered at the next reboot rather than at the time. The space
is free; the failure is not.

## Alternatives Considered

1. **`/boot` inside `@`** — rejected. Standard and works, and couples the boot
   path to the subvolume layout for no gain on a machine with 256 GiB of disk.
2. **A `@boot` subvolume** — rejected. Worst option: still btrfs, still inside
   the default-subvolume question, and adds a mount unit while providing none of
   the isolation of a separate partition.
3. **512 MiB** — rejected on the kernel-retention argument above.

## Consequences

### Positive

- GRUB never resolves a subvolume. The `rootflags=subvol=@` handoff is the
  kernel's job, and it is visible in `/proc/cmdline`
  (*measured 2026-08-30*: `root=UUID=<FSUUID_vda3> ro rootflags=subvol=@
  net.ifnames=0 console=tty0 quiet`).
- Root filesystem restructuring cannot brick the boot path.
- `/boot` is not compressed and not snapshotted, which is what one wants for a
  directory whose contents must be readable by a bootloader.

### Negative

- **`/boot` is not covered by any btrfs snapshot.** A rollback of `@` restores a
  root filesystem alongside whatever kernel happens to be in `/boot` — they can
  disagree. In practice netcup's hypervisor snapshots cover the whole disk and
  are the primary rollback mechanism here
  ([[20260830-subvolume-layout|ADR-0026]]), so this is less severe than it would
  be on arch-thinkpad. It is still the sharp edge of this decision and the reason
  in-guest snapshots are **not** the rollback story on this machine.
- One more partition and one more fstab line, with `0 2` in the fsck column where
  the btrfs entries are `0 0`.
- Resizing later means moving `vda3`'s start, which is not a live operation.
  1 GiB was chosen so that it will not need to be.

## Revisit when

Nothing plausible. If in-guest snapshot-based rollback ever becomes the primary
recovery mechanism on this host, the `/boot`-outside-snapshots consequence needs
re-examining — but that would mean ADR-0026 had been reversed first.
