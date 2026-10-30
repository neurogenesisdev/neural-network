---
type: index
device: netcup-vps
updated: 2026-09-24
tags: [index]
---

# Runbooks

Procedures with a trigger, entered at need. The first two were written from
things that went wrong during the 2026-08-30 build; the 2026-09-11 pair are
planned procedures written *before* execution and marked `inferred` until run.

| Runbook | Fires when | Urgency |
|---------|-----------|---------|
| [[ssh-lockout-recovery\|Locked out of SSH]] | `ssh` refuses, times out, or the host key changed unexpectedly | High — the machine may be fine and unreachable, or may not be booting |
| [[rescue-system-entry\|Entering and working in the rescue system]] | The machine does not boot, or the disk must be manipulated offline | High — you are here because normal access failed |
| [[restic-b2-backup\|restic → B2 backup: install, verify, restore]] | Setting up backups (plan step 0.2), or restoring a file from the VPS repository | Medium — planned work, not an incident; the restore section is the one entered under pressure |
| [[panel-firewall-outer-ring\|Configure the netcup panel firewall]] | Closing the outer-ring backlog item (plan step 0.3) | Medium — planned; the rollback is the VNC console |
| [[coding-rig-stage-1\|Coding rig stage 1]] | Building or rebuilding the `dev` principal, the uid-keyed egress rules, or any of the three harnesses; also the paste rule for tmux sessions | Medium — planned work; the state-as-of-tonight record for resuming stage 1. **Amended 2026-09-24:** sandbox off, headless Chromium for browser MCP, zsh for `dev`/`luiswg`/`root`, laptop `ssh <user>@vps` → tmux |
| [[automations-stage-2\|Automations stage 2]] | Building or rebuilding the `auto` or `hermes` principal, the inbox repo, the vault mirror, any stage-2 timer, or the Telegram secretary; unit conventions for `auto-*` system units; the Hermes traps | Medium — planned work; all sections `measured` 2026-09-12/13 |
| [[mcp-endpoint\|MCP endpoint for claude.ai]] | Building, checking or rebasing the claude.ai vault connector: `mcpread`, obsidian-remote-mcp v1.5.0 with its 11 local commits, the two fences, the Funnel path mounts, the monthly upstream rebase; **the kill switch for the public endpoint** | High when used as the kill switch; otherwise planned care. Rewritten 2026-10-29 for the read/write connector (ADR-0041); all sections `measured`, acceptance passed |

The two chain: a lockout that the console cannot resolve escalates to rescue.
`ssh-lockout-recovery` says so at the end.

## The recovery path this machine actually has

Worth stating once, because two of its properties are counterintuitive and both
were discovered the hard way.

**Root is locked** (`passwd -l root`, *measured 2026-08-30*: `passwd -S root`
returns `root L`). Console login is `luiswg` plus `sudo -i`. There is no root
console login and there will not be one.

**The netcup panel's root-password reset does not help.** It works by booting
rescue and writing into the target's `/etc/shadow` — it is not a live lever
against a running system. Whether it also clears the `!` lock prefix is
**`inferred`, not tested**; if it only replaces the hash, the lock persists and
the reset silently does nothing. Do not rely on it. The `luiswg` console
password in KeePassXC is the real fallback.

**The rescue system's root password is a third, separate credential**, shown in
the SCP at activation time. It is not the panel root password and not the
`luiswg` password. This cost time on 2026-08-30.

## What belongs here

- A procedure with a trigger, entered under pressure, possibly from a phone
- Anything where the correct first move is *look at the console* rather than
  *retry the command*

Decisions with a rationale go in [[../decisions/_index|decisions/]]. The
narrative of how these were discovered is in
[[../sessions/20260830-netcup-vps-rebuild|the session log]].
