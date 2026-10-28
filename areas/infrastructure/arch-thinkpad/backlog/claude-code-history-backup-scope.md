---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [backups]
tags: [restic, claude-code, agent-state, backups]
created: 2026-08-21T00:00:00Z
updated: 2026-08-26T00:00:00Z
resolved: 2026-08-25
commit: pending
---

> [!SUCCESS] Resolved 2026-08-25 by [[20260825-harness-state-and-identity|ADR-0017]]
> **Narrow the exclusion.** Session history, `projects/` and `history.jsonl` are backed
> up; `.credentials.json`, `.claude.json`, `auth.json` and the cache directories are
> excluded by name. [[20260818-restic-exclusions|ADR-0008]] carries the amendment.
>
> **This item predicted its own residual and was right.** The *Consideration* below
> warns that narrowing "means enumerating credential files by name and keeping that
> list current as tooling changes — a maintenance burden with a silent failure mode."
> That is exactly what was accepted: restic is exclude-based, so the list **fails
> open**, and a harness that invents a new credential file is backed up until someone
> notices. It is named as a residual in ADR-0017 rather than solved.
>
> One thing this item did not anticipate: the transcripts it argues for keeping are a
> verbatim record of everything pasted into a session, so they are the likeliest place
> for a copy of the very credentials the exclusion removes. ADR-0017 accepts that too,
> with a behavioural mitigation rather than a detector.

# Decide whether Claude Code history belongs in the backup set

> [!NOTE] Summary
> `~/.local/state/devbox` is excluded from restic wholesale. When that exclusion was written the directory meant "OAuth tokens." It now also holds `sessions/`, `projects/`, `history.jsonl` and `plugins/` — actual work history, discarded on every backup.

## The question

Narrow the exclusion to the credential files (`.credentials.json`, `auth.json`, `.claude.json`) and back up the rest? Or accept that agent session history is disposable?

## Consideration

Backing up the directory as-is puts long-lived OAuth tokens in the restic repo. Narrowing the exclusion means enumerating credential files by name and keeping that list current as tooling changes — a maintenance burden with a silent failure mode.

## Related

- [[20260818-restic-exclusions|ADR-0008]] — raised there as an open consequence
