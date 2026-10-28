---
title: "TPM2 enrollment binds PCRs 0+2+7"
type: decision
id: ADR-0013
aliases: [ADR-0013]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-23
commit: pending
impact: high
created: 2026-08-23
updated: 2026-08-23
tags: [decision, tpm2, secure-boot, luks, firmware]
constrains: spec/boot
---

# ADR-0013 — TPM2 enrollment binds PCRs 0+2+7

- **Status:** accepted
- **Date:** 2026-08-23
- **Provenance:** measured (SPI write protection, the current enrollment, PCR 2's
  value, PCR semantics), inferred (that PCR 2 is stable across peripheral
  attachment — untested)

## Summary

The LUKS TPM2 enrollment binds PCRs **0+2+7** with a PIN, accepting an
invalidated enrollment on every firmware update, because firmware modification
on this machine is reachable from a root shell rather than requiring physical
access.

> **Applied 2026-08-23.** The machine is enrolled against `[0,2,7]` with a PIN
> (measured — LUKS2 header, collector run 2026-08-23), and the enrollment
> survived a reboot with the PIN prompt returning. The spec's
> [[../spec/boot#tpm2-enroll|`tpm2-enroll`]] postcondition passes. The procedure
> used was [[../runbooks/tpm2-reenroll|runbooks/tpm2-reenroll]] — the same one
> every firmware update will trigger from now on.
>
> *Amended 2026-08-23: replaces a "decided, not yet applied" callout. The
> Decision and Rationale below are unchanged.*

## Context

The build spec cannot write `--tpm2-pcrs=` without choosing, and the choice is a
security trade rather than a detail.

**The deciding measurement.** `fwupdmgr security` reports
`org.fwupd.hsi.Amd.SpiWriteProtection: Disabled`, flagged `action-contact-oem`.
The SPI flash holding system firmware is not write-protected. Modifying firmware
on this machine therefore does not require a flash programmer, chassis access,
or physical possession — it is reachable from a root shell. `action-contact-oem`
means the remedy is Lenovo's, on a 2020 machine they are unlikely to revisit, so
this is a durable property of the hardware rather than a backlog item.

That distinction is the whole decision. It is not "PCR 0 until SPI write
protection is fixed"; it is "PCR 0 for the life of this machine."

What each PCR measures, quoted from `systemd-cryptenroll(1)` (read 2026-08-23):

| PCR | Name | Measures |
|-----|------|----------|
| 0 | `platform-code` | "Core system firmware executable code; changes on firmware updates" |
| 2 | `external-code` | "Extended or pluggable executable code; includes option ROMs on pluggable hardware" |
| 7 | `secure-boot-policy` | "Secure Boot state; changes when UEFI SecureBoot mode is enabled/disabled, or firmware certificates (PK, KEK, db, dbx, ...) changes." |

**Prior state.** The machine was enrolled against PCR 7 alone with a PIN
(measured, collector run 2026-08-23: `tpm2-pcrs: [7]`, `tpm2-pin: true`,
`tpm2-pcr-bank: "sha256"`). PRD §12.2 specified `0+2+7` but was filed under
"Deferred" and never applied as written (`inherited-unverified`).

**The systemd default is no PCRs at all.** `systemd-cryptenroll(1)`: binding to
no PCRs "is also the default, if this option is not used." An unbound enrollment
and a bound one are indistinguishable by the presence of a token, which is why
the spec's postcondition reads the PCR mask from the LUKS2 header rather than
checking that a token exists.

## Decision

Enroll with `--tpm2-pcrs=0+2+7 --tpm2-with-pin=yes`.

## Alternatives considered

**PCR 7 only — the prior state.** Rejected. PCR 7 covers Secure Boot state and
the certificate databases, so it defeats an attacker who disables Secure Boot or
enrols their own keys. It does not cover the firmware itself. Given SPI write
protection is disabled, an attacker with root can modify firmware while leaving
the enrolled keys intact — PCR 7 does not move, and the TPM releases the key to
a compromised boot chain. The earlier framing of PCR 0 as protecting against an
"evil maid with a flash programmer" set the bar too high and quietly argued for
this option; the real threat needs no hardware at all.

**PCR 0+7, dropping 2.** Rejected on evidence. The reasoning for dropping PCR 2
was that a laptop with no discrete GPU has near-constant `external-code` — but
that premise is false. This machine has integrated Radeon graphics, and
`systemd-analyze pcrs` reports PCR 2 (`external-code`) as **non-empty**:
`ec3a64e9…` (measured 2026-08-23). Something extends it. What, is not
established.

## The frequency cost, stated honestly

Binding PCR 0 means every firmware update invalidates the enrollment. This
machine takes Lenovo updates through fwupd, so that is a recurring, expected
event. It presents as a passphrase prompt at an unexpected boot — correct
behaviour that will still be surprising the first time. See the re-enrollment
runbook; the response is documented rather than rediscovered at that boot.

PCR 7 alone was never invalidation-free either. It measures `dbx`, the
revocation list, and fwupd delivers dbx updates — so the prior configuration
also dropped to a passphrase prompt after some firmware-adjacent updates. The
change is in frequency, not in kind.

## Consequences

### Positive

- Firmware modification, which is software-reachable on this hardware, now
  invalidates the TPM unlock rather than being transparent to it.
- The spec can encode a decided value and point here.

### Negative

- Every firmware update requires re-enrollment. This is now a routine
  maintenance step, not an exception.
- The passphrase must stay memorised, and stay correct, indefinitely — the TPM
  path cannot be relied on as the only way in.
- **Open gap:** PCR 2 is measured non-empty but unexplained, and its stability
  across peripheral attachment — docks, Thunderbolt devices, external GPUs — is
  untested. If it proves to move on hotplug, the enrollment will invalidate for
  reasons unrelated to firmware, and this record needs revisiting.

  *Amended 2026-08-23:* all three sealed-against PCRs are now measured
  byte-identical across a reboot (`state/boot.md`, `## tpm2 pcrs`, two collector
  runs either side of one). That narrows the gap without closing it — it
  establishes that nothing in an ordinary boot extends PCR 2, which is the
  weaker of the two properties. Peripheral attachment remains untested, and it
  is the case that would actually bite. What the reboot result does buy is that
  a future diff in that section is now known to mean something changed, rather
  than being noise nobody had ruled out.

## Revisit when

- PCR 2 is observed changing across peripheral attachment, or its contents are
  explained.
- `org.fwupd.hsi.Amd.SpiWriteProtection` ever reports enabled — which would
  reopen the question this record closed.
