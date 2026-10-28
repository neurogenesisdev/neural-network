---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [vault, devenv]
tags: [collector, awk, gawk, portability, testing]
created: 2026-08-26T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# Lint the collector's awk and shell, now that the container matches the host

> [!NOTE] Summary
> [[../decisions/20260825-devbox-base-is-arch|ADR-0015]] makes the container's `awk` gawk, identical to the host's. That closes the mawk class — and it removes the accidental cross-implementation testing that *caught* it. [[devbox-contents-audit]] named this as the condition attached to installing gawk. It is now due.
>
> **Scope warning up front:** this covers portability. It does **not** cover vacuous correctness — a check that is valid in every implementation and asserts nothing. Both defects this round produced were valid gawk. See the scope note.

> [!INFO] Full review of the collector 2026-09-03; shellcheck deferred to the unmetered session
> Every block was read end to end. No portability defect found: the awk programs
> use `match`/`substr`/`index`/`sub`/`gsub` and `-v`, all POSIX; the one gawk-only
> construct in the file's history (`\<`) is gone. Findings were **churn**, not
> correctness, and were fixed in the same pass: removable disks in `lsblk` (letters
> shift per plug), ephemeral ports in `ss` (Syncthing, tailscaled's TCP side,
> Firefox), usbguard rule numbers, drkonqi transient units in the session graph,
> and a snapper *count* recorded where the configured *limits* were meant. One new
> exception was needed immediately — tailscaled's fixed UDP port lies inside the
> ephemeral range and is read from `/etc/default/tailscaled` to stay literal. The
> collector also now runs as root natively (`SUDO_USER` or uid 1000 as the
> operator, `--machine` for the user manager) instead of through a `PATH` shim.
> `shellcheck` is 3 MB plus Haskell deps behind a stale mirror database; `-Sy`
> alone would be a partial upgrade, so it waits for the next `-Syu`. The item
> stays open for that one run.

## The trade this pays for

The `\<` word-boundary escape is a gawk extension. It matched nothing under the
container's mawk, so a collector verification **passed while exercising nothing at
all** — `devbox-contents-audit` calls that "the worst failure shape available, a check
that fails the same way as a pass."

It was found *because the container was stricter than the target.* ADR-0015 removes that
asymmetry deliberately, on the grounds that a divergence which catches one bug by
accident and hides the next by the same mechanism is not a testing strategy. But the
audit's caveat stands and was accepted as a consequence:

> If gawk is installed, the collector needs a POSIX-only lint from somewhere else, or
> the next extension goes in unnoticed.

Nothing currently provides that. Every `awk` program in `collect/collect.sh` and in the
spec's postconditions is now written and executed against gawk on both sides, and a
second gawk extension would be invisible until it ran somewhere else — which, for the
collector, is nowhere, and for a step record copied into a runbook, is anywhere.

## What would actually close it

Not "be careful." Options, roughly in order of cost:

- **`gawk --posix` or `--traditional` as a lint pass** over the collector's awk
  programs. Cheapest, and it is the same binary answering about itself — channel 1.
  Needs the programs to be extractable, which today they are not: they are inline
  single-quoted strings inside a shell script.
- **`busybox awk` or `mawk` as a second reader** in CI or in a pre-commit hook. Restores
  the asymmetry deliberately instead of by accident, and unlike the accident it reports
  rather than silently passing. Costs a second implementation on the host, which is the
  thing ADR-0015 just removed — worth noting the tension rather than pretending there
  is none.
- **Nothing, and accept it.** Defensible only if written down as an accepted
  consequence with the failure named, which is what this item would become.

## Scope note — this covers one of two gaps, and not the one that bit hardest

This is about the **collector and the spec's postconditions**, not about general shell
style. The reason those two matter is that both are checks: a broken check reports
clean, and a clean report is what nobody investigates.

**But portability is only half the problem, and the round that produced this item
demonstrated the other half twice.** Both defects were **valid gawk**, so neither a
POSIX lint nor a second implementation would have seen either:

| Defect | Why a lint misses it | What would catch it |
|---|---|---|
| A counter reader whose `$2` was the literal string `packets`, so `n += $2` coerced to 0 and the check reported "no drops" under every condition | Syntactically and semantically valid in every awk. It computes something; the something is wrong | A **negative control** — run the check against input known to contain drops and assert it reports non-zero |
| `$STATE_DEVBOX` referenced before assignment under `set -u`, which would have aborted the collector partway and left five state files holding the previous run's content | Not an awk problem at all, and `bash -n` passes | **shellcheck (SC2154)** |

So there are two gaps: **portability** — the same program meaning different things to
different implementations — and **vacuous correctness** — a program that is valid
everywhere and asserts nothing. This item covers the first. The second wants a
different instrument, and the general shape is a check that must **demonstrate it can
fail**: assert a non-zero reading against known-positive input, not merely that two
readers agree, because two broken readers agree at zero.

shellcheck is the cheap immediate win and is not currently run against `collect.sh` or
against any postcondition block. Worth doing regardless of what happens to the awk
question.

## Related

- [[devbox-contents-audit]] — where the condition was set
- [[../decisions/20260825-devbox-base-is-arch|ADR-0015]] — where it came due
- [[../spec/_schema#Conventions|`_schema.md`]] — the conventions that most depend on awk
  behaving as expected
