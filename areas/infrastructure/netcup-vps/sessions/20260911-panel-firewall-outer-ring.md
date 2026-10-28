---
title: "2026-09-11 — stage 0 complete: outer-ring firewall, restic → B2, ADR-0034"
type: session
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-11
commit: pending
created: 2026-09-11
updated: 2026-09-11
tags: [session, netcup, firewall, tailscale, credentials, backups, devenv]
---

# 2026-09-11 — stage 0 complete: outer-ring firewall, restic → B2, ADR-0034

Stage 0 of the coding-rig plan, start to finish. The vault gained ADR-0034 and
two runbooks (written before execution, marked `inferred`, now `measured`), and
both runbooks were run:
**the netcup panel firewall now drops all public inbound except Tailscale's
UDP port and ICMP**, verified from off-tailnet and across a reboot. Then **restic → B2 with a restore verified from the laptop.** Backlog items
`panel-firewall-outer-ring` and `restic-client` closed. Nothing on this machine
is "required" anymore.

## Context: why this session exists

`arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`
(2026-09-10, revised twice on 2026-09-11 after two rounds of second-model
review) makes netcup-vps the primary coding host. Its stage 0 is this device's
two outstanding "required" backlog items plus one new record:

- 0.1 [[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] —
  the enforceable form of ADR-0021's residual. Written; `proposed` until stage 1
  matches its credential table.
- 0.2 [[../runbooks/restic-b2-backup|restic → B2]] — written, **not run**.
- 0.3 [[../runbooks/panel-firewall-outer-ring|panel firewall]] — written and
  run, below.

## The panel firewall

### Two false starts, both operator-side

1. **Wrong screen.** The first look landed on a "Firewall policies" view that
   read *No firewall policies defined*. The actual per-server firewall is
   under the server → Firewall, and it showed exactly what the 2026-08-30
   session recorded: three outgoing SMTP drops, ping allow in/out for v4 and
   v6, then the *Implicit Rules* accepting everything both ways. Checked
   against a PDF print of the page, not from memory.
2. **The UI renders "any" as `*:*`, but you must type nothing.** Entering
   `*:*` into Sources fails validation ("any IP (null or empty array)…").
   Entering the port into *Destinations* (an IP field) fails the same way; it
   belongs in *Dst Ports*. Both errors were visible and immediate — the panel
   validates before applying, which the runbook did not know and is worth
   knowing: it is safer than the backlog item assumed.

A first draft carried nine rules (v4/v6 duplicates, ICMP drops). Reduced to
three, because empty address fields cover both families and ICMP is already
matched by the *Ping allow* policy above.

### What was applied

Policy `tailnet-only`, placed after *netcup Ping allow* and before *Implicit
Rules*. Policies are separate objects and can be dragged into order in the
server's firewall page.

| # | Direction | Protocol | Action | Sources | Src Ports | Destinations | Dst Ports |
|---|---|---|---|---|---|---|---|
| 1 | INCOMING | UDP | ACCEPT | any | any | any | 41641 |
| 2 | INCOMING | TCP | DROP | any | any | any | any |
| 3 | INCOMING | UDP | DROP | any | any | any | any |

The panel offers no `ANY` protocol, hence two drop rules. Outbound untouched:
implicit accept plus netcup's SMTP block. Port 41641 confirmed on the host
first: `ss -ulpn | grep tailscaled` → `0.0.0.0:41641` and `[::]:41641`.

### Verification, in the order run

| Check | Result |
|---|---|
| Fresh SSH over tailnet, `-o ControlPath=none` (lockout runbook §0) | `NEW-SESSION-OK` |
| `tailscale status` before and after | arch-thinkpad `direct` over IPv6 :41641 both times — the UDP accept matches |
| The SSH session that was open *before* saving | **stayed open.** Either the panel tracks connection state or its drop applies to new flows only; the record can't tell which, and the operational lesson is the same as 2026-08-31: an old session proves nothing |
| Off-tailnet, phone on mobile data: `nc -zvw3 <NETCUP_IPV4> 22` | `timed out`, exit 1 |
| Off-tailnet: `ping -c 3 <NETCUP_IPV4>` | 3/3 replies, ttl 53 — packets reach the host, only the port is dropped |
| `sudo systemctl reboot` (run on the host; `ssh host sudo …` prompts for a password and is the wrong tool), then fresh SSH | back in ~35 s; `tailscale status` `direct` again |

Not tested: whether *IPv6* inbound TCP is dropped from off-tailnet — the phone
test was IPv4. Empty address fields are documented as any-family, so this is
`inferred` true; a `nc -6` from an IPv6-capable off-tailnet host would close it.

### What the outer ring now means

Two independent controls: nftables on the host (`input policy drop`, ADR-0029
makes it the single host-side control) and the panel firewall, which a root
compromise of the host cannot edit. Both had to be right for the phone test to
time out; both were.

The failure mode the backlog item warned about — Tailscale silently falling
back to DERP relays — did not occur: `direct` before and after.

## restic → B2 (runbook §0–§8, all run)

**B2 console/CLI (laptop).** Bucket `netcup-vps-restic-k7q2` (the plain name
was taken), private, SSE on, Object Lock off, lifecycle *keep prior versions
30 days*. The web console only offers Read / Write / Read+Write presets, so the
write-only key was made with the `b2` CLI (`pipx install b2`; not in Arch's
repos) using the **master key**, then `b2 account clear`. A first attempt with
`restic-arch-thinkpad`'s key failed correctly — bucket-scoped, no `writeKeys`.
`b2 key list --long`: `netcup-vps-restic`, bucket-scoped, exactly
`listBuckets,listFiles,readFiles,writeFiles`. The console never showed the
API-created key even after logout/login; `b2 ls` with the key itself proved it.
Second key `laptop-vps-restic-admin` (Read+Write preset) for the laptop.

**A key secret was pasted into the chat by mistake** (the `key create` output
carries ID and secret on one line, and the delete-echo above it looks similar).
Deleted and recreated on the spot. Lesson for ADR-0034: a key **ID** is safe to
paste; a key **secret** never is, and `key create` is the one place both appear.

**VPS.** `apt install restic`; `/etc/credstore/{restic-password,restic-b2-key-id,restic-b2-key}`
written with `read -rs` (no history, no `ps`), 0600, no trailing newline;
wrapper `/usr/local/bin/restic-b2` reads `$CREDENTIALS_DIRECTORY` or
`/etc/credstore`. `findmnt`: `@ @snapshots @cache @home @log @containers`, so
paths are `/ /boot /home /var/log` with `--one-file-system` (`/boot` is ext4,
ADR-0025, and would otherwise be skipped). First backup 870 MiB / 409 MiB
stored in 14 s. Units: `restic-b2.{service,timer}` daily + 1 h jitter,
`restic-b2-check` monthly 5 %, `restic-b2-stale` daily 09:00, all with
`LoadCredential=`; `OnFailure=ntfy-alert@%n.service` posting to a random
ntfy.sh topic (topic in `/etc/credstore`, phone subscribed). The timer-driven
run under systemd succeeded first time (the laptop's units had failed here on
the missing cache dir). **Both `*-wait-online` units were disabled**, making
`network-online.target` hollow; `ifupdown-wait-online.service` enabled.

**Measured outcomes.**

| Check | Result |
|---|---|
| `list locks` after two backups with the write-only key | empty both times — locks are hidden, not deleted |
| `forget <id> --prune` from the VPS | **succeeds** — soft delete; the key can make snapshots disappear from restic's view |
| `b2 ls --versions …/snapshots/` from the laptop | the two forgotten snapshots (`3e9ff1e1`, `7fd8722d`) still present as prior versions under hide markers — **recoverable for 30 days** |
| Restore from the laptop, `diff` against live `/etc/hostname` | `RESTORE=PASS` |
| `restic forget … --prune` from the laptop | 1 snapshot removed, prune clean |
| Deliberate `ntfy-alert@` fire | notification on the phone in seconds |

So the deletion-resistance property is: *the host can hide, not destroy*; the
30-day lifecycle is load-bearing and the VPS key cannot change it. ADR-0034's
table records this. Object Lock and a "recover after malicious hide" drill are
stage 3.

**Transport faults, again operator-side, none on the machine:** `apt` run as
`luiswg` instead of root; `HISTFILE= zsh` pasted as the first line of a block
(the new shell swallows the rest — twice); bash `read -rsp` on zsh (`-p` is
coprocess; zsh wants `read -rs 'VAR?prompt'`); `--keep-last 0` rejected, so the
negative test needed a forget-by-ID. All folded into the runbook.

## Stage 1, evening: 1.1–1.3c

Done after the restic work, all as planned, all measured (details and configs
in [[../runbooks/coding-rig-stage-1|the stage-1 runbook]]):

- `dev` and `auto` principals; `dev` gets linger, tmux, rootless podman
  (subuid ranges allocated by `useradd`), a 5 G slice cap; sshd's
  `AllowUsers luiswg` had to gain `dev`.
- uid-keyed egress in the existing `output` chain: `dev`/`auto` cannot reach
  RFC 1918, link-local or the tailnet; internet and loopback open. Counter
  proved it. **And the rule catches rootless-container traffic too** — the
  plan's last unknown, closed from inside the OpenCode container.
- Claude Code 2.1.269 with the sandbox fail-closed and `sandbox.credentials`
  denies; the three acceptance tests pass. Denied writes exit 0 and produce
  no file — verify from outside.
- Codex 0.154.0 with beta permission profiles. Two corrections found by
  running it: denying `~/.codex` breaks Codex itself (its binary lives there;
  deny `auth.json` instead), and Linux bwrap has no native `**`, so
  `glob_scan_max_depth` is required for the `.env` deny to mean anything.
- OpenCode 1.18.30 in a Debian container, two mounts, uid 1001 — `~/.ssh`
  absent, tailnet blocked, internet OK. Logged in to OpenAI and DeepSeek.

**The interruption in the middle:** after a shift and a hotspot change,
`ssh` returned `Permission denied (publickey)` for every user. `ssh -v` said
`agent contains no identities` — KeePassXC had locked (ADR-0004's timer), so
the agent held no key. Not a server fault. Added to the lockout runbook's
diagnosis list.

**Transport faults, this half:** `sudo -i` at the top of a paste block
swallowed the whole block (same shape as `HISTFILE= zsh` earlier); three
heredocs in one paste through tmux garbled the second; bash `read -rp` vs zsh
`read 'VAR?prompt'` again, this time the other way round (root on the VPS is
bash). The runbook now opens with the paste rule.

## Records touched

- `backlog/panel-firewall-outer-ring.md` and `backlog/restic-client.md` → resolved.
- `runbooks/restic-b2-backup.md` → `provenance: measured`, §4a outcome and the paste/zsh gotchas folded in.
- ADR-0034 table: B2 key capabilities measured; ntfy topic row; master/admin keys noted as laptop-only.
- `backlog/_index.md`, `_index.md` status paragraph.
- `runbooks/panel-firewall-outer-ring.md` → `provenance: measured`, with the
  two UI facts (validation, empty-means-any, Dst Ports) folded in.
- `sessions/_index.md`.
- The panel rule set lives here until `state/` exists
  ([[../backlog/netcup-collector|netcup-collector]]).

## Redaction

The PDF print of the panel page and the terminal pastes carried the public
IPv4, the IPv6 prefix, the NIC MAC and both Tailscale addresses. None of them
enter this vault; tokens per `_index.md`. The PDF stays in `~/Downloads`.

## Next

Stage 0 of the plan is complete. Stage 1.1: `dev` user, linger, tmux, zram,
rootless podman, `MemoryMax` — the coding rig proper.
