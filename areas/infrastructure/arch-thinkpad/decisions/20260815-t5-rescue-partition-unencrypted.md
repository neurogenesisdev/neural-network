---
title: The T5 rescue partition is unencrypted by design
type: decision
id: ADR-0009
aliases: [ADR-0009]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: high
created: 2026-08-15
tags: [decision, backups, encryption, recovery, veracrypt]
constrains: spec/backups
---

# ADR-0009 — The T5 rescue partition is unencrypted by design

- **Status:** accepted
- **Date:** 2026-08-15
- **Provenance:** measured

## Context

The Samsung T5 (931.5 GB) holds both full-disk images and the restic
repository. Encrypting the whole drive would mean that recovering from a dead
machine requires the tooling to open the container — which requires a working
system, which is the thing that is unavailable.

## Decision

Two partitions:

| Partition | Size | Format | Contents |
|---|---|---|---|
| `sda1` | 465 GB | plain ext4, label `rescue` | Full-disk images **only** |
| `sda2` | 466.5 GB | VeraCrypt → ext4 | restic repo, `archive/`, `backups/`, `recovery/` |

## Rationale

`sda1` is readable from bare rescue media with no network and no extra tooling.
The images it holds are of a LUKS-encrypted disk and are opaque without the
root passphrase, so leaving the partition unencrypted discloses nothing.

The Arch ISO does not ship VeraCrypt. Restoring from `sda2` therefore requires
installing VeraCrypt first, which requires network. That dependency is
acceptable for the restic repo — a data restore is not time-critical and
implies a working system — but unacceptable for boot-chain artifacts, which is
why those live on `sda1` and on the Ventoy stick.

## Consequences

- **Nothing else goes on `sda1`.** A stray document there is unencrypted
  personal data on a portable drive. This is the rule the partition split
  depends on.
- The restic repo path `/run/media/veracrypt1/backups/restic/arch-thinkpad` is a
  VeraCrypt mapping, unrelated to udiskie's automounting at
  `/run/media/luiswg/<label>`.
- Since 2026-08-19 udiskie auto-mounts plaintext partitions at login, so `sda1`
  appears at `/run/media/luiswg/rescue` without being asked for. The assumption
  "nothing is mounted unless I mounted it" no longer holds, and `df` before a
  backup will show entries that were not there before.

  **Amended 2026-08-28.** `udiskie` is uninstalled and `udiskie.service` is
  disabled — the desktop moved to KDE Plasma
  ([[20260828-plasma-replaces-hyprland-userland|ADR-0018]]) and removable media is
  Plasma's now. The rescue-partition decision is untouched; what changed is **who
  mounts `sda1` and when**, which was the operative detail in both bullets above.
  The mount point is also Plasma's to choose and is no longer assumed to be
  `/run/media/luiswg/<label>`.

  Whether Plasma automounts on attach or waits for a click is **`inferred`** — not
  checked, and the two differ exactly where this bullet cares. Until it is, keep
  the conservative reading: assume something may be mounted that you did not
  mount, and run `df` before a backup rather than relying on either behaviour.
- The LUKS header backup is GPG-encrypted (AES256, passphrase distinct from the
  LUKS one) and stored on the T5. Distributed to the Pixel and Proton Drive via
  Syncthing, and to the Ventoy stick's `tools/`.
