---
title: sshd binds the wildcard; nftables is the control
type: decision
id: ADR-0029
aliases: [ADR-0029]
status: accepted
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-31
commit: pending
impact: medium
created: 2026-08-31
updated: 2026-08-31
tags: [decision, ssh, nftables, tailscale, boot-order, lockout]
constrains: spec/ssh, spec/tailscale
---

# ADR-0029 — sshd binds the wildcard; nftables is the control

- **Status:** accepted
- **Date:** 2026-08-31
- **Provenance:** measured (both halves verified across a reboot, see below)

## Summary

sshd continues to listen on `0.0.0.0`/`[::]`. Reachability is enforced by
nftables — input policy `drop`, with `iifname "tailscale0" accept` as the only
path to port 22. **Step 6 of [[../backlog/stage-3b-lockdown|stage-3b-lockdown]],
binding sshd to the Tailscale address, is deliberately not done.**

## Context

Stage 3b closed public inbound. With `policy drop` and only `tailscale0`
accepted, sshd's bind address no longer determines who can reach it — nftables
does, before a packet reaches the socket. Binding sshd to the tailnet address as
well would be defence in depth: two independent controls, so a flushed or
unloaded ruleset would not immediately expose sshd to the internet.

It also introduces a dependency that did not exist before. sshd can only bind an
address that is present at start time, so it now depends on `tailscaled` having
brought `tailscale0` up first. If that ordering is not expressed — or is
expressed and still races — sshd exits on a bind error and the machine comes back
reachable only through the VNC console.

The failure is silent in the way that matters: it presents as a dead service, not
as a networking fault, and it presents *after a reboot*, which is exactly when
nobody is watching. This machine is checked roughly monthly by one person whose
only fallback is a VNC console at the wrong keyboard layout.

The hazard is not hypothetical here. The same area bit the decommissioned hive
build, where `ssh.socket` had to be disabled and `ssh.service` run directly
before `ListenAddress` was respected at all — socket activation ignores the
config's bind address. Any future attempt at step 6 inherits that too.

## Decision

Leave sshd bound to the wildcard. Treat the nftables ruleset as the control, and
`/etc/nftables.conf` as the file that must be correct for the machine to be both
reachable and closed.

## Alternatives Considered

1. **Bind sshd to the Tailscale address, with `After=`/`Wants=` on
   `tailscaled.service`.** The textbook form of defence in depth. Rejected on the
   asymmetry of the two failure modes, not on effort — see below.
2. **Bind sshd to the Tailscale address without ordering directives.** Rejected
   outright; this is the version that strands the operator, and it works until
   the first reboot, which is the worst possible schedule for discovering it.
3. **Both controls, verified by repeated reboot testing.** Defensible, and the
   right answer on a machine with out-of-band access that is pleasant to use.
   Not this machine.

## Consequences

### Positive

- No boot-order dependency between sshd and `tailscaled`. Nothing new can fail at
  reboot.
- The failure modes are asymmetric in the useful direction. If the firewall fails
  to load, the machine is **reachable but exposed** — recoverable remotely, and
  visible. If sshd failed to bind, the machine would be **closed and unreachable**
  — recoverable only by console. Preferring the first is the whole decision.
- One control to reason about, in one file, with one verification command.

### Negative

- **A flushed or unloaded ruleset exposes sshd to the public internet
  immediately.** Key-only authentication is then the only thing standing between
  the machine and the internet's background scanning. That is a real reduction in
  depth and should not be argued away.
- `/etc/nftables.conf` becomes load-bearing beyond its apparent role. Anyone
  running `nft flush ruleset` for a debugging session opens the box without
  intending to.
- The obligation lands on `nftables.service` staying enabled. Nothing currently
  alarms if it is not.

### Residual

`systemctl is-enabled nftables` is the check that matters and nothing re-runs it.
The right home for it is the collector this device does not have —
[[../backlog/netcup-collector|netcup-collector]].

## Verification

`measured` 2026-08-31, after a deliberate reboot:

```bash
# NetCup server
sudo nft list chain inet filter input | head -3   # policy drop
systemctl is-active tailscaled nftables           # active, active
```

```bash
# Arch (ThinkPad)
ssh netcup-tailnet 'echo ok'                              # ok
ssh -o ControlPath=none -o ConnectTimeout=10 netcup-public # timed out
```

The `ControlPath=none` on the negative test is not decoration — see
[[../sessions/20260831-stage-3b-lockdown#The verification that lied|the session log]].

## Revisit when

The machine gains out-of-band access that is genuinely usable — a working serial
console, or a second independent network path — or when a service on it justifies
the extra depth. Either changes the asymmetry this record turns on.

## Related

- [[../backlog/stage-3b-lockdown|stage-3b-lockdown]] — step 6, closed by this
- [[../runbooks/ssh-lockout-recovery|Locked out of SSH]]
- [[20260830-vps-holds-no-credentials-upward|ADR-0027]]
