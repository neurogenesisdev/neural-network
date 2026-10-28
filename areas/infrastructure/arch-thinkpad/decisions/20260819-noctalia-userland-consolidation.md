---
title: Consolidate the desktop userland onto Noctalia
type: decision
id: ADR-0004
aliases: [ADR-0004]
status: superseded
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-19
updated: 2026-08-28
tags: [decision, hyprland, noctalia, desktop, tlp, power]
constrains: spec/desktop
superseded_by: ADR-0018
---

# ADR-0004 — Consolidate the desktop userland onto Noctalia

- **Status:** superseded 2026-08-28 by
  [[20260828-plasma-replaces-hyprland-userland|ADR-0018]]
- **Date:** 2026-08-19
- **Provenance:** measured — migration performed and the resulting stack
  verified in use

> **Superseded by events, not by reasoning.** The trade argued below — ten packages
> for one beta, on the grounds that rollback is cheap and well understood — was never
> re-opened and never lost. Its *subject* stopped existing: the desktop was replaced
> with KDE Plasma on 2026-08-26/27 and Noctalia, hyprlock and udiskie are uninstalled.
> Nothing here was found to be wrong. Do not read this as a reversal.
>
> **Three consequences outlived this record and were re-homed rather than entombed.**
> Clipboard history as a persistent on-disk artifact, and the unanswered
> `x-kde-passwordManagerHint` question, moved with Klipper to
> [[20260828-plasma-replaces-hyprland-userland|ADR-0018]], as did the `upower`
> requirement. The TLP / `power-profiles-daemon` conflict is a constraint about power
> rather than about a desktop and has its own record now,
> [[20260828-power-management-stays-with-tlp|ADR-0020]] — which also corrects the
> charge-threshold figures below.

## Context

The desktop was an à-la-carte Wayland stack: waybar, wofi, mako, swaybg,
hypridle, wlsunset, hyprshot, cliphist, Thunar, xarchiver. Ten packages, each
with its own config format and failure mode. There was no volume/brightness
OSD at all, and repeated attempts to add one had failed (see the EC/PipeWire
note in the state notes).

## Decision

Replace the stack with Noctalia 5.0.0_beta.8 from `[extra]`, plus `yazi` for
file management and `udiskie` for removable-media automount. Retain `hyprlock`
as a lock-screen fallback.

## Rationale

A large reduction in moving parts, traded against a dependency on one beta
package from a project roughly a year old with ~222 open issues at adoption.

The trade was judged acceptable because rollback is cheap and well understood:
re-enable the waybar/wofi/mako autostarts and `pacman -Rns noctalia`.

Do not install `noctalia-shell*` from the AUR — that is the unmaintained v4
Quickshell line and its packages were withdrawn. The current package is native
C++/OpenGL ES.

## Decision, secondary — configuration authority

Noctalia reads `~/.config/noctalia/*.toml` (merged alphabetically) *and*
`~/.local/state/noctalia/settings.toml`. **The state file loads last and wins.**
The Settings UI writes only to the state file.

**The Settings UI is authoritative. `~/.local/state/noctalia/settings.toml` is
the real config file** and must be in the backup set. `~/.config/noctalia/` is
empty by design.

## Consequences

- Noctalia is unconfined by construction under ADR-0002 — it requires Hyprland
  IPC.
- Clipboard history is now a persistent on-disk artifact under
  `~/.local/state/noctalia/`, reachable by anything running as `luiswg`.
  Passwords pasted from KeePassXC land there. Whether Noctalia honours the
  `x-kde-passwordManagerHint` clipboard hint is **not yet determined**.
- `power-profiles-daemon` must not be installed — it conflicts with TLP, which
  owns power management and the 75/80 charge thresholds. Noctalia's
  `power-set`/`power-cycle` verbs are therefore inert. Leave it that way.

  **Amended 2026-08-28.** The figures in that bullet are wrong. Measured on the
  host: `charge_control_start_threshold` is **85** and
  `charge_control_end_threshold` is **90**. The constraint is unaffected — only the
  numbers were. Both the constraint and the corrected values now live in
  [[20260828-power-management-stays-with-tlp|ADR-0020]], which also records *why*
  the figure was able to rot: nothing in `state/` collects these attributes, so the
  number here had nothing to be checked against.
- `ddcutil` not installed: external-monitor DDC/CI only; the internal panel uses
  the backlight sysfs interface `brightnessctl` already drives.
- `upower` is required for the battery widget and was reinstalled after being
  removed by cascade.
- Idle management moved from `hypridle` to Noctalia. As of 2026-08-21 the idle
  *inhibitor* (caffeine) is confirmed present and functioning; an idle timeout
  reaching the locker unattended has **not been observed**. `settings.toml` is
  48 lines and contains no idle key, meaning the timeout is an unwritten
  default not discoverable from the CLI.

## Revisit when

5.0 stable is released. Re-evaluate the beta dependency then.
