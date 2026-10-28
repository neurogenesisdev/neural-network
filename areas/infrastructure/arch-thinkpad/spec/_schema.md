---
title: Step record schema and executor contract
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-23
updated: 2026-08-24
tags: [spec, schema, contract]
---

# Step record schema and executor contract

The build spec is a set of step records embedded as fenced blocks in Markdown
prose. Ordering is a DAG expressed through `requires`, not a numbered sequence —
promoting a section does not mean appending it, it means adding edges and
letting a topological sort re-derive a legal order.

There is no executor. This document defines the shape one could be written
against later; `spec-run` does not exist and is not planned for this round.

## The five fields

```yaml
id:            # unique, kebab-case. DAG node identity and transclusion target.
requires:      # list of ids. Edges, not line numbers.
precondition:  # optional. Shell. Must the world be a certain way before this runs?
command:       # optional. Shell. Absent means a human must satisfy this step.
postcondition: # shell. Did the step achieve what it was for?
```

`postcondition` is the only field that is always present. A step with no
postcondition is not a step, it is a suggestion.

## What is deliberately absent

Schema churn is the named failure mode for this artifact shape, so the argument
for each omission is recorded here rather than rediscovered.

**`rationale`.** The prose paragraph above the block is the rationale. A field
would give one idea two homes, and the paragraph is the one that gets edited.
The invariant is structural instead: **a step record must be preceded by prose,
and the unit of transclusion is the section, never the bare block.** A block
with no prose above it is a defect.

**`supersedes` / `superseded-by`.** `decisions/` is append-only; `spec/` is not.
The spec describes how to build the machine now. Retaining dead steps with
supersession pointers turns it into a changelog — the failure
`decisions/_index.md` warns about. A rejected configuration is recorded two
ways instead: the prose links the ADR, and the surviving step carries a
**negative postcondition** asserting the rejected state is absent. That fails
the build rather than sitting there being ignorable.

**`type` / `manual` / `interactive`.** Absence of `command` carries it, under
this rule:

> **A step with no `command` is one the executor cannot make true. It can only
> check it.**

That covers four shapes with one convention — a firmware menu or PIN entry, a
step whose command could not be looked up, a standing invariant that no action
establishes (`btrfs-default-matches-root` is the third kind), and an action that
destroys the executor performing it (`first-boot`: `reboot` does not return, so
no process that issues it can observe whether it worked). The earlier
formulation, "a human must satisfy this," was narrower and fitted neither the
invariant nor the reboot; this one covers both without being widened for them,
which is the argument for having phrased it this way. What matters is never the
menu path, which decays and appears in no man page; it is the observable result,
which is what the postcondition asserts.

*Known cost:* those four render identically in the block and are distinguished
only by the prose. Accepted deliberately — the alternative is an enum, and an
enum is how schema churn starts. The cost grows with each shape the rule turns
out to cover, so it is worth noting when a fifth appears.

**`provenance` / `verified` / `last_run`.** The governing principle: **the spec
records intent; `state/` and executor output record observation.** A
postcondition resolving at run time is `measured` by construction. Provenance
markers are claims about prose, so they live in prose and in frontmatter.

## The executor contract

This is what makes a judgment field unnecessary, so it is normative rather than
advisory.

> **A failing precondition halts and reports. The executor never remediates,
> never continues, and never re-runs a step to make a check pass.**

Under that rule every judgment step is already handled. TPM2 enrollment against
PCR 7 is not special-cased — it is the step where halting is *most often*
correct, not the only one where it is correct at all. A failing precondition
there usually means a firmware update legitimately moved the PCR, and the
repair is a different step, named in the prose by `id`.

**Trip-wire:** if an executor is ever built that continues past a failed check,
this argument collapses and the judgment field comes back. Revisit here.

## Conventions

**Assert against live state, never against constants.** Postconditions resolve
values at run time. `sbctl verify`, not a path under `/var/lib/sbctl/`.
`blkid`, not the UUID from the last build. Anything varying per machine or per
rebuild cannot be a literal in a step record. A UUID copied from a previous
install is wrong *and* an unredacted identifier.

**A parameter is named in prose and validated against the DAG, never merely
typed.** Some values cannot be derived from anything that exists before the step
runs: the target of `sgdisk -Z` is the clearest case, and `/dev/nvme0n1` is
exactly the enumeration-order literal the previous convention forbids. Three
steps in [[disk|spec/disk]] therefore take a parameter — `$DISK`, `$LUKSDEV`,
`$ESPDEV`. **No schema field records this**, for the same reason execution
context has none: it is already visible in the record, and a field duplicating
what the shell says is a field that goes stale. The convention is three parts:

1. **Named in the prose above the record**, with its shape and an example value.
2. **Guarded with `${VAR:?message}` in every field that reads it**, precondition
   and postcondition alike. A postcondition must stand alone — it is what a
   re-run checks and what gets duplicated into a runbook — so it cannot lean on
   a guard in the precondition having run first.
3. **Validated against something the DAG established, not against its type.**
   `luks-format`'s precondition first tested `test -b "$LUKSDEV"` and
   `! cryptsetup isLuks "$LUKSDEV"`. Both pass for `sda2` — 466 GB of
   unformatted space on the rescue drive (ADR-0009), which is attached during
   the build and exists to be readable when this machine is not. No property of
   the value's *type* separates the right partition from a catastrophic one.
   Tying it back to what `disk-partition` produced — partition 2 of `$DISK`, by
   `PARTN` — does.

Every destructive step will have that shape. The parameter names what gets
destroyed, so the check that matters is not what the value *is* but which
earlier step made it.

*Clause 3 is scoped to that hazard, and a later parameter showed where it
stops.* [[containers#podman-first-pull|`podman-first-pull`]] takes
`$DEVBOX_IMAGE_DIGEST`, and **nothing in the DAG establishes an image
reference** — it comes from outside the machine. Clause 3 has nothing to bind
to. But the hazard is also different: `luks-format`'s is *pointing at the wrong
object*, and [[../decisions/20260818-base-image-pinned-by-digest|ADR-0007]]'s is
*a reference that moves* — "A tag is a moving reference. A digest is
content-addressed and immutable." A shape check for `@sha256:` plus 64 hex
digits is not a weak stand-in for provenance there; it **is** the property the
ADR requires, because being pinned by digest is a fact about the string and
nothing else. So the clause reads: **validate against what the DAG established
when the hazard is a wrong target; validate the shape when the hazard is the
value changing underneath you.** A type check is the wrong answer to the first
question and the right answer to the second, and which question a step is asking
is decided by what going wrong would cost.

**Prefer output parsing to exit codes where the exit code is undocumented.**
`sbctl verify` and `btrfs subvolume show` both have undocumented exit-status
behaviour on failure. Postconditions that depend on them grep output and say so
in the prose, accepting the format fragility as the lesser risk.

**Emit a one-line verdict.** A postcondition should end with a single
`RESULT=PASS` / `RESULT=FAIL` line rather than a wall of output. A check whose
answer is buried under 400 lines is a check nobody reads and an agent cannot
parse.

**A check must not fail the same way as the thing it detects.** Learned from the
collector: `"tpm2-pcrs": null` from a wrong jq field name reads identically to an
enrollment bound to no PCRs. Where a check has an ambiguous failure output, the
prose names the disambiguator.

**A postcondition must reach its verdict line on every input it is meant to
judge.** Under `set -e`, an assignment from a failing command substitution ends
the script — and under `set -o pipefail` a filter anywhere in a pipe carries its
status out past everything downstream of it. So **any filter whose "found
nothing" answer is an exit status will delete the report of the thing it found
nothing of.** The check does not report zero; it reports silence, which is
indistinguishable from an executor that never ran the step.

That is the [[#Conventions|don't-fail-the-same-way]] rule applied to control
flow rather than to output, and it has been the most common defect in this spec
by a wide margin: **four instances across two chapters and three tools**, all
found by testing rather than by reading.

| Where | Construct | What the zero case was |
|---|---|---|
| `disk-partition` | `n=$(… \| grep -cx part)` | `sgdisk` did nothing — the most destructive step in the spec |
| `btrfs-subvol-mount` | `n=$(findmnt … \| grep -c '^/mnt')` | nothing mounted; two tools aborting on one line |
| `resume-offset-matches-swapfile` | `n=$(… \| grep -c .)` | the `options:` label moved, directly under a comment promising to distinguish it |
| `cmdline-write` | `live_uuid=$(blkid …)` | no LUKS mapping — and the `[ -n "$live_uuid" ]` check on the next line was unreachable |

The last two share the worst property: **a written diagnostic that could never
fire.** Reading the block does not show it. Only running it against the input it
claims to handle does.

Three repairs, in order of preference:

1. **Count with `awk`**, which exits 0 and prints `0` for no matches:
   `awk '$1 == "part" { n++ } END { print n + 0 }'`. Prefer this to `grep -c`
   anywhere a count feeds a comparison.
2. **Let the assignment fail to empty** — `x=$(…) || x=` — where the tool's own
   exit status carries nothing worth keeping. `findmnt(8)` is the worked case:
   "The exit value is 0 if there is something to display, or 1 on any error (for
   example if no filesystem is found…)", so the status cannot separate
   *not found* from *broken*, and collapsing it discards nothing.
3. **Drop `set -e` from the postcondition**, keeping `set -uo pipefail`, when
   several checks legitimately count zero and guarding each one would be noise.
   [[base#fstab-generate|`fstab-generate`]] is the worked case and says so in a
   comment.

**This applies to postconditions, not to preconditions.** A precondition that
aborts has done its job: the executor contract above says a failing precondition
halts and reports, and a precondition emits no verdict to lose. A postcondition
is the half that must produce a `RESULT=` line, and the half that gets
duplicated into a runbook — the same asymmetry that makes a `${VAR:?}` guard
mandatory in both.

*Residual, and left as one:* a check may still abort when the tool it reads is
absent or errors outright. That is a broken check rather than the condition
being detected, it reads differently from a reported zero, and chasing it would
mean wrapping every command. Name it in the prose where it is plausible.

**A command must survive being copied out with its indentation.** Step records
are indented inside their fenced block, and the unit of transclusion is the
section — so the way this text reaches a human is copied, with leading
whitespace, out of a rendered page. Shell ignores leading whitespace before a
command, which is why every other construct in these chapters survives that
trip.

*The worked case is a heredoc, which does not.* `bootloader-install` first wrote
`loader.conf` with `cat > … <<'EOF'`. That is correct as YAML — a literal block
scalar strips the common indentation and the terminator lands at column zero —
and correct nowhere else. Copied with two spaces of indent, the terminator no
longer matches: the shell either waits for a delimiter that never arrives, or
writes indented keys, after which the step's own
`grep -qE '^editor[[:space:]]+no$'` fails on a run that did exactly what it was
told. `<<-` is not the fix; it strips tabs, not spaces. `printf '%s\n'` with a
quoted argument per line is indentation-immune and is what the step uses.

The general form: **prefer constructs whose meaning does not depend on column
position.** Heredocs are the main offender; so is anything relying on a YAML
block scalar being parsed rather than read.

**The container has a plausible wrong answer to every host question.** The devbox is
Arch and the machine is Arch
([[../decisions/20260825-devbox-base-is-arch|ADR-0015]]), so `pacman`, `awk`, `grep`
and `sed` all exist on both sides and all answer. Before, a host command typed in the
container failed loudly; now it succeeds against the wrong data. Two mechanisms, and
both are about making the correct invocation *look* different from the incorrect one.

**1. Host package questions go through `--dbpath /host/pacman`.** The container's own
repositories build the container and answer nothing about the machine. The host's
database is bind-mounted read-only at `/host/pacman`, and every question about what the
machine has names that path explicitly. A bare `pacman -Q` in a step record is a defect
even when it returns something.

Measured 2026-08-25, against a read-only mount, identical under `--userns=keep-id`:

| Query | Result |
|---|---|
| `-Q` | works, 892 packages, no lock contention |
| `-Qi`, `-Ql` | work |
| `-Si` | works — returned `podman 6.0.2-3` from `extra` |
| `-Qo` | resolves against the **host** database, not the container filesystem |
| `-Qu` | exits **1** on a fully-updated host |

Three of those rows are traps:

- **`-Qo` is right for the wrong-feeling reason.** It answered correctly for
  `/usr/bin/sbctl` and `/usr/bin/tailscale`, neither of which exists in the image. It
  is resolving the path against the database, not the filesystem — so it is usable, and
  the reason it works is not the reason it looks like it works.
- **`-Si` reflects the last `pacman -Sy` the *host* ran.** A postcondition asserting
  availability is asserting the freshness of the host's sync database, not the state of
  the repositories. Say which one is meant.
- **`-Qu` exits 1 when there is nothing to list.** No postcondition may assert that
  `-Qu` succeeded: "fully updated" and "the command failed" share an exit status. Under
  `set -e` an assignment from it aborts the script, so this is the
  [[#Conventions|must-reach-the-verdict-line]] rule with a new tool attached — the
  zero case carried out as a status, deleting the report of the thing it found none
  of.

No lock is taken for read-only queries, so no copied snapshot of the database is needed
and the mount stays `:ro`.

**2. A tool is identified by its resolved path, never by the name the shell shows.**
Use `readlink -f "$(command -v <tool>)"`. Two measured failures, on two different axes,
on the same day:

- **The shell's name is not the binary.** `.zshrc` aliased `grep` to `rg` and `diff` to
  `difft`. The first reading of the host's toolchain recorded **"ripgrep 15.2.0" as the
  host's grep** — a wrong answer produced by a correct-looking command. The aliases are
  now commented out, which changes nothing about the rule: an alias is one edit away at
  all times, and [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]] already records that `alias grep='rg'`
  silently changes the *interface*, not just the output.
- **`command -v` is not the resolved path.** In the container `command -v gawk` returns
  `/usr/sbin/gawk` where `readlink -f` gives `/usr/bin/gawk`, because `/usr/sbin` is a
  symlink. A parity check comparing `command -v` output would report a divergence that
  does not exist — a check failing red on a correct system, which is the permanently-red
  shape [[disk#luks-format|`spec/disk`]] avoids.

The two mechanisms are one rule — **the host-question rule**: *a question about the
machine, asked from inside the container, must name what makes it a host question.*
`--dbpath /host/pacman` for packages; a resolved path for tools.

**A path shown to a human must name which side it is on, or be true on both** — **the
which-side rule**.

*Worked case, measured 2026-08-25.* The `/plan` command printed
`/home/dev/.claude/plans/…` — a container path, rendered into a terminal a human reads
on the host, where `/home/dev` does not exist at all. The file was real: it is at
`~/.local/state/devbox/claude/plans/…` host-side, which `/proc/self/mountinfo` gives
directly. Nothing errored, and nothing about the string said which filesystem it
belonged to. That is the previous convention's `command -v` trap again — **a
correct-looking string answering about the wrong side** — moved from what a check reads
to what a person is told.

Two repairs, and which one applies is decided by whether the path is a mount:

1. **Make it true on both sides.** Where the path is bind-mounted, mounting it at its
   host-identical absolute path means the printed string resolves for both readers and
   needs no annotation. This is
   [[../decisions/20260825-harness-state-and-identity|ADR-0017]]'s path parity, and it
   is the better repair in principle because it removes the failure rather than
   labelling it. **It is deferred**, so it is not available for project paths yet —
   which makes repair 2 the operative one for now, including for the `/plan` case that
   prompted this convention.
2. **Name the side.** Where a path is deliberately container-only, say so:
   `/host/pacman` is the worked example, and it was named that way before this
   convention existed — the prefix *is* the annotation.

The rule binds prose, diagnostics and postcondition output alike. A `RESULT=FAIL` line
quoting a container path to someone standing at a host shell has told them something
they cannot act on.

*Known limit:* this cannot be enforced mechanically, because a path is just a string
and both sides are plausible. It is a review rule, and the review question is one
sentence — *if this line were read on the other side, would it still be true?*

## Placeholders

Hand-written documents use the vault's placeholders — `<MACHINE_ID>`,
`<TS_IPV4>`, `<TS_IPV6>`, `<FSUUID_nvme0n1p2>`, `<NVME_SERIAL>`. A step record
should rarely need one: if a real identifier is about to be written into a
command, that is the signal the value should be derived at run time instead.
