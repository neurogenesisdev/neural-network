---
title: No full-disk encryption on an always-online VPS
type: decision
id: ADR-0021
aliases: [ADR-0021]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
impact: high
created: 2026-08-30
updated: 2026-09-11
tags: [decision, encryption, luks, threat-model, vps]
constrains: build/stage2, spec/services
---

# ADR-0021 — No full-disk encryption on an always-online VPS

- **Status:** accepted
- **Amended 2026-09-11:** the negative consequence "no long-lived secret in plaintext on disk" is restated in enforceable form by [[20260911-persistent-credentials-on-netcup-vps|ADR-0034]] (five properties + a named credential table). The decision and rationale here are unchanged.
- **Date:** 2026-08-30
- **Provenance:** measured

## Summary

The netcup server runs unencrypted at rest. Sensitive data is protected at the
application layer instead — restic's own encryption, Vaultwarden's client-side
crypto — and the standing obligation that replaces FDE is that **no long-lived
secret is written to disk in plaintext**.

## Context

arch-thinkpad runs LUKS2 with TPM2+PIN, so the default expectation carried into
this build was that the server would be encrypted too. It is the same operator,
the same threat model, the same discipline.

It is not the same machine. Two properties differ and both cut the same way:

- **The hypervisor can read guest RAM.** Any key material in the running
  system's memory is readable by the host. FDE cannot defend against the party
  that operates the hardware.
- **There is nobody at the console.** A passphrase-protected root volume needs a
  human at boot. On a remote server that means either VNC console entry on every
  unattended reboot — including the kernel-update reboots `unattended-upgrades`
  will produce — or `dropbear-initramfs`: a second SSH daemon, a second keypair,
  a firewall exception, and a second thing to be broken when it is needed most.

## Decision

No LUKS. `/dev/vda3` is plain btrfs.

## Rationale

The framing that settled it came from the operator, and it is sharper than the
cost/benefit version that preceded it:

> FDE protects data at rest. A server that is never at rest never gets the
> protection.

Uptime is expected to exceed 99.9%. The windows in which an encrypted disk would
protect anything are precisely the windows in which nobody is present to unlock
it. The only realistic threat FDE addresses here is **disk decommissioning** —
netcup reprovisioning shared storage without wiping it — and that is a narrow
scenario to buy with a permanent operational liability.

## Alternatives Considered

1. **LUKS with VNC console unlock** — rejected. Every unattended reboot leaves
   the machine down until the operator notices and opens a browser. This
   converts a self-healing system into one requiring attendance, against the
   stated requirement that solutions fail gracefully unattended.
2. **LUKS with `dropbear-initramfs` remote unlock** — rejected. Works, and is the
   right answer for a machine where FDE is actually required. Here it adds a
   separate SSH keypair, an initramfs network stack, and an early-boot listener
   to maintain — to defend against a threat the hypervisor already defeats.
3. **Encrypt only a data subvolume** — rejected as the worst of both. Still needs
   a key available to unattended services, so the key lands on the unencrypted
   root and the encryption becomes decorative.

## Consequences

### Positive

- Boots unattended. `unattended-upgrades` can reboot for a kernel without the
  machine staying down until someone notices.
- **netcup's snapshots stay useful.** They are taken at the hypervisor and can be
  exported off-site or imported into another netcup server. Against an encrypted
  guest they would be opaque blobs. This is a real gain, not a consolation: it is
  a better rollback primitive than anything running inside the guest.
- The panel's disk-usage reporting works, which it does not for encrypted guests.

### Negative

- **Anyone with host-level access, or the physical disk, can read everything.**
  That is accepted, not mitigated.
- **The obligation this creates is the important half of the record.** With no
  FDE underneath, `chmod 600` on a plaintext credential file is the *entire*
  protection. Concretely, for the service rebuild:
  - Container secrets go through `podman secret` or systemd `LoadCredential=`,
    **not** a `.env` file beside the quadlet. The decommissioned hive put
    `VAULTWARDEN_ADMIN_TOKEN` in `/opt/docker/.env`; that pattern must not
    survive into the rebuild.
  - The restic repository password and the Hetzner Storage Box credentials are
    the two that matter most — they are the keys to everything else this machine
    can reach.
  - A `gh` OAuth token on disk is plaintext by design and cuts directly against
    this rule; see [[20260830-vps-holds-no-credentials-upward|ADR-0027]].
- SSH host keys are necessarily passphrase-free and readable by root. That is
  true of every SSH server and is not a consequence of this decision, but it is
  the first thing an unencrypted disk hands over.

## Revisit when

netcup offers guest-transparent volume encryption, **or** this machine is asked
to hold data whose disclosure at decommissioning would be materially harmful and
which cannot be encrypted by the application that owns it. Neither condition is
close.
