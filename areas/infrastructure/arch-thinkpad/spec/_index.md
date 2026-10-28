---
title: Build specification — chapter map
type: index
device: arch-thinkpad
provenance: measured
stale: false
last_verified: 2026-08-26
commit: pending
created: 2026-08-23
updated: 2026-09-02
tags: [index, spec]
---

# Build specification

The refactor of `_source/Arch_Linux_Setup_PRD_T14-AMD-Gen1_v5.md` into step
records. Read [[_schema|_schema.md]] first — it defines the five fields and the
executor contract, and the chapters assume both.

Nothing here is executable. `spec-run` does not exist and is not planned for
this round; the value is that a claim which cannot be written as a postcondition
is a claim nobody had actually checked.

## Chapters

| Chapter | Status | Covers |
|---------|--------|--------|
| [[boot\|spec/boot]] | written 2026-08-23, extended and swept 2026-08-24 | cmdline, mkinitcpio config, UKI generation, systemd-boot install and its NVRAM entry, `loader.conf`, sbctl key enrollment, UKI signing, Secure Boot, TPM2 enrollment |
| [[disk\|spec/disk]] | written 2026-08-23, re-tested 2026-08-24 | partitioning, LUKS format, btrfs filesystem, subvolume creation and mounting, the default subvolume, ESP, swapfile |
| [[base\|spec/base]] | written 2026-08-24, swept the same day | `pacstrap`, the generated fstab and its two edits, user and sudo, the first boot |
| [[containers\|spec/containers]] | written 2026-08-24, amended 2026-08-26 | rootless podman and its socket, the container storage subvolume, the digest-pinned first pull |
| [[devenv\|spec/devenv]] | written 2026-08-26 | the agent image's toolchain parity, its ephemeral writes, its identity markers — **not** the build itself |
| [[backups\|spec/backups]] | written 2026-08-30 | restic against B2 and the T5 — credentials, the exclusion file, repository init, the daily timer, the restore test |

`_worked-examples.md` is gone. It was a review artifact holding five examples
written to test the schema; the last two left for `spec/containers` on
2026-08-24, and its own instruction was to delete it rather than leave an empty
shell. Everything it said survives in the chapters that absorbed it, including
the corrections made on the way — the two promoted this round each had a defect
the file did not record.

The two 2026-08-23 session logs still link to it, and those links are left
broken on purpose: a session log records what existed when it was written, and
editing one to tidy a link would make it a worse record of the round it
describes.

## The DAG is not a reading order

Ordering is expressed through `requires`, and a topological sort re-derives a
legal order. Chapters group by subsystem so a human can find things; they are
deliberately *not* an execution sequence, and the two orders genuinely differ.

**`first-boot` is an execution-context boundary, not an ordinary node.**
Everything reachable before it runs in the installer or under `arch-chroot`.
Everything after runs on the booted system. That boundary is not decoration: it
is what produced the `swapon --show` bug caught while writing these steps — a
precondition that consulted the kernel's idea of the swap path, which inside the
chroot is the path recorded at `swapon` time (`/mnt/swap/swapfile`) rather than
the one the step meant. Prose carried that error indefinitely without anyone
noticing which namespace it ran in. A precondition could not.

`spec/boot` straddles the boundary: `cmdline-write`, `mkinitcpio-config` and
`uki-generate` are chroot-side; `firmware-setup-mode` onward are not. So the
chapter cannot be executed top to bottom, and `spec/base` — which defines
`first-boot` — has to interleave with it.

`spec/disk` sits entirely on the near side, and shows that the boundary hides a
finer one. Its ten steps run in the **live installer**, before `pacstrap` and
before any chroot exists — which the derived rule cannot distinguish from
chroot-side, because the observable tell it uses (`sudo` or not) is the same for
both. The distinction that does bite is paths: `/mnt/swap/swapfile` in that
chapter is `/swap/swapfile` to `cmdline-write` in this one, and they are one
file.

**A step's context is derived, not declared.** It is chroot-side if `first-boot`
is absent from its transitive `requires` closure, and booted-system if present.
No schema field records this, on purpose: `_schema.md` names schema churn as the
failure mode for this artifact shape, and a field that can be computed from
edges already present is exactly the kind that gets added and then goes stale.
The observable tell is corroborating — chroot-side commands run as root and
write directly, booted-system ones use `sudo`.

**The derived rule computes privilege; the namespace question has its own
answer, in [[base#What the boundary actually is|spec/base]].** There are four
namespaces, not two — live installer, `arch-chroot` from the installer, booted
system, and rescue chroot entered from the booted system — and `first-boot`'s
postcondition separates them with two readings that are each insufficient
alone. `systemd-detect-virt -r` answers "is this a chroot" and cannot tell the
installer from the booted system; `rd.luks.name=` in `/proc/cmdline` answers
"did this kernel boot our cmdline" and cannot tell a rescue chroot from the
system it was entered from. Each covers exactly the other's blind spot. That
pair is what the `swapon --show` bug needed and prose did not have.

## Unresolved `requires`

Every id referenced by a `requires:` list but not yet defined by a step record.
This is the real progress measure; the count alone is not, since a chapter both
resolves and introduces references.

| id | Referenced by | Belongs to |
|----|---------------|------------|
| *none* | — | `devbox-image-build` was the last, defined 2026-09-02 in [[devenv\|spec/devenv]] from the mirrored recipe; its postcondition is unexecuted |

**One entry, 2026-08-26, and it is the entry this table spent a round complaining it
could not see.** On 2026-08-24 the table read empty while the spec was plainly
unfinished, and the reason given was that it "measures resolution, not coverage" —
with `spec/devenv` as the worked case: *named by two ADRs and by no `requires:` line, so
the table cannot see it.* Three `requires:` lines now name it. The measure was not wrong
about `spec/devenv`; it was blind to it, and it is not blind any more.

**`devbox-image-build` is undefined on purpose rather than by omission.** The recipe
repo — `~/dev/devbox`, holding the `Containerfile` and the `devbox` wrapper — is not in
this vault and has never been read from it. Writing the step from
`_source/Block-C-Dev-Environment.md`'s 2026-08-18 transcript would produce a build
specification that is `inherited-unverified` while looking authoritative, which is the
failure the root `CLAUDE.md` exists to prevent. Mirroring the repo is the unblocking
task; see [[../backlog/devbox-recipe-repo-unmirrored|the backlog item]].

`podman-install` was the previous last entry and
[[containers#podman-install|`spec/containers`]] defines it.

`home-subvol-mount` was folded into `btrfs-subvol-mount` on 2026-08-23. It had
been written in isolation for the container example, and there is no point in
the build where `@home` is mounted and the others are not.

**Eight to four, and this time the arithmetic was dull.** `spec/disk` resolved
its four and introduced none: every id it referenced it also defined, apart from
`first-boot`, which was already on the list. That is what the previous round
learned the hard way — a chapter that satisfies the dependencies it introduces is
the normal case, so *new ids* and *new unresolved references* are different
quantities.

**Four to one, through two that were never on the table.** `spec/base` resolved
three — `base-install`, `user-create`, `first-boot` — and introduced
`bootloader-install`, the first genuinely new unresolved reference since the
table was written. It was not a chapter that does not exist: `bootctl install`
and `loader.conf` are PRD §3.4, `spec/boot` had promoted only the `mkinitcpio
-P` half of that section, and `uki-sign` was signing two files that existed
solely because of a step nobody wrote. `spec/boot` was extended the same day
with `bootloader-install` and `bootloader-nvram-entry`, so both are now defined
and only `podman-install` remains.

**So the count measures what someone thought to reference, and a missing step
that nothing names is invisible to it.** That is a sharper limit than the
new-ids-versus-new-references confusion of the previous round: that one made the
number wrong, this one makes it silent. The table has never been an inventory of
what the spec needs — only of what the spec has asked for, and for a whole round
it read four while the true figure was five. The way to find the rest is to read
a PRD section against the chapter that claims to replace it, which is how this
one surfaced and how ADR-0014 surfaced the round before.

**One to zero, and zero is the number that proves the limit.** `spec/containers`
resolved `podman-install` and introduced nothing, so the table is empty — and the
spec is plainly not finished. Three things the round found are invisible to it:

- **`spec/devenv` does not exist and two decision records name it.** ADR-0007's
  frontmatter says `constrains: spec/devenv` outright, and ADR-0006 has two
  rules of which `spec/containers` implements one — container storage is a
  nested subvolume; project data is bind-mounted rather than held in named
  volumes. The second has no step anywhere. No `requires:` line mentions it, so
  the table cannot.
- **PRD §3.6–§3.8, §5, §6 and §7 have no chapters**, and §3's vconsole ordering
  requirement belongs to one of them and currently exists nowhere. §6.1 is worse
  than absent: it prescribes a sysctl that would break the chapter written this
  round, and only `spec/containers`' prose says so.
- **PRD §12 declaring something out of scope is not the same as it being out of
  scope.** Containers were "deliberately out of scope" in the document and are
  installed and running on the machine. The chapter written this round replaces
  no PRD section at all — the first one that does not — so a reader auditing
  coverage by walking PRD sections would never have reached it.

The honest reading is that **the table measures resolution, not coverage**, and
those are different quantities that happened to move together for four rounds.
Coverage is measured by reading a PRD section against the chapter claiming to
replace it — and now also by reading a decision record's `constrains:` field
against the chapters that exist.

**Run that second measure and `spec/desktop` comes back, larger — 2026-08-28.**
Five records now carry `constrains: spec/desktop` and no such chapter exists:
ADR-0004 and ADR-0011 (both superseded), and ADR-0018, ADR-0019 and ADR-0020,
written this round for the KDE switch. That is the same shape as `spec/devenv` on
2026-08-24 — named by decision records, referenced by no `requires:` line, and so
**invisible to the table above**, which has read empty or near-empty throughout.

Not filed as work, because a desktop chapter is a larger question than a missing
step: the three live records constrain a userland that was installed by one
metapackage, and it is not yet clear that its configuration is reproducible enough
to be written as postconditions. Recorded here so the next coverage pass does not
have to rediscover it.

**The same measure found `spec/backups`, and this time the chapter was written —
2026-08-30.** Three records carried `constrains: spec/backups` (ADR-0008, ADR-0012
and now ADR-0028) against no chapter, exactly as `spec/devenv` and `spec/desktop`
did. It is the first chapter in this spec that exists **because the coverage
measure named it**, rather than because a PRD section was walked. That is the
measure working as intended, on its third instance and its first success.

One asymmetry is worth recording: unlike every other chapter, `spec/backups`
describes something that was **built while the chapter was being written**, so its
postconditions were drafted against commands whose output was in front of the
author. Two of them — `restic-b2-timer` and `restic-restore-test` — assert things
that have **never been observed true**, and both are red by design until
[[../backlog/restic-restore-untested|restic-restore-untested]] closes. A chapter
with a deliberately failing postcondition is new here; the precedent is
`devbox-identity-markers`, red by design until
[[../backlog/devbox-shell-configuration|devbox-shell-configuration]] lands.

## Conventions that are not in the schema

- **A step record must be preceded by prose**, and the unit of transclusion is
  the section, never the bare block. Each step's heading *is* its `id`, so
  `![[boot#tpm2-enroll]]` transcludes the record together with the reasoning
  that justifies it.

  *Known limit:* transclusion is all-or-nothing. A step record is one fenced
  block, so embedding a section brings its `command:` along with its
  `postcondition:`. Where a consumer wants only the assertion — a runbook, where
  the wrong command is destructive and the reader is under pressure —
  **duplicate it and say so on both sides.** Do not split the record into
  separate fences to make half of it targetable; that is schema churn driven by
  a consumer rather than by the spec. `runbooks/tpm2-reenroll` is the worked
  case and records the trade.
- **The spec records intent; `state/` records observation.** Where a chapter
  claims something is currently true of the machine, it cites the `state/`
  section that shows it.
