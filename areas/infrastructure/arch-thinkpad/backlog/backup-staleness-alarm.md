---
type: backlog-item
status: resolved
priority: high
device: arch-thinkpad
projects: [backups]
tags: [backlog, backups, restic, systemd, metered]
created: 2026-08-31
updated: 2026-09-02
commit: pending
---

# Notify when the newest snapshot is older than N days

> [!NOTE] Summary
> The metered guard makes a skipped backup **silent by design** — `inactive`, one journal line, no failure. That is correct behaviour on a tethered day and the wrong behaviour on a tethered fortnight, and nothing distinguishes the two.

## Why this exists now and did not before

[[../decisions/20260818-backup-cadence-open|ADR-0012]] listed four options and chose
D, a remote plus a timer, on the reasoning that A, B and C were all workarounds for
a target that is only *sometimes attached*. The metered guard added on 2026-08-31
reintroduces exactly that property through a different door: the target is always
reachable, but the **link is only sometimes acceptable**, and the backup is
suppressed on the rest.

So **option B — "a timer that does not back up but notifies when the last snapshot
is older than N days" — is back**, and it is now the right answer rather than a
stopgap. The record's own words for it apply unchanged: *preserves the manual model,
removes the "forgot" failure. Requires deciding N.*

The difference is that in August the thing being forgotten was plugging in a drive.
Now it is noticing that a run has been quietly skipping for a while, which is harder
to notice precisely because the mechanism is working as specified.

> [!SUCCESS] Resolved 2026-09-02 — `measured`
> Built as the item leaned: a **local marker**, not a B2 query. `restic-b2.service`
> gained `StateDirectory=restic` and an `ExecStartPost=touch /var/lib/restic/last-success`,
> which systemd runs only after every `ExecStart=` succeeded — so the marker
> means *backup and prune both completed*. A user-session timer
> (`/etc/systemd/user/restic-b2-stale.timer`, daily, `Persistent=true`, plus
> three minutes after login) runs `/usr/local/bin/restic-b2-stale-check`, which
> exits 1 and raises a **critical desktop notification** when the marker is missing,
> older than `N`, or `restic-b2-check.service` is in `failed` — the check's
> "second case" from `restic-restore-untested` rides along. The user unit shows
> `failed` in `systemctl --user --failed` until the next successful backup.
>
> **Three decisions taken by assumption, each cheap to revise:**
> - **N = 7 days**, the item's own first guess, as `RESTIC_STALE_DAYS` in the
>   script's environment. Not argued from a loss tolerance; revise there.
> - **Desktop notification only.** The item's own objection stands — a closed
>   laptop sees nothing — and the phone route is still the netcup question.
> - **Unit failure is the second channel.** It is not gated on the metered guard.
>
> First run, on install: `No successful B2 backup has ever been recorded by
> restic-b2.service`, unit `failed`, notification shown. That is the true state —
> the timer has never got past the guard — and it will nag daily until an
> unmetered night. Spec step: [[../spec/backups#restic-staleness-alarm|`restic-staleness-alarm`]].

## Shape

A second timer, daily, that reads the newest snapshot's timestamp and warns past a
threshold. It must **not** be gated on the metered guard — the warning is cheap and
is most needed exactly when the backup is being skipped.

Three decisions to make:

- **N.** A week is the obvious first guess and has no argument behind it yet. The
  useful framing is the one ADR-0012 used: how much work is acceptable to lose, given
  that snapper covers mistakes and the T5 covers drive failure. This is only about the
  off-site copy.
- **How it reaches a human.** A desktop notification is easy and is missed when the
  laptop is closed, which correlates with the tethered condition. Sending it to the
  phone would cross into the netcup server's territory and is a bigger question.
- **Whether it reads B2 or a local marker.** Querying `restic snapshots` needs the
  network and the credentials — on a metered link, for a check about being on a
  metered link. Recording the last successful run locally (a timestamp file written by
  a third `ExecStart=`) answers the same question offline and cannot be fooled by a
  repository that is unreachable rather than stale. **The local marker is probably
  right**, and it is the less obvious of the two.

## Related

- [[../decisions/20260818-backup-cadence-open|ADR-0012]] — option B, resurrected for a reason it did not anticipate
- [[../spec/backups#restic-metered-guard|`restic-metered-guard`]] — the mechanism that created the gap
- [[restic-restore-untested]] — the other half of "nobody has watched this work"
