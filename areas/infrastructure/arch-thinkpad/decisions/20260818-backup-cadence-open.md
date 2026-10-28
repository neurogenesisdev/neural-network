---
title: "Backup cadence: manual-when-attached"
type: decision
id: ADR-0012
aliases: [ADR-0012]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-30
commit: pending
impact: medium
created: 2026-08-18
updated: 2026-08-30
tags: [decision, backups, restic, open-question]
constrains: spec/backups
---

# ADR-0012 — Backup cadence: manual-when-attached

- **Status:** accepted 2026-08-30 — **option D**, see *Decision 2026-08-30* at the
  end of this record. The text above it is left as written on 2026-08-18/21.
- **Date:** 2026-08-18 (raised), unresolved as of 2026-08-21, decided 2026-08-30
- **Provenance:** measured (the gap), inferred (that it is a problem)

## Context

Two restic snapshots exist, six days apart, both taken manually. There is no
timer, because the Samsung T5 is not always attached — a timer would fail more
often than it would succeed.

The six-day gap spanned a new development environment, SSH configuration
changes, and a new repository. None of that was externally backed up during the
window.

The remote restic repository (PRD §9.4) that would cover this is blocked: the
Hetzner VPS has been suspended since July.

## Options

**A. Keep manual-when-attached, unchanged.** Zero effort. Accepts that the
external backup lags by however long the drive stays unplugged, which is
unbounded.

**B. Manual, with a reminder.** A timer that does not back up but notifies when
the last restic snapshot is older than N days. Preserves the manual model,
removes the "forgot" failure. Requires deciding N.

**C. Opportunistic automation.** A udev rule or systemd path unit that triggers
a backup when the VeraCrypt volume is mounted. Removes the decision entirely;
adds a moving part that runs unattended against an encrypted volume, and the
mount is manual anyway so the trigger is only marginally more automatic than
running the command.

**D. Unblock the remote repo.** Restores the VPS or provisions a new target,
giving a backup path that does not depend on physical attachment. Largest
effort; also the only option that survives the laptop and the drive being lost
together.

## Current state

None chosen. Practice is A by default rather than by decision, which is the
reason this record exists.

Snapper covers the short-horizon case — `snap-pac` takes pre/post snapshots on
every pacman transaction, so package-level mistakes are recoverable without
restic. The gap is specifically **off-machine** recovery.

## Deciding factor

What failure is being insured against. Snapper handles mistakes; restic-on-T5
handles disk failure; only a remote handles theft, fire, or both devices
failing together. If the third case is in scope, D is the only real answer and
the others are stopgaps.

## Decision 2026-08-30 — option D, and the cadence question dissolved rather than being answered

**Option D.** The remote repository exists
([[20260830-remote-repository-is-b2|ADR-0028]]), and with it the cadence is a
`systemd` timer: `restic-b2.timer`, `OnCalendar=daily`, `RandomizedDelaySec=1h`,
`Persistent=true`, followed in the same unit by
`forget --keep-daily 7 --keep-weekly 5 --keep-monthly 12 --prune`.

**The interesting part is that the question this record was written to answer
stopped applying.** Options A, B and C are all shapes of *how to cope with a
target that is only sometimes present* — accept the lag, be reminded of it, or
trigger on the mount. None of them is a cadence policy; all three are workarounds
for physical attachment. The moment the target is reachable whenever the machine
has a network, "how often" has an ordinary answer and needs no policy at all.

So D did not beat B and C on their merits. **It removed the constraint that made
them candidates**, which is why the deciding factor above was the right one to
name and the four options were the wrong frame.

`Persistent=true` is the clause that carries the laptop case: a run missed because
the machine was asleep or shut is taken shortly after the next boot rather than
silently skipped. That is the whole of what options B and C were reaching for.

### What this does not settle

- **The T5 keeps its manual cadence**, and option A is now its policy *by
  decision* rather than by default — the drive is a second copy behind a daily
  remote, so the unbounded lag this record objected to is no longer unbounded
  exposure. It is run when the drive is attached; nothing automates it.
- **The granularity cost in [[20260818-restic-exclusions|ADR-0008]] drops as that
  record predicted**, and its *Revisit when* is satisfied: a daily remote snapshot
  buys back off-disk history with correct parent matching, so excluding the snapper
  tree returns to being uncontroversial.
- **The timer is installed and has not yet fired.** Measured 2026-08-31: enabled,
  `systemd-analyze verify` silent, next elapse `Tue 2026-09-01 00:10:28 CEST`. Both
  existing snapshots were still taken by hand.

## Amendment 2026-08-31 — the link, not the target, turns out to be the intermittent thing

The decision above assumed that once the target is reachable whenever the machine
has a network, cadence stops being a question. **That was wrong in a way that took
one day to surface.** The laptop is frequently on a phone hotspot with a data
allowance, so the target is reachable and the *link* is not acceptable — which is
the same intermittency the four options were arguing about, relocated one layer
down.

The backup is therefore gated on `ExecCondition=/usr/local/bin/restic-not-metered`
([[../spec/backups#restic-metered-guard|`restic-metered-guard`]]). A non-zero exit
makes systemd **skip** the unit rather than fail it, so a tethered day produces an
`inactive` unit and one journal line instead of a red timer that gets ignored.

Measured 2026-08-31, on the hotspot: `Skipped due to 'exec-condition'`,
`systemctl is-failed` → `inactive`. **The skip path is the only one that has been
observed**; no unattended backup has yet run.

**`Persistent=true` and the guard pull against each other**, and the interaction is
the reason to record this here rather than in the spec alone. A catch-up run fires
shortly after boot, which is disproportionately often when tethered — so the clause
this record chose *specifically to cover the laptop case* is also the clause most
likely to attempt an upload at the worst moment. The guard is what makes it safe,
not an optional refinement of it.

**And this brings option B back.** A skip is silent by design, so a long tethered
stretch leaves an old remote copy with nothing complaining — the unbounded-lag
failure this record was written to object to, arriving through a door it did not
anticipate. Option B's own text still fits: *preserves the manual model, removes the
"forgot" failure. Requires deciding N.* Filed as
[[../backlog/backup-staleness-alarm|backup-staleness-alarm]], and it is now the
right answer rather than the stopgap this record called it.
