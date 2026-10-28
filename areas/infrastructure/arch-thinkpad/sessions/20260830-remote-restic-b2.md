---
title: "A blocked item's premise had expired, and the exclusion list was rewritten twice in opposite directions"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-31
commit: pending
created: 2026-08-30
updated: 2026-08-31
tags: [session, backups, restic, b2, netcup, exclusions, credentials, metered]
---

# 2026-08-30 — A blocked item's premise had expired, and the exclusion list was rewritten twice in opposite directions

Seventh round. The remote restic repository was set up against Backblaze B2 and
the T5 was re-pointed at the same path set, closing
[[../backlog/remote-restic-repository|remote-restic-repository]] and deciding
[[../decisions/20260818-backup-cadence-open|ADR-0012]] after twelve days.

Artifacts: [[../decisions/20260830-remote-repository-is-b2|ADR-0028]], the
decision on ADR-0012, a third amendment to
[[../decisions/20260818-restic-exclusions|ADR-0008]],
[[../spec/backups|spec/backups]], [[../runbooks/restic-restore|restic-restore]],
and two new backlog items. This is what did not fit in them.

## The item was blocked on a resource that no longer existed

`remote-restic-repository` listed three options. The Hetzner VPS was known gone.
The Storage Box was described as *"already owned (`u531077`, 1 TB, BX11), already
reachable over SFTP"* — and it is also gone, which the operator said when asked
and which nothing in this vault could have established.

**The nine days matter more than the fact.** That line was written 2026-08-21 and
was `inherited-unverified` from the moment the box lapsed. For the whole of that
window it was the *load-bearing input to the one decision the item blocked*: a
reader planning this work would have gone straight to SFTP, and the netcup
comparison that turned out to be the actual question would never have been asked.

The vault has a rule for exactly this — *ground claims in `state/`, not in
`_source/`* — and it does not reach here, because **`state/` has no section for
anything off-machine.** A collector that runs on the laptop cannot see whether a
subscription at another company is still paid. The same blindness covers the B2
bucket written tonight.

This is the [[20260828-desktop-kde-migration|previous round]]'s negative finding in
a new place. There it was *the vault cannot answer whether a package is installed*;
here it is *the vault cannot answer whether a resource still exists*, and the
second is worse, because a package question has a command that would answer it.

## Two claims about netcup, and the reason was wrong on the one that was right

The round opened with a prior from a web search: netcup's local block storage is
*unsuited for remote restic backup due to reachability*. Both halves of the
comparison came out differently.

**Block storage is reachable.** It attaches to the RS 1000 as a device; a
`rest-server` or SFTP endpoint on that host over the Tailscale mesh is a working
restic target. The reason to reject it is **correlation** — the copy would sit on
the same machine at the same provider as one of the systems it insures — and that
reason survives into the future-second-remote question, where "unreachable" would
have wrongly closed the door.

**Storage Space is the one that is unsuited**, and nothing in the prior mentioned
it. netcup's help centre says it is **NFS only**, access-controlled by IP
allowlist. The client is a laptop that roams; there is no address to allowlist,
and NFS over the public internet is unencrypted anyway. It is also the most
expensive option in the comparison and is deleted if the vServer is cancelled.

**The general shape:** a rejection reached for the wrong reason looks identical to
one reached for the right reason, and only the wrong one fails when circumstances
move. The lookup rule's insistence on the primary source is what separated them —
the vendor's own documentation, fetched rather than recalled.

## The exclusion list was rewritten twice, in opposite directions, in one round

This is the round's most useful sequence and the one an artifact would flatten.

**First rewrite: too little.** The list was ADR-0017's, and the first B2 backup
scanned **41.913 GiB**. It was interrupted two minutes in — not by a check, but
because the progress line happened to show it uploading
`~/.config/Claude/vm_bundles/claudevm.bundle/rootfs.img`. A `du` then put
`vm_bundles` at 12 GiB of the 22 GiB in `$HOME`.

**Nothing would have caught that.** No postcondition asserts a scan size, no
report compares runs, and the daily timer would have paid a fresh delta on a
virtual disk every night, silently and forever. It was found by a human reading
scrolling output, which is not a mechanism.

**Second rewrite: too much.** The repair over-corrected into tidiness — excluding
a 237 MB application payload, a 4 MB log database, a cache JSON, a lock directory,
`.cargo/registry`. The operator stopped it with one question: *it isn't that much
data though, right?*

He was right, and the arithmetic settles it. At $6.95/TB/month **237 MB costs about
a fifth of a cent per month**, while every line added to an exclude-based list
widens the fail-open surface ADR-0017 had already named. The rewrite produced the
four grounds now in ADR-0008 — correctness, credentials, churn, large-and-rebuildable
— and **size is deliberately not one of them.**

### And then the same round made the error it had just defined

Simplifying dropped four ADR-0017 lines, including `paste-cache` — the directory
holding pasted content as files, and the one mechanical measure against ADR-0017's
own stated residual that *transcripts are the likeliest place for a copy of the
very credentials the exclusion removes*.

**It was dropped by a rule about storage cost, two paragraphs after establishing
that storage cost is not a ground.** The four lines are credentials-ground; the
simplification never applied to them.

Both snapshots taken tonight contain `paste-cache`. **The order of the repair is
the part worth keeping.** The defect was written into ADR-0008 *before* it was
fixed — declared, handed to the operator with the argument, applied, and only then
recorded as repaired. Writing the fix and the finding in one motion would have left
the list correct and the mechanism invisible, and the mechanism is what recurs.

The four lines are back as of the same evening; the two `codex-state` lines
(`log/`, `tmp/`) stayed out, because for those the simplification legitimately
applied. Snapshots already taken are not rewritten — expunging one directory is not
worth a `forget`/`prune` cycle, and ADR-0017 already prescribes treating a restore
of the transcript trees as containing secrets.

**The pattern worth keeping:** a principle established mid-round does not
retroactively govern the edits made while establishing it. Nothing re-read the
list against the grounds it had just been given — which is why
[[../spec/backups#restic-exclusions|`restic-exclusions`]] now asserts `paste-cache`
by name, and why the surviving backlog item is about a **trigger** rather than
about four lines.

### The fail-open residual fired, with a five-day fuse

ADR-0017 enumerated `$STATE/claude` and `$STATE/codex-state` on 2026-08-25. Listing
them on 2026-08-30 found thirteen entries that had not existed then — `session-env/`,
six SQLite databases with journals, `models_cache.json`, `cache/`, `rules/`,
`skills/`, `installation_id`.

None was a credential. **The finding is the interval**, not the contents: a list
with a documented silent-failure mode and no review trigger drifted by thirteen
entries in five days. Options for a trigger are in the backlog item; the cheapest
adds a listing to the collector, so drift arrives in a diff instead of waiting for
someone to look.

## Two credentials that existed in only one place, one of them unknown to its owner

Asked where the T5 repository password was stored, the operator's answer was that
he had *never saved it — we always used the password file*. It was at
`/root/.restic-password`, found by grepping shell history for `--password-file`.

**Using a password file was correct.** The defect is that the file lived on the
encrypted disk of the machine it was backing up, so the repository and the only
copy of its key shared a failure mode — and for the duration of the first B2
upload, so did the new one. A backup nobody can open after the event it insures
against is a long-running job, not a backup.

Both are now in KeePassXC, confirmed synced off-device. That is what makes ADR-0028
a decision about recovery rather than about storage.

**The vault never recorded where that password lived**, which is why the question
had to be asked at all rather than looked up. `spec/backups` names both locations
now.

## Revised conclusions

| Claimed | Corrected to | Why |
|---|---|---|
| The Storage Box is available (backlog item, 2026-08-21) | It is gone | Operator, asked before writing any steps |
| netcup block storage is unusable for restic — unreachable | Usable; rejected for **correlation** | Vendor docs; the objection is topology, not connectivity |
| The relevant netcup product is block storage | **Storage Space** is the one that disqualifies itself — NFS-only, IP-allowlisted | netcup help centre |
| ADR-0008's 73 GiB → 7.4 GiB ratio predicts this repo | It does not; that ratio was **deduplication of near-identical snapshot copies**, and this input has none | 20.015 GiB → 12.809 GiB measured |
| The repo will be ~7.6 GiB (the T5's size, assumed transferable) | 12.809 GiB — the T5 covered `/etc`, B2 covers `/` | `restic snapshots` on the T5 |
| Excluding the 237 MB payload, the log DB and the caches is worth doing | It is not; size is not a ground | $0.0016/month, against a wider fail-open surface |
| Run the two backups serially — they would fight for I/O | Fine in parallel; separate repos, separate caches, and B2 is upload-bound | Reversed within the same round, before either run |
| `tee` with placeholder content had to be cleaned up | `tee` truncates; the second run replaced the file wholesale | `awk`/`wc`/`stat` check, printed no secret |

Row 4 is the one that would have misled someone else. A compression ratio recorded
in a decision record reads as a property of the data, and it was a property of
*that input's redundancy*. The record does say so — "deduplication works correctly"
— but the memorable number is the ratio.

Row 8 is the round's cheapest good habit: verifying a credential file by
**shape rather than content** — first four characters per key, line count, mode and
owner. It catches the two failures that matter (placeholders left behind, a
trailing newline in the password) without putting the secret into scrollback.

## The one check that failed loudly, and why that is worth recording

The upload hit `Temporary failure in name resolution` when the WiFi dropped, retried
the lock save six times over ~33-second backoffs, and completed:
`Save(<lock/761378b14a>) operation successful after 6 retries`.

Every other finding in this round is about something that failed *silently* — a
stale subscription, a drifting list, a 12 GiB image nobody was watching for. This
one announced itself, retried, said how many attempts it took, and carried on. It
is the behaviour the nightly timer depends on and the only part of tonight that
demonstrated it under a real fault.

## Accepted limitations

- **Nothing in this round was run by the agent.** Every measurement is terminal
  output pasted by the operator from the host. The two are not the same: a
  transcript is trimmed, and no `RESULT=` line was produced by anything. The
  postconditions in `spec/backups` were written *against* that output, not *from*
  executing it.
- **`spec/backups` ships with two postconditions that have never been true.**
  `restic-b2-timer` and `restic-restore-test` are red by design until
  [[../backlog/restic-restore-untested|restic-restore-untested]] closes. Precedent
  is `devbox-identity-markers`; the difference is that these two assert the
  *purpose* of the chapter rather than a mitigation within it.
- **The bucket, endpoint and key are deliberately absent from every artifact.**
  Postconditions read them from `/etc/restic/b2.env` at run time. That follows
  *assert against live state* and the redaction rules, and it costs something: a
  reader without the machine cannot reconstruct the repository URL from this vault.
  `runbooks/restic-restore` names the two places to find it instead.

## Continued 2026-08-31 — the decision's premise was wrong within a day

The timer was not installed on the 30th, and the reason turned out to be a finding
rather than an oversight: **the laptop is often on a phone hotspot with a data
allowance.**

ADR-0012's decision, written hours earlier, argues that once the target is reachable
whenever the machine has a network, cadence stops being a question. That is wrong,
and the correction sits one layer down: the target is reachable, the *link* is not
acceptable, and **that is the same intermittency options A, B and C were arguing
about**, relocated from the drive to the connection. A decision that dissolved a
question had in fact moved it.

The mechanism is `ExecCondition=`, which makes systemd **skip** a unit on a non-zero
exit rather than fail it. The distinction is the whole value: a skipped run leaves
`inactive` and one journal line, where a failed one leaves a red timer that gets
ignored after the third occurrence.

**The guard had to be given its answer.** NetworkManager guesses `metered=no` for a
phone hotspot, because a hotspot over wifi looks like ordinary wifi — so the check is
only worth `nmcli connection modify <hotspot> connection.metered yes`, which is
per-connection state on the machine and nothing the script can establish. A guard
that silently guesses wrong is worse than no guard: it converts *I forgot* into
*I was told it was handled*.

It also had to be told to ignore `tun` devices. With Mullvad as a Tailscale exit node
the default route is `tailscale0`, so the obvious implementation — ask whether the
route is metered — answers about the tunnel rather than about what carries it. That
is the *host-question rule* from `_schema` in a new place: a correct-looking query
answering about the wrong layer.

### The negative path was the one that got tested, and that was luck

Being tethered at the time meant the guard could be exercised where it matters:
`METERED`, then `Skipped due to 'exec-condition'`, then `is-failed` → `inactive`.

**Every check written in this vault so far has been confirmed by seeing it pass.**
This one was confirmed by seeing it refuse, which is the harder half and usually the
untested one — the round's own `restic-b2-timer` postcondition is green while the
thing it guards has never run. The asymmetry is now recorded in the backlog item:
*a guard that has only ever said no is indistinguishable from a guard that always
says no.* Both branches need seeing, and only one has been.

### The fix reopened the question the decision had closed

A silent skip is correct for a tethered day and wrong for a tethered fortnight, and
nothing distinguishes them. That is the unbounded-lag failure ADR-0012 was written
to object to, arriving through the door its own fix opened, and it makes **option B
the right answer rather than the stopgap that record called it** —
[[../backlog/backup-staleness-alarm|backup-staleness-alarm]].

Worth noting where the alarm should read from: querying `restic snapshots` needs the
network and the credentials, on a metered link, to answer a question *about* being on
a metered link. A local last-success marker answers offline and cannot confuse
*unreachable* with *stale*. The obvious implementation is the wrong one again, for
the third time in two days.

### One convention bent, deliberately

The guard script is stored at `scripts/restic-not-metered` and the step record reads
it from there, rather than embedding it. `_schema` forbids heredocs in step records
because a terminator does not survive being copied with indentation — but the
`printf` alternative, for a script containing an `awk` program that uses both quote
characters, needs three layers of escaping and was unreadable enough that a
transcription error in it would have passed review unseen. **Obeying the convention
produced the failure the convention exists to prevent**, so the step names a file
instead. First time a `command:` in this spec has pointed outside its own block.

## Still open

- **No restore has been performed** from B2. The operator reports one having been
  done at some earlier point; which repository it was against is unestablished, and
  a T5 restore exercises neither the B2 password nor the S3 transport — the two
  things that are new. Left open rather than closed on an ambiguity. The runbook is
  `inferred` for the same reason and the backlog item leads with it.
- **The timer is enabled but has never taken a backup**, and neither
  `Persistent=true` nor the guard's proceed branch can be verified except by
  letting a window pass on an unmetered link.
- ~~`NetworkManager-wait-online.service` has not been checked.~~ **Closed
  2026-08-31: `enabled`.** Promoted from prose to a postcondition
  ([[../spec/backups#network-online-is-real|`network-online-is-real`]]) on the
  grounds that it is a dependency living outside the unit that needs it — the class
  of thing that gets disabled by an unrelated change and reports nothing. Had it
  been `disabled`, the symptom would have been indistinguishable from the wifi drop
  of the 30th.
- **`restic check` is scheduled by nothing** — a habit where ADR-0012 is the vault's
  own worked example of what happens to backup habits.
- **`state/` cannot see backups at all.** No collector section reports repository
  size, snapshot age, timer status or last-run outcome, so the freshest thing this
  vault will ever know about the backups is whatever a human last pasted into a
  session. Belongs with
  [[../backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]] rather
  than as its own item, but it is a different axis: that one is *the collector does
  not look*, this one is *there is nothing on this machine to look at* for the
  remote half.
- **The netcup server still has no backup**, and its item wants an off-provider
  target too. Whether both hosts share one is the condition under which ADR-0028's
  Storage Box arithmetic is re-read.
