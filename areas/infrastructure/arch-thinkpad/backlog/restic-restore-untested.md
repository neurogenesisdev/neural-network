---
type: backlog-item
status: active
priority: high
device: arch-thinkpad
projects: [backups]
tags: [backlog, backups, restic, b2, verification]
created: 2026-08-30
updated: 2026-09-08T00:00:00Z
commit: pending
---

# The backups are unverified in three separate ways

> [!NOTE] Summary
> Two repositories now hold a current snapshot of `/`, `/home` and `/var/log`. Nothing has been restored from B2, and nothing has watched either timer do the thing it exists to do. The same failure twice: a mechanism nobody has observed working.

> [!INFO] Partial progress 2026-08-31
> The timer is **installed and enabled** — next elapse `Tue 2026-09-01 00:10:28
> CEST`, `systemd-analyze verify` silent — and its **skip** path was exercised
> deliberately on the hotspot: `Skipped due to 'exec-condition'`, unit `inactive`
> rather than `failed`. What remains of item 2 is the path that actually uploads
> something, which only an unattended run on an unmetered link demonstrates.

> [!SUCCESS] Item 3 resolved 2026-08-31
> The monthly integrity check is scheduled rather than intended.

> [!WARNING] Item 3's first scheduled run did not happen — 2026-09-01/02, `measured`
> The timer fired on 2026-09-01 18:25 while the machine was offline: restic retried
> DNS for the B2 endpoint for fifteen minutes and the unit ended `failed` —
> **`network-online.target` was satisfied by a link with no route out**, the
> `network-online-is-real` step's stated risk. Re-run by hand on 2026-09-02 15:09 on
> the phone hotspot: `Skipped due to 'exec-condition'`, guard correct, unit back to
> `inactive`. So the check has still never read a byte from B2; the next automatic
> attempt is `Thu 2026-10-01`. A failed-check state is now surfaced by
> [[backup-staleness-alarm]] rather than only by the journal.

> [!SUCCESS] Items 1 (cheap version) and 3 measured 2026-09-02; item 2 still open
> **Restore works.** `restic restore latest --host arch-thinkpad --target <scratch>
> --include /etc/fstab --include /home/luiswg/.zshenv` from the 2026-08-30 B2
> snapshot; both files `cmp` byte-identical to the live copies. Password file,
> index and transport all exercised together. The runbook's commands were run as
> written and can lose their `inferred` marker for the single-file path — the
> off-host version, from KeePassXC only, has still not been done.
> **Check passes.** Run by hand as a transient unit bypassing the metered guard
> once, on the operator's say-so: `restic check --read-data-subset=5%` →
> `no errors were found`, 52 s wall clock. The scheduled unit itself has still
> never got past its guard. The repository held **one** snapshot, not two —
> the earlier one had been pruned by the retention policy.
> **Item 2 remains**: the timer has never taken a backup; every elapse so far was
> on the hotspot. `backup-staleness-alarm` now nags daily until it does.

> [!SUCCESS] T5 side measured 2026-09-02, and a latent defect in both B2 units fixed
> **T5 backup taken** by hand (`snapshot 137ea200`, 747 MiB added, 27 s) and a
> **two-file restore** from it — `/etc/fstab` and `~/.ssh/config` — compared
> identical. Both repositories have now given a file back.
> **The B2 timer would have failed on its first real run.** The first T5 attempt,
> run as a transient system unit, died with `unable to locate cache directory:
> neither $XDG_CACHE_HOME nor $HOME are defined`. System services get no `$HOME`;
> `restic backup` requires a cache and `restic check` does not (it makes a
> temporary one, which is why the check passed earlier the same day). The
> scheduled `restic-b2.service` had the same shape and had never got past its
> guard to show it. Both units now carry `CacheDirectory=restic` and
> `--cache-dir /var/cache/restic`; verified with a transient unit of the same
> shape listing T5 snapshots. Diagnostic in [[areas/infrastructure/arch-thinkpad/state-notes|notes]].

> [!WARNING] Item 2, 2026-09-08 — the timer has now fired past the guard three times and failed each time
> 2026-09-04 17:33, 09-05 21:30, 09-06 00:09: the `Persistent=true` catch-up ran on
> resume while the Wi-Fi was associated with no internet behind it
> (`CONNECTED_LOCAL`); DNS for the B2 endpoint failed for fifteen minutes, exit 1.
> So the *proceed* branch has been reached and the unit itself works up to the
> network — the cache-dir fix held — but no backup has landed. The guard now also
> requires NM connectivity `full` (spec amended), so this class skips cleanly
> instead of failing. Still waiting for one unmetered, online night. The staleness
> alarm has been correctly red throughout.

## The three

**1. No restore has been performed.** This is the one that matters. Until a file
comes back out, the evening's work produced an upload, not a backup — password,
index and transport are all untested together.
[[../spec/backups#restic-restore-test|`restic-restore-test`]] is the step;
[[../runbooks/restic-restore|runbooks/restic-restore]] is the procedure, and it is
marked `inferred` for this reason.

*Cheap version:* one small file into `/tmp/restic-test` and `cmp` against the live
copy. Ten seconds.

*The version that would actually establish something:* a restore performed from a
machine that is **not** this laptop, using only the KeePassXC copy of the password
and the B2 console for the endpoint. That is the scenario the remote exists for,
and it is the one the cheap version cannot reach — the netcup server's own item
makes the same argument in its own words: *a restore procedure that requires the
host being restored is not a procedure*.

**2. The timer has never taken a backup.** Installed and enabled on 2026-08-31, and
both existing B2 snapshots were still taken by hand. Two clauses remain unobserved
and both fail quietly: `Persistent=true` only shows itself after a missed window,
and the metered guard's *unmetered* branch — the one that proceeds — has never been
reached, because every test so far ran on the hotspot. **A guard that has only ever
said no is indistinguishable from a guard that always says no.**

**3. ~~Nothing schedules `restic check`.~~ Done 2026-08-31.**
`restic-b2-check.timer` is enabled — monthly, `--read-data-subset=5%`, behind the
same metered guard, next elapse `Tue 2026-09-01 01:56:36 CEST`.
[[../spec/backups#restic-check-timer|`restic-check-timer`]] is the step.

**What it does not do is tell anyone.** A failed check is not self-correcting the way
a missed backup is — it means the repository is damaged — and it reports to the
journal and nowhere else. The postcondition asserts `is-failed` as well as
`is-enabled` for that reason, but a postcondition nobody runs is not a notification.
That half moved to [[backup-staleness-alarm|backup-staleness-alarm]].

## Why one item and not three

They resolve together in a single sitting, and separating them invites the cheap
two to be closed while the expensive one stays open. The order is 1, 2, 3 by
consequence.

**Not in scope here:** noticing when the backup has been skipped for a long time.
That is a different failure — a mechanism working as specified and producing a bad
outcome, rather than one nobody has watched — and it is
[[backup-staleness-alarm|backup-staleness-alarm]].

## Related

- [[../decisions/20260830-remote-repository-is-b2|ADR-0028]] — its *Revisit when* names items 1 and 2
- [[../decisions/20260818-backup-cadence-open|ADR-0012]] — item 2 is the confirmation it lacks
- [[../../netcup-vps/backlog/restic-client|netcup-vps/restic-client]] — *an untested backup is a hypothesis*, and the off-host restore argument
- [[backup-staleness-alarm|backup-staleness-alarm]] — the adjacent gap the metered guard opened
