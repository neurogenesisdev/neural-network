---
title: BIOS boot retained; the stock image's ESP was dropped
type: decision
id: ADR-0022
aliases: [ADR-0022]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: medium
created: 2026-08-30
updated: 2026-08-30
tags: [decision, boot, grub, bios, uefi, secure-boot]
constrains: build/stage1, build/stage2
---

# ADR-0022 — BIOS boot retained; the stock image's ESP was dropped

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

The machine boots legacy BIOS with GRUB in a 1 MiB `ef02` partition. The stock
netcup image's 244 MiB EFI System Partition was not carried forward.

## Context

The pre-existing netcup Debian image presented a contradiction, measured
2026-08-30 before the wipe:

- `/dev/vda2` was a 244 MiB vfat partition mounted at `/boot/efi`
- `/dev/vda1` was a 1 MiB partition with no filesystem signature — a `bios_grub`
  partition
- `/sys/firmware/efi` **did not exist**

So the image was built dual-capable and the firmware booted it in legacy mode.
The ESP was present, populated, and never read. The SCP's *Settings → UEFI Boot*
toggle was `off`, confirming the same thing from the other side.

Since the disk was being repartitioned anyway, both modes were genuinely open.

## Decision

BIOS. GPT with a 1 MiB `ef02` BIOS-boot partition, `grub-pc`, `grub-install
--target=i386-pc /dev/vda`. No ESP.

## Rationale

UEFI's substantive advantage over BIOS on this class of machine is Secure Boot,
and Secure Boot is not worth having here for the same reason LUKS is not — see
[[20260830-no-luks-on-always-online-vps|ADR-0021]]. A verified boot chain
defends against an attacker who can modify the boot artifacts but not observe
runtime memory. The hypervisor can do both. Verifying the chain against a host
that can read the running kernel's memory is theatre.

That removes the only reason to prefer UEFI, and leaves the tiebreak to
operational simplicity: legacy GRUB on a VM is one fewer moving part, the
firmware is *demonstrably* booting it today, and there is no ESP to keep signed,
mounted, or in sync.

**This is the opposite conclusion to arch-thinkpad, and correctly so.** That
machine runs Secure Boot with its own keys, sbctl-signed UKIs and TPM2
enrollment, because it is a laptop that leaves the house and whose disk can be
stolen while powered off. Neither property holds here.

## Alternatives Considered

1. **Switch to UEFI via the SCP toggle** — rejected. Buys Secure Boot, which is
   inert here; costs an ESP to maintain and a firmware mode change on a machine
   whose current boot path is known-good.
2. **Keep the ESP unused, boot BIOS** — rejected. 244 MiB of vfat that nothing
   reads is a partition that will confuse whoever next looks at `lsblk`, and it
   invites a future `grub-install --target=x86_64-efi` that half-works.

## Consequences

### Positive

- One bootloader target, one install command, no signing step.
- 244 MiB reclaimed, and more usefully, one fewer partition to explain.
- No `sbctl`, no key enrollment, no re-signing after a kernel update. The whole
  [[../../arch-thinkpad/runbooks/bootloader-reinstall|bootloader-reinstall]]
  failure class does not exist on this machine.

### Negative

- **Switching to UEFI later means repartitioning**, since there is no ESP and no
  free space reserved for one. `vda3` starts immediately after `/boot`. This is
  the expensive-to-reverse half of the decision and is the reason it was settled
  before `sgdisk` ran.
- No Secure Boot, and therefore no defence against modification of the boot
  chain by anything that already has root. Accepted: anything with root on this
  box has already won.
- **The SCP's UEFI Boot toggle must stay off.** Flipping it on a machine with no
  ESP produces a server that will not boot, recoverable only by flipping it back.
  Cheap to reverse, but the symptom — a machine that stops booting after a panel
  change nobody connects to the disk layout — is exactly the kind that costs an
  evening.

## Revisit when

Never, realistically. If it is revisited, the trigger is a requirement for
measured boot that the hypervisor threat model does not already defeat — which
would mean netcup had started offering confidential computing.
