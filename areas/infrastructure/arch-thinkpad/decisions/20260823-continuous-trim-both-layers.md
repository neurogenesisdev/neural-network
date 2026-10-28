---
title: "Continuous TRIM is granted at both layers"
type: decision
id: ADR-0014
aliases: [ADR-0014]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-23
commit: pending
impact: high
created: 2026-08-23
updated: 2026-08-23
tags: [decision, luks, btrfs, trim, discard, privacy, ssd]
constrains: spec/disk
---

# ADR-0014 — Continuous TRIM is granted at both layers

- **Status:** accepted
- **Date:** 2026-08-23
- **Provenance:** measured (the cmdline, the mount options, the LUKS header
  flags, the enabled timer, the kernel version, both man page quotes), inferred
  (that `fstrim` cannot work without the dm-crypt grant; that the coupling
  between the two layers behaves as described)

## Summary

Keep the unprefixed `rd.luks.options=discard` and leave btrfs at its default
`discard=async`, **accepting the documented metadata leak** — because the
alternative the PRD proposed, weekly batch TRIM, discloses the same information
by the same mechanism *and* cannot run without the very grant the PRD refuses.

## Context

This record exists because the machine has been doing something the PRD forbids,
in two places, and nobody wrote down why.

**What the PRD decided.** §1.2: "this document does not pass `--allow-discards`
at format time and does not include `rd.luks.options=discard` in the cmdline.
Online TRIM through dm-crypt leaks block-allocation patterns to the SSD
controller. Combined with omitting `discard=async` at the btrfs layer (§1.4),
this is a deliberate privacy-leaning trade; SSD wear is managed by weekly batch
TRIM via `fstrim.timer` instead." §1.4 repeats it from the other side, listing
`discard=async` among the options "omitted deliberately — using `fstrim.timer`
instead (pick one, not both)."

**What the machine does.** All measured, collector run 2026-08-23:

| Layer | State | Where |
|---|---|---|
| dm-crypt | `rd.luks.options=discard`, unprefixed | `state/boot.md`, `## cmdline (configured)` and the signed `.cmdline` of all four UKIs |
| LUKS header | `Flags: (no flags)` | `state/storage.md`, `## luks` |
| btrfs | `discard=async` on every one of the six mounts | `state/storage.md`, `## mounts` |
| batch | `fstrim.timer` enabled | `state/services.md`, `## system enabled` |

So both layers discard continuously, the weekly timer runs as well — the "both"
§1.4 warned against — and the LUKS header carries no persistent flag, meaning
the grant is per-activation and its **sole source is one token on the kernel
command line**.

**The two halves diverged from the PRD for different reasons, and only one of
them is a reversal.**

*The btrfs half was overtaken.* `btrfs(5)` on `discard`: "(default: async when
devices support it since 6.2, async support since: 5.6)" (*measured*, read
2026-08-23). This machine runs 7.1.7 (*measured*, `state/boot.md`,
`## kernels`). Omitting the option stopped meaning "off" at kernel 6.2, and
§1.4's deliberate omission quietly became a deliberate nothing. Turning it off
now requires writing `nodiscard`, which the PRD never contemplated because it did
not have to. **No decision was reversed here; a decision stopped being
expressible by omission and nobody noticed.**

*The dm-crypt half was reversed in the act and never in writing.* Somebody typed
`discard` into `/etc/kernel/cmdline`, it was baked into four UKIs and signed, and
the reasoning is not recorded anywhere in this vault. That is the gap this record
closes.

**The leak itself is not overtaken.** `cryptsetup-open(8)` on `--allow-discards`
still carries it verbatim (*measured*, read 2026-08-23):

> **WARNING:** This command can have a negative security impact because it can
> make filesystem-level operations visible on the physical device. For example,
> information leaking filesystem type, used space, etc., may be extractable from
> the physical device if the discarded blocks can be located later. If in doubt,
> do not use it.

That is a real cost and this record accepts it rather than arguing it away. What
does not survive scrutiny is the PRD's *mitigation*.

**The PRD's alternative is not implementable as written.** `--allow-discards`
means "Allow the use of discard (TRIM) requests for the device" — so without the
grant, discard requests are not allowed, and `fstrim.timer`, whose entire job is
to issue them, has nothing to issue them through (`inferred`: the option's
semantics are documented, but nobody has run `fstrim` on this machine with the
grant absent). §1.2 refuses the grant and then names `fstrim.timer` as the
replacement. The configuration it describes therefore yields **no TRIM at all**
on a QLC drive, which is not what it claims to yield.

And the privacy argument does not distinguish the two in the way §1.2 implies.
Batch TRIM discards the same blocks through the same dm-crypt grant; the
difference between weekly and continuous is *when* the allocation map is
disclosed, not *whether*. The only configuration that actually honours §1.2's
reasoning is no TRIM at any cadence.

## Decision

Grant discards at the dm-crypt layer through the unprefixed
`rd.luks.options=discard` on the kernel command line, and leave btrfs at its
kernel default. Do not persist the flag into the LUKS header. Retain
`fstrim.timer`.

## Alternatives considered

**Reverse both — drop the cmdline token and mount `nodiscard`.** Rejected, and
it is the only alternative that honours the original reasoning. Its cost is no
TRIM at all on a QLC drive with 36 GiB of swapfile on it, since batch TRIM dies
with the grant. It is also not a mount-option edit: the token lives inside four
signed UKIs, so reversing means `cmdline-write`, `uki-generate` and `uki-sign`,
in that order. Worth knowing before anyone reaches for it as the "safe" default.

**Keep the grant, disable btrfs's continuous discard with `nodiscard`, rely on
the weekly timer.** Rejected. This is §1.4's "pick one" resolved toward batch,
and it buys no privacy at all for the reason above — the disclosure is the same,
one burst a week instead of steadily. It trades the kernel's default, which is
tuned to avoid latency spikes, for a configuration that has to be written down
and maintained against a default that already moved once.

**Persist the flag with `cryptsetup --persistent`.** Rejected. It would make the
grant survive independently of the cmdline, which sounds like robustness and is
the opposite of what this machine wants: the cmdline is the thing under Secure
Boot measurement, and a header flag would grant discards to *any* activation,
including a recovery boot from foreign media. `Flags: (no flags)` stays.

## The coupling, which is the part worth remembering

**Two layers are reached by one token on a kernel command line inside a signed
UKI.** dm-crypt advertises discard support only when it was activated with the
grant; btrfs enables its default async discard only "when devices support it".
So if the unprefixed `rd.luks.options=discard` is ever lost, dm-crypt stops
advertising, btrfs's *default* silently declines to apply, and `fstrim` fails as
well — three effects, one cause, no error message (`inferred`: both behaviours
are documented, the interaction has not been observed here).

Losing it is not hypothetical. PRD §12.2 instructs adding
`rd.luks.options=<UUID>=tpm2-device=auto`, and a UUID-prefixed list "will
override the options for the given UUID" while an unprefixed one applies "to any
UUIDs not specified elsewhere" (`systemd-cryptsetup-generator(8)`, see
[[areas/infrastructure/arch-thinkpad/state-notes|notes.md]]). Following §12.2 removes discard from all three
layers by adding an unrelated option. The correct form carries both:
`rd.luks.options=<UUID>=discard,tpm2-device=auto`.

The enforcement point is
[[../spec/boot#cmdline-write|`cmdline-write`'s negative postcondition]], which
fails the build if a UUID-prefixed `rd.luks.options` appears. That assertion was
written before this record existed and is what makes this decision hold.

`fstrim.timer` is retained partly for this: with continuous discard it has little
left to do, and its remaining value is diagnostic — it is the unit that would
begin failing if the grant went away (`inferred`; the failure has not been
observed).

## Consequences

### Positive

- TRIM works at all, which the configuration the PRD describes would not have
  achieved.
- btrfs sits on its documented default, so there is no local setting to drift
  against a kernel that has already changed this once.
- The controller on a QLC drive gets accurate free-space information
  continuously rather than in weekly bursts.

### Negative

- **The documented leak is accepted.** Filesystem type and used space may be
  extractable from the physical device by an adversary who can locate discarded
  blocks. Bounded to metadata, not content: an attacker holding the powered-off
  disk still faces LUKS2, and the TPM2 unlock is bound to PCR 0+2+7 with a PIN
  (ADR-0013). What they gain is the shape of the used space (`inferred` — this
  is reasoning about the quoted warning, not an observed extraction).
- The threat this most helps is the one nobody has modelled here: an adversary
  with **repeated** raw-disk access over time, for whom the deltas in the
  allocation map are more informative than any single snapshot.
- Reversing this decision costs a UKI regeneration and re-sign, not a config
  edit, because the token is inside signed images with `editor no` in
  `loader.conf`.
- Both layers depend on one token, and its loss is silent. See the coupling
  above.

## Revisit when

- The NVMe drive is replaced with a TLC or pSLC part, which weakens the
  endurance side of the trade. Current wear does not argue for continuous TRIM
  on its own: `percentage_used: 0%`, `available_spare: 100%`, `media_errors: 0`
  (*measured*, `state/storage.md`, `## nvme health`).
- The threat model gains an adversary with repeated physical access to the
  powered-off disk.
- `btrfs(5)` changes the default again, in either direction — the reason the
  btrfs half of this record exists is that it changed once already.
