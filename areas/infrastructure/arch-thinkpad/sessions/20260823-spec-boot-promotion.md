---
title: "Churn fixed by opposite means; spec/boot promoted; lookup rule reordered"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-23
commit: pending
created: 2026-08-23
updated: 2026-08-23
tags: [session, spec, collector, boot, documentation, devbox]
---

# 2026-08-23 — Churn fixed by opposite means; spec/boot promoted

Second round of the same day. The first
([[20260823-build-spec-schema|session log]]) settled the schema and traced the
TPM2 unlock; this one fixed two post-reboot churn sources, promoted three of the
five worked examples into [[../spec/boot|spec/boot]], and reordered the
`CLAUDE.md` lookup rule after the documentation channel it named turned out to
be unreachable.

Artifacts: [[../spec/boot|spec/boot.md]], [[../spec/_index|spec/_index.md]], a
reduced [[../spec/_worked-examples|_worked-examples.md]], an amended
[[../decisions/20260823-tpm2-pcr-selection|ADR-0013]],
[[../backlog/devbox-wiki-mount|backlog/devbox-wiki-mount]] and
[[../backlog/devbox-contents-audit|backlog/devbox-contents-audit]]. This is what
did not fit in any of them.

## Two churn sources, opposite treatments

Both were reported the same way — a diff after a reboot that meant nothing — and
the instinct was to reach for the same fix twice. The man pages are what
separated them.

**`lsblk` reorders `MOUNTPOINTS` and has no deterministic alternative.**
`lsblk(8)`: the column "displays by multi-line cell all mount points associated
with the device," with no documented order. `--sort` does not help, because it
orders *lines* and this is one cell. The singular `MOUNTPOINT` looked like the
escape and is not — it "displays only one mount point (usually the last mounted
instance of the filesystem)," which moves for the same reason. So there was
nothing to stabilise: the column had to go, and the information had to be
re-sourced from a tool that *can* be made deterministic. `findmnt --list`
suppresses the tree formatting that `findmnt(8)` otherwise puts in `TARGET`, and
extending it to `vfat` keeps `/boot` present as an *observation* rather than
leaving it only in `## fstab`, which is intent.

**`list-dependencies` churn is the command's documented contract.**
`systemctl(1)`: "this command only lists units currently loaded into memory by
the service manager. In particular, this command is not suitable to get a
comprehensive list at all reverse dependencies on a specific unit." So
`gvfs-udisks2-volume-monitor.service` appearing only while running is not an
anomaly sitting inside otherwise-static output — the output was never static.
Filtering it by name would have suppressed one instance of a general property
and gone stale on the next D-Bus-activated unit to join the session. The honest
fix was to relabel the heading as a collect-time snapshot of loaded units and
quote the limitation in the collector comment, converting an unexplained diff
into an expected one.

**The generalisation worth keeping:** "fix the churn" resolved to *remove* in one
case and *explain* in the other, and the deciding question was the same both
times — does the tool document a stable mode? Only the man page answers that.
Reaching for a filter first would have produced a plausible patch in both cases,
one of which would have been wrong.

A third change went in alongside: `## mounts` had tree glyphs assigned in
mount-table order and was then piped through `sort`, so it looked stable while
its prefixes could still move. Same defect as the `lsblk` one, one level down,
and not yet observed — fixed because the section was about to become the sole
carrier of what `MOUNTPOINTS` used to show.

## The transclusion that was structurally right and operationally wrong

`runbooks/tpm2-reenroll` had carried a duplicate of the `tpm2-enroll`
postcondition with a note saying it would become a transclusion once `spec/boot`
existed. `spec/boot` now exists, so it was written as
`![[../spec/boot#tpm2-enroll]]`, satisfying the schema's rule that the unit of
transclusion is the section.

That was wrong, and the reason is worth preserving because the mechanism was
never in doubt. **A step record is one fenced block.** Embedding the section
therefore carries `command:` along with `postcondition:` — and that command is
the *first* enrollment, with no `--wipe-slot=tpm2`. Run from the runbook it
would add a second TPM2 token to a volume that already has a stale one, after
which the assertion directly above it fails for a reason the reader had just
created.

The first repair was a `[!WARNING]` callout beside the transclusion. That is
worse than the duplicate it replaced: this page is read at an unexpected
passphrase prompt, which is the worst available condition under which to ask
someone to notice that one of two adjacent commands must not be run. The second
option — targeting only the `postcondition:` — would need the step record split
into separate fences, which is schema churn driven by a consumer rather than by
the spec, and `_schema.md` names schema churn as the failure mode for this
artifact shape.

So the duplication stayed, with a note on both sides that they must move
together. The drift risk is one `jq` expression and cheap to compare; the risk
being traded against is a destroyed working enrollment. **Single-sourcing is a
means, not the goal** — and the way to see that is to ask what each option costs
*when it fails*, rather than which is tidier.

The general limit is now recorded in [[../spec/_index|spec/_index.md]]:
transclusion is all-or-nothing, and a consumer that wants only the assertion
should duplicate and say so.

## The lookup rule had the ordering backwards

The rule named `man.archlinux.org` then the ArchWiki for Arch-packaged tools.
Both halves failed in one session.

**The live ArchWiki is unreachable to any non-browser client.**
`wiki.archlinux.org` sits behind Anubis, a JavaScript proof-of-work
interstitial. It returns HTTP 200 with a body reading `Access Denied: error code
<hex>`, so it presents as a block or a rate limit — the natural responses being
to wait, retry, or try a different article, none of which work. `?action=raw`
does not bypass it either. A turn was spent on this before the shape of the
failure became clear. Now in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]], because the error
message points somewhere other than the cause.

**More useful: the tool documented itself better than any page, twice.**
`mkinitcpio.conf(5)` describes the `HOOKS` array but not a single hook, and
explicitly defers to `mkinitcpio -H <hook>` — which was never run until the
gap had already been written up as unclosable. And from the earlier round,
`sbctl verify --json` retired two hours of increasingly careful glyph-parsing
and had been in `sbctl.8` the whole time.

Two instances is enough to reorder rather than annotate. The rule now reads:
the tool itself, then `man.archlinux.org`, then the *local* ArchWiki
(`arch-wiki-lite`, installed at `20260702` and so weeks behind), then Context7
for upstream projects. Reaching for prose documentation before asking the binary
is the expensive mistake, not the safe one.

## A hazard written off as unassertable, then closed

Without per-hook documentation, `mkinitcpio-config` was written to assert
membership only — `systemd` present, `udev` absent, `sd-encrypt` present,
`encrypt` absent — with a prose paragraph conceding that ordering claims were
not sourced and naming `mkinitcpio -H` as the lookup that would close it. That
concession was correct procedure and still left the step weaker than it needed
to be.

The `-H` output closed it and added a hazard that had not been considered at
all. `autodetect`: "This hook must be run before other subsystem hooks in order
to take advantage of auto-detection." And `microcode`: "If the autodetect hook
runs before this hook, it will only add early microcode update files for the
processor of the system the image is built on."

In the measured array `autodetect` does precede `microcode`, so the UKIs carry
AMD microcode only. **Rebuilding this spec on a different CPU produces a
wrong-microcode image with no error at any stage and no diff in `state/` to show
it.** The ordering is now asserted — not because the assertion prevents the
hazard, but so that reversing it is a decision someone makes on purpose rather
than a position that drifted.

`sd-encrypt`'s help also narrowed a gap from the earlier round. The finding had
been recorded as "the man pages do not account for the observed behaviour,"
which was too strong: the hook "allows systemd to unlock encrypted LUKS1 volumes
with a password, and encrypted LUKS2 volumes with a password, FIDO2 token or
TPM2 key." The TPM2 path *is* documented. What remains unexplained is narrower —
the PIN interaction specifically, since `cryptsetup-open(8)` documents automatic
token use only for tokens "unprotected by PIN." And it confirmed the crypttab
negative from the other side: "if the file */etc/crypttab.initramfs* exists, it
will be added to the initramfs as */etc/crypttab*" names the one documented path
by which a crypttab could reach the initrd, and that file does not exist here.
The search was not merely exhaustive; the documented mechanism is confirmed
absent.

The postcondition that resulted was tested against the measured array and seven
mutations — PASS on the first, a specific diagnostic on each of the rest —
rather than being reasoned about and committed.

## A prediction that came out wrong

Before writing `spec/boot`, the dangling `requires:` set was enumerated at ten,
and the round was predicted to end at ten again: two resolved
(`uki-generate`, `secureboot-enable`), two introduced (`mkinitcpio-config`,
`esp-mount`). The actual figure is **eight**.

Two errors, in opposite directions and unequal in interest. The dull one:
folding `home-subvol-mount` into `btrfs-subvol-mount` removed an id the estimate
had not accounted for. The useful one: **`mkinitcpio-config` was counted as
introduced when it is defined in the same chapter that introduced it.** It was
never dangling for a moment. The estimate had counted *new ids* rather than *new
unresolved references*, which are different quantities whenever a chapter is
internally coherent — and a chapter that introduces a dependency it also
satisfies is the normal case, not the exception.

The count is a bad progress measure for exactly this reason, which is why
`spec/_index.md` carries a table naming each unresolved id and the chapter that
will define it. That table was checked against a recount rather than
transcribed, and matches.

## The container is Debian; the target is Arch

Raised as a candidate for the devbox audit: install `man-db` and `man-pages`, on
the reasoning that a local man page reflects the *installed* version where
`man.archlinux.org` is upstream-current.

The premise does not hold here. **The devbox is Debian 13 trixie** (measured,
`/etc/os-release`). `man cryptsetup` inside it would report Debian's cryptsetup —
neither the version on the ThinkPad nor reliably current, and strictly worse than
the fetch it would replace. The two Arch-specific tools that prompted the whole
question, `mkinitcpio` and `sbctl`, are not in Debian's archive at all.

Two further obstacles, both measured and both invisible until looked for: there
is no `man` binary, and `/etc/dpkg/dpkg.cfg.d/docker` carries `path-exclude
/usr/share/man/*`, so `/usr/share/man/man{1,5,7,8}` exist and are **empty**.
Installing `man-db` would populate nothing without overriding the exclusion and
reinstalling every package already present.

The version-accurate fix is a read-only mount of the host's `/usr/share/man`,
the same shape as the wiki mount — which would make the lookup rule's second
channel local *and* more accurate than the web fetch, rather than less. Recorded
in [[../backlog/devbox-contents-audit|the audit item]] rather than acted on,
because it wants the same rebuild as the wiki mount and the two should be
decided together.

The same measurement pass closed one candidate outright — `jq` is present, so
the worry that spec postconditions depend on a missing tool was unfounded — and
surfaced a sharper one: `awk` here is mawk 1.3.4 where the target is gawk. That
divergence has already produced the worst available failure shape, a `\<`
word-boundary escape that matched nothing and let a verification pass while
testing nothing at all. Installing gawk would stop the bug class and also remove
the accidental cross-implementation check that caught it.

## A standing limitation, said out loud

`btrfs`, `cryptsetup`, `sbctl`, `systemctl`, `podman`, `mkinitcpio`, `ukify` and
`bootctl` are **all absent from the devbox container** (measured 2026-08-23).

The consequence, which had been operating unstated for two rounds of spec work:
**every postcondition in this vault is written and reviewed in a container that
cannot execute any of them, and is only ever run on the machine itself.** No
step record in `spec/` has been executed by whoever wrote it.

This is probably the right arrangement — those tools need a real LUKS volume and
a real ESP, and installing Arch binaries into a Debian container to run them
against nothing would be worse than useless. It is recorded here as an accepted
limitation with a named consequence rather than as a gap to close. What follows
from it is a working rule: **a postcondition's correctness rests on the man page
and on measured output pasted back from the host, never on having watched it
run.** That is why the `mkinitcpio-config` check was tested against synthesised
config files, and why `sbctl verify --json`'s output shape had to be measured on
the machine before the assertion could be written against it.

## Conclusions revised

| Claimed | Corrected to | Why |
|---|---|---|
| Drop `MOUNTPOINTS`, `## mounts` already covers it | Covers only btrfs; `/boot` would survive as intent alone | `## mounts` was `findmnt -nt btrfs`; extended to `vfat` |
| The runbook block becomes a transclusion once `spec/boot` exists | Duplication is correct here | One fence carries `command:` too, and the reader is under pressure |
| A warning callout makes the transclusion safe | It does not | Wrong command adjacent to right command, read at a failed boot |
| Hook ordering cannot be sourced, mark `inferred` | Fully sourced | `mkinitcpio -H`, which `mkinitcpio.conf(5)` had pointed to all along |
| The man pages do not account for TPM2 unlock | They do; only the PIN interaction is unaccounted for | `mkinitcpio -H sd-encrypt` |
| Dangling references stay at 10 | 8 | Counted new ids, not new *unresolved* ones |
| `man-db` would give installed-version pages | Debian's versions, in a Debian container | `/etc/os-release` |
| `jq` may be missing from the container | Present | `apt-mark showmanual` |

The transclusion and the dangling-count entries are the two that matter. Both
came from applying a rule that was correct in the abstract — single-source the
assertion, count what a chapter introduces — without checking what it produced
in the specific case.

## Still open

- `spec/disk`, `spec/base`, `spec/containers`. Eight unresolved ids, each
  assigned to a chapter in [[../spec/_index|spec/_index.md]].
- The mawk/gawk divergence needs a decision, not a default. Installing gawk
  removes a bug class and an accidental check at the same time.
- The devbox man page and wiki mounts want one rebuild and one decision between
  them.
- PCR 2 remains measured non-empty and unexplained. All three sealed-against
  PCRs are now byte-identical across a reboot, which narrows the gap to
  peripheral attachment without closing it.
