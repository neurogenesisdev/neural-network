---
title: KDE Plasma replaces the Hyprland/Noctalia userland
type: decision
id: ADR-0018
aliases: [ADR-0018]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-27
commit: pending
impact: medium
created: 2026-08-28
updated: 2026-08-28
tags: [decision, desktop, kde, plasma, hyprland, noctalia, wayland]
supersedes: [ADR-0004, ADR-0011]
constrains: spec/desktop
---

# ADR-0018 — KDE Plasma replaces the Hyprland/Noctalia userland

- **Status:** accepted
- **Date:** 2026-08-28
- **Provenance:** measured — the package and unit changes below are read from
  `state/packages.md` and `state/services.md`, both re-collected 2026-08-27.
  Claims about `~/.config/systemd/user/` are the operator's, made the same day,
  and are marked where they appear.

## Summary

The desktop was replaced on 2026-08-26/27. This record supersedes
[[20260819-noctalia-userland-consolidation|ADR-0004]] and
[[20260820-thunar-retained-during-yazi-transition|ADR-0011]] and exists mainly to
carry forward the consequences of the first, which outlived it.

## Context

**Superseded by events, not by reasoning.** ADR-0004 traded ten userland packages
for one beta and argued the trade was acceptable because rollback was cheap and
well understood. That argument was never re-opened and never lost. Its *subject*
stopped existing: Hyprland, Noctalia and hyprlock are uninstalled, and a record
whose Decision names packages that are not on the machine cannot be left
`accepted`.

The distinction matters for anyone reading ADR-0004 later. Nothing there was
found to be wrong. Do not read this record as a reversal, and do not treat the
Noctalia consolidation as a mistake that was corrected — it was a working
arrangement that was replaced wholesale for reasons outside the record.

Measured from the `state/packages.md` diff, 2026-08-27. Removed:

    blueman  grim  hyprland  hyprlock  hyprpicker  hyprpolkitagent
    noctalia  slurp  thunar  udiskie  xdg-desktop-portal-hyprland

Added: `plasma-meta`, `dolphin`, `ark`, `konsole`. (`claude-desktop` appears in
the same diff and has nothing to do with the desktop switch.)

Three user units left the enabled set — `noctalia.service`,
`udiskie.service`, `hyprpolkitagent.service` — and one system unit joined it,
`plasmalogin.service`. That last one is a login-path change and is not covered
here; see [[20260828-secret-service-provider-is-ksecretd|ADR-0019]] and
[[../backlog/login-path-unreviewed|the backlog item]].

## Decision

KDE Plasma, installed via `plasma-meta`, is the desktop. Dolphin is the file
manager. ADR-0004 and ADR-0011 are superseded.

**Three of ADR-0004's consequences survive the record that carried them** and are
restated below rather than left in a tombstone. A superseded record is not read
by anyone looking for a live constraint.

## Consequences

### Clipboard history is still a persistent on-disk artifact

It does not disappear with Noctalia, it **moves to Klipper**. The exposure is
unchanged in kind: history is readable by anything running as the user, and it is
where a password pasted out of KeePassXC lands.

ADR-0004 recorded that it was "not yet determined" whether Noctalia honours the
`x-kde-passwordManagerHint` clipboard hint. That question is **inherited open,
not reopened** — it now applies to Klipper, where it is if anything more
answerable, since the hint is a KDE convention and Klipper is the implementation
it was written for. Nothing in this round establishes it (`inferred`). The lookup
that closes it is Klipper's own configuration — whether it exposes an
ignore-password-manager option — plus `klipperrc` on the host.

Until then, treat clipboard history as an unprotected on-disk copy of anything
pasted, exactly as before.

### `upower` is still required

Still explicit in `state/packages.md`. ADR-0004 reinstalled it after a `-Rns`
cascade removed it (see the cascade entry in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]]); it is
now also pulled by Plasma's power stack, so it is held by two claims rather than
one. The practical note stands: check reverse-optional-dependencies before any
`-Rns` of a userland package.

### Power management is a separate record now

The TLP / `power-profiles-daemon` conflict was an ADR-0004 consequence and is a
standing constraint about power, not about a desktop. It has been re-homed once
already and is not re-homed a third time: it lives in
[[20260828-power-management-stays-with-tlp|ADR-0020]], which also corrects
ADR-0004's charge-threshold figures. Plasma makes the constraint sharper rather
than milder — read that record before touching anything in the Power Management
settings page.

### Two disabled unit files remain in `~/.config/systemd/user/`

`noctalia.service` and `udiskie.service` are still present and are **disabled**
(operator, 2026-08-27). They are inert, and they are the same residue class as
the `hyprland-session.target` file that produced this round's collector defect: a
hand-written unit belongs to the user, so package removal never touches it.

Left in place deliberately rather than swept, because "disabled and present" is a
different and safer state than "enabled and dangling" — but they are on the list
for the next cleanup, and the removal order in
[[areas/infrastructure/arch-thinkpad/state-notes#Removing a package does not stop its process or remove its autostart|notes.md]]
now checks for `*.target` files as well as `*.target.wants/` symlinks.

### The unconfined-by-construction set got larger

ADR-0002 names Noctalia as the application that cannot be confined because it
requires compositor IPC. With Noctalia gone the *instance* changes and the
*class* does not: the equivalent privilege under Plasma belongs to the session's
own core rather than to one optional shell package.

This record does not amend [[20260816-hyprland-ipc-never-granted|ADR-0002]] and
does not restate its rule. See the analysis in
[[../sessions/20260828-desktop-kde-migration|the 2026-08-28 session log]], which
is where the finding lives until ADR-0002 is taken up deliberately.

### The vault can no longer see the desktop it documents

`state/packages.md` is `pacman -Qqe`, so `plasma-meta` is a single line standing
in for a userland that was previously ten visible entries. Components that matter
to other records — `ksecretd`, `klipper`, `ksshaskpass`, `powerdevil`,
`plasmalogin` — appear in **no** section of any generated file. Filed as
[[../backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]].

## Revisit when

**If Klipper turns out not to honour `x-kde-passwordManagerHint`, the clipboard
consequence above reopens** — it would mean passwords pasted from KeePassXC persist
on disk with no mitigation available from configuration, which is a different
decision from "an open question inherited from ADR-0004." Stated here as a trigger
rather than left in Consequences, because the question has now survived two records
without being answered.

Also when Plasma is replaced, or `plasma-meta` is unpicked into explicit component
packages — the second would close the visibility gap above and is worth doing for
that reason alone if the metapackage ever needs pinning.
