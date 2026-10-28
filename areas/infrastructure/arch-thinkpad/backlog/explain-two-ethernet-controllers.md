---
type: backlog-item
status: resolved
priority: low
device: arch-thinkpad
projects: [hardware]
tags: [hardware, ethernet, unexplained]
created: 2026-08-21T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Explain the two Realtek ethernet controllers

> [!NOTE] Summary
> `enp2s0f0` (02:00.0, rev 0e) and `enp4s0` (04:00.0, rev 15) — both RTL8111/8168 `[10ec:8168]`, both Lenovo subsystem `[17aa:5081]`, both driven by `r8169`, both DOWN. Distinct PCI addresses and different revisions, so not one controller presenting twice. A T14 Gen 1 AMD has one RJ45.

> [!SUCCESS] Resolved 2026-09-02 — PCI facts `measured`, the mapping `inferred`
> They are two different chips on two different root ports, so neither
> hypothesis of one device presenting twice survives. `journalctl -k`:
> `02:00.0` is **RTL8168ep/8111ep** (rev 0e, "DASH disabled", `eth0`), parent
> `00:02.2`; `04:00.0` is **RTL8168h/8111h** (rev 15, `eth1`), parent `00:02.6`.
> Their permanent MACs are consecutive, which is how a vendor provisions two
> onboard NICs, not how a dock enumerates. `dmidecode -t slot` lists no populated
> slot for either. **Inferred:** the `ep` variant with DASH is the RJ45 port's
> controller (DASH is Lenovo's out-of-band management on that port), and the `h`
> is the *Ethernet extension connector* beside the USB-C ports, which on this
> generation is a second wired NIC that only carries traffic with Lenovo's
> adapter attached. What the PSREF says was not checked — no web lookup this
> round. Both are `DOWN` and both sit behind the default-drop input chain; no
> action.

## Hypotheses

- An Ethernet extension connector populated on this unit.
- A dock enumeration that persisted in the PCI tree.

## Why it matters

Low, but non-zero: an unexplained network interface is an unexplained attack surface, and both carry permanent MACs that are now in the redaction map. Worth resolving before the build spec claims to describe this hardware.

## Steps

`lspci -vvnn -s 02:00.0` and `-s 04:00.0`, plus `dmesg | grep r8169`. Compare against the Lenovo PSREF for this machine type.
