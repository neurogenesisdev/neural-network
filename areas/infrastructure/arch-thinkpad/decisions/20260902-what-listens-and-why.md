---
type: decision
id: ADR-0031
aliases: [ADR-0031]
status: accepted
impact: medium
provenance: measured
device: arch-thinkpad
constrains: spec/base
stale: false
created: 2026-09-02T17:40:00Z
updated: 2026-09-02T17:40:00Z
last_verified: 2026-09-02
commit: pending
supersedes:
superseded_by:
tags: [decision, listeners, network, avahi, passim, ollama, ssh, syncthing, hardening]
---

# ADR-0031 — What listens on this machine, and why

## Summary

Four listeners are deliberate — `sshd` on the tailnet address, `syncthing`, and
nothing else on the network; `passim` and `avahi-daemon` are masked and `ollama`
is disabled and started by hand. The nftables default-drop input chain remains the
control that makes any listener harmless on a foreign network; this record is
about not depending on it more than necessary.

## Context

[[../backlog/undeliberate-listeners-review|undeliberate-listeners-review]] found
in August that "nothing listens that I did not put there" was false: `ollama`
(loopback), `passimd` (all interfaces) and `avahi-daemon` (all interfaces) were
listening without having been chosen. All three were unreachable from outside —
`policy drop`, with only `tailscale0` and the Syncthing ports accepted — so the
cost was exposure *if the ruleset ever failed to load*, on a laptop that joins
café networks. Re-measured 2026-09-02: `avahi-daemon.service` was disabled yet
running, socket-activated; `passim.service` is `static` and a hard dependency of
`fwupd`; `ollama.service` was enabled.

The operator has no printers, no network storage and no LAN devices to
discover, and uses `ollama` occasionally.

## Decision

- `passim.service`: **masked**. It shares firmware metadata with other machines
  on the LAN; this machine has nothing to share with and nothing to receive.
  `fwupd` keeps working without it (`fwupdmgr get-devices` answers).
- `avahi-daemon.service` and `.socket`: **masked**. mDNS discovery has no
  consumer here. The package stays — `libcups`, `pipewire-pulse` and others link
  its library — only the daemon is stopped.
- `ollama.service`: **disabled**, not masked. `systemctl start ollama` when it is
  wanted; it listened on loopback only and was never a network exposure, so this
  is tidiness, not hardening.
- `sshd`: deliberate, bound to `<TS_IPV4>:22` — the reasoning and the boot-order
  drop-in are in the backlog item and `notes.md`.
- `syncthing`: deliberate, `*:22000` and UDP discovery, the only thing accepted
  inbound from non-tailnet networks; it carries the KeePassXC database to
  pixel-graphene.

## Alternatives Considered

1. **Leave all three, trust the firewall.** That is the state the item found and
   it is defensible; rejected because each one is free to remove and the firewall
   is load-bearing on a roaming machine with nothing re-verifying it.
2. **Socket activation for ollama.** It has no native support; a proxy unit adds
   a moving part for a loopback-only service. Rejected as not worth it.
3. **Uninstall avahi.** Not possible — six packages require the library.

## Consequences

### Positive

- `ss -tulnp` on a foreign network now shows exactly two things reachable by
  design (Syncthing) and one bound to the tailnet (sshd).
- One record to check against `state/network.md`'s listening-sockets section.

### Negative

- A `.local` hostname will stop resolving. That is the symptom to expect if
  something one day depends on mDNS; the fix is `systemctl unmask
  avahi-daemon.socket`.
- `ollama` must be started by hand. If it is used daily, re-enable it.

## Verification

`measured` 2026-09-02:

```bash
# arch-thinkpad
ss -tulnp | grep -E '5353|27500|11434'     # nothing
systemctl is-enabled ollama passim avahi-daemon.socket avahi-daemon.service
                                            # disabled masked masked masked
fwupdmgr get-devices >/dev/null && echo ok  # ok
```

Rollback: `systemctl unmask passim avahi-daemon.socket avahi-daemon.service`,
`systemctl enable --now ollama`; snapper snapshot 469 predates the change.

## Revisit when

A printer, a cast target or another LAN device arrives, or `ollama` is used
often enough that starting it by hand is friction.

## Related

- [[../backlog/undeliberate-listeners-review|undeliberate-listeners-review]] — closed by this
- [[../../netcup-vps/decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]] — the opposite sshd decision, and why
