---
title: Operational notes — arch-thinkpad
type: notes
status: living
device: arch-thinkpad
updated: 2026-09-08
tags: [notes, gotchas, diagnostics]
---

# Operational notes — arch-thinkpad

Hand-written. **The collector does not own this file.** Everything else in
`state/` is generated and must not be edited by hand.

Each entry carries a provenance marker:

- **measured** — observed directly on this machine, with the command that showed it
- **inferred** — reasoned, not demonstrated
- **inherited-unverified** — carried from an earlier document, not re-checked

The section is for diagnostics whose error message points somewhere other than
the actual cause. If an entry does not have that shape, it is probably a
decision and belongs in `decisions/`.

---

## Shell and tooling

### `alias grep='rg'` changes the interface, not just the output
**measured 2026-08-21**

`grep -n 'a\|b'` returns nothing rather than erroring. GNU grep treats `\|` as
BRE alternation; ripgrep treats it as a literal pipe character and searches for
the literal string `a|b`. A silent empty result is indistinguishable from a
genuine negative.

The 2026-08-19 session notes record this alias as "set and then removed the
same night." It was not removed — `type grep` confirms it is still live.
`alias diff='difft'` is also live and was retained knowingly; difftastic takes
exactly two paths and has no `-r`, so `diff -r` fails with
`unexpected argument '-r'`.

Bypass with `command grep` / `command diff`. Scripts are unaffected — aliases
are interactive-only.

**"Scripts are unaffected" is true and reads as a smaller blast radius than it
is.** The interactive shell is where checks get *composed and tried* before they
become postconditions, so the alias reaches the script by way of the person
writing it. The measured failure came from exactly there: an interactive reading
of the host's `grep` version, copied forward. A rule that only binds at
execution time does not bind where the mistake is made.

**Amended 2026-08-25.** Both aliases are now commented out at `.zshrc:36` and
`:37`. The finding stands and the reason for keeping this entry is sharper than
it was: the alias produced **a wrong reading of the machine's own toolchain**.
The first attempt to record the host's `grep` version returned "ripgrep 15.2.0"
— a correct-looking command answering about the wrong binary. That is why the
collector resolves tools with `readlink -f` and why
[[areas/infrastructure/arch-thinkpad/spec/_schema#Conventions|the host-question rule]] names the shell-visible
name as untrustworthy. An alias is one edit from returning; the rule does not
depend on this one staying commented out.

Note the two divergence axes are independent. This one is *interactive shell vs
script*; the other is *container vs host*, and closing the second
([[areas/infrastructure/arch-thinkpad/decisions/20260825-devbox-base-is-arch|ADR-0015]]) does nothing about the
first.

### `$SHELL` is stale after `chsh`, and so is kitty
**measured 2026-08-19**

`$SHELL` is set once at login and inherited; it reports the old shell until the
next full logout regardless of `/etc/passwd`. Worse, kitty resolves the login
shell when the *kitty process* starts, so new windows from a pre-`chsh` kitty
instance also open the old shell.

Authoritative: `getent passwd $USER | awk -F: '{print $NF}'` for configured,
`ps -p $$ -o comm=` for actual. `exec zsh` switches in place without a logout.

### Pasting a multi-line block beginning with `sudo -i`
**measured**

`sudo -i` opens a new shell and the remaining lines are delivered to it
mid-stream as bracketed-paste input rather than executed. Symptoms: the block
appears to do nothing, then runs unexpectedly when you type `exit`, and stray
`^[[200~` sequences appear. Run `sudo -i` alone, or use `sudo <command>` per
line.

### `sed` delimiters and shell pipes
**measured 2026-08-21**

`s|pattern|replacement|` breaks when the pattern contains a shell pipe — sed
splits fields at every `|`. Symptom is `unknown option to 's'`, but a botched
substitution can also *succeed* and mangle the file silently. Use `#` or `,` as
the delimiter when the content contains pipes, or edit the file directly for
anything non-trivial.

### The live ArchWiki is unreachable to any non-browser client
**measured 2026-08-23**

`wiki.archlinux.org` sits behind Anubis, a proof-of-work interstitial. A fetch
returns HTTP 200 with a page whose body is `Access Denied: error code
<hex>` — so it presents as a block or a rate-limit, and the natural response is
to wait and retry or to try a different article. Neither works. `?action=raw`
does not bypass it either; the interstitial is served before MediaWiki sees the
request. Nothing is wrong with the URL, the article name, or the network.

The cause is that solving the challenge requires running JavaScript. Any client
that does not — `curl`, an agent's fetch tool — gets the same page forever.

Use the local copies instead. `arch-wiki-lite` provides `wiki-search <term>`
against a plain-text dump; `arch-wiki-docs` holds the HTML. Both were installed
at version `20260702`, so they lag live by roughly seven weeks — fine for hook
semantics and config syntax, not for anything that changed recently.

Better still, ask the tool. `mkinitcpio -H <hook>` and `sbctl verify --json`
both turned out to document things the wiki did not, and `mkinitcpio.conf(5)`
explicitly defers to `-H`. See the lookup rule in the root `CLAUDE.md`, which
was reordered on 2026-08-23 for this reason.

**Not reachable from the devbox container.** `/usr/share/doc/arch-wiki*` is not
mounted, so an agent inside it cannot run `wiki-search` — see
[[areas/infrastructure/arch-thinkpad/backlog/devbox-wiki-mount|backlog/devbox-wiki-mount]].

---

## Networking

### `SelfNode.capMap` is not a reliable Mullvad-grant diagnostic
**measured 2026-08-21**

State reference §1.4 asserted that an empty `capMap` means the Tailscale
control plane is not sending the Mullvad grant, and that "nothing on this
machine can fix it."

That is **wrong**. As of 2026-08-21 `sudo tailscale debug netmap | jq
'.SelfNode.capMap'` returns `null` while `tailscale exit-node list
--filter=Germany` returns a full list of working nodes. The grant is arriving
by some path the netmap dump does not expose, or the field moved between client
versions.

**The functional test is `tailscale exit-node list`.** Do not infer grant
status from the netmap.

What changed: the device was enabled in the Tailscale Mullvad add-on settings
*and* `tailscaled` was restarted (`systemctl restart tailscaled`, then
`tailscale down` / `tailscale up`). **Which of the two was responsible was not
isolated.** If this recurs, try the restart first — it is cheaper than a
console round trip.

Same failure shape as the Fn media keys: a correct observation, a wrong
inference drawn from it, recorded as fact.


### Three daemons listen that were never deliberately enabled
**measured 2026-08-21**

`ss -tulpn` shows, beyond Syncthing and tailscaled:

| Listener | Bind | Origin |
|---|---|---|
| `ollama` | `127.0.0.1:11434` | Installed and enabled at boot; in no document |
| `passimd` | `0.0.0.0:27500` | Ships with `fwupd`; advertises firmware metadata over mDNS |
| `avahi-daemon` | `0.0.0.0:5353` | mDNS service discovery |

None is reachable — the nftables `input` chain is default-drop and none of these
ports is opened. But two bind all interfaces, and the working assumption
"nothing listens that I did not put there" is therefore false.

Recorded, not acted on. Whether to disable `passim` and `avahi` is a hardening
decision for after the documentation work.

---

## SSH and credentials

### `IdentitiesOnly yes` filters agent-held keys, not just on-disk keys
**measured 2026-08-17**

With `IdentitiesOnly yes` under `Host *`, SSH offers only identities named by an
`IdentityFile`. With the private key held in KeePassXC and no matching
`IdentityFile`, the agent's key is discarded before the offer is made. The
connection fails on publickey despite the key being loaded and available.

Fix: point `IdentityFile` at the **public** key. SSH reads it for the
fingerprint and requests that identity from the agent; the private half never
touches disk.

    Host github.com
        User git
        IdentityFile ~/.ssh/id_ed25519_t14.pub
        IdentitiesOnly yes

The verbose trace is distinctive and reads like a server-side rejection:

    debug1: get_agent_identities: agent returned 1 keys
    debug1: No more authentication methods to try.

The key is present, loaded, and never offered — there is no `Offering public
key` line between those two. Agent returns keys, then immediate failure, means
client-side filtering, not the server.

This defect was present from 2026-08-14 and corrected on all four host entries
2026-08-17. **Supersedes** an earlier note calling the stale `IdentityFile`
harmless — that was an untested inference recorded as verified behaviour.

### `sshd` "start-limit-hit" after boot is the tailnet bind racing `tailscaled`
**measured 2026-09-02**

`systemctl status sshd` after a reboot showed `failed (Result: start-limit-hit)`.
The status line names a rate limit, and the five journal lines above it say

    error: Bind to port 22 on <TS_IPV4> failed: Cannot assign requested address.
    fatal: Cannot bind any address.

Neither points at the cause. `sshd_config.d/20-tailnet-only.conf` binds only the
tailnet address; that address exists only once `tailscaled` has created
`tailscale0`. The stock unit orders sshd after `network.target` only, so sshd
started at the same instant as `tailscaled`, failed to bind, and systemd retried
five times at 100 ms intervals — all five attempts were over **90 ms before** the
interface appeared (`journalctl -b -o short-precise`: last bind error at
10:08:17.561, `tailscale0` created at 10:08:17.653). `Restart=always` is
defeated by the default start-rate limit, so the unit stays `failed` for the whole
uptime with no further attempts.

Fix, in `/etc/systemd/system/sshd.service.d/10-wait-for-tailnet.conf`: order
`After=`/`Wants=tailscaled.service`, an `ExecStartPre=` that polls for an IPv4
address on `tailscale0` (90 s per attempt), and `StartLimitIntervalSec=0` with
`RestartSec=5` so the unit keeps trying instead of giving up. Reproduced the race
deliberately — `systemctl stop sshd tailscaled`, then `systemctl start sshd` — and
sshd pulled tailscaled in, waited, and bound one second later. A machine where the
tailnet never comes up shows `activating (auto-restart)` rather than `failed`,
which is the intended loud state.

**The decision this implements** — bind to the tailnet address here, the opposite
of netcup's ADR-0029 — is recorded in
[[areas/infrastructure/arch-thinkpad/backlog/undeliberate-listeners-review|undeliberate-listeners-review]], not
here.

---

### A locked KeePassXC database presents as an invalid key file
**measured 2026-08-17**

    Load key "/home/luiswg/.ssh/id_ed25519_t14.pub": invalid format
    git@github.com: Permission denied (publickey).

Nothing in that message points at KeePassXC. SSH is reporting it could not load
the `.pub` as a *private* key — which it only attempts after the agent supplies
nothing. The key file is fine.

Same symptom when the lock timer fires mid-session and when the agent is
cleared manually. Lock and unlock re-adds the key; nothing is lost.

**First diagnostic for any sudden publickey failure is `ssh-add -l`**, not
inspecting key files.

### A greyed-out Secret Service checkbox in KeePassXC is not a disabled feature
**measured 2026-08-27**

KeePassXC's **Application Settings → Secret Service Integration** control was found
greyed out. The natural reading is "this was switched off" — by an update, by a
profile reset, by someone.

It was not. **Another provider holds `org.freedesktop.secrets`**, and KeePassXC
detects that and declines rather than competing for the name. The control is
disabled *because the feature cannot be offered*, not because the feature is off.
Since Plasma was installed, the holder is `ksecretd`.

The diagnostic points at KeePassXC and the cause is on the bus:

    busctl --user list | grep secrets

Nothing prompted, nothing errored, and no choice was presented — the only visible
trace of the handover was this checkbox going grey. See
[[areas/infrastructure/arch-thinkpad/decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] for what it
means for credentials already stored, and for the KWallet per-request-prompt
question that is still open.

**Corollary worth knowing before you go looking:** a client configured with
`"password-store": "gnome-libsecret"` is bound to the *bus name*, not to a
provider, so it follows a handover silently and keeps working. Client behaviour is
not a signal here either.

---

### `aa-enforce`/`aa-complain` fail on apparmor.d profiles with a path-syntax error
**measured 2026-09-02**

    ERROR: Path doesn't start with / or variable: =/org/bluez/hci@{int}/dev_…

from `aa-enforce /etc/apparmor.d/discord`. The message names a D-Bus path in an
abstraction the profile includes, so it reads as a broken profile. The profile is
fine — `apparmor_parser -r` loads it and the kernel enforces it. The Python
`aa-*` tools carry their own parser, and apparmor.d 0.4910 uses syntax it does not
understand. Change modes by editing `flags=(...)` on the `profile` line and
running `apparmor_parser -r <file>`; that is what `/usr/local/bin/apparmor-reenforce`
does. Do not use `aa-enforce` on this machine.

### `runuser -u luiswg -- git …` reports "Permission denied" on a directory luiswg owns
**measured 2026-09-02**

`warning: could not open directory '.local/bin/': Permission denied`, from a
bare-repo `git add` run as luiswg via `runuser` from a root shell. The directory
is `0700 luiswg`. The relative path is the tell: `runuser` does not change the
working directory, so `.local/bin/` resolved under **root's** home, which is
`0700 root`. The add itself succeeded because the work tree is absolute. Either
`cd` to the target home first or use `runuser -l`; `ls` via the same `runuser`
"works" only because it was given an absolute path.

---

### A restic backup unit dies on "neither $XDG_CACHE_HOME nor $HOME are defined"
**measured 2026-09-02**

    unable to open cache: unable to locate cache directory: neither $XDG_CACHE_HOME nor $HOME are defined

from `restic backup` inside a system unit, exit 1 before a byte is read. The
message names two environment variables the unit never mentioned, so it reads
as a restic packaging fault. It is systemd: **system services run with no
`$HOME`** (`systemd-run --wait /usr/bin/env` shows none), and restic resolves its
cache from `$XDG_CACHE_HOME`, then `$HOME/.cache`, then gives up. `restic check`
hides the same gap — it builds a temporary cache under `/tmp` and continues — so
a check unit passing says nothing about a backup unit. Fix on the unit:
`CacheDirectory=restic` plus `--cache-dir /var/cache/restic` on every
`ExecStart=`. Do not set `Environment=HOME=` — it works and it is the wrong
tool for a service.

### `collect.sh` run as root aborts on the user bus, and a shim on `$PATH` leaks into `packages.md`
**measured 2026-09-02**

    Failed to connect to user scope bus via local transport: Operation not permitted

from the `systemctl --user` calls in the services block, when the collector is
run by root with `HOME` pointed at the operator's home. Root cannot join another
user's session bus; `systemctl --user --machine=<user>@.host` can. A wrapper
script named `systemctl` earlier on `$PATH` makes the run succeed **and** writes
`systemctl /root/shim/systemctl …` into `## tool versions`, because that section
resolves the binary with `readlink -f` on purpose. The collector is designed to
run as the operator with `sudo` inside it; run it that way. Before the staging
change of the same day this abort left `services.md` 71 lines short and looking
current — the case [[areas/infrastructure/arch-thinkpad/backlog/collector-stale-file-detection|collector-stale-file-detection]]
described.

---

### A timer job fails on DNS although the machine is "connected"
**measured 2026-09-08**

    dial tcp: lookup s3.eu-central-003.backblazeb2.com: Temporary failure in name resolution

from `restic-b2.service`, and the same from `reflector.service`, both minutes
after a resume, both while `nmcli` said `connected`. The message points at DNS;
the cause is that there was no network behind the link. NetworkManager's state
was `CONNECTED_LOCAL` — associated to an access point, no route out — which
still activates the device (`state 100`) and still satisfies
`network-online.target`. `Persistent=true` timers fire the moment the system
resumes, which is exactly when that state is most likely. The only signal that
distinguishes it is NetworkManager's connectivity probe:
`nmcli networking connectivity` returns `full` only when the internet answers.
Both units now carry that as an `ExecCondition`; a skip shows as `inactive
(condition failed)` rather than `failed`. Because `/etc/resolv.conf` is written by
Tailscale and points at its MagicDNS resolver, DNS fails identically whether the
link is local-only or the tailnet is down — do not read a DNS failure as a
Tailscale or DNS problem before checking `nmcli general status`.

## Packages and services

### Removing a package does not stop its process or remove its autostart
**measured 2026-08-19**

After `pacman -Rns waybar mako`, both continued to run and reappeared after
multiple logouts. Three mechanisms:

1. A running process survives deletion of its binary — Linux keeps the unlinked
   inode alive. `pacman -Q waybar` reported "not found" and `/usr/bin/waybar`
   did not exist while PID 921075 was still drawing a bar.
2. `pacman -Rns` removes `/usr/lib/systemd/user/waybar.service` but **does not
   remove the enablement symlink** at
   `~/.config/systemd/user/graphical-session.target.wants/waybar.service`,
   because that file belongs to the user, not the package. The dangling symlink
   caused the respawn across logouts.
3. `$SHELL`, `pgrep`, and visual identification all misled during diagnosis.
   Definitive: `readlink /proc/<pid>/exe` (names the exact file, including
   `(deleted)`) and `hyprctl layers` (names the owner of every layer-shell
   surface).

Correct removal order for anything with a user unit:

    systemctl --user disable --now <name>.service
    pacman -Rns <name>
    rm ~/.config/systemd/user/<name>.service      # only if it was your own file
    ls -la ~/.config/systemd/user/*.target.wants/ # verify nothing dangling
    ls -la ~/.config/systemd/user/*.target        # and no hand-written target survives

Verified clean as of 2026-08-21 — no dangling `.wants` symlinks remain.

**Amended 2026-08-28 — the audit was one line short.** The `.wants` check finds
dangling *symlinks*; it does not find a hand-written `.target` **file**, which is
what `hyprland-session.target` was. That file survived the desktop switch and
produced a wrong answer in generated state — see the entry two below. The fourth
line above is new for that reason.

`noctalia.service` and `udiskie.service` are still in `~/.config/systemd/user/`
today, disabled. Present-and-disabled is a safer state than dangling-and-enabled,
so they were left rather than swept, but they are the same residue class and the
same mechanism put them there: **a file you wrote belongs to you, and no package
manager will ever remove it.**

### `-Rns` cascades through optional dependencies without blocking
**measured 2026-08-19**

`pacman -Rns waybar wofi mako swaybg` pulled 19 dependencies including
`upower`, which Noctalia needs for the battery widget. pacman printed
`:: noctalia optionally requires upower` in the dependency check and proceeded
anyway. **Optional dependencies do not block a cascade** — the warning is
printed but not enforced.

Before any `-Rns` of a userland package, check whether anything already
installed lists the cascade candidates as optional dependencies.

### What starts a given process on this machine
**measured 2026-08-19**

Three mechanisms are in play: own user units, package-shipped user units, and
units generated by systemd from XDG autostart `.desktop` files
(`app-*@autostart.service`, pulled in via `xdg-desktop-autostart.target`).

`systemctl --user list-dependencies graphical-session.target` covers all three.

This is why `systemctl --user is-active fcitx5` reported `inactive` while the
process was running — that command queries the *package's* unit, which is not
the one in use. fcitx5 is started by `app-org.fcitx.Fcitx5@autostart.service`.
The Bluetooth applet started the same way until `blueman` was removed on
2026-08-26.

**Supersedes** an earlier guess that a Hyprland `exec-once` line was
responsible. `hyprland.lua` contained no such line — it had exactly one autostart
line, which started `hyprland-session.target`.

**The conclusion it reached is the durable part and it survived the desktop
switch: adding an autostart on this machine means writing a user unit.** The
contrast it was drawn against — "not editing `hyprland.lua`" — is dead, since
neither the compositor nor its config exists.

Whether Plasma's *Autostart* settings page adds a **fourth** mechanism or writes an
XDG `.desktop` entry and so reaches the third one above is **`inferred`** — not
checked. It matters, because the three-mechanism list is the reason this entry is
worth running at all. Lookup: add something via that page and see what appears in
`~/.config/autostart/`, then re-run the `list-dependencies` command above.

**Amended 2026-08-28 — the target name in this entry was wrong, and wrong in a way
that answered anyway.** It read `hyprland-session.target` until today. That is a
compositor-specific name, and naming one here is the same defect the collector
carried: it binds the general question *"what starts things on this machine"* to a
particular desktop. `graphical-session.target` is the freedesktop unit every DE
populates, so it keeps answering across a desktop switch. The entry directly below
this one is what the old name did.

The three-mechanism finding above is unchanged and still the reason to run this at
all. Only the target name moved.

### A stale session target answers for the desktop that replaced it
**measured 2026-08-27**

`~/.config/systemd/user/hyprland-session.target` was hand-written on 2026-08-13.
`pacman -Rns hyprland` never touched it — it is the user's file, per the entry two
above. It carried `BindsTo=graphical-session.target`.

So after the switch to KDE,
`systemctl --user list-dependencies hyprland-session.target` **did not fail**. It
resolved through that dependency and printed the entire Plasma service tree:
`plasma-kwin_wayland`, `plasma-plasmashell`, `plasma-powerdevil`, the portals, the
lot. A full, healthy-looking dependency graph — for a compositor that is not
installed.

**Nothing errors, and the output is the opposite of a hint.** An empty result or
`Failed to get dependencies` would have pointed straight at the cause. A rich,
plausible tree reads as confirmation that the query was right.

This is the [[#Removing a package does not stop its process or remove its autostart|dangling `.wants` symlink]]
failure one level up, and worse in the one way that matters here: that one
respawned a visible process, which is annoying and self-announcing. This one fed a
**wrong answer into `state/services.md`**, which is generated output the rest of
the vault treats as ground truth. The collector was reading a compositor-specific
target name; it now reads `graphical-session.target`, the freedesktop unit every DE
populates, so it describes whichever session is actually running.

The general form is the vault's own rule reached from a new direction: **assert
against live state, never against constants** — where the constant was a *unit
name*. `hyprland-session.target` encodes a choice about which desktop is installed.
`graphical-session.target` encodes none.

Two places named the same stale target and only one was found first;
`state/_index.md` described the collector's section by it too. When you fix a name
like this, grep for it.

### `snapper create` is silent on success
**measured 2026-08-19**

No output, no confirmation, exit 0. Verify with
`snapper -c <config> list | tail -3`. Note `snapper -c home list` requires sudo
— the config is root-owned, so an unprivileged call returns "No permissions"
rather than an empty list.

---

## Boot and storage

### A UUID-prefixed `rd.luks.options` replaces the unprefixed one rather than merging
**measured 2026-08-23** (the behaviour) / **inferred** (the consequence)

`systemd-cryptsetup-generator(8)` on `rd.luks.options=`: "Takes a LUKS super
block UUID followed by an '=' and a string of options separated by commas as
argument. **This will override the options for the given UUID.** If only a list
of options, without a UUID, is specified, they apply to any UUIDs **not
specified elsewhere**, and without an entry in /etc/crypttab."

The cmdline on this machine carries `rd.luks.options=discard` with no UUID
prefix. PRD §12.2 instructs you to add
`rd.luks.options=<FSUUID_nvme0n1p2>=tpm2-device=auto`. Doing so specifies that
UUID *elsewhere*, so the unprefixed `discard` stops applying to the root volume.
The correct form would be a single prefixed entry carrying both:
`rd.luks.options=<FSUUID_nvme0n1p2>=discard,tpm2-device=auto`.

**Nothing errors.** The volume unlocks, the system boots, and discard is simply
no longer passed through to the SSD. The symptom is degraded write performance
on a QLC drive surfacing months later, and nothing about that points at a kernel
command line — which is additionally embedded in a signed UKI with `editor no`
in `loader.conf`, so it cannot be inspected or corrected from the boot menu.

`inferred`: nobody has observed discard being lost on this machine. The
override behaviour is documented; the performance consequence is reasoned.

The instruction is also unnecessary. TPM2 unlock works here with no
`tpm2-device=` anywhere — see [[areas/infrastructure/arch-thinkpad/sessions/20260823-build-spec-schema|the 2026-08-23 session]]
for the three locations checked. The spec encodes this as a negative
postcondition on the cmdline step, so re-introducing the prefixed form fails the
build rather than silently costing discard.

---

## Hardware

### The volume keys emit no evdev event, but the volume changes
**measured 2026-08-19**

`wev` shows nothing for the volume keys, so the `XF86Audio*` binds in
`hyprland.lua` are inert for volume. Yet PipeWire's sink volume changes when
they are pressed — the embedded controller talks to the codec directly and the
kernel learns only about the result.

**Consequence: any OSD that works by observing key events cannot work on this
machine.** That includes `swayosd` in both modes — the libinput backend watches
for `KEY_VOLUMEUP`, and `swayosd-client` is invoked from a keybind. Neither has
anything to trigger on. swayosd was not misconfigured.

The OSD must subscribe to PipeWire state instead. Noctalia's does. Verified by
`wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+` from a terminal with no key
involved — the OSD appears.

**Amended 2026-08-28.** The finding above is unchanged and still load-bearing: the
EC talks to the codec directly, no evdev event is emitted, and **an event-driven
OSD cannot work on this machine** no matter which desktop is installed. That is a
property of the hardware and it survived the switch.

What did *not* survive is the worked example. Noctalia is uninstalled. **Whether
Plasma's OSD subscribes to PipeWire state or watches for key events is not
verified** (`inferred`) — and if it does the latter, it will appear broken for
volume and fine for brightness, which is the confusing half-working symptom this
entry exists to explain. The re-test is one command and needs no key press:

    wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+

If an OSD appears, Plasma's is PipeWire-driven and this entry is closed. If not,
the hardware constraint has claimed a second victim and the reason is written
above.

The `hyprland.lua` reference in the first paragraph is historical — the binds went
with the compositor.

BIOS "F1–F12 as Primary Function" is enabled and stays that way; it was never
the cause. Brightness and media-transport keys do emit events and work
normally. The `XF86Audio*` volume binds are retained with a comment recording
this — harmless, and correct if the machine ever changes.

**Supersedes** the note in PRD §12.5 concluding "no OSD daemon can catch them,
waybar's volume module is the working answer." The symptom was right, the
conclusion was wrong, and it cost two sessions.

### Two Realtek ethernet controllers are present
**measured 2026-08-21**

`enp2s0f0` (02:00.0, rev 0e) and `enp4s0` (04:00.0, rev 15), both RTL8111/8168
`[10ec:8168]`, both Lenovo subsystem `[17aa:5081]`, both driven by `r8169`,
both DOWN. Distinct PCI addresses and different revisions, so not one
controller presenting twice.

A T14 Gen 1 AMD has one RJ45. **Not yet explained** — possibly an Ethernet
extension connector, possibly a persisted dock enumeration. Recorded rather
than resolved.

---

## Noctalia — historical, superseded 2026-08-28

**Noctalia is uninstalled.** The desktop moved to KDE Plasma on 2026-08-26/27
([[areas/infrastructure/arch-thinkpad/decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]]), and nothing
in this section describes the machine as it is now. It is kept rather than deleted
because two of its findings are about *shapes of failure* rather than about
Noctalia, and both have successors worth recognising:

- **A settings file that the GUI writes and the docs do not mention** (the
  `~/.local/state/` file winning over `~/.config/`). Plasma is an entire desktop
  configured this way; the `klipperrc`/`kwalletrc` question in
  [[areas/infrastructure/arch-thinkpad/decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] is the same
  shape.
- **Three idle-lock failure modes that look identical from outside.** Lid, lock and
  suspend are `plasma-powerdevil`'s now and work out of the box, but the diagnostic
  sequence below — inhibitor list, then configured timeout, then "not idle long
  enough" — is the one to follow when they stop.

The verbs, paths and file sizes below are dead. Do not run them.

### Settings written by the UI can appear not to apply
**measured 2026-08-19**

Two causes, in order of likelihood: the value went to
`~/.local/state/noctalia/settings.toml` (correct and authoritative — see
ADR-0004) but needs `noctalia msg config-reload`; or a hand-edited
`~/.config/noctalia/*.toml` is being overridden by the state file. Confirm the
value landed with `grep` on the state file before assuming the UI failed.

Changes take effect on `noctalia msg config-reload` or a service restart, not
on save.

### Distinguishing idle-lock failure modes
**measured 2026-08-21**

Three states look identical from the outside when the screen does not lock:

1. **Caffeine enabled.** `systemd-inhibit --list` shows
   `noctalia … idle … Caffeine … block`. With caffeine off, only the
   `sleep … Lock before sleep … delay` inhibitor remains. Confirmed by
   toggling `noctalia msg caffeine-toggle` and watching the inhibitor list.
2. **No idle timeout configured.** `settings.toml` is 48 lines and contains no
   idle key. `noctalia msg --help` exposes no verb printing effective settings
   — only `settings-open`, which is the GUI. The duration is not discoverable
   from the CLI.
3. **Not idle long enough.**

The verb for manual lock is `noctalia msg session lock`, not `noctalia msg
lock`. `Super+L` is bound to hyprlock as the retained fallback.

Session lock state: `loginctl show-session $XDG_SESSION_ID -p LockedHint`.

---

## Inherited, not verified

Entries carried from earlier documents that have **not** been re-checked on the
current system. Treat as leads, not facts.

### `sudo` rejecting a password you are sure is correct is `pam_faillock`
**inherited-unverified**

Carried from PRD §3.5. Three failed attempts lock the account for ten minutes,
after which the *correct* password is also rejected — so the symptom is
indistinguishable from having the password wrong, and retrying extends the
lockout. Clear it with `faillock --user <name> --reset`.

§3.5 says this "will look identical at the login prompt after first boot", which
is the reason it is worth recognising before it happens rather than during.

Not re-checked here, and the specific numbers are the PRD's: nothing in
`state/` records `/etc/security/faillock.conf`, and the defaults are
configurable. `faillock --user luiswg` on the machine reports the current tally
and would confirm both the tool name and the mechanism.

### `chezmoi cd` opens an interactive subshell
**inherited-unverified**

Chaining `chezmoi cd && git add -A && git commit` does not work — the git
commands are queued and run only after the subshell exits. Use
`chezmoi git -- <args>`.

chezmoi is **not installed** on this machine (confirmed 2026-08-19 and again by
the package collector 2026-08-21). This note is inherited from general usage and
cannot be verified here until chezmoi is installed.

### Reads of `/sys/class/power_supply/AC/*` hang in D state; `tlp`, `tlp-stat`, powerdevil stall
**measured 2026-09-09**

`cat /sys/class/power_supply/AC/online` never returns and cannot be killed, even
under `timeout`. `ps -eo stat,wchan,comm` shows ~90 `kworker/*+kec_query` and
`+kacpi_notify` threads in `D`; `/proc/<pid>/stack` puts the EC query workers in
`acpi_ex_system_do_sleep` inside `acpi_ec_event_processor` (an EC `_Qxx` firmware
method looping in `Sleep()`) and every AC reader in `acpi_os_wait_semaphore` behind
the ACPI mutex that method holds. First `hung task` report 10:21:13, so the wedge
began ~10:19; a Seagate Expansion USB disk (`0bc2:231a`) was attached on the
`06:00.4` xHCI at 10:18:25, blocked by usbguard, then allowed. Nothing else in the
journal at that time. It looks like enforcement or `tlp` broke — it is not: the
AppArmor change was 10:27 and the log has zero `DENIED`. Only a reboot clears it;
expect suspend and `shutdown` to stall on ACPI, hold power if they do.
**Revised 2026-09-10:** the USB disk was coincidence. The same stall recurred with
no dock and no disk attached (09-09 17:12, transient, after an adapter-cable pull)
and again on resume from suspend (09-10 10:39, permanent). See the next entry.

### Embedded controller misbehaves since 2026-09-09: ACPI stalls, black-screen resume, POST "fan error"
**measured 2026-09-10**

Three events with one signature — kernel ACPI workers (`kacpi_notify`, `kec_query`)
piling up in `D` behind an EC event handler that never returns:

| when | trigger | outcome |
|------|---------|---------|
| 09-09 10:19 | bus-powered USB disk on a USB-C port | permanent; GPU decode hang and Plasma freeze at 11:09–11:23 sat behind the same lock |
| 09-09 17:12 | adapter cable pulled and restored | cleared itself after a few minutes |
| 09-10 10:39 | resume from S3 (suspended 00:47) | kernel resumed (`PM: suspend exit`), EC restarted, then hung; screen stayed black, power LED solid. Hard shutdown 10:43 |

The hard shutdown that followed produced **"Fan error"** at POST, then a boot that
timed out waiting for the root device (no journal — root never mounted); the next
power-on booted normally. The fan is not dead: under 30 s of six-thread load it
spun to 2700–2900 rpm, though only from 77 °C. Fan and thermal are EC-owned, so
the POST message and the stalls are the same subsystem. Read as the EC in a bad
state that a warm reboot and even a power-button hold do not clear, because the
EC keeps running from the battery.

Ruled out: the Thunderbolt dock (never connected during any event), kernel
version (7.1.9 and 7.2.3 both), AppArmor enforcement (first stall predates it,
zero `DENIED`), CPU throttling (`bios_limit` 2.1 GHz is the base clock under
`acpi-cpufreq`, boost on), firmware age (BIOS 1.56 / EC R1BHT87W is the latest
on LVFS as of today). Boot -2 (09-03 → 09-09) completed 10 suspend cycles with
no stall before the first event, so this is a change in the machine, not in the
config. Remedy to try first: full EC reset via the emergency-reset hole on the
underside with AC disconnected, then reconnect AC and power on (Lenovo user
guide). If a stall recurs after that, it is hardware.

**Resolved 2026-09-10, `measured`.** A 30 s press of the emergency-reset hole with
AC disconnected cleared it: the next boot had no stuck workers, one suspend/resume
cycle and a direct USB-C plug of the same Seagate disk both completed with zero
`D`-state tasks and zero hung-task reports, and the disk stayed enumerated instead
of dropping within a second as it had on 09-09. Yesterday's serial disconnects
were the EC, not the disk. If the signature returns, escalate to hardware.

### ThinkPad Thunderbolt 3 Dock Gen 2 on the T14 Gen 1 AMD: USB-C fallback, and only one port brings up USB 3
**measured 2026-09-10**

This machine has no Thunderbolt controller, so the dock runs in USB-C mode. On
the first USB-C port tried (xHCI `06:00.4`, bus 5) the SuperSpeed link failed
repeatedly — `usb usb5-port1: Cannot enable. Maybe the USB cable is bad?` a dozen
times — and only the dock's USB 2 branch (hubs, audio, billboard) enumerated;
the dock's front and rear USB 3 ports and its ethernet were absent, and a disk
plugged into a front port produced no kernel event at all. On the other USB-C
port (xHCI `06:00.3`, bus 3) the USB 3.1 hub links at 10 Gbps and the dock
ethernet (`r8152`, appears as `eth0`, not `enx…`) and rear ports work at 5 Gbps.
Which physical port that is was not recorded — check `lsusb -t` for a
`10000M` hub under bus 3 after docking. A device that shows at `480M` under the
USB 2 hub tree is on a port or cable without SuperSpeed, not a policy problem.
usbguard: every dock device is allowed by its own hash rule in `rules.conf`;
new devices behind the dock still need allowing once. Plugging the dock in
caused a short EC stall (stuck ACPI workers for a few seconds) that cleared by
itself — see the EC entry above.

### Root-run `git` in the vault leaves `.git/index` owned by root; luiswg then gets "index file open failed: Permission denied"
**measured 2026-09-10**

Any `git status`/`commit` as root in `~luiswg/dev/infra-docs` rewrites
`.git/index` (and objects) as root. The next git call as luiswg fails on the
index, which reads like a corrupt repo or an AppArmor denial — it is ownership.
Fix: `chown -R luiswg:luiswg .git`. Rule from now on: from a root shell run vault
git as the operator, `sudo -u luiswg -H git -C /home/luiswg/dev/infra-docs …`,
with `SSH_AUTH_SOCK=/run/user/1000/ssh-agent.socket` for push. This also avoids
the `dac_override` capability the enforced git profile would otherwise need.
