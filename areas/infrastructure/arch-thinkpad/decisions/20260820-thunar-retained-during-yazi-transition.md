---
title: Thunar retained during the yazi transition
type: decision
id: ADR-0011
aliases: [ADR-0011]
status: superseded
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: low
created: 2026-08-20
updated: 2026-08-28
tags: [decision, desktop, transitional]
constrains: spec/desktop
superseded_by: ADR-0018
---

# ADR-0011 — Thunar retained during the yazi transition

- **Status:** superseded 2026-08-28 by
  [[20260828-plasma-replaces-hyprland-userland|ADR-0018]]
- **Date:** 2026-08-20
- **Provenance:** measured — reinstall confirmed by the package collector
  2026-08-21

> **Superseded by the desktop switch, and its own trigger never fired.** The
> *Revisit when* below names one condition — `yazi` becoming comfortable for daily
> use — and that is not what happened. Thunar left with the Hyprland userland on
> 2026-08-26/27; `dolphin` and `ark` arrived with `plasma-meta`. So the transitional
> state this record describes ended without the transition completing, and a GUI
> file manager is still installed — a different one.
>
> **The reasoning was not tested and is not retired.** "Removing the fallback before
> the replacement is fluent means fighting an unfamiliar tool under time pressure"
> still applies to `yazi` and now to Dolphin. Whether `yazi` has displaced daily GUI
> file management is unknown here and was never the reason the package left.
>
> **Amended in the same breath:** the `udiskie` consequence below is stale.
> `udiskie` is uninstalled and its unit disabled; removable-media handling is
> Plasma's. `thunar-volman` remains removed and remains irrelevant.

## Context

The 2026-08-19 migration removed Thunar in favour of `yazi`, and recorded "No
GUI file manager is installed." Thunar was reinstalled on 2026-08-20.

## Decision

Keep Thunar installed until `yazi` is comfortable for daily use. This is a
transitional state with an intended end, not a reversal of the migration.

## Rationale

A TUI file manager is a genuine change in working method, not a drop-in
replacement. Removing the fallback before the replacement is fluent means
either fighting an unfamiliar tool under time pressure or having no file
manager at all.

The satellites were **not** reinstalled — `thunar-archive-plugin`,
`thunar-volman` and `xarchiver` remain removed. Only the file manager itself is
back.

## Consequences

- `udiskie` continues to own removable-media automounting. `thunar-volman` is
  not reinstalled and must not be, or the two will contend.
- Archive handling stays with `yazi` + `7zip`. `xarchiver` stays removed.
- `tumbler` and `gvfs` were promoted to explicit before the original removal so
  they could not cascade; that promotion still stands and is still correct —
  they are not Thunar's to provide.
- The 2026-08-19 claim "No GUI file manager is installed" is **superseded** by
  this record.

## Revisit when

`yazi` is comfortable for daily use. At that point remove Thunar again and
follow the user-unit removal order in the operational notes, even though Thunar
has no user unit — the `.wants` audit is cheap and the failure it catches is
not obvious.
