---
type: index
device: netcup-vps
updated: 2026-09-15
tags: [index, backlog]
---

# Backlog

Everything known to be undone on netcup-vps. Ordered by consequence.

**As of 2026-09-11 nothing here is required anymore.** The machine is on the
tailnet, closed to the public internet at both rings, and backed up off-provider
with a tested restore. What remains is improvements — and the service/coding-rig
build-out, which is a project (see the plan in `arch-thinkpad/_scratch/`).

## Active Items

| Item | Priority | Area | Summary |
|------|----------|------|---------|
| [[mcp-read-access-from-desktop]] | medium | tooling | Claude chats (incl. phone app, laptop off) read VPS vault mirrors + inbox via a remote MCP connector: read-only principal `mcpread`, Tailscale Funnel, OAuth required. First public-facing service — ADR first. Requested "very soon" 2026-09-15, ≈ 3–4 h |
| [[netcup-collector]] | medium | vault | No `collect.sh`, so no `state/`. Every claim in this folder is a point-in-time measurement nothing will re-verify — including `systemctl is-enabled nftables`, which ADR-0029 now depends on |
| [[apparmor-baseline]] | medium | security | Debian ships the framework; nothing here enforces a profile. The RHEL alternative rejected in [[../decisions/20260830-debian-over-rhel-family\|ADR-0024]] would have arrived with SELinux enforcing — that delta is unpaid |
| [[tailscale-ssh-decision]] | medium | security | Unblocked by ADR-0027's acceptance. The question changed shape though — ADR-0029 declined a boot-order dependency on `tailscaled`, and Tailscale SSH deepens exactly that dependency |
| [[cpu-topology-and-ipv6-rdns]] | low | config | Two panel-side corrections that need a power cycle and a form respectively. Bundled because neither justifies its own reboot. **A reboot was spent on 2026-08-31** verifying the lockdown and this was not folded into it |

## Resolved 2026-09-12

| Item | Outcome |
|------|---------|
| ~~[[vps-trust-direction]]~~ | Per-repo deploy keys are the measured norm: read-only for the two context vaults, write for `site-template`; `git push --dry-run` from a read-only clone is refused by GitHub. ADR-0027 stays in this folder; moving it to `shared/` is cosmetic and deferred |

## Resolved 2026-09-11

| Item | Outcome |
|------|---------|
| ~~[[restic-client]]~~ | restic → B2 daily timer with `LoadCredential=`, monthly integrity check, staleness alarm → ntfy on the phone, restore verified byte-identical from arch-thinkpad, retention runs from the laptop. **Measured:** the write-only key hides but cannot destroy; hidden snapshots visible as prior versions with the admin key. Target changed from the Storage Box (does not exist) to a dedicated B2 bucket |
| ~~[[panel-firewall-outer-ring]]~~ | Policy `tailnet-only` in the SCP: UDP 41641 accept, TCP and UDP drop, after *Ping allow*. Off-tailnet port 22 times out, ping answers, Tailscale stays `direct`, verified across a reboot. Runbook + session log written. **The outer ring exists**; ADR-0029's single-control residual is now two controls |

## Resolved 2026-08-31

| Item | Outcome |
|------|---------|
| ~~[[stage-3b-lockdown]]~~ | **Done.** Tailnet joined, tailnet SSH verified *before* anything was tightened, nftables to `policy drop`, public inbound confirmed dead from a fresh connection, and the whole thing verified across a deliberate reboot. Step 6 closed as won't-do by [[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control\|ADR-0029]]. **Two things the item did not anticipate**: Debian's stock `flush ruleset` would have wiped Tailscale's own tables during a revert, and SSH connection multiplexing made the first verification report success against a firewall that was already closed. See [[../sessions/20260831-stage-3b-lockdown\|the session log]] |

## Deferred deliberately

**Service rebuild — Vaultwarden, Radicale, n8n.** Not filed as backlog items
because they are a project rather than undone work, and because every one of them
depends on the four high/medium items above. Filing them now would put six
service items above a firewall item in a list ordered by consequence.

The constraint that will govern them is already recorded:
[[../decisions/20260830-no-luks-on-always-online-vps#Consequences|ADR-0021]]
requires `podman secret` or systemd `LoadCredential=` rather than a `.env` file
beside the quadlet — the pattern the decommissioned hive used.

**A `spec/` chapter.** [[../build/_index|build/]] holds the executable form.
Promotion to step records is worth doing when a second netcup machine exists or
when a runbook needs to transclude a build step; neither is true.

## Deliberately not tracked here

Decisions with a rationale go in [[../decisions/_index|decisions/]]. Discovery
narrative goes in [[../sessions/_index|sessions/]]. This file is only for work
that is undone.
