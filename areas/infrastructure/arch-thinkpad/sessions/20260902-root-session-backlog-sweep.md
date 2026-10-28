---
title: "First root session: a boot race that had already fired, a backup unit that could never have run, and the lane question answered by doing the work"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-09-08
commit: pending
created: 2026-09-02
updated: 2026-09-08
tags: [session, root, sshd, tailscale, restic, apparmor, luks, dotfiles, collector, listeners, containers, shutdown]
---

# 2026-09-02 → 09-08 — First root session: a boot race that had already fired, a backup unit that could never have run, and the lane question answered by doing the work

Eighth round, and the first run **as root on the host**. Measured from inside
the session: `id` is `uid=0`, `SUDO_USER=luiswg`, `HOME=/root`, and the process
tree is `kitty → zsh (uid 1000) → sudo -i → bash → claude --resume`. So this
was `sudo -i` from the operator's own desktop session, not a root login and not
a container. Everything below was done with that authority; where a step needed
the operator's own session (the user bus, the SSH agent, GitHub, KeePassXC), it
was reached through `systemd-run --machine=luiswg@.host --user` or asked for.

Artifacts: [[../decisions/20260902-dotfiles-bare-git-repo|ADR-0030]],
[[../decisions/20260902-what-listens-and-why|ADR-0031]],
[[../decisions/20260902-plasma-login-path-accepted|ADR-0032]],
[[../decisions/20260903-agent-containment-two-lanes|ADR-0033]], the
[[../devenv/_index|devenv/]] mirror, `spec/backups` amended three times,
`spec/devenv` completed, thirteen backlog items closed, two opened, seven notes
entries. This is what did not fit in them.

## The sshd finding was already a decision; the fix needed two things the decision had not named

The failed unit was the first thing looked at. The backlog already said what to
do — bind to the tailnet, `After=`/`Wants=tailscaled.service` if it ever raced —
so the question was only whether to follow netcup's ADR-0029 (wildcard bind)
instead. The item's own argument, that physical access flips the acceptable
failure mode, held, and it was followed.

**What the item had not anticipated:** ordering alone is not enough, because
`tailscaled` signals ready before `tailscale0` has an address, and the default
start-rate limit turns five 100 ms retries into a permanent `failed`. Measured
from the journal: the last retry was 90 ms before the interface appeared. The
drop-in polls for the address and disables the rate limit. Reproduced with
tailscaled stopped, then verified for real by the 2026-09-03 reboot. Both halves
are in [[areas/infrastructure/arch-thinkpad/state-notes|notes]].

## `restic check` passing said nothing about `restic backup`

The scheduled check had failed offline and the manual re-run was guard-skipped
on the hotspot; when the operator allowed a bypass, the check passed in 52 s.
That looked like the B2 units were sound. They were not: the first T5 backup,
run as a transient unit, died on *neither $XDG_CACHE_HOME nor $HOME are
defined*. System services have no `$HOME`; `restic backup` needs a cache
directory and `restic check` builds a temporary one and carries on. **The
scheduled B2 backup had never got past its guard, so it had never had the chance
to show this.** Both units gained `CacheDirectory=` and `--cache-dir`. Verified
five days later in the least pleasant way: the timer fired three times past the
guard and got as far as the network every time.

## The guard passed on a network with nothing behind it

Those three runs (09-04, 09-05, 09-06) all fired on resume while the Wi-Fi was
associated to a network with no route out. NetworkManager's state was
`CONNECTED_LOCAL`, the device was `100 (connected)`, `network-online.target` was
satisfied, and the metered field was not `yes`. Fifteen minutes of DNS retries
each, then exit 1. The error text names DNS and the resolver is Tailscale's, so
the obvious misreading is a tailnet problem. The only signal that separates the
cases is NM's connectivity probe. The guard now requires `full`. **Still no B2
backup has landed after eight days**, which the staleness alarm has reported
correctly the whole time — the alarm's first run, on install, was its truest.

## Firefox was withdrawn from the AppArmor plan by reading the profile

The item said enforce Firefox, Discord, Spotify. apparmor.d ships
`/etc/apparmor.d/firefox` as `flags=(unconfined)` with a comment saying the
profile exists only to give the process a name; the loaded mode read `complain`
only because August's sweep had rewritten the flag. Confining Firefox is a
decision against upstream, not a step. Discord and Spotify took four
launch-and-read rounds each. Two things about the method: `aa-enforce` cannot
parse apparmor.d's syntax at all (the error names a D-Bus path, the profile is
fine — `apparmor_parser -r` loads it), and every forced stop of a test instance
left a core dump, which was resolved only by the operator opening and quitting
the apps normally the next day. The re-enforce hook is in place; the ~500
profiles that attach to binaries actually present here are Sunday's job, not the
1954 the count suggested.

## Two of the round's own mistakes, in the host lane

- A `pkill -f` pattern matched the agent's own shell command line, twice, and
  killed the session's command mid-run. The bracket trick (`spotif[y]`) is the
  fix; the lesson is that `-f` matches the caller.
- The LUKS keyslot was first keyed with the clipboard's expired contents — the
  command line itself — because KeePassXC clears the clipboard on a timer and
  `luksAddKey` prompts twice. `luksDump` cannot tell a good slot from a bad one.
  `luksKillSlot` on the bad slot, asking for the *other* passphrase, was the
  safe reversal. Rule recorded in the item: paste a secret somewhere visible
  before typing it into a prompt that echoes nothing.

Both were recoverable because of procedural controls, not a sandbox. That fact
is written into ADR-0033's negative consequences on purpose.

## The lane question was answered by the work rather than argued

Every task in the round — sshd, restic units, AppArmor, LUKS, usbguard, the
collector, masking daemons, a history rewrite — needed root and `/etc`, `/var`,
the journal or block devices. No mount set makes that container work. The
operator raised the design question directly (why re-add tools one by one to
a neutered container; whether the IPC hardening was even proportionate) and
ADR-0033 came out of it: coding in the devbox with tools baked in, administration
on the host under the vault's rules. Two revisions during the draft, both from
the operator: the tailnet stays reachable (n8n on the VPS), and VS Code is made
as safe as the wrapper by launching it without the agent socket rather than
being replaced.

## Things that were not what they looked like

- **The Codex manual "failure"** was a helper with no `--help`, not a container
  limitation; the documented invocation works from inside.
- **The ArchWiki** was not fixed by the Arch image; the packages were never in
  it. Two words on the package line at the rebuild.
- **The Seagate** was not an NTFS problem. usbguard had silently blocked a drive
  that had never been plugged in since the policy was generated; then the
  dirty flag; then it just worked under `ntfs3`. The migration item closed as
  *stay on NTFS*.
- **"Programs came back after reboot"** was Plasma session restore, not
  hibernation — `uptime -s` showed a fresh boot.
- **A `tailscaled` "restart" during shutdown** was its `ExecStopPost` cleanup
  run, not the new sshd drop-in pulling it back.
- **The T5's I/O errors** were the drive being unplugged while mounted, at the
  minute the Wi-Fi reassociated — the operator leaving for a café.

## Negative findings

- The shutdown hang leaves nothing in the journal: both occurrences end
  normally where journald stops. The debug hook is installed and has not yet
  had a shutdown to observe. `/shutdown-log.txt` absent as of 2026-09-08.
- The collector has no portability defect; a full read found only churn. The
  POSIX-lint item stays open for one `shellcheck` run, which the stale mirror
  database blocks until the next `-Syu`.
- The vault could not see that the recipe repo already had a GitHub remote,
  that the Seagate serial would reach `state/` unmapped, or that the Ethernet
  extension connector exists. Each got a mechanism, not a note.

## Accepted limitations with a named consequence

- **The host lane has no technical containment.** Named in ADR-0033.
- **Two pushed commits carried a USB drive serial** for about a day before the
  history was rewritten and force-pushed; the pre-commit hook has no serial
  pattern and still does not. The map generator now covers serials of any
  connected device, so the next new drive is mapped the first time the collector
  sees it — but only if the collector runs before the commit.
- **The collector runs as root now**, resolving the operator from `SUDO_USER`.
  Every other clone still expects to be run as the operator with `sudo` inside;
  both paths are supported and neither is asserted by a postcondition.
- **The KeePassXC agent lifetime** stalled every push in the round until the
  operator set it to three hours. Not a vault matter; recorded because the
  symptom — *Permission denied (publickey)* with a key that was there a minute
  ago — will recur and looks like a GitHub problem.
