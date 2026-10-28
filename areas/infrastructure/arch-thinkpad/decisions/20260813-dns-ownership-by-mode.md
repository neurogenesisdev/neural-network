---
title: DNS ownership is split by connection mode
type: decision
id: ADR-0005
aliases: [ADR-0005]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-13
updated: 2026-08-21
tags: [decision, networking, dns, privacy, tailscale, mullvad]
supersedes: PRD-7.3
constrains: spec/networking
---

# ADR-0005 — DNS ownership is split by connection mode

- **Status:** accepted, but **the default-mode question is now open** — see
  Revisit
- **Date:** 2026-08-13
- **Amended:** 2026-08-21 — Tailscale Mullvad exit nodes became available on
  this device, which changes which row of the table is the common case
- **Provenance:** measured
- **Supersedes:** PRD §7.3 (DNS over TLS via systemd-resolved + NextDNS), which
  is **rejected**, not deferred

## Context

PRD §7.3 specified NextDNS over DoT via `systemd-resolved`. Applying it would
have put three components in contention for `/etc/resolv.conf`:
`systemd-resolved`, NetworkManager, and the Mullvad client daemon.

## Decision

No single global resolver. `systemd-resolved` stays **disabled**. Resolution is
whoever owns the tunnel at the time:

| Mode | Who resolves | Filtering |
|---|---|---|
| Tailscale exit node active | the exit node, internally — queries stay in-tunnel | **none** |
| Native Mullvad client connected | the Mullvad client, via `mullvad dns set default --block-ads --block-trackers` | ads + trackers |
| Neither | DHCP-provided resolver | **none** |

## Rationale

NextDNS over DoT sends account-identifying queries *outside* the tunnel. That
is a worse privacy position than letting the tunnel operator resolve, and it
was the deciding argument.

uBlock Origin covers the filtering gap in the browser, which is where it
matters most. **It does not cover anything else on the machine.**

A tailnet-wide Global nameserver with "Override local DNS" was considered and
rejected: it applies to every client whenever Tailscale is connected —
including the Pixel, and including this machine on the home network with no
exit node — and with an exit node active the query is answered by the exit node
anyway. The effective behaviour would be state-dependent and hard to reason
about.

## Consequences

- Filtering is **not** uniform across modes. Only the native Mullvad client
  filters. This was accepted when the exit-node mode was unavailable; it is
  more consequential now that it is not.
- `mullvad dns set` configures the **client daemon only**. It has no effect
  when routing through a Tailscale Mullvad exit node — the two are alternative
  products sharing a brand, not layers of one product.
- No DoT/DoH is configured on this machine at all.
- The Pixel's NextDNS configuration is a separate device decision, unaffected.

## Amendment, 2026-08-21 — exit nodes are now available

`tailscale exit-node list --filter=Germany` returns a populated list. The
device was enabled in the Tailscale Mullvad add-on settings and `tailscaled`
was restarted; **which of the two produced the change was not isolated.**

`SelfNode.capMap` is still `null`. The claim in state reference §1.4 that an
empty `capMap` proves the grant is not being sent **is therefore wrong** — the
grant is evidently arriving by some other path, or the field moved between
client versions. The functional test is `tailscale exit-node list`. See the
operational notes.

As of the amendment no exit node is selected (`STATUS` is `-` for all nodes),
so the machine is in the third row: DHCP resolver, no filtering.

## Revisit when

**Now.** Two modes that previously could not coexist now both work, and they
differ in filtering. The open question is which is the default:

- **Exit node as default** — better transport privacy story, no filtering
  outside the browser.
- **Native client as default** — filtering everywhere, and exposes DAITA and
  multihop, which the Tailscale add-on does not.
- **Neither as default** — current state by inertia rather than by choice.

This needs deciding rather than drifting. The deciding factor is whether
DNS-level ad/tracker filtering outside the browser is worth giving up DAITA and
multihop, or vice versa.
