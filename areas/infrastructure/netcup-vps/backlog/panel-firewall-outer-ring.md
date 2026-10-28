---
type: backlog-item
status: resolved
priority: medium
device: netcup-vps
projects: [netcup-build]
tags: [backlog, security, firewall, netcup, smtp]
created: 2026-08-30
updated: 2026-09-11
commit: pending
---

# panel-firewall-outer-ring

> [!NOTE] Summary
> **Resolved 2026-09-11.** Policy `tailnet-only` (UDP 41641 accept, TCP/UDP drop) added after *Ping allow*; verified off-tailnet (port 22 times out, ping answers), direct Tailscale path kept, survives reboot. The pre-existing SSH session stayed open through the change — whether the panel tracks state or drops only new flows is not distinguishable from that. Procedure: [[../runbooks/panel-firewall-outer-ring|runbook]]; narrative: [[../sessions/20260911-panel-firewall-outer-ring|session]].
>
> netcup's cloud firewall is a genuine second layer — one the decommissioned hive
> did not have — and it currently accepts everything. Configuring it is the last
> step of the lockdown, deliberately after the host firewall and Tailscale, so
> that a mistake at the outer ring can still be diagnosed from inside.

## Context

Measured 2026-08-30 in the SCP. Rules evaluate top-to-bottom until first match,
and the **Implicit Rules** block at the end accepts all inbound and all outbound.
So the only live restrictions are:

- **netcup Mail block** — outbound TCP 25, 465, 587 dropped
- **netcup Ping allow** — ICMP and ICMPv6 accepted in both directions

Everything else is permitted inbound.

The outbound SMTP block is worth knowing about before it causes a confusing
failure: any future service that sends mail directly will fail silently at the
network layer, not in its own logs.

## Steps

1. Complete [[stage-3b-lockdown]] first — host firewall tightened, Tailscale
   verified, sshd bound.
2. Confirm the tailnet path is working and has been working for at least a few
   days. This is the outer ring; if it is wrong, the inner ring is what saves the
   session.
3. Add explicit accepts for what the machine actually needs inbound: UDP 41641
   for Tailscale direct connections, and ICMP (keep the existing default policy).
4. Change the implicit inbound accept to a drop, or add an explicit final deny.
5. Leave outbound alone. The SMTP block stays; there is no reason to lift it and
   a reason to keep it.
6. Verify from off-tailnet — a phone on mobile data — that the public IP no
   longer answers on 22.

## Blocked on

[[stage-3b-lockdown]]. Configuring the outer ring while the inner ring is
permissive means a mistake locks out the only working path, with no second layer
to fall back through.

## Note

**This is the one control in the estate that survives a compromised host.** nftables
runs on a machine an attacker with root can reconfigure; the panel firewall does not.
That makes it worth configuring properly rather than treating as redundant with
nftables — they defend against different things.

**Tailscale needs UDP 41641 inbound for direct connections.** Blocking it does not
break Tailscale — it falls back to DERP relays — but it does make every connection
slower and dependent on Tailscale's infrastructure. The failure mode is *degraded
and still working*, which means it will not be noticed and will be blamed on
something else later.

**Changes here take effect without a reboot and without warning.** Unlike the host
firewall there is no `nft -c -f` dry run and no scheduled-revert trick. Make one
change, verify, make the next.

## Related

- [[stage-3b-lockdown]] — the blocker
- [[../sessions/20260830-netcup-vps-rebuild#Panel state, recorded here because there is no `state/`|Session log — measured panel state]]
