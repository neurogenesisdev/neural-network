---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [vault]
tags: [git, hooks, redaction, verification]
created: 2026-08-26T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# The pre-commit hook is tracked now, but activation is still per-clone

> [!NOTE] Summary
> The redaction gate moved from `.git/hooks/pre-commit` to `.githooks/pre-commit` on 2026-08-26, so its text is finally in the repository. Two things remain outside it: `core.hooksPath` must be set by hand in every clone, and nothing enforces that the hook's pattern set matches `redact.sh`'s.

> [!WARNING] Scope note — the subject is repo-wide, the file is not
> This item is filed under `arch-thinkpad/` and carries `device: arch-thinkpad`,
> but **the control it describes is repo-scope**: `.githooks/pre-commit` guards
> every device folder, including `hive/` and `pixel-graphene/` when they arrive.
>
> `device: shared` was considered and rejected. It is an established value — the
> template offers it and the root `_index.md` lists it — but no content note has
> ever used it, `shared/` is empty, and the root index scopes that area to
> cross-device **decisions** ("SSH key model, Tailscale ACLs"). A backlog item
> would not belong there even once it is populated. Filing it here with the scope
> in prose is the smaller inaccuracy; the alternative was inventing a
> `shared/backlog/` area for one item.
>
> Revisit if a second device is migrated and this list needs splitting.

> [!SUCCESS] Resolved 2026-09-02 — parity enforced, activation documented and warned, `measured`
> **Parity:** the hook now extracts its own `-e '…'` lines and `redact.sh`'s,
> and refuses to run if the sorted sets differ — five patterns, identical on
> the day. `redact.sh` itself is untouched, as the item wanted.
> **Activation:** still per-clone, and accepted as such. The step is written
> into the repo's `CLAUDE.md`, and `collect.sh` prints a stderr warning when
> `core.hooksPath` is not `.githooks` — the collector is the thing run just
> before a commit, so the warning lands where the gate matters. The residual
> is a fresh clone that never runs the collector and commits prose; that is
> the documented step's job.

## What was wrong, and what the move fixed

`.git/hooks/` is not part of a repository. The gate blocking unredacted UUIDs, MACs
and Tailscale addresses therefore existed on **exactly one machine**, could not be read
from the vault, and was not reviewable — while `state/_index.md` described it as though
it were part of the system. A second device, which the per-device layout explicitly
anticipates, would have cloned with **no gate and no indication one was missing**.

Moved to `.githooks/pre-commit` at the repository root rather than under
`arch-thinkpad/collect/`, because the control is repo-scope: it guards `hive/` and
`pixel-graphene/` too when those arrive. It now diffs, reviews and versions alongside
`collect.sh` and `redact.sh`.

## What is still outside version control

**`core.hooksPath` is local config.** A fresh clone gets the file and no gate until
someone runs:

    git config core.hooksPath .githooks

That is strictly better than before — *visible and inert* beats *invisible and absent*
— but it is not automatic, and the failure mode is silent in the same way the old one
was: nothing reports that the hook is present but unused. Options, in rough order of
cost: document it in the root `README`/`CLAUDE.md` as a clone step; assert it in a spec
step record beside `collector-run`; or accept it and say so.

**Pattern-set parity is unenforced.** `redact.sh`'s warning pass and the hook must
detect the same shapes. They silently did not until 2026-08-26 — `redact.sh` carried
`\b[0-9a-f]{32}\b` (machine-id shaped) and the hook did not, so the **weaker** detector
was the one with the final say before a push to a remote. They match today because both
were edited in the same sitting, which is not a mechanism.

The two are deliberately asymmetric in **severity** and must not be in **coverage**:

| | `redact.sh` | `.githooks/pre-commit` |
|---|---|---|
| Severity | warns, exits 0 — a collector must finish writing | blocks, exits 1 — last point a human is present |
| Coverage | **must be identical** | **must be identical** |

A single shared pattern file sourced by both is the obvious fix and the reason it was
not done here: it changes `redact.sh`, which is on the collector's critical path, and
that deserves its own change rather than riding along with a documentation fix.

## Related

- [[../state/_index|state/_index.md]] — the MAC paragraph and the layering explanation
- [[collector-stale-file-detection]] — the other verification gap found the same day
- [[collector-posix-awk-lint]] — the third; that one is portability, this is enforcement
