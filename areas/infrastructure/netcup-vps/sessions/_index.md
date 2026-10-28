---
type: index
device: netcup-vps
updated: 2026-09-12
tags: [index]
---

# Sessions

Working narrative for netcup-vps. Same purpose as
[[../../arch-thinkpad/sessions/_index|arch-thinkpad's]]: dead ends, discovery
order, revised conclusions, and negative findings — the residue no other artifact
holds.

| Session | Subject |
|---------|---------|
| [[20260830-netcup-vps-rebuild\|2026-08-30]] | Full rebuild from the netcup image to a debootstrapped Debian 13. Six decisions settled before the wipe; the install itself was uneventful and **every problem came from the transport between operator and machine** rather than from the machine |
| [[20260831-stage-3b-lockdown\|2026-08-31]] | Public inbound closed, tailnet-only SSH, verified across a reboot. ADR-0027 accepted, ADR-0029 written. The transport pattern repeated — but this time a transport fault produced a **false confirmation** rather than a visible error: SSH multiplexing reported public SSH working against a firewall that was already correctly closed |
| [[20260911-panel-firewall-outer-ring\|2026-09-11]] | **Stage 0 complete.** ADR-0034 written; panel firewall applied (public inbound dropped except UDP 41641 + ICMP, verified off-tailnet and across a reboot); restic → B2 daily with `LoadCredential=`, ntfy alerts, restore verified from the laptop. **Measured: the write-only B2 key can hide but not destroy** — 30-day prior versions are the recovery window. Faults again operator-side: wrong panel screen, `*:*` as input, bash `read` on zsh, a block paste swallowed by a new shell, a key secret pasted into chat (rotated) |
| [[20260912-coding-rig-stage-1-complete\|2026-09-12]] | **Stage 1 complete.** Deploy keys, worktrees + peer messaging (on by default), the Codex review gate (four sandbox openings, each found by running it), laptop alias, `site-template` scaffolded by Claude Code and reviewed by Codex — **two real defects found** that the coder's own checks could not. ADR-0035 written; ADR-0034 accepted. An agent's "I noticed" was wrong once; an empty-tree SHA hung the plugin ten minutes |
| [[20260912-stage-2-0-auto-bootstrap\|2026-09-12 (evening)]] | **Stage 2 built in one weekend (2.4/2.5 parked): `auto` + inbox repo, heartbeat, digest, vault mirror, `claude -p` triage, hours, a contained Telegram secretary, and n8n behind `tailscale serve`.** `auto` bootstrapped: write-only deploy key to `freelance-automation-inbox` (refused on the vault repo — measured), root commit, `auto-*` system-unit pattern with `ProtectSystem=strict` and the root ntfy template; success and forced failure both seen, alert reached the phone; healthchecks heartbeat via a wrapper after `ExecStopPost=` turned out to have no credentials dir; first trend digest committed from a 130-line script at ≈ $0.002/run; root vault mirror + `claude -p` weekly triage — **the first triage was confidently wrong because the mirror was stale** (nothing pushed the vault); fixed with a laptop push timer. Hermes: `DropCapability=ALL` kills its init; `debugging` in `disabled_toolsets` silently strips `file` (ChatGPT found it from a handoff); a second `hermes` process beside the gateway corrupts `state.db`. ADR-0036 Decision: system units with `User=auto` instead of user units. No transport faults |

## The finding that spans 2026-08-30

Worth surfacing here because it is the shape of the whole session and no single
ADR carries it.

**Nine of the eleven interruptions were transport failures, not system failures.**
kitty's paste dialog, a `^C` smuggled into a clipboard, zsh swallowing a heredoc,
`sudo` eating a script line as a password, a rescue shell with no bracketed-paste
support, Grml lacking `xterm-kitty` terminfo, commands run against the wrong
machine, an SSH session that had silently ended, and `IdentitiesOnly` filtering an
agent key. The machine did what it was told throughout.

The corrective that emerged — **write the script on arch-thinkpad, review it,
pipe or copy it, never paste into a remote shell** — is now the standing pattern
and is why [[../build/_index|build/]] exists in the form it does. It was arrived
at three separate times before it stuck.

## The finding that spans 2026-08-31

The same shape, one turn worse.

**A transport fault can confirm rather than fail.** Every 2026-08-30 interruption
announced itself as an error. On 2026-08-31, SSH connection multiplexing made
`ssh netcup-public` return a prompt *after* the firewall had correctly closed the
public path, because the session rode a master connection opened before the
change. The natural response to that reading — conclude the ruleset is broken and
start editing it — would have been destructive on a machine whose only fallback is
a VNC console.

The corrective: **anything verified over SSH after a change to shells, keys, or
the firewall runs on a fresh connection** — `ssh -O exit <host>` first, or
`-o ControlPath=none` on the check. Otherwise the answer describes a connection
made in the past.

Both sessions also lost a round to `ssh_config`'s first-match-wins rule via a
`Host *` block placed at the top of the file — `IdentitiesOnly` in the first,
`ControlPath` in the second. Two settings, one mechanism, two sessions. It is a
property of that file, not a coincidence.
