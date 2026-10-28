---
type: backlog-item
status: active
priority: low
device: netcup-vps
projects: [netcup-build]
tags: [backlog, netcup, panel, cpu, dns, ipv6]
created: 2026-08-30
updated: 2026-08-30
commit: pending
---

# cpu-topology-and-ipv6-rdns

> [!NOTE] Summary
> Two panel-side corrections, bundled because neither justifies its own trip to
> the SCP. CPU topology is presented as 4 sockets × 1 core, which tells the guest
> scheduler there is NUMA distance between every core. IPv6 reverse DNS is unset,
> so forward and reverse now disagree after the address changed during the
> rebuild.

## Context

Both measured 2026-08-30 in the SCP.

**CPU topology.** *Settings → CPU Topology* shows Sockets 4, Cores per Socket 1,
for 4 vCPUs. A guest reading that assumes four separate physical packages and
schedules as though cache and memory locality differ between them. `1 socket ×
4 cores` describes what the hypervisor is actually providing.

The practical effect on three small containerised services is somewhere between
negligible and unmeasurable. It is filed because it is free to correct during a
reboot that is happening anyway, not because it is costing anything now.

**IPv6 rDNS.** The rebuild changed the v6 address from netcup's MAC-derived
EUI-64 address to a clean `<NETCUP_IPV6>` — see
[stage 2](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage2-configure-boot.sh) for why. The panel's IPv6 reverse
DNS field was empty before and is still empty. IPv4 rDNS is the
`<SERVER_ID>.ultrasrv.de` default and still correct.

## Steps

1. Next time the machine is powered off — most likely for
   [[stage-3b-lockdown]] or a kernel update — set *Settings → CPU Topology* to
   Sockets 1, Cores per Socket 4, save, then power on.
2. In *Network → IPv6 Addresses*, set the reverse DNS entry for
   `<NETCUP_IPV6>` to match the IPv4 rDNS name.
3. Verify:

   ```bash
   lscpu | grep -E 'Socket|Core|Thread'
   dig -x <NETCUP_IPV6> +short
   ```

## Blocked on

Nothing. Waiting for a reboot window rather than warranting one.

## Note

**The CPU change requires a full power cycle, not a reboot.** Guest-visible
topology is set at domain start. A `systemctl reboot` will not pick it up, which
makes this look like it did not work.

**IPv6 rDNS is not cosmetic if this machine ever sends mail** — receiving mail
servers check that forward and reverse agree, and a missing v6 PTR on a
dual-stack host is a common cause of delivery failures that present as spam
filtering. Note that netcup blocks outbound SMTP by default anyway
([[panel-firewall-outer-ring]]), so this only becomes live if that block is ever
lifted. Two related facts worth keeping adjacent.

**Do not change the storage or network driver while in this panel.** Both are
`virtio` and correct; netcup's documentation warns that a storage driver change
can render the server unbootable and that SATA/IDE severely affects snapshots.
Mentioned here because the CPU topology control sits on the same screens.

## Related

- [[stage-3b-lockdown]] — the likely reboot window
- [[panel-firewall-outer-ring]] — the SMTP block
- [[../sessions/20260830-netcup-vps-rebuild#Panel state, recorded here because there is no `state/`|Session log — measured panel state]]
