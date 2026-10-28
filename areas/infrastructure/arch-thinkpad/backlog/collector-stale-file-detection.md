---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [vault]
tags: [collector, state, verification, staleness]
created: 2026-08-26T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Detect a `state/` file left behind by a previous collector run

> [!NOTE] Summary
> If `collect.sh` aborts partway, the files it never reached keep the previous run's content. They exist, they are non-empty, and the git diff shows them unchanged — which reads as "nothing changed" rather than "nothing ran". Nothing currently detects this.

> [!SUCCESS] Resolved 2026-09-02 — `measured`, and by a different shape than proposed
> The collector now writes every block into a `mktemp -d` staging directory and
> copies the set into `state/` **only after the last block completed** (`set -e`
> takes care of the rest). An aborted run therefore changes nothing — no mixed
> state — which is the failure this item described, and it happened for real
> the same day: a run aborted on the user-bus block and left `services.md` 71
> lines short while every other file was fresh. The per-file marker was not
> built; instead one generated file, `state/collector-run.md`, carries the run
> timestamp and the collector's git revision. That is the one churning line the
> item's constraints allowed, and it separates *nothing changed* from *nothing
> ran* in the diff. Cross-file equality is now true by construction.

## How it surfaced

Adding the harness block on 2026-08-26 introduced a reference to an undefined variable
under `set -u`. Caught before it ran, but the failure it would have produced is the
interesting part: the collector aborts at that line, and `services.md`, `hardware.md`,
`boot.md`, `storage.md` and `network.md` are simply not rewritten. Every one of them
still holds the last good run.

**`bash -n` passes on that file.** So does
[[../spec/base#collector-run|`collector-run`]]'s postcondition, which asserts each state
file is present and non-empty — both true of a stale file. The prose justifying that
check originally claimed it caught this; it does not, and the claim has been corrected.

## What does cover it today, and what does not

| Mechanism | Covers |
|---|---|
| `set -euo pipefail` + the executor contract | the abort, **if the exit status is looked at** |
| Block ordering — harness moved last 2026-08-26 | blast radius, not detection |
| `collector-run`'s `-s` check | a file that is empty or missing |
| — | **a file that is stale** |

The gap is narrow but it is the one with no symptom. A human running
`./collect/collect.sh` interactively sees the error; a human running it as part of a
longer sequence, or an agent summarising "collector run, committed", does not.

## What would close it

A **per-run marker** written into every file and asserted to match across all of them.
One value generated once per invocation, emitted into each block's output, and checked
for equality by a postcondition — files disagreeing means they came from different runs.

Two constraints on the shape, both from rules this vault already has:

- **It must not churn meaninglessly.** `_index.md`'s anti-patterns forbid recording what
  the system manages on its own. A wall-clock timestamp in every file makes every run a
  diff in every file, which is exactly the noise that made the full package version list
  a bad idea. A monotonic counter or a short random token has the same problem.
- **The check is equality across files, not a value.** So the marker's *content* need
  not be meaningful and must not be asserted against a constant — only that all seven
  agree. That also means the diff noise is unavoidable by construction, and the trade is
  seven changed lines per run against a failure mode with no symptom.

Worth weighing against simply **asserting the exit status is looked at**, which is free
and covers the realistic case. The marker covers the case where someone ignores a
non-zero exit, which is a different and rarer failure.

## Related

- [[../spec/base#collector-run|`collector-run`]] — the step whose prose this corrects
- [[collector-posix-awk-lint]] — the other verification gap this round found, and the
  distinction between them: that one is portability, this one is detection
