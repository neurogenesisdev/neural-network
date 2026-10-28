---
title: "A deferred one-token fix was ten; the last chapter had no PRD section behind it"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-24
updated: 2026-08-24
tags: [session, spec, containers, podman, testing, postconditions, schema]
---

# 2026-08-24 — A deferred one-token fix was ten; the last chapter had no PRD section behind it

Fifth round. The first ([[20260823-build-spec-schema|log]]) settled the schema,
the second ([[20260823-spec-boot-promotion|log]]) promoted `spec/boot`, the
third ([[20260823-spec-disk|log]]) wrote `spec/disk`, the fourth
([[20260824-spec-base|log]]) wrote `spec/base` and deferred two `grep -c`
aborts. This round paid that debt — and found the debt had been mis-sized by a
factor of five — then wrote [[../spec/containers|spec/containers]], deleting
`_worked-examples.md` as it emptied.

Artifacts: [[../spec/containers|spec/containers.md]], a seventh convention and
an amended fourth in [[../spec/_schema|_schema.md]], changed postconditions in
[[../spec/disk|spec/disk]] (3 steps), [[../spec/boot|spec/boot]] (5) and
[[../spec/base|spec/base]] (2), an aligned
[[../runbooks/tpm2-reenroll|tpm2-reenroll runbook]], an amended
[[../spec/_index|spec/_index.md]], and one deletion. This is what did not fit in
them.

## The deferral was right and its arithmetic was wrong

The previous round found two `grep -c` aborts in `spec/disk` and deliberately
did not fix them, on the grounds that "those records were tested against
mutation sets that were never written down, and a one-token change to a
destructive step's postcondition should arrive with its re-test rather than
ahead of it." That judgment was correct. **The count in it was not.**

Writing the mutation sets found **four** instances in `spec/disk`, not two. The
extra two are both in `resume-offset-matches-swapfile`, a step nobody had
flagged, tested only because the other two were being tested. Sweeping
`spec/boot` and `spec/base` afterwards found **six more** — four and two.

Ten instances, three chapters, and the tool was different almost every time:
`grep -c`, `grep -oE` inside a pipe under `pipefail`, `blkid`, `jq`, `findmnt`,
`passwd -S`, `cryptsetup status`, and a glob that matches no files and reaches
`sed` as a literal. Naming the class after `grep -c` was itself part of why it
had been under-counted.

**This is the third time a count in this spec has read plausibly while being
incomplete**, after the dangling-id measure failed twice. The pattern across all
three is the same: a number derived from *what someone had already noticed*,
presented as a measure of *what is there*.

## Two of the ten were checks no input could reach

This is the finding that outlives the fix, and it is why the general form went
into `_schema.md` rather than staying in a chapter's prose.

`resume-offset-matches-swapfile`:

```
signed_offset=$(grep -oE 'resume_offset=[0-9]+' <<<"$opts" | cut -d= -f2)
...
[ -n "$signed_offset" ] || { echo "  no resume_offset= in the signed entry"; fail=1; }
```

`grep -oE` exits 1 on no match, and `pipefail` carries that out past `cut`,
which exits 0. So a signed boot entry with no `resume_offset=` ended the
postcondition **three lines above the diagnostic written for exactly that
case**. `cmdline-write` in `spec/boot` had the identical shape with `blkid`, and
its dead line was `[ -n "$live_uuid" ] || fail=1`.

Both were written by someone thinking carefully about the empty case. Both
blocks *say* they handle it. Neither could. **Reading a block does not show
this** — the code looks complete, and the branch that cannot execute is the one
that would have told you. Running it against the input it claims to handle is
the only thing that shows it, which is the whole argument for a mutation set
over a careful re-read.

## The general form, and why it went where it went

> Under `set -e`, any filter whose "found nothing" answer is an exit status will
> delete the report of the thing it found nothing of.

That is now the seventh convention in `_schema.md`. It was first drafted as a
paragraph inside `spec/disk`, which was wrong twice over: it is a rule about how
checks get written rather than a fact about one chapter, and `spec/boot` turned
out to hold the worst instance in the vault. A convention living in the chapter
where it was noticed is a convention the next chapter will not consult.

**Six of the seven conventions now come from a specific failure rather than from
foresight.** The one-line verdict is still the only one proposed in advance. The
third round put that ratio at four of five and said it was worth noticing before
adding a convention that sounds sensible; two rounds later it has moved in the
same direction twice.

The convention also gained a scope clause the instances forced: **it applies to
postconditions, not to preconditions.** A precondition that aborts has done its
job — the executor contract says a failing precondition halts and reports, and a
precondition emits no verdict to lose. That asymmetry is the same one that makes
a `${VAR:?}` guard mandatory in both, arrived at from the opposite direction.

## Testing one thing found three others

The mutation sets were built to verify a one-token change. What they produced:

**A permanently-red check on a correct disk.** `disk-partition`'s type parse,
`sed -nE 's/^Partition GUID code:.*\((.*)\)$/\1/p'`, is greedy and binds to the
*last* parenthesis in the line. Against a type name that itself contains
parentheses — `Linux x86-64 root (/)` is the shape in question — it returns
`/)`, and the step reports `p2 type is '/)', want x86-64 root` **on a correctly
partitioned disk**. `[^(]*` binds to the first parenthesis and is identical on a
name without any, so the fix is safe under both readings. It was found because
the baseline case failed on the very first run of its own mutation set.

Whether the defect is live is `inferred`, and establishing that was itself a
finding: `sgdisk(8)` documents `-i` and **gives no example of its output**
(*measured*, read 2026-08-24). So the `Partition GUID code:` label and the
parenthesised format are as unsourced as the type strings the chapter already
marked `inferred`. Three ungrounded things in three lines, which strengthens an
open item that previously asked for one: a collector section recording partition
types would ground the label, the format and the strings at once. The set was
therefore run twice, against a type name with nested parentheses and one
without, and the fixed block behaves identically on both.

**A count that matched a prefix.** `grep -c '^/mnt'` counted a btrfs mount at
`/mnt2` toward the six. Nothing in this build creates one — but it is the
`' path @'`-matches-`@home` defect from `btrfs-subvols-create`, four steps up, in
a chapter that already names that class. Its mutation is the only row where the
fixed and unfixed columns disagree other than on the abort, and it disagrees in
the direction that matters: the unfixed block reports a failure that is not
there.

**A stack trace where a diagnostic belonged.** `tpm2-enroll`'s `jq` printed
`null (null) cannot be sorted` above its verdict when a token was bound to no
PCRs — which is *precisely* the case the step exists to detect, and which its
prose spends a paragraph explaining. The verdict was right; the message buried
it, which is the failure `_schema.md` names about output drowning an answer.
`// []` fixes it, and the same change had to go into the runbook copy.

## `findmnt` documents its exit status to be uninformative, and that is useful

`findmnt(8)`: "The exit value is 0 if there is something to display, or 1 on any
error (for example if no filesystem is found based on the user's filter
specification, or the device path or mountpoint does not exist)." (*measured*,
read 2026-08-24.)

That sentence settled two repairs. Collapsing the status with `|| mounts=`
discards **no distinction the tool was ever willing to make**, so the fix needed
no judgment call about whether information was being lost. And it means two
mutations — "nothing mounted" and "findmnt errored" — are deliberately
indistinguishable in the output, recorded rather than papered over.

The documented-exit-status question has now come up five times, and the set is
worth stating together:

| Tool | Its status is | So the check |
|---|---|---|
| `visudo -c` | documented and means one thing | uses it |
| `sbctl verify` | documented as 0 regardless | parses output |
| `bootctl` verbs in a chroot | documented, and `--graceful` disarms them | asserts the result instead |
| `findmnt` | documented, and conflates two causes | collapses it deliberately |
| `systemctl is-enabled` | documented, and 0 covers five non-enabled states | parses the word |

**Having a documented exit status is not the same as having a useful one.** The
lookup rule gets you the sentence; deciding what the sentence is worth is a
separate step, and it has gone the "do not trust it" way four times out of five.

## The last chapter had no PRD section behind it

`spec/containers` is the first chapter that replaces nothing. PRD §12 files a
container runtime under "Deliberately out of scope" — and the machine has run
rootless podman since 2026-08-17. Its sources are three ADRs and a build log.

**The PRD does touch it once, destructively.** §6.1 prescribes
`kernel.unprivileged_userns_clone=0` and then warns, in its own text, that this
"breaks rootless Podman… you will hit it the first time you install a container
runtime." No chapter has promoted §6.1, and `state/` **records no sysctls at
all**, so the vault cannot say whether it was applied here.

The step routes around that rather than asserting a knob name that may not exist
on this kernel: `podman info`'s `.host.security.rootless` is a live reading of
the capability §6.1 would remove. **Assert against live state, never against
constants** — reached from a direction the convention was not written for. Its
worked examples are about values that *vary* (a UUID, a path); this is a
constant that might not *exist*, and the same answer falls out.

## A check that succeeded the same way as the thing it was checking

`containers-subvol` arrived from `_worked-examples.md` requiring
`[btrfs-subvol-mount, user-create]`. Both are chroot-side, so the derived-context
rule classifies the step as chroot-side — and the step uses `$HOME`, which under
`arch-chroot` as root is `/root`. It would create a subvolume at
`/root/.local/share/containers`, on `@`, silently.

**Its postcondition passes on that**, because the check resolves the same `$HOME`
the command did.

`_schema.md` has a rule that a check must not fail the same way as the thing it
detects, and `btrfs-subvol-mount` already recorded the mirrored case — a check
that would *pass* the same way, via `findmnt -T` resolving to a parent
filesystem. This is a third face of it: not a shared failure and not a shared
path resolution, but **a shared unstated assumption**. The command and the check
both take `$HOME` on trust, so no disagreement between them is possible.

The repair is not a better postcondition, it is a precondition that reads
something neither of them assumed: `findmnt -no SOURCE -T "$HOME"` must report
`@home`. That is also, independently, the ADR-0006 nesting requirement — one
reading doing two jobs. And it is the chapter where `-T` is the *right* flag,
against `spec/disk` insisting on `-M`, decided by whether the path is supposed
to be a mountpoint. Same shape as `swapon --show` being correct in `spec/disk`
and wrong one chapter over.

## An open item that was closable, in the wrong place

`_worked-examples.md`'s last entry asked for a check that `@home` "holds no
orphaned overlay store" before `containers-subvol` runs, against the case where
podman ran, was uninstalled, and left a populated `graphRoot` behind.

**The scenario as written does not hold.** Rootless podman's store *is*
`~/.local/share/containers/storage` — the images live at the path being tested.
If the directory is gone they went with it. For images to survive its removal
they must have been elsewhere, which means a `storage.conf` had moved
`graphroot`.

**That version is real and was already asserted, one step later.**
`podman-first-pull`'s `podman info` reading reports `graphRoot` *after* both
`storage.conf` locations and the environment have been applied, so one reading
covers every way the store can be pointed elsewhere — better than any directory
scan, and it was in the example already. The entry asked for a check on `@home`
before `containers-subvol`; the check that answers it is on `podman`, after
`podman-install`.

**What remains is not closable and is not the step's business.** A previous
installation with a non-default `graphroot` can leave a store elsewhere under
`@home`, costing snapshot-pinned space — the 142 GB failure ADR-0006's context
cites. There is no documented signature for "abandoned containers store"; a
check would have to `find` for directory names upstream is free to change, and a
stale check that reports clean is worse than none. Recorded as an accepted
limitation with a named consequence.

**And examining it found something larger next to it.** The wrong-namespace
defect above has a silent failure mode and a postcondition that passes on it,
against which "podman might have run before" is narrow. That is the second round
running in which reading a recorded open item surfaced a different and bigger
defect beside it, after `spec/base` found `bootloader-install` while writing
`first-boot`'s `requires`.

## The dangling-id table reached zero and the spec is not finished

`podman-install` was the last entry. The table is empty, and three things this
round found are invisible to it:

- **`spec/devenv` does not exist and two ADRs name it.** ADR-0007's frontmatter
  says `constrains: spec/devenv`. ADR-0006 has two rules and `spec/containers`
  implements one; the bind-mount-not-named-volume half has no step anywhere.
- **PRD §6.1 is worse than absent**, since it prescribes a sysctl that would
  break the chapter written this round.
- **A chapter replacing no PRD section is invisible to a coverage audit that
  walks PRD sections** — which is the mechanism the previous two rounds
  recommended for finding what the count cannot see.

So the measure has failed in three distinct ways across four rounds: wrong
(counted new ids rather than new unresolved references), silent (a step nothing
references), and now **complete while the work is not** — because it measures
*resolution*, not *coverage*, and those happened to move together until they did
not. Recorded in `spec/_index.md` along with the way to measure the other thing.

## The harness is not the machine, in one way that is new

The standing limitation is that no postcondition in this vault has been executed
by whoever wrote it. Mutation sets narrow that and do not close it: `lsblk`,
`findmnt`, `sgdisk`, `bootctl`, `btrfs`, `podman`, `loginctl` and `systemctl`
were all stubbed, so what was tested is shell logic against output shapes taken
from `state/` and from man pages.

Two toolchain divergences bound how far that goes. `awk` in the devbox is **mawk
1.3.4** where the machine runs gawk — already recorded, and already responsible
for the `\<` escape that matched nothing and let a check pass while testing
nothing. New this round: `grep` there is **ugrep 7.5.0, not GNU grep**
(*measured*, 2026-08-24). The `grep -c` behaviour every fix in this round turns
on is POSIX-specified and holds for both, so nothing here is affected — but
anything resting on a GNU extension would test clean in the container and fail
on the machine, which is the mawk bug's exact shape waiting to happen again.

Every block was extracted from its finished document rather than retyped, which
is the one thing that makes the sets worth anything: a test written against a
copy of the code tests the copy. One correction came out of that discipline
rather than out of the blocks — a mutation was first recorded as gaining a
diagnostic from the fix, and the difference was a `grep` filtering the harness
output. Same shape as the `compress=zstd:1`-on-`/swap` reading the third round
retracted: a comparison that looked like evidence and was an artifact of how it
was read.

## Conclusions revised

| Claimed | Corrected to | Why |
|---|---|---|
| Two `grep -c` aborts in `spec/disk` | Four there, ten across three chapters | Mutation sets, then a sweep |
| The class is `grep -c` | Seven tools, incl. `grep -oE` under `pipefail`, `blkid`, `jq`, a missed glob | Naming it after one tool is why it was under-counted |
| The aborts suppress a diagnostic | Two suppress a diagnostic **no input could reach** | The `-n` guard sits below the assignment that aborts |
| The general form belongs in `spec/disk`'s prose | `_schema.md`, seventh convention | It is a rule about writing checks; `spec/boot` had the worst instance |
| `disk-partition`'s type parse works | Greedy `.*` binds to the last paren; `/)` on a correct disk | Its own baseline case failed |
| The type *strings* are the ungrounded part | The label and the output format are too | `sgdisk(8)` gives no example of `-i` output |
| `btrfs-subvolume(8)` does not specify `show`'s exit status | It has an EXIT STATUS section; it does not map conditions for `show` | Read 2026-08-24 |
| `containers-subvol requires [btrfs-subvol-mount, user-create]` | `[first-boot]` — the example put it chroot-side | `$HOME` is `/root` there, and the postcondition passes |
| `containers-subvol` must precede podman *pulls* | It must precede podman being *installed* | `podman.socket` lets any client create `graphRoot` |
| `podman-first-pull`'s postcondition checks the pull | It re-ran its own precondition and checked nothing | It asserted `graphRoot` and never asked for the image |
| The open item wants a scan of `@home` | Half is asserted already, one step later; half is not closable | The store *is* the path being tested |
| `_schema.md`'s parameter clause 3 applies to every parameter | Scoped to wrong-target hazards; shape is right for moving-reference ones | `$DEVBOX_IMAGE_DIGEST` has nothing in the DAG to bind to |
| An empty dangling-id table means the spec is resolved | It measures resolution, not coverage | `spec/devenv` is named by two ADRs and by no `requires:` |

**The two that matter are rows 3 and 8.** Both belong to the class the previous
round named as the dangerous one — things that would have looked like they
worked. A diagnostic below an aborting assignment, and a postcondition that
resolves the same variable as its command, are each *invisible to review* and
produce no symptom until the case they cover arrives, at which point the
artifact reports nothing rather than reporting wrongly.

Row 13 is the quieter one and may be the most durable: the measure this work has
used for progress across five rounds has now been wrong, silent, and complete
while the work was not. It should stop being cited as a progress measure at all.

## Still open

- **`spec/devenv` does not exist** and holds ADR-0006's bind-mount rule,
  ADR-0007's `constrains:` target, the `devbox` wrapper, `CODEX_HOME`,
  `CLAUDE_CONFIG_DIR` and the devcontainer template.
- **`state/` records no sysctls and no lingering.** Both are needed by
  `spec/containers` and both are currently `inferred`; the `Linger` property
  name is not in `loginctl(1)` at all, and the check reports an empty read as an
  empty read rather than as lingering being off. Two small collector sections
  close both.
- **`spec/containers` has no mutation sets and nothing in it has been executed.**
  `podman-first-pull`'s `RepoDigests` parse is the likeliest to be wrong: for a
  multi-architecture image the pull digest and the manifest-list digest differ,
  and which one `RepoDigests` reports is not on the page. If they differ, the
  check is red on a correct pull.
- **`luks-format` still has no mutation set**, and it is the other step in this
  vault whose failure mode is destroyed data.
- **`mkinitcpio-config` was tested against seven mutations that were never
  written down** — the same defect that deferred this round's fixes for a round,
  still outstanding in `spec/boot`.
- **PRD §§3.6–3.8, §5, §6 and §7 have no chapters**, and §3's vconsole ordering
  requirement still exists nowhere.
