---
title: "2026-08-30 — netcup-vps rebuild"
type: session
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
created: 2026-08-30
updated: 2026-08-30
tags: [session, netcup, debootstrap, install, kitty, ssh]
---

# 2026-08-30 — netcup-vps rebuild

From the stock netcup Debian image to a `debootstrap`ped Debian 13 with a
chosen disk layout, key-only SSH and a verified console fallback. Seven ADRs
(ADR-0021…ADR-0027), two runbooks, three build scripts.

## The premise was wrong and the correction reframed the task

The opening request was to reinstall from the panel's *Official Images* picker
and select partitions and subvolumes during install. **The picker cannot do
that** — netcup's documentation gives exactly two fixed layouts, no filesystem
choice, no subvolumes. The Arch image in the same list is subject to the same
installer.

So the question was never *which image*. It was *don't use the image installer*.
That reframing produced [[../decisions/20260830-debootstrap-over-image-installer|ADR-0023]]
and everything downstream.

**Reading the vendor documentation before answering was the highest-value single
action of the session.** The intuitive answer — pick a different image — would
have consumed a reinstall cycle and produced the same unusable layout.

A secondary finding from the same page: **the panel's image list is stale in a
way that matters.** Debian 11 Bullseye and openSUSE Leap 15.6 were both offered
as fresh installs; Leap 15.6 reached EOL on 2026-04-30. That killed Leap as a
candidate in [[../decisions/20260830-debian-over-rhel-family|ADR-0024]] on a
checkable fact rather than a preference.

## Decisions front-loaded, and the operator sharpened the biggest one

Six ADRs were settled before `sgdisk` ran. That ordering was deliberate: LUKS,
BIOS-versus-UEFI and the `/boot` filesystem are all repartition-to-reverse.

The LUKS argument is worth recording because **the operator's version was better
than the advisor's**. The advisor's case was cost/benefit — hypervisor reads
guest RAM, unattended reboots need a passphrase, dropbear is a lot of apparatus.
The operator's was one line:

> LUKS encryption is meaningless as the server will be online 99.99% of the time.

Same conclusion, and it identifies the actual mechanism: at-rest encryption
protects data at rest, and this machine is never at rest. That framing is what
[[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]] records, and it
generates the residual obligation — no long-lived plaintext secret on disk — more
directly than the cost/benefit version does.

**The `@cache` question is the one the operator caught and the advisor had
dropped.** Asked mid-build, after stage 1 had already run. The correct answer was
that it is cheap to add, but the *procedure* is not obvious: `debootstrap` had
already written into `/var/cache`, and mounting a fresh subvolume over a
populated directory hides the content rather than moving it — silently. Recorded
as [[../decisions/20260830-subvolume-layout#The `@cache` addendum, and why it is the useful part of this record|ADR-0026's addendum]],
which is the most reusable paragraph produced this round.

## Everything that went wrong was transport

The install ran clean on the first attempt in both stages. Eleven interruptions
occurred; **nine were between the operator and the machine.**

### kitty's paste dialog, twice, for two different reasons

A full-screen *"The text to be pasted contains terminal control codes"* prompt
appeared on every paste. Initially misattributed by the operator to netcup —
understandably, since it appeared during an SSH session — and it is entirely
local: kitty 0.48.2's `paste_actions`, rendered by the `ask` kitten.

The first occurrence was real: a literal `^C` was in the clipboard, visible as
`hostnamectl^C` in the pasted output. Neither dialog option helped — *Sanitize*
flattened the block into one line, *Paste anyway* fed the control code through.

Fixed with one line in a config file that **did not exist**, which caused a
reasonable doubt from the operator:

```conf
paste_actions replace-dangerous-control-codes,confirm-if-large
```

An empty `~/.config/kitty/` is what an untouched kitty install looks like —
kitty ships no default config file, all defaults are compiled in.

**Two advisor errors here, both from recalling rather than looking up.** The
terminal was identified as kitty from the dialog's wording before confirming it
(correct, but a guess at the time — Ghostty has a similar feature). Then
`kitty --debug-config` was offered as the verification and **does not exist in
0.48.2**; sources disagree about whether it was removed in 0.21.0 and re-added.
The empirical test — paste two lines, see whether a dialog appears — was the
authoritative check all along and should have been first.

### Then the same symptom with a different cause

After the config fix, multi-line paste still failed in the **rescue** shell: each
line overwrote the previous and nothing executed. Different mechanism — Grml's
minimal shell has no bracketed-paste support, so line endings arrive as carriage
returns. And a heredoc pasted into local zsh was swallowed whole as a single
quoted string by `bracketed-paste-magic`.

Three distinct paste failures, three mechanisms, one corrective:

> **Never paste multi-line content into any shell. Write it to a file, review it,
> pipe or copy it.**

This is the same hazard class as the vault's existing note that the Hetzner VNC
console corrupts pasted SSH keys. It has now cost time on two providers.

### `bash -s` and interactive `sudo` cannot coexist

Piping a script with `ssh 'bash -s' < file` gives sudo no TTY. `ssh -t` fails
("stdin is not a terminal"); `ssh -tt` succeeds and is worse — **sudo reads the
password from the same stdin as the script and consumes the next line as a
password attempt.** Observed: `Sorry, try again` while the operator had typed
nothing, and a line of the probe script was submitted to PAM and logged.

Rule: a piped script is fully non-interactive (`sudo -n`, accept loud failures)
or it is `scp`'d and run in an interactive session. Not both. This is recorded in
[[../runbooks/rescue-system-entry#Terminal quirks — read before typing|the rescue runbook]]
and in stage 3's header comment.

### Commands run against the wrong machine

A block intended for arch-thinkpad was pasted into an SSH session
(`nvim: command not found` on a minimal Debian). Later, and more consequentially,
**a `sudo systemctl reboot` on the server partially succeeded** — the D-Bus error
it printed was systemd's client failing to *report*, while the reboot itself went
through and killed the session. The operator did not notice the disconnection and
ran the next six commands locally, entering the *server's* console password into
arch-thinkpad's `sudo` six times.

Per the vault's existing note, three failures trips PAM faillock for ten minutes.
The tell was in the output the whole time — `zsh: command not found: dpkg`, eza's
listing format, `/run/systemd/` timestamps from the laptop's boot.

Corrective: a red `PS1` on the server. Cheap, and it would have prevented the
entire detour.

### The rescue password, and a wrong hypothesis

`ssh root@…` into rescue rejected the password. The operator suspected earlier
SSH hardening. A search of past sessions showed that hardening belonged to the
**decommissioned hive server**, not netcup — and more importantly that the
hypothesis was structurally wrong: rescue is a separate OS with its own
`/etc/shadow`, so nothing configured on the installed system applies.

The actual cause: **netcup's rescue system has its own generated password, shown
in the SCP at activation.** The panel's root-password reset targets the installed
system. Resetting it repeatedly, as the operator had, cannot work.

This also surfaced something worth knowing before it is needed: `passwd -l root`
means the panel's reset is **not a live lever** against the running system — it
is a rescue cycle — and whether it clears the `!` lock prefix at all is
`inferred`, not tested. The `luiswg` console password is the real fallback. In
[[../runbooks/_index#The recovery path this machine actually has|the runbook index]].

### `IdentitiesOnly` filtered the agent key on first contact

After a clean boot and a correctly-matching host key fingerprint, SSH returned
`Permission denied (publickey)`. `ssh -v` showed **no `Offering public key:` line
at all** — six `Will attempt` lines, every one a default filename, every one
`type -1` (does not exist).

Cause: `IdentitiesOnly yes` under `Host *` restricts offers to keys named by
`IdentityFile`; with none specified it falls back to default filenames, and the
agent-held key is filtered out despite being loaded. **This is already a known
gotcha in the vault** and it still cost a round, because the symptom presents as
a server-side rejection.

The fix — a host block with `IdentityFile` pointing at the **`.pub`** file, above
any `Host *` — is in
[[../runbooks/ssh-lockout-recovery#A — `Permission denied (publickey)`|the lockout runbook]].
Every new host needs its own block; that is a consequence of the `Host *`
setting, not a quirk of any server.

## Two real system findings

Everything above is transport. Two things were genuinely about the machine.

**`net.ifnames=0` is load-bearing and invisible.** The interface presents as
`eth0` with `enp0s3`/`ens3`/`enx…` as altnames — meaning something forces the
legacy name. Confirmed in the stock image's `/proc/cmdline`. A `debootstrap`
without carrying it forward boots with the NIC as `ens3`, the `eth0` stanza
matching nothing, and no network — recoverable only by console. It became a stage
2 postcondition asserting on the **generated `grub.cfg`**, not on
`/etc/default/grub`, since asserting on the input would pass while `update-grub`
failed.

**`--no-install-recommends` omitted D-Bus, and the failure looked like something
else.** First boot: `hostnamectl` returned *"Failed to connect to system scope
bus"*. The hostname was correct — only the query tool was broken. But the same
gap breaks `systemctl --user`, which rootless podman quadlets require, so it
would have resurfaced later as a mysterious podman failure.

The diagnosis needed care: PID 1 *was* systemd, and the install log line
*"dbus.service is a disabled or a static unit, not starting it"* is normal —
`dbus.socket` is the enabled unit. The packages were installed into an already
running system, so the socket was never activated in that boot. A reboot fixed
it. The general lesson is in
[[../decisions/20260830-debootstrap-over-image-installer#Consequences|ADR-0023]]:
a minimal base needs an explicit check that the *intended workload's*
prerequisites exist, because nothing will pull them in.

## Panel state, recorded here because there is no `state/`

Measured 2026-08-30 in the SCP. Belongs in generated state; there is no collector
([[../backlog/netcup-collector|netcup-collector]]), so it lives here.

- **Cloud firewall is effectively off.** Rules evaluate top-to-bottom until first
  match; the implicit system rules at the end accept all inbound *and* all
  outbound. Only live restrictions: outbound SMTP blocked on 25/465/587, and
  explicit ICMP/ICMPv6 allows both directions. The SMTP block will silently break
  any future mail-sending service.
- **Storage and network drivers are `virtio`.** netcup's documentation warns that
  changing the storage driver can render the server unbootable and that SATA/IDE
  severely affects snapshots. Read, never change.
- **UEFI Boot toggle is off**, consistent with the absent `/sys/firmware/efi`.
- **CPU topology is 4 sockets × 1 core**, which presents false NUMA distance to
  the guest scheduler. 1 × 4 is more accurate. Needs a power cycle; filed rather
  than done.
- **IPv6 rDNS is unset**; IPv4 rDNS is the `<SERVER_ID>.ultrasrv.de` default.
  The v6 address changed during the rebuild from netcup's MAC-derived EUI-64 to a
  clean `::2`, so forward and reverse now disagree by omission.
- One exportable snapshot slot remains. Not spent — the disk was empty.

## What was not done

Stage 3 was written and the operator elected to run all of it. **Stage 3b — the
lockdown — was deliberately not written**, because it is gated on verifying
Tailscale reachability and that verification has not happened. Writing it in
advance would produce a script whose safe execution depends on a condition it
cannot check. See [[../backlog/stage-3b-lockdown|stage-3b-lockdown]].

The trust-direction question from
[[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]] was raised by a
concrete request — server-side SSH for file copy and git — and tabled by the
operator rather than decided. It is `proposed`, not `accepted`, and it blocks the
Tailscale ACL.
