---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [vault]
tags: [collector, state, coverage, packages, power, kde]
created: 2026-08-28T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Close the collector blind spots the desktop switch exposed

> [!NOTE] Summary
> Three gaps, all the same shape: **the collector answers confidently about something it cannot see.** A metapackage is invisible in both the package view and the service view at once, and the battery charge thresholds are not collected at all. The consequence is concrete and already happened — the vault could not answer "is `power-profiles-daemon` installed?" from any file, and [[../decisions/20260828-power-management-stays-with-tlp|ADR-0020]]'s central constraint therefore rests on a host reading `state/` cannot reproduce.

> [!SUCCESS] Resolved 2026-09-02 — `measured`
> All three steps. `hardware.md` gains `## battery charge thresholds` (85 / 90
> on the first run, matching ADR-0020). `packages.md` gains `## presence of
> packages named by records`, a **bounded** list resolved with `pacman -Q` and
> printing `<absent>` rather than a missing line — `power-profiles-daemon
> <absent>` and `chezmoi <absent>` are now vault-answerable, `tlp`, `kwallet-pam`,
> `plasma-login-manager` and `ksshaskpass` carry versions. Two names in the item
> were not packages at all: `ksecretd` lives in `kwallet` and `klipper` in
> `plasma-workspace` (`pacman -Qo /usr/bin/klipper` owns nothing — it is not a
> binary), so those packages are listed instead. The session graph was left
> alone, as the item asked.

## Context

Found on 2026-08-27/28 while grounding the records for the Hyprland → KDE switch.
Nothing here is a defect in what the collector reports; each section is accurate.
The problem is **coverage**, and coverage shrank without anything changing in the
collector — the machine changed shape underneath it.

Under Hyprland the desktop was roughly ten explicit packages, each visible in
`state/packages.md` by name. Under Plasma it is the single line `plasma-meta`.

## The three gaps

### 1. `packages.md` is explicit-only, so a metapackage hides its userland

`collect.sh` runs `pacman -Qqe`. `plasma-meta` is one line standing in for the
entire desktop. Components that other records depend on — `ksecretd`, `klipper`,
`ksshaskpass`, `powerdevil`, `plasmalogin` — appear nowhere.

**The absence of a package from this file is not evidence of anything**, because
dependencies were never in it. That was harmless when the things worth knowing
about were installed explicitly, and it stopped being harmless on 2026-08-26.

### 2. `services.md` reports *enabled* units and a loaded-session graph, so it
does not cover the gap either

The natural assumption is that the two files complement each other — what the
package list misses, the service list catches. It does not, and the failure is
**simultaneous**.

Checkable against the current file: `ksecretd` is D-Bus-activated and so appears in
no enabled list and no session graph; `klipper` runs inside `plasmashell` and has no
unit at all; `ksshaskpass` is a binary invoked on demand with no unit. **None of the
three appears in any of `services.md`'s three sections.**

So a metapackage's contents are invisible in both views at once, and there is no
third view.

### 3. Battery charge thresholds are not collected

`hardware.md` records `energy_full_design`, `manufacturer` and `model_name` for
`BAT0` and stops. `charge_control_start_threshold` and
`charge_control_end_threshold` are not read.

**This is why the 75/80 figure in ADR-0004 was able to rot.** A hand-written number
in a decision record was the only record of these values, and nothing regenerated
it to disagree with. The measured values are 85 and 90. The repair is a collector
line, not a more carefully typed constant — that is the root `CLAUDE.md` rule
*assert against live state, never against constants*, and the thresholds are
exactly the kind of per-machine value it names.

## Steps

1. Add the two `charge_control_*` sysfs reads to the hardware block. Static
   configuration, so it will not churn the diff.
2. Decide how to make an installed-package question answerable. `pacman -Qqe` is
   correct for *what was chosen* and the file should keep it; the gap is *what is
   present*. A full `pacman -Qq` is ~1500 lines and would churn on every `-Syu`,
   which is the anti-pattern `state/_index.md` names. A bounded list — the
   packages other records depend on, resolved with `pacman -Qq <name>` and reported
   present/absent — matches the `## tool versions` precedent, which is already
   bounded on purpose and for the same reason.
3. Whatever is chosen, make an absent package report **`<absent>` rather than a
   missing line**, exactly as the tool-versions loop already does. A tool that
   disappears must show as absent rather than as a shorter section.

## Note

**The trap is fixing gap 3 and calling it done.** The thresholds are the visible
damage; gaps 1 and 2 are the reason nobody noticed. A record that cannot be checked
against `state/` decays silently, and this round produced two more of them —
ADR-0019's bus-ownership readings and ADR-0020's PPD absence are both host
measurements with no vault-side counterpart.

**Do not solve gap 2 by expanding the session graph.** It is already a
collect-time snapshot of loaded units and it grew substantially under Plasma. The
question "is X installed" is a package question and belongs in the package file;
answering it from the service view is what produced the
[[areas/infrastructure/arch-thinkpad/state-notes#A stale session target answers for the desktop that replaced it|stale target]]
defect's cousin — asking a component about something it only indirectly knows.

## Related

- [[../decisions/20260828-power-management-stays-with-tlp|ADR-0020]] — the constraint that rests on an unreproducible reading
- [[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] — three of its four readings have no vault-side basis
- [[collector-stale-file-detection]] — sibling collector gap; a stale file also reports confidently
- [[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]] — the change that exposed all three
