---
title: Accept SSH agent forwarding into VS Code devcontainers
type: decision
id: ADR-0003
aliases: [ADR-0003]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: high
created: 2026-08-18
tags: [decision, ssh, credentials, devcontainer, keepassxc]
constrains: spec/devenv
---

# ADR-0003 — Accept SSH agent forwarding into VS Code devcontainers

- **Status:** accepted
- **Date:** 2026-08-18
- **Provenance:** measured — the forwarding behaviour was observed; the upstream
  issue confirms it is not configurable

## Context

The purpose of the devcontainer is that third-party dependencies and
agent-issued shell commands cannot reach `~/.ssh`, the KeePassXC database,
`~/Sync`, Firefox profiles, or the host home directory. The VS Code Dev
Containers extension forwards `$SSH_AUTH_SOCK` into the container
unconditionally. `remoteEnv` overrides are ignored and no setting disables it
(vscode-remote-release#11413).

This partially undermines the stated purpose: code in the container cannot read
the private key, but it can ask the agent to sign with it for as long as the
key is loaded.

## Decision

Accept the forwarding. Do not abandon Dev Containers over it. Bound the
exposure by time rather than by isolation.

## Rationale

The alternatives were each worse:

- **Abandon Dev Containers, use the `devbox` wrapper only.** The wrapper does
  not forward the agent unless `--with-agent` is passed. But this gives up the
  editor integration that made containment usable day to day, and an unusable
  control gets bypassed.
- **Run without an agent, type the passphrase per operation.** The key lives in
  KeePassXC; there is no passphrase to type independently of unlocking it.
- **Accept it, and bound the window.** Chosen. The KeePassXC inactivity timer
  becomes the control — see ADR-0004.

The exposure is *use of* the key, not *disclosure of* it. A compromised
dependency can sign during the window; it cannot exfiltrate the key and use it
later.

## Consequences

- The KeePassXC inactivity timer is load-bearing, not a convenience setting.
- `gh` is deliberately host-only. Its token is a copyable file rather than an
  agent socket, so putting it in the container would be strictly worse than
  forwarding the agent.
- The attached devcontainer also receives IPC bridges back to the host
  (`VSCODE_IPC_HOOK_CLI`, `REMOTE_CONTAINERS_IPC`, `GIT_ASKPASS`). A `.bashrc`
  shim unsetting these is on the backlog. It stops opportunistic credential
  harvesting; it does not stop anything that enumerates `/tmp`. Mitigation,
  not a boundary.

## Revisit when

SSH key diversification is implemented — a GitHub-only key that may be
forwarded freely, with the infrastructure key (hive, n8n, Storage Box) never
leaving the host. That changes a forwarded-agent compromise from "re-key
everything" to "one forge account" and supersedes the one-key-per-device model.
