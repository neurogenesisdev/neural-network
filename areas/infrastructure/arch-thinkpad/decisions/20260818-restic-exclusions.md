---
title: restic excludes the snapper snapshot tree and agent state
type: decision
id: ADR-0008
aliases: [ADR-0008]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-18
updated: 2026-08-26
tags: [decision, backups, restic, snapper, btrfs]
constrains: spec/backups
---

# ADR-0008 — restic excludes the snapper snapshot tree and agent state

- **Status:** accepted
- **Date:** 2026-08-18
- **Amended:** 2026-08-21 — the original text presented this as cost-free. It is
  not. The cost is named below.
- **Amended:** 2026-08-30 — the *fails open* residual fired, and the list was
  rewritten around four grounds rather than around tidiness. A rewrite dropped four
  of ADR-0017's credential exclusions without re-arguing them; declared as a defect
  and **repaired the same evening**. See *Amendment 2026-08-30* at the end.
- **Amended:** 2026-08-25 — the open question below is **answered**. The agent-state
  exclusion is narrowed from the whole directory to the credential and cache files,
  by [[20260825-harness-state-and-identity|ADR-0017]]. See *Amendment 2026-08-25* at
  the end of this record; the Decision block above is left as written.
- **Provenance:** measured

## Context

restic was descending into `/home/.snapshots`, re-reading the entire live tree
once per snapper snapshot. The 12 August run scanned 543 MiB; the 18 August run
scanned **73.005 GiB**. Nothing about the machine's data had changed by that
factor — snapper's timeline had simply filled.

## Decision

Add two exclusions to the restic invocation:

    --exclude '/home/.snapshots'
    --exclude '/home/luiswg/.local/state/devbox'

## Rationale

### What excluding the snapshot tree costs

**Off-disk historical granularity.** snapper retains 5 hourly / 7 daily / 2
monthly. restic currently holds two snapshots six days apart. Excluding the
tree means the T5 can only restore a file as it existed at a restic snapshot,
not as it existed at snapper snapshot #40.

That is a real loss, and the original version of this record did not state it.

### Why including the tree is nonetheless the wrong way to buy it back

**Parent matching breaks.** 186,844 of 189,689 files counted as "new" despite a
valid parent, because snapper's numbering means paths differ between runs
(`/home/.snapshots/40/...` vs `/45/...`). restic re-hashes byte-identical
content and writes a metadata tree that is stale by the next run. You pay a
large scan cost for history that is recorded badly.

**It is a step change, not a drift.** Every further snapper tick adds another
full copy of `~/Videos` to the scan set. The 73 GiB figure grows without bound.

**Storage was never the problem.** Deduplication works correctly: the 18 August
run scanned 73.005 GiB and stored 7.455 GiB, with the whole repo at 7.617 GiB
in raw-data mode. **No prune is needed.**

### The correct mitigation

**Increase restic cadence** — see ADR-0012. More frequent restic snapshots buy
back the granularity with correct parent matching and bounded scan cost. Same
benefit, properly recorded.

### Why snapshots have no disaster-recovery value regardless

btrfs snapshots live on the same NVMe as the live filesystem. If the drive
fails, every snapshot fails with it. No retention policy changes this. Their
job is undo, not recovery.

### Agent state

`~/.local/state/devbox` holds agent OAuth tokens, regenerable with one login.
Backing them up puts long-lived credentials in the repo for no recovery
benefit. Snapper `home` snapshots still capture them; that is unavoidable
without another nested subvolume and is not worth one.

## Consequences

- Off-disk history is only as granular as the restic cadence. That cadence is
  currently undecided (ADR-0012) and is now **load-bearing for this decision**.
- Agent tokens must be re-authenticated after a restore. One login each.
- Full exclusion set: `--exclude-caches`, `/home/.snapshots`, `~/.cache`,
  `~/.local/share/Trash`, `~/.local/share/containers`,
  `~/.local/state/devbox`, `/var/log/journal`.
- **Open question raised 2026-08-21.** Claude Code session history is not agent
  state in the disposable sense — `~/.local/state/devbox/claude/` holds
  `sessions/`, `projects/`, `history.jsonl` and `plugins/` alongside
  `.credentials.json`, and the current exclusion discards all of it. If that
  history is worth keeping, narrow the exclusion to the credential files rather
  than the whole directory.
- If Flatpak is adopted, exclude `~/.local/share/flatpak` but **not**
  `~/.var/app` — the latter is per-app data. One glob would catch both; write
  them as separate lines.

## Revisit when

ADR-0012 is decided. If the answer is a remote repo with a frequent cadence,
the granularity cost of this exclusion drops to near zero and this record
returns to being uncontroversial.

## Amendment 2026-08-25 — the open question is answered

The consequence above asked, in its own words, to "narrow the exclusion to the
credential files rather than the whole directory" if the session history is worth
keeping. [[20260825-harness-state-and-identity|ADR-0017]] decides that it is, and
narrows it. Derived state — `sessions/`, `projects/`, `history.jsonl`,
`file-history/`, `plans/` — is now **included**; credentials and cache are excluded by
name:

    # replaces:  --exclude '/home/luiswg/.local/state/devbox'
    --exclude '/home/luiswg/.local/state/devbox/claude/.credentials.json'
    --exclude '/home/luiswg/.local/state/devbox/claude/.claude.json'
    --exclude '/home/luiswg/.local/state/devbox/claude/plugins'
    --exclude '/home/luiswg/.local/state/devbox/claude/paste-cache'
    --exclude '/home/luiswg/.local/state/devbox/claude/shell-snapshots'
    --exclude '/home/luiswg/.local/state/devbox/claude/backups'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/auth.json'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/log'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/tmp'

**The cost this record did not name, and which decides the credentials row.** The
*Agent state* section above argues from recovery benefit — tokens regenerate with one
login, so backing them up buys nothing. True, and not the strongest argument. The
stronger one is about **revocation**: a restic repository's retention is not a
credential's lifetime. A token deliberately revoked after a suspected compromise
persists in every snapshot taken before the revocation, and a semi-automatic restore
re-materialises it. That turns "revoked" into "revoked unless someone restores," which
is a property nobody chose.

**`.claude.json` joins the exclusion** because it carries the OAuth account. That is
affordable only because the collector now extracts the MCP server list and per-project
trust into `state/harness.md`
([[../spec/base#collect-harness-config|`collect-harness-config`]]) — the record
survives although the file does not.

**Two residuals, accepted and named.** restic is exclude-based, so the list **fails
open**: a harness that invents a new credential file is backed up until someone
notices. And the transcripts now included are a verbatim record of everything typed and
pasted into the container, which is how a copy of a credential gets into a backup even
when the credential store itself is excluded. ADR-0017 argues both at length and
records the behavioural mitigation: *do not paste credentials into a session*, and
*treat a restore of `projects/` as containing secrets*.

## Amendment 2026-08-30 — the list failed open on schedule, and was rewritten around four grounds

Written while standing up the remote repository
([[20260830-remote-repository-is-b2|ADR-0028]]). Everything in this section is
`measured` on the host on 2026-08-30 unless marked otherwise.

### The residual fired, and the interval was five days

The *fails open* warning above is not hypothetical any more. A directory listing of
`$STATE/claude` and `$STATE/codex-state` found entries that did not exist when
ADR-0017 enumerated them on 2026-08-25: `session-env/`, and in `codex-state` a whole
SQLite set — `logs_2.sqlite` with its `-wal` and `-shm`, `state_5.sqlite`,
`queue_1.sqlite`, `goals_1.sqlite`, `memories_1.sqlite`, `thread_history_1.sqlite` —
plus `models_cache.json`, `cache/`, `shell_snapshots/`, `thread-writer-locks/`,
`rules/`, `skills/` and `installation_id`.

None of them is a credential, so nothing leaked. **The finding is the interval.** The
list was written on 2026-08-25 with a named silent-failure mode, and five days later
thirteen new entries had appeared under it. Whatever the right maintenance cadence is,
it is shorter than a fortnight, and nothing currently triggers it.

### The scan was 41.913 GiB, and 12 GiB of it was a virtual disk

The first backup against B2 was interrupted after two minutes because the progress
line showed it uploading
`~/.config/Claude/vm_bundles/claudevm.bundle/rootfs.img` — the Claude Desktop VM's
disk image. `du -xh --max-depth=1` puts `vm_bundles` at 12 GiB against 22 GiB for the
whole of `$HOME`.

This is the [[#Context|`/home/.snapshots`]] shape again in a new location: not a large
file, but a large file **whose internals change on every use**, so each nightly run
would pay a fresh delta for content with no recovery value. Excluding it, with
`.cache`, `discord/` and `baloo/`, took the scan to 20.015 GiB.

### Size alone is not a ground for an exclusion, and this round nearly made it one

An intermediate version of the list excluded `claude-code/2.1.246` (237 MB, a
versioned application payload), `models_cache.json`, `shell_snapshots/`,
`thread-writer-locks/`, `logs_2.sqlite*` and `.cargo/registry`. The operator's
objection was that this is not much data, and it was correct.

At B2's $6.95/TB/month, **237 MB costs about a fifth of a cent per month.** Set
against that: every line is a line that can go stale, and every line widens the
fail-open surface documented above. A rule that costs more attention than the storage
it saves is a net loss.

So the grounds are now explicit, and a candidate exclusion has to name one:

| Ground | Examples |
|---|---|
| **Correctness** — backing it up is wrong, not merely wasteful | pseudo-filesystems, `/swap` (a 36 GB swapfile), `/.snapshots` and `/home/.snapshots` recursion |
| **Credentials** — the revocation argument, above | `.credentials.json`, `.claude.json`, `auth.json` |
| **Churn** — a large delta every run for content with no recovery value | `vm_bundles`, `.cache`, `baloo` |
| **Large and rebuildable** — restored by reinstalling, not by restoring | `discord/`, `/var/cache`, `/var/log/journal`, `containers`, `Trash` |

Retention caps the churn case independently: `--keep-daily 7 --keep-weekly 5
--keep-monthly 12` means at most ~24 surviving copies of any file, so a 4 MB
database rewritten daily costs under 100 MB in the worst case (`inferred` —
arithmetic, not observed).

### The list as installed, 2026-08-30

`/etc/restic/excludes.txt`, used by both the B2 and T5 invocations:

    /proc /sys /dev /run /tmp /var/tmp /mnt /media /lost+found
    /swap
    /.snapshots
    /home/.snapshots
    /home/luiswg/.config/Claude/vm_bundles
    /home/luiswg/.config/discord
    /var/cache
    /home/luiswg/.cache
    /var/log/journal
    /home/luiswg/.local/share/Trash
    /home/luiswg/.local/share/containers
    /home/luiswg/.local/share/baloo
    /home/luiswg/.local/state/devbox/claude/.credentials.json
    /home/luiswg/.local/state/devbox/claude/.claude.json
    /home/luiswg/.local/state/devbox/codex-state/auth.json
    /home/luiswg/.local/state/devbox/claude/plugins
    /home/luiswg/.local/state/devbox/claude/paste-cache
    /home/luiswg/.local/state/devbox/claude/shell-snapshots
    /home/luiswg/.local/state/devbox/claude/backups

The last four were appended after the defect below was declared; the ordering of the
file reflects that history and nothing else.

`--exclude-caches` and `--one-file-system` are passed on the command line rather than
in the file.

### The defect this amendment is required to declare

**Four lines from ADR-0017's set are missing from the list above**, and they were
dropped on cost grounds during the simplification, not by re-arguing the record that
put them there:

    /home/luiswg/.local/state/devbox/claude/plugins
    /home/luiswg/.local/state/devbox/claude/paste-cache
    /home/luiswg/.local/state/devbox/claude/shell-snapshots
    /home/luiswg/.local/state/devbox/claude/backups

**`paste-cache` is the one that matters.** ADR-0017's second residual says in terms
that the transcript stream is *"the likeliest place for a copy of the very credentials
the exclusion removes"*, and `paste-cache` is the directory that holds pasted content
as files. Excluding it was the one mechanical measure against that residual, and it
was removed by a rule about storage cost — which is exactly the category error the
section above was written to prevent, committed in the same round, two paragraphs
later.

The two `codex-state` lines (`log/`, `tmp/`) went the same way with less at stake.

**Repair performed 2026-08-30**, after the operator read this argument. The four
`claude/` lines are back in `/etc/restic/excludes.txt` and appear in the list above.
They are credentials-ground exclusions, not size-ground ones, so the simplification
never applied to them.

The two `codex-state` lines (`log/`, `tmp/`) were **not** restored. They are
churn-and-rebuildable, not credentials, so the simplification did apply to them and
they stay out on their own merits.

**The snapshots taken before the repair are unaffected by it.** The first B2 snapshot
(`02cd13b3`) and the T5 snapshot (`2d1578be`) contain `paste-cache` as taken, and
nothing rewrites them — a `forget`/`prune` cycle to expunge one directory is not worth
its risk here. A restore of either should be treated as containing secrets, which is
the handling ADR-0017 already prescribes for the transcript trees regardless.
Snapshots from the next run onward do not carry it.

**Declared, then repaired, in one round.** Worth noting as a sequence rather than
collapsing into "the list is correct": the defect was written down before it was
fixed, so the record of *how a credentials-ground exclusion got dropped by a
storage-cost argument* survives the fix. That mechanism is the reusable part; the four
lines are not.
