---
title: Power management stays with TLP; power-profiles-daemon must not be installed
type: decision
id: ADR-0020
aliases: [ADR-0020]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-27
commit: pending
impact: medium
created: 2026-08-28
updated: 2026-08-28
tags: [decision, power, tlp, battery, kde, plasma, powerdevil]
constrains: spec/desktop
---

# ADR-0020 — Power management stays with TLP; `power-profiles-daemon` must not be installed

- **Status:** accepted
- **Date:** 2026-08-28
- **Provenance:** measured — `tlp.service` enabled is read from `state/services.md`.
  PPD's absence and the threshold values are the operator's host readings on
  2026-08-27 and are **not reproducible from `state/`**; that asymmetry is stated
  below rather than smoothed over.

## Summary

TLP owns power management on this machine. `power-profiles-daemon` conflicts with
it and must not be installed. This was a consequence of
[[20260819-noctalia-userland-consolidation|ADR-0004]]; it is a standing constraint
about power rather than about a desktop, and it is given its own record so it does
not need re-homing every time the desktop changes.

## Context

The constraint predates Plasma and survives it. What changed on 2026-08-26/27 is
that it became **easier to violate by accident**.

Under Noctalia the conflict was inert. ADR-0004 recorded that Noctalia's
`power-set` / `power-cycle` verbs were therefore dead and said "leave it that way"
— nothing in the UI invited anyone to install PPD.

Under Plasma it is live. `plasma-powerdevil.service` is loaded in the
`graphical-session.target` graph — that part is **measured**, from
`state/services.md`.

**That powerdevil consumes `power-profiles-daemon`, and that its settings page will
therefore show an inert profile switcher, is `inferred`** — reasoned from what the
two components are for, not observed on this machine and not looked up. The lookup
was attempted and failed: see the note on channels below. The honest statement is
that *if* powerdevil surfaces a profile switcher, it cannot work without a daemon
that is not installed, and the obvious repair — install the daemon it is asking for
— is the thing that breaks the battery charge thresholds.

**The constraint does not depend on that inference.** TLP and PPD conflict whether
or not powerdevil ever displays anything. What the inference affects is only *how
the trap presents itself*, and it is recorded because a visible-but-dead control is
what would prompt someone to install the package.

That is the shape this vault exists to catch: a symptom whose natural fix is the
damage.

## Decision

TLP owns power management. **Do not install `power-profiles-daemon`**, regardless
of what the Plasma Power Management page implies, what a forum answer suggests, or
what `powerdevil` logs about a missing service.

If the profile switcher's inertness becomes intolerable, that is a decision to
replace TLP wholesale — a new record, weighing charge thresholds and TLP's other
settings against profile switching. It is not a package install.

## Rationale

The two daemons both drive the same kernel interfaces and are documented upstream
as mutually exclusive; running both produces last-writer-wins behaviour on
settings neither one believes it shares. The concrete cost here is the charge
thresholds, which are the reason TLP is installed at all on a machine that spends
its life on AC.

## The charge thresholds, corrected

ADR-0004 recorded these as "the 75/80 charge thresholds." **That was wrong.**

Measured on the host, 2026-08-27:

| Attribute | Value |
|---|---|
| `/sys/class/power_supply/BAT0/charge_control_start_threshold` | **85** |
| `/sys/class/power_supply/BAT0/charge_control_end_threshold` | **90** |

The constraint is unaffected — only the numbers were wrong. Installing PPD would
still break them.

**The TLP configuration keys that set these are deliberately not named here.** The
lookup rule ran out of channels: `tlp` is not installed in the devbox where this
was written, `man.archlinux.org/man/tlp.conf.5` returns 404, and the local ArchWiki
is not mounted in the container (see
[[../backlog/devbox-wiki-mount|devbox-wiki-mount]] — this is that item firing). Per
channel 5 of the rule, the requirement is written and the flag is not: *the TLP
configuration must set a start threshold of 85 and a stop threshold of 90 for
`BAT0`.* The lookup that closes it is `tlp-stat -b` or `tlp.conf(5)` on the host,
where the tool is installed and can answer about itself.

**Why the figure rotted in the first place, which matters more than the figure.**
Nothing regenerates it. `state/hardware.md` collects `energy_full_design`,
`manufacturer` and `model_name` for `BAT0` and stops — the `charge_control_*`
attributes are not collected, so a hand-written number in an ADR was the only
record of them and had nothing to be checked against. That is exactly the decay
the root `CLAUDE.md` predicts under *assert against live state, never against
constants*, and the repair is a collector line, not a more carefully typed
constant. Filed in
[[../backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]].

Treat the table above as correct on 2026-08-27 and as an unrooted constant
thereafter.

## What is verifiable from the vault, and what is not

Stated explicitly because this record's central constraint rests on a reading the
vault cannot reproduce:

| Claim | Basis |
|---|---|
| `tlp.service` is enabled | **`state/services.md`** — reproducible from the vault |
| `plasma-powerdevil.service` is loaded | **`state/services.md`** — reproducible |
| `power-profiles-daemon` is not installed | **Host measurement.** `pacman -Qi power-profiles-daemon` returned *not found*, run by the operator 2026-08-27 |
| The threshold values | **Host measurement**, sysfs, as above |

`state/packages.md` is `pacman -Qqe` — explicit installs only — so it would not
show PPD whether it were absent or present as a dependency. **Its absence from
that file is not evidence.** Anyone re-checking this constraint must run
`pacman -Qi` on the machine; reading the vault cannot answer it.

## Consequences

- The Plasma power-profile switcher, if it appears at all, is expected to be inert.
  **That is the correct state**, not a fault to be fixed, and it will look like one.
  (`inferred` — see the Context note.)
- `upower` remains required, as it was under ADR-0004.
- TLP's other settings are undocumented in this vault. Only the thresholds have
  ever been recorded, and only in prose.

## Revisit when

TLP stops being maintained, or upstream establishes a supported way for
`power-profiles-daemon` and TLP to coexist. Either makes this a real choice again
rather than a constraint.
