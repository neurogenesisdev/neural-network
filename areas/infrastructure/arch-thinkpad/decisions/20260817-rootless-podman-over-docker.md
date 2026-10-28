---
title: Rootless podman rather than Docker for containers
type: decision
id: ADR-0001
aliases: [ADR-0001]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-17
tags: [decision, containers, podman, nftables, firewall]
constrains: spec/containers
---

# ADR-0001 — Rootless podman rather than Docker for containers

- **Status:** accepted
- **Date:** 2026-08-17
- **Provenance:** measured

## Context

The contained development environment needs a container runtime. Docker's daemon
writes its own iptables/nftables rules on start and on container creation, which
would sit underneath a hand-written `/etc/nftables.conf` with a default-drop
`input` chain and `forward policy drop`. Whether that interference actually
occurs on this machine was the deciding question, not a matter of preference.

## Decision

Use rootless podman. Do not install Docker. `podman-docker` provides the
`docker` command alias and `podman.socket` provides the Docker-compatible API
that the VS Code Dev Containers extension requires.

## Rationale

Measured 2026-08-17: the `inet filter` ruleset hash was identical before and
after building images and running containers.

    4bc7713546e40b948acb7375090a4798d5cab2ab461d18edd44edec8cc47e17f

The default-drop `input` chain and `forward policy drop` were untouched.
Rootless podman writes no nftables rules at all. Docker would have required
either accepting daemon-managed rules or fighting them.

Secondary: rootless containers run as a mapped UID rather than root, so a
container escape lands as an unprivileged user rather than as root on the host.

## Consequences

- File operations on podman's storage tree require `podman unshare`, not `sudo`
  — files are owned by mapped UIDs starting at 100000.
- `~/.local/share/containers` is a nested subvolume, excluded from `@home`
  snapshots and from restic. Container storage is disposable by design.
- The Dev Containers extension talks to `podman.socket`, which requires
  `loginctl enable-linger luiswg` to survive logout.

## Revisit when

Re-verifying the firewall claim: hash `inet filter` specifically, not the full
ruleset. The full ruleset includes Tailscale's `iptables-nft` tables, which
change on their own during exit-node and network transitions and will produce
false alarms.
