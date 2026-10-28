---
title: "A hand-written target outlived its compositor and answered for the one that replaced it"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-28
commit: pending
created: 2026-08-28
updated: 2026-08-28
tags: [session, kde, plasma, hyprland, collector, secret-service, kwallet, power, apparmor]
---

# 2026-08-28 — A hand-written target outlived its compositor and answered for the one that replaced it

Sixth round, and the first that is not about the spec. The desktop was replaced
with KDE Plasma on 2026-08-26/27; this round wrote the records for it and swept
what the switch made untrue.

Artifacts: [[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]],
[[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] and
[[../decisions/20260828-power-management-stays-with-tlp|ADR-0020]]; supersessions
on [[../decisions/20260819-noctalia-userland-consolidation|ADR-0004]] and
[[../decisions/20260820-thunar-retained-during-yazi-transition|ADR-0011]];
amendments to [[../decisions/20260815-t5-rescue-partition-unencrypted|ADR-0009]],
four entries in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]] and one bullet in
[[../backlog/apparmor-enforce-stage-2|apparmor-enforce-stage-2]]; two new backlog
items and one closed. This is what did not fit in them.

## The defect: a stale target that answered instead of failing

`~/.config/systemd/user/hyprland-session.target` was hand-written on 2026-08-13.
`pacman -Rns hyprland` never touched it, because it is the user's file — the
mechanism already recorded in `notes.md` for dangling `.wants` symlinks. It
carried `BindsTo=graphical-session.target`.

The collector ran `systemctl --user list-dependencies hyprland-session.target`.
After the switch that command **did not fail**. It resolved through the binding
and printed the entire Plasma tree — `plasma-kwin_wayland`, `plasma-plasmashell`,
`plasma-powerdevil`, the portals — a full, healthy dependency graph for a
compositor that is not installed.

**The output is the opposite of a hint.** An empty result, or `Failed to get
dependencies`, would have pointed straight at the cause. A rich and plausible tree
reads as confirmation that the query was right. The collector was fixed to read
`graphical-session.target`, the freedesktop unit every DE populates.

### Why this is worse than the failure it resembles

`notes.md` already records the dangling-`.wants` case from the waybar removal, and
this is the same family: a user-owned artifact that survives package removal. The
plan called it "one level up" and that was right for the wrong reason.

The `.wants` failure **respawned a process**. It was annoying, visible, and it
announced itself by drawing a bar on the screen. Diagnosis was hard; noticing was
free.

This one produced **a wrong answer in generated state** — the artifact the rest of
the vault treats as ground truth, and the one place the root `CLAUDE.md` says the
whole system exists to protect. Nothing announced it. Had the fix not landed the
same morning, `state/services.md` would have been committed describing a Hyprland
session on a KDE machine, and the diff would have looked unremarkable.

### The general form is a rule the vault already had

*Assert against live state, never against constants* — where the constant is a
**unit name**. `hyprland-session.target` encodes a claim about which desktop is
installed. `graphical-session.target` encodes none.

The rule was written for UUIDs and paths, and reads as being about values that
vary per machine or per rebuild. A unit name looked like neither: it is stable, it
is not an identifier, and it is not per-machine. It was still a constant standing
in for a live reading, and it decayed the moment the thing it named changed.

That is the second time in three rounds the same convention has been reached from
a direction it was not written for. `spec/containers` found it as *a constant that
might not exist* (a sysctl §6.1 would remove); this round found it as *a constant
that names a choice*.

### One fix, two places, and only one was found first

`state/_index.md` described the collector's section as the
"hyprland-session.target graph." Hand-written, and wrong for exactly the same
reason as the collector line — the vault describing a generated section by a
compositor-specific name.

It was found by grepping, not by thinking. **This is the `grep -c` lesson again**:
the 2026-08-24 round recorded that naming a defect after the first instance is why
it gets under-counted, and that a sweep is not optional. The instinct after fixing
the collector was that the job was done, and the second instance was two minutes
of grep away.

## Two changes arrived with a metapackage and neither announced itself

### The Secret Service provider

`org.freedesktop.secrets` is held by `ksecretd`. Discovery was indirect: a setting
in KeePassXC had gone grey.

**The first reading of that was wrong and is worth recording as a dead end.** A
greyed-out checkbox reads as "this was switched off" — by an update, by a reset, by
someone. The initial framing of this round's brief described KeePassXC's provider
as *being disabled deliberately*, and it was not. KeePassXC detects that another
process holds the bus name and **declines rather than competing**. The control is
disabled because the feature cannot be offered.

That distinction changes the record from a decision into a discovery, which is how
[[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] is written.

**KWallet won uncontested, not by a race.** Only `keepassxc-proxy` was running at
the time. Recording this mattered because the natural assumption — whoever asked
first won, so re-ordering startup would flip it back — is wrong and would cost a
session to disprove.

**The consumer never noticed, and that is the mechanism.** `_source/`
§3.8 configured VS Code with `{ "password-store": "gnome-libsecret" }` and
explained, in its own text, that this means *"use libsecret against whatever owns
`org.freedesktop.secrets`."* The setting names a **bus name, not a provider**. So
the one component configured to care about the Secret Service was configured in a
way that could not detect a change of provider. Nothing broke; therefore nothing
reported.

**The finding that outlives the switch** is what §3.8 said made the arrangement
tolerable in the first place: *"Keep `Confirm when passwords are retrieved by
clients` ticked — it is the per-request prompt that makes exposure tolerable."*
Everything in an exposed group is readable by any client, and the prompt bounded
that. A KWallet unlocked by PAM at login has no obvious equivalent. **Not
established, and not guessed at** — it is the condition in ADR-0019's *Revisit
when*, because a security control that may have silently evaporated is exactly the
kind of thing that must reopen a record rather than sit in a bullet.

Two smaller consequences fell out of the same paragraph: the token lived in the
KDBX and therefore **synced** to the Pixel and Proton Drive, and the KeePassXC lock
timer is what made VS Code re-prompt. Neither transfers to a local, PAM-unlocked
wallet. The same credential moved from *synced, timer-gated* to *local,
unlocked-for-the-session* without a decision.

### The login path

`plasmalogin.service` is enabled where previously only `getty@.service` was. It is
the **only** addition to the system-enabled unit list across the whole switch, and
`getty@.service` is still there — so a graphical login path was added in front of
the console one rather than replacing it.

It connects to the paragraph above: `plasma-kwallet-pam.service` is in the session
graph, so the display manager is the mechanism that unlocks the wallet. The login
path and the credential store became one subject without either being decided.
Filed as [[../backlog/login-path-unreviewed|login-path-unreviewed]], deliberately
framed as the same class as
[[../backlog/undeliberate-listeners-review|undeliberate-listeners-review]] — with
the note that the blast radius is not comparable. An unreviewed listener behind a
default-drop firewall is contained; an unreviewed PAM stack is the authentication
path itself.

## The negative finding: the vault could not answer a question it needed

Grounding ADR-0020 required knowing whether `power-profiles-daemon` is installed.
**No file in `state/` can answer that**, and establishing *why* took longer than
the lookup would have.

- `packages.md` runs `pacman -Qqe`. Explicit installs only, so a dependency has
  never been in it. Absence from that file is not evidence.
- `services.md` reports enabled units and a loaded-session graph. The natural
  assumption is that the two files complement each other — what one misses, the
  other catches. **They do not, and the gap is simultaneous.** `ksecretd` is
  D-Bus-activated; `klipper` runs inside `plasmashell` and has no unit; `ksshaskpass`
  is a binary with no unit. None of the three appears in any section of either file.

Under Hyprland this was invisible because the desktop was ~10 explicit packages,
every one of them named in `packages.md`. Under Plasma it is the single line
`plasma-meta`. **The collector did not change; the machine changed shape underneath
it, and coverage fell without any file looking different.**

Third gap, found the same way: the charge thresholds. ADR-0004 recorded "the 75/80
charge thresholds" and the measured values are 85 and 90. `hardware.md` collects
`energy_full_design`, `manufacturer` and `model_name` for `BAT0` and stops — so a
hand-written number in a decision record was the only record of these values, and
**nothing existed that could disagree with it**. That is not a typo that survived;
it is a constant with no live counterpart, which the root `CLAUDE.md` predicts will
rot. All three are
[[../backlog/collector-desktop-blind-spots|filed together]], because the third is
the visible damage and the first two are why nobody noticed.

## The lookup rule ran out of channels, inside the devbox

ADR-0020 wanted TLP's configuration variable names for the thresholds. All three
channels failed in order: `tlp` is not installed in the devbox, so the tool cannot
answer about itself; `man.archlinux.org/man/tlp.conf.5` returns **404**; the local
ArchWiki is not mounted in the container.

That last one is [[../backlog/devbox-wiki-mount|devbox-wiki-mount]] firing in
practice rather than in principle — the item predicts exactly this and now has an
instance.

**Channel 5 was used as written**: the requirement is recorded (*the TLP
configuration must set a start threshold of 85 and a stop threshold of 90 for
`BAT0`*), the variable names are not written, and the lookup that closes it is
named — `tlp-stat -b` or `tlp.conf(5)` on the host, where the tool is installed and
can answer about itself.

Worth noting that channel 1 failed **for a reason that is about where the work
happens, not about the tool**. The rule's ordering assumes the machine being
documented and the machine doing the documenting are the same. They have not been
since the devbox, and this is the first time that has cost a lookup rather than a
convenience.

## ADR-0002 was re-read and deliberately not amended

Reported rather than changed, at the operator's instruction. Three findings:

**The premise moves rather than vanishing, and gets harder to express.** `hypr/**`
was a *file* rule over a unix socket — `apparmor.d` can write it, `aa-logprof` can
propose it, a human can grep for it. The equivalent privilege under Plasma is on
the **session bus**, which AppArmor mediates with `dbus` rules keyed on
bus/name/path/interface/member. A single broad `dbus (send) bus=session` line in a
shared abstraction re-opens the class for many profiles at once, and does not read
as a grant of compositor control.

**The candidate interfaces are not named.** Nothing about KDE's D-Bus surface can
be looked up from the devbox, and the root `CLAUDE.md` forbids recalling it. The
lookup that closes it is `busctl --user introspect` on the host. This is the
correct place for the report to stop, and stopping there is the finding: **the
premise is unverified in the new world**, which is a different state from being
wrong.

**The open question moved in the harder direction.** ADR-0002 prices itself as
costing "confinement of the compositor shell, not of everything." Under Hyprland
the unconfined-by-construction set was Noctalia — one optional package that could
have been removed. Under Plasma the same argument reaches `plasmashell`,
`kwin_wayland`, `kded6`, `krunner` and the portal: **the session's own core**, all
present in the graph in `state/services.md`. Same rule, materially larger residual,
and it lands on the `apparmor-enforce-stage-2` decision the record defers to.

The root `_index.md` open-question row still reads "Whether to enforce AppArmor
given the Hyprland IPC rule." Left alone on purpose — rewording it is amending the
record by another route. It is the first item for whoever takes ADR-0002 up.

## A decision about where a constraint lives

ADR-0004's TLP / `power-profiles-daemon` conflict could have gone into ADR-0018 as
a Consequences bullet. It went into its own record instead, and the argument is
worth keeping because it generalises.

The constraint is about **power**, not about a desktop. It had already been
re-homed once — it was born as a Noctalia consequence — and putting it inside the
record that supersedes Noctalia guarantees a third re-homing at the next desktop
change. A constraint that keeps moving is a constraint that will eventually be
dropped in transit.

It also got sharper rather than milder in the move. Under Noctalia the conflict was
inert: `power-set`/`power-cycle` were verbs nobody used. Under Plasma,
`plasma-powerdevil` is the KDE-native consumer of PPD, so the settings page
surfaces **a profile switcher that silently does nothing**, and the obvious repair
— install the daemon it is asking for — is the thing that breaks the charge
thresholds. The symptom's natural fix is the damage.

## Conclusions revised

| Claimed | Corrected to | Why |
|---|---|---|
| `list-dependencies hyprland-session.target` covers all three autostart mechanisms (`notes.md`) | It covers whatever the stale target happens to bind to | The target outlived the compositor and resolved downward |
| The stale target would make the collector fail | It made the collector **succeed**, with a plausible wrong tree | `BindsTo=graphical-session.target` |
| One fix, in the collector | Two places; `state/_index.md` named the same target | Grep, not thought |
| KeePassXC's Secret Service provider was disabled | It is **greyed out**; KeePassXC declined on detecting another holder | Operator correction mid-round |
| KWallet won a race | KWallet won **uncontested** — only `keepassxc-proxy` was running | " |
| VS Code broke when the provider changed | It followed the swap silently — `gnome-libsecret` binds to the bus name | `_source/` §3.8, `inherited-unverified` |
| ADR-0004's "75/80 charge thresholds" | 85 start, 90 end | Measured on the host 2026-08-27 |
| `packages.md` can tell you whether PPD is installed | It cannot; `pacman -Qqe` never held dependencies | Read the collector |
| `services.md` covers what `packages.md` misses | Neither covers it; the gap is simultaneous | `ksecretd`, `klipper`, `ksshaskpass` are in no section of either |
| Noctalia is the unconfined-by-construction case — stated in ADR-0002 §Consequences, ADR-0004 §Consequences and `apparmor-enforce-stage-2` §Constraints | The session's own core is | **Corrected in two of the three.** ADR-0002's copy stands, ring-fenced this round |
| udiskie auto-mounts plaintext partitions at login (ADR-0009) | Removable media is Plasma's; the unit is disabled | `packages.md` diff |
| ADR-0011's transition would end when `yazi` became fluent | It ended when the package left with the desktop; its trigger never fired | " |

**The two that matter are rows 2 and 9.** Both are cases where a check *succeeded*
and therefore reported nothing. A command that resolves through a stale binding,
and a pair of files each assumed to cover the other's gap, produce no error and no
empty result — they produce confident output. That is the same class as the
2026-08-24 round's "a diagnostic below an aborting assignment": invisible to
review, and silent until the case arrives.

Row 4 is the round's cheapest lesson. **A disabled control in a UI is not evidence
that a feature was disabled.** It took an operator correction to catch, and the
wrong version had already been written into a plan.

## Accepted limitations

- **Everything in this round was written from inside the devbox, and the host was
  unreachable.** `collect.sh` refuses to run in a container by design. So of the
  four load-bearing readings behind ADR-0019 and ADR-0020, only one —
  `plasmalogin.service` being enabled — can be re-checked from the vault. The other
  three are operator measurements with no vault-side counterpart, marked as such in
  both records rather than smoothed over.
- **The defect's own artifact does not survive in git.** The committed
  `state/services.md` at HEAD is the *pre-switch* graph; the false-healthy
  Plasma-under-`hyprland-session.target` output was seen at run time and the
  collector was fixed before any commit. The evidence for the finding is the
  reasoning and the two files, not a diff.

## Still open

- **Whether KWallet has a per-request retrieval prompt.** ADR-0019's reopening
  condition. `kwalletmanager` access control and `kwalletrc`.
- **Which askpass services a given prompt.** `ksshaskpass` is installed alongside
  the KeePassXC SSH-agent arrangement. `SSH_ASKPASS` / `SSH_ASKPASS_REQUIRE`, read
  from a graphical session rather than a TTY.
- **Whether Klipper honours `x-kde-passwordManagerHint`.** Inherited open from
  ADR-0004, now against the implementation the hint was written for.
- **Whether Plasma's OSD is PipeWire-driven.** The hardware finding stands
  regardless — no evdev event is emitted — but the worked example was Noctalia's.
  One command, no key press: `wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+`.
- **ADR-0002's D-Bus premise**, unverified, and the root `_index.md` row that
  quotes its old phrasing.
- **The flat-battery symptom that opened the lid item** has not been re-tested.
  Suspend on lid close is not hibernation before the battery is gone.
- **KDE lid/lock/suspend timings** are unset — GUI work, deliberately deferred.
