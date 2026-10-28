---
title: "2026-08-31 — stage 3b lockdown"
type: session
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-31
commit: pending
created: 2026-08-31
updated: 2026-08-31
tags: [session, netcup, nftables, tailscale, ssh, terminfo, multiplexing, zsh, termux, pixel-graphene]
---

# 2026-08-31 — stage 3b lockdown

The machine went from a reachable base with a permissive firewall to a host
closed to the public internet and reachable only over the tailnet, verified
across a reboot. One ADR accepted (ADR-0027), one written (ADR-0029), one backlog
item closed.

**The pattern from the rebuild session repeated exactly.** Everything that went
wrong was between the operator and the machine; the machine did what it was told
throughout. This time the transport failures were more interesting, because one
of them produced a *false confirmation* rather than a visible error.

## The session did not start with the firewall

It started with the operator asking what was in the backlog, and then declining
the first thing recommended.

`luks-second-keyslot` was carried at **high** priority on arch-thinkpad on
lockout grounds. The operator's objection: the passphrase is memorised, held in
KeePassXC, synced to the phone via Syncthing, and the database is backed up to
Proton Drive. **The objection is correct and the priority was wrong.** A second
keyslot insures against losing or corrupting the one passphrase; the header
backup — already taken — is what insures against losing the header. With three
independent copies of the passphrase, the item guards a risk that has already
been retired by other means.

Worth recording as a class, not an incident: **an item filed under "lockout"
inherits urgency from the word rather than from the operator's actual recovery
position**, and the vault has no view of what is in KeePassXC. Repricing that
item is left to a session touching arch-thinkpad's backlog.

## The terminal detour, which was the actual productivity work

The operator asked for two things before any server work: multi-line paste that
does not execute on arrival, and `nvim`. What followed was four rounds of wrong
diagnosis before the real cause surfaced, and it is worth writing down in order,
because the *sequence* is the lesson.

1. **Bracketed paste was blamed and enabled.** `set enable-bracketed-paste on`
   in `~/.inputrc`. Paste then arrived as one visual line with `^J` where the
   newlines were. Reported as "not quite what I expected" — correctly.
2. **That was described as working-as-designed**, which was true but unhelpful.
   bash's readline holds the newlines in the buffer and executes nothing until
   Enter, so the *safety* property was satisfied; the *legibility* one was not,
   and the operator's actual complaint was legibility.
3. **zsh was recommended** for its native multi-line buffer display. Installed,
   `.zshrc` written, `chsh` run. The prompt stayed `$`.
4. **The operator then reported the real symptom**: history was being truncated
   on multi-line commands, "this isn't just an aesthetic fix". That reframing is
   what led anywhere.

The cause of all of it was one line that had been on screen for several rounds
and was read as noise:

```
WARNING: terminal is not fully functional
Press RETURN to continue
```

kitty announces itself as `TERM=xterm-kitty`. SSH passes that name to the remote,
and **Debian had no terminfo entry for it**, so every program needing cursor
addressing fell back to dumb-terminal behaviour: readline could not redraw a
multi-line buffer, and `less` refused to page git's output. Same shell, same
paste, different result — purely because the far end did not know what the
terminal could do.

```bash
# NetCup server
sudo apt install -y kitty-terminfo     # NOT ncurses-term — see below
```

**`ncurses-term` was tried first and was wrong.** It is the obvious package and
it does not carry kitty's entry; kitty's terminfo is shipped by kitty and Debian
packages it separately as `kitty-terminfo`. One wasted install, and a reminder
that the vault's *look it up, never recall it* rule applies to package contents
as much as to command flags.

The packaging-independent form, which works against any host and needs no root:

```bash
# Arch (ThinkPad)
infocmp -x xterm-kitty | ssh <host> 'tic -x -o ~/.terminfo /dev/stdin'
```

**The `xterm-kitty` gap is already in the vault** — the 2026-08-30 session
records Grml lacking it in the rescue environment. It cost time again on a
different host, which makes it a standing property of this terminal rather than a
quirk of any one machine. `kitten ssh` copies terminfo automatically on connect
and is the durable fix; it has not been adopted.

With terminfo correct, **bash rendered multi-line pastes properly too.** The
shell was never the problem. zsh was still adopted — it is what arch-thinkpad
runs, and it keeps pasted newlines as real lines in history rather than flattening
them — but it was adopted for parity, not as the fix, and the two were conflated
for most of the detour.

Two smaller zsh findings, both from pasting this vault's own command blocks:

- **`# NetCup server` was executed as a command.** zsh does not treat `#` as a
  comment in interactive shells by default. `setopt INTERACTIVE_COMMENTS` fixes
  it. Every platform-labelled code block in this vault triggers this on a fresh
  zsh.
- **bash's active-region highlight persists after `^C`.** readline marks pasted
  text in reverse video and `^C` aborts before clearing it. `set
  enable-active-region off`. Moot under zsh.

## The verification that lied

This is the finding worth carrying forward.

The `chsh` to zsh appeared not to take: `getent passwd` reported `/usr/bin/zsh`,
the login shell was bash. Then, after nftables went to `policy drop` and the
blanket port 22 accept was removed, `ssh netcup-public` **returned a prompt**.

Both had the same cause, visible in a line that had been printed on every
disconnect all session:

```
Shared connection to <SERVER_ID>.ultrasrv.de closed.
```

That message means the session was a channel inside a multiplexed master
connection, not a login of its own. `Host *` in the operator's `~/.ssh/config`
sets `ControlMaster auto` with `ControlPersist 10m`. A master opened before the
`chsh` kept serving the old shell; a master opened before the firewall change kept
working after it, because **`policy drop` filters new connections and says
nothing about established ones** — the `ct state established,related accept` rule
is doing exactly its job.

So the first lockdown verification reported *public SSH still works* against a
firewall that was already correctly closed. The natural next move from that
reading — assume the ruleset is broken, start editing it — would have been
destructive, on a machine whose only fallback is a VNC console.

Three things made it recoverable rather than expensive:

- The scheduled revert was armed. It fired during the confusion and returned the
  machine to permissive on its own. **A revert firing mid-session is the
  mechanism working**, not a mistake; the cost was re-running two commands.
- The negative test was re-run with `-o ControlPath=none`, which the config
  cannot override because command-line options win.
- The `ControlPath none` written into the `netcup-tailnet` block was itself being
  ignored, which exposed the second half of the cause.

**`ssh_config` takes the first value it obtains for each option, so a `Host *`
block at the top of the file overrides every block below it.** The operator's
`Host *` was first. This is the same mechanism as the `IdentitiesOnly` gotcha
already recorded in the lockout runbook from the rebuild session — *first match
wins, and `Host *` at the top wins everything* — surfacing a second time through
a different option. It has now cost a round on two separate settings and should
be treated as a property of that config file, not as two coincidences.

The standing rule, now in the runbook: **anything being verified over SSH after a
change to shells, keys, or the firewall must run on a fresh connection.**
`ssh -O exit <host>` first, or `-o ControlPath=none` on the check itself.
Otherwise the answer describes a connection made in the past.

## The lockdown itself was uneventful

Which is the point of how it was sequenced. Steps in
[[../backlog/stage-3b-lockdown|the item]]; outcome there too.

One hazard the item did not anticipate, caught before it could fire:
**Debian's stock `/etc/nftables.conf` opens with `flush ruleset`, which wipes
every table — including the four that Tailscale manages through `iptables-nft`
and marks *"do not touch"*.** A revert file made by copying the stock config
would have torn out the tailnet's own firewall rules at the moment the tailnet
was the only way in. `tailscaled` rebuilds them eventually; *eventually* is not a
property to depend on during a recovery.

Both files are now scoped to the one table this machine owns:

```nft
table inet filter
delete table inet filter

table inet filter { ... }
```

The bare declaration before the `delete` is load-bearing: `delete` on a
non-existent table aborts the whole file, which in the revert file means a
recovery that fails on the one path it exists for.

`iifname` rather than `iif` matters here too and was already right in the stage 3
baseline: `iifname` resolves the interface at packet time, so a ruleset
referencing `tailscale0` loads correctly at boot even though `tailscaled` has not
yet created it. `iif` would require the interface to exist at load time and would
have made the boot order load-bearing.

## Step 6 was closed rather than done

The operator recalled, unprompted, that sshd and `tailscaled` had raced on the
decommissioned hive. The vault's surviving record of that is narrower — hive's
problem was `ssh.socket` needing to be disabled before `ListenAddress` was
respected at all — but the instinct pointed at the right hazard, and it is the one
the backlog item itself had flagged.

Binding sshd to the tailnet address would mean sshd depends on `tailscale0`
existing at start. The two failure modes are asymmetric: a firewall that fails to
load leaves the machine **reachable but exposed**, recoverable remotely; an sshd
that fails to bind leaves it **closed and unreachable**, recoverable only by
console — and it fails at reboot, when nobody is watching. Preferring the first is
[[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]].

The cost is real and is recorded there: nftables is now a single point of
control, and `nft flush ruleset` during a debugging session opens the box to the
internet without anyone intending it.

## ADR-0027 was accepted, and the blocker was a misreading

The record had been `proposed` since 2026-08-30, blocking the Tailscale ACL. The
operator's condition for accepting was an explanation of how git authentication
works without a server-side SSH key.

**The premise of the question was wrong, and clearing it settled the record.**
ADR-0027 does not forbid a key on the VPS; it forbids a key pointing *upward*.
GitHub sits to the right of netcup-vps in the trust order, so a key authenticating
to it was permitted all along. The only convenience actually surrendered is
server-initiated `scp` to the endpoints, which the operator accepted without
argument once it was the only thing on the table.

A per-repository deploy key differs from an account key in scope, not mechanism:
generated on the server, public half added under that repository's Deploy keys,
addressed through a `Host` alias because every deploy key talks to `github.com`
and SSH otherwise cannot tell them apart. A keypair was generated and its public
half stored in KeePassXC; **registration in the repository settings is not
verified** and stays in
[[../backlog/vps-trust-direction|vps-trust-direction]].

The misreading is available to the next reader, which is why it is recorded in
the ADR itself: *"the VPS holds no credentials"* is not what that record says, and
an agent reading the title alone would refuse a deploy key it should be writing.

## Reboot as verification

A deliberate reboot closed the round: `policy drop` persisted, `tailscaled` and
`nftables` came back active, tailnet SSH worked, public SSH timed out.

Worth stating plainly because the distinction is the vault's central rule:
before the reboot the lockdown was *currently true*; after it, *persistent*. The
`systemctl reboot` also looked like it had failed — the prompt returned and the
session stayed responsive for several seconds before the connection reset. That
is the same ambiguity that cost the 2026-08-30 session six commands typed into
the wrong machine, and it resolved correctly this time only because the operator
waited instead of acting.

The reboot was not used to also clear
[[../backlog/cpu-topology-and-ipv6-rdns|cpu-topology-and-ipv6-rdns]], which needs
a power cycle and was the obvious thing to fold in. Filed, not done.

---

# After the commit

The vault was committed and pushed at this point. Everything below happened
afterwards and is appended rather than woven in, because the ordering is part of
the story — the push itself is what surfaced the next fault.

## The same `ssh_config` mechanism, a third time, and a real loss

`git push` failed with `Permission denied (publickey)` on a repository that had
worked minutes earlier.

Cause: `~/.ssh/config` had been **replaced wholesale** with the two netcup blocks
plus `Host *`, destroying the `Host github.com` block that had been there.
Without it, `IdentitiesOnly yes` from `Host *` applies with no `IdentityFile`,
SSH falls back to six default filenames that do not exist, and the agent-held key
is never offered — **exactly fault A from the lockout runbook**, reached by a
third route.

The advisor's error, stated plainly because it is the reusable part: **a config
fragment was presented in a way that read as a whole file.** A labelled
`~/.ssh/config` block containing three `Host` stanzas, with no instruction to
*add* rather than *replace*, is a whole file to anyone who pastes it. The fix
would have been one clause — "add these two above your existing `Host *`" — or
showing it as a diff.

This is now the **third instance in two days of `ssh_config` first-match-wins
semantics costing a round**: `IdentitiesOnly` on 2026-08-30, `ControlPath` earlier
today, `IdentitiesOnly` again via a deleted block. Three routes, one mechanism.

**The loss was cheap by luck, not by design.** Only GitHub had a block, and it
failed loudly on the next push. A rarely-used host would have failed silently
weeks later with nothing to reconstruct from — `~/.ssh/config` is unversioned, and
restic's timer has never actually taken a backup, so there was no snapshot either.
Two open arch-thinkpad items compounding turned an editing mistake into an
unrecoverable one. Recorded in
[[../../arch-thinkpad/backlog/dotfiles-versioning|dotfiles-versioning]], which
previously had only a hypothetical cost.

## luks-second-keyslot was repriced, and the vault's blind spot named

The session opened with the operator declining the item recommended to him; it
closed with that decision written down. `luks-second-keyslot` moved **high →
low** — not because the work is wrong, but because the risk was already retired by
the operator's passphrase custody: memorised, in KeePassXC, synced to
pixel-graphene, backed up to Proton Drive.

The generalisation is in the item and is worth more than the reprice:
**the vault cannot see what is in KeePassXC or what the operator has memorised,
so it systematically overprices any risk whose mitigation lives outside the
machine.** Items filed on recovery or lockout grounds inherit urgency from the
word. That failure is not fixable by a collector — the missing information is not
on the machine at all.

## Phone ↔ laptop SSH, and an assumption that was backwards

The operator asked for SSH between arch-thinkpad and pixel-graphene (Termux) over
the tailnet, both directions.

ThinkPad → phone worked immediately. Phone → ThinkPad failed with
`Permission denied (publickey)`, and the first diagnosis offered was the
authorized_keys concatenation trap — plausible, since the key had just been
appended to an existing file, and **wrong**.

The actual cause was in the first line of the check that was run alongside it:
**`sshd` was `inactive` on arch-thinkpad.** It had never been enabled. There was
no server. The empty journal was the confirming evidence and was read past.

Worth recording as a diagnostic ordering, not an incident: **`systemctl
is-active` belongs before every key-and-permissions theory.** A key problem and
an absent daemon present identically to the client, and one of them is checkable
in two seconds. The operator also supplied the disconfirming fact unprompted —
Termux → netcup-vps already worked, which cleared the phone's client and the key
in one observation.

A second correction from the same exchange: the advisor said to expect three keys
in arch-thinkpad's `authorized_keys`, including the machine's own. **A machine
does not authorise its own key.** Two were correct.

### The bind, and why it is the opposite of ADR-0029

Enabling sshd on the laptop adds its first deliberate inbound listener, on a
machine that roams onto untrusted networks. It was bound to the tailnet address:

```
# /etc/ssh/sshd_config.d/20-tailnet-only.conf
ListenAddress <TS_IPV4_arch-thinkpad>
PasswordAuthentication no
```

`ss -tlnp` confirms port 22 on the tailnet address only.

**This is precisely what
[[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]]
rejected for netcup-vps, and the reasoning is unchanged.** There, sshd failing to
bind because `tailscaled` had not come up strands the operator behind a VNC
console. On a laptop the same failure costs a reboot and thirty seconds, because
the machine is in his hands. **Physical access flips which failure mode is
acceptable** — the same argument, applied honestly, produces opposite decisions on
the two machines. That is the useful shape to keep.

`sshd -t` failed at first with `no hostkeys available`; Arch generates them via
`sshdgenkeys.service` when `sshd.service` first starts, so enabling the service
succeeded anyway. Not a fault, but it looks like one.

Recorded in
[[../../arch-thinkpad/backlog/undeliberate-listeners-review|undeliberate-listeners-review]],
whose premise — *nothing listens that I did not put there* — would otherwise have
rediscovered this as a finding.

### pixel-graphene got a stub folder, and an expiry warning

[[../../pixel-graphene/_index|pixel-graphene/]] existed as an empty directory
until this session. It now holds what was measured: tailnet address, the two SSH
directions, Termux's port 8022, and the fact that the phone's private key sits on
disk rather than in an agent — a different trust tier from arch-thinkpad's, not
yet recorded as one anywhere.

**The live risk on that page is node key expiry.** The Tailscale admin panel
showed pixel-graphene as *"Expires 17 hours from now"*, alone among the four
machines. When the key lapses the phone leaves the tailnet and both SSH routes
set up today stop working. Whether it was disabled in the panel is **not
verified**.

Also noted and not adopted: **Taildrop** (`tailscale file cp`) moves files between
tailnet devices with no server, no keys and no config. If file transfer rather
than a shell was the actual goal, the Termux sshd was more apparatus than the job
needed.
