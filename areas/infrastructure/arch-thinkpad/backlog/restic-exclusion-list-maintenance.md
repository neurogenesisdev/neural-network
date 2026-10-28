---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [backups]
tags: [backlog, backups, restic, credentials, agent-state]
created: 2026-08-30
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Give the exclusion list a review trigger

> [!NOTE] Summary
> Two things, one cause. **Half is done:** four exclusions dropped from ADR-0017's set on 2026-08-30 were restored the same evening. What remains is the reason they could be dropped unnoticed — an exclude-based list with a documented silent-failure mode, thirteen unreviewed new entries in five days, and nothing that triggers a re-read.

> [!SUCCESS] Part 1 resolved 2026-08-30
> The four `claude/` lines are back in `/etc/restic/excludes.txt`, and
> [[../decisions/20260818-restic-exclusions|ADR-0008]]'s amendment records the repair
> alongside the defect rather than in place of it. The two `codex-state` lines
> (`log/`, `tmp/`) stay out — they are churn-and-rebuildable, so the simplification
> legitimately applied to them.
>
> **Snapshots taken before the repair still contain `paste-cache`** — B2 `02cd13b3`
> and T5 `2d1578be`. Not rewritten; a restore of either is treated as containing
> secrets, per ADR-0017.

> [!SUCCESS] Part 2 resolved 2026-09-02 — the collector option, `measured`
> `harness.md` now ends with `## agent state directories (top level)` — the
> names one level under `$STATE/claude` and `$STATE/codex-state` — followed by
> `## restic exclusions`, the live `/etc/restic/excludes.txt`. A new entry in
> either directory arrives as a diff beside the list that should mention it,
> which is the review trigger this half asked for. Names only, one level deep,
> through the redaction filter like every block.

## 1. Put the four lines back — done 2026-08-30

    /home/luiswg/.local/state/devbox/claude/plugins
    /home/luiswg/.local/state/devbox/claude/paste-cache
    /home/luiswg/.local/state/devbox/claude/shell-snapshots
    /home/luiswg/.local/state/devbox/claude/backups

`paste-cache` holds pasted content as files.
[[../decisions/20260825-harness-state-and-identity|ADR-0017]] says the transcript
stream is *"the likeliest place for a copy of the very credentials the exclusion
removes"*, and excluding this directory was the one mechanism standing against
that. It was removed by an argument about storage cost, in the same round that
established storage cost is not a ground for exclusions.

These are **credentials-ground**, not size-ground, so the simplification never
applied to them.

**Kept here after the fact** because the *mechanism* is the reusable part and the
four lines are not: a credentials-ground exclusion was removed by an argument about
storage cost, in the same round that established storage cost is not a ground. Part
2 exists because nothing would have caught that except someone re-reading the list
against the rules it had just been given.

[[../spec/backups#restic-exclusions|`restic-exclusions`]] now asserts `paste-cache`
by name for the same reason — a postcondition that would not have caught the one
failure this list has actually suffered is not much of a postcondition.

## 2. Give the list a trigger

Between 2026-08-25 and 2026-08-30, thirteen new entries appeared under
`$STATE/claude` and `$STATE/codex-state` — `session-env/`, six SQLite databases and
their journals, `models_cache.json`, `cache/`, `rules/`, `skills/`. None was a
credential, so this instance cost nothing. The next one might not be.

An exclude-based list has a silent failure mode by construction and **no schedule**.
Options, cheapest first:

- **A collector section.** `collect.sh` already generates `state/harness.md`; a
  listing of the two state directories would put new entries into a diff, where
  they get reviewed by the same mechanism as everything else. This is the one that
  fits the vault's existing machinery and needs no new habit.
- **A named review cadence** in ADR-0008. Weakest option — it is the habit that
  just failed.
- **Invert the list.** An include-based backup fails *closed*: a new file is missed
  rather than captured. That trades a confidentiality failure for an availability
  one, which is arguably the better trade for a credential directory and the worse
  one for everything else. Would need its own ADR.

## Related

- [[../decisions/20260818-restic-exclusions|ADR-0008]] — the 2026-08-30 amendment declares the defect and recommends the repair
- [[../decisions/20260825-harness-state-and-identity|ADR-0017]] — the record being narrowed
- [[claude-code-history-backup-scope]] — predicted the fail-open residual in 2026-08-21, before it had a name
- [[collector-desktop-blind-spots]] — the same shape in a different subsystem: the collector cannot see what it was not told to look for
