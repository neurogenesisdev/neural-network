---
title: "Build-spec schema settled; TPM2 unlock mechanism traced"
type: session
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-23
commit: pending
created: 2026-08-23
updated: 2026-08-23
tags: [session, spec, tpm2, secure-boot, collector]
---

# 2026-08-23 — Build-spec schema settled; TPM2 unlock mechanism traced

Refactoring the PRD into `spec/` started with two questions and turned into a
four-hour hunt for where TPM2 unlock gets its instruction. The artifacts are
[[../spec/_schema|spec/_schema.md]], [[../spec/_worked-examples|spec/_worked-examples.md]],
[[../decisions/20260823-tpm2-pcr-selection|ADR-0013]], and
[[../runbooks/tpm2-reenroll|runbooks/tpm2-reenroll]]. This is the part that
didn't fit in any of them.

## Order of discovery

The sequence matters, because two conclusions were reached, retracted, and then
re-reached for different reasons.

**1. Three findings from `state/` before any writing.** The cmdline step has two
per-build literals, not one — `resume_offset=533760` is as rebuild-specific as
the LUKS UUID, and it looks like a constant precisely because it hasn't changed
yet. Hazard 4 turned out live: no `rootflags=subvol=` in the cmdline, so the
btrfs default subvolume is the sole selector for `/`, baked into a signed UKI.
And the collector couldn't see it — `get-default` was never captured.

**2. A conclusion reached too fast.** From "a `systemd-tpm2` token exists" and
"the cmdline has no `tpm2-device=auto`" I concluded §12.2's cmdline edit was
unnecessary and the DAG cycle dissolved. That conflated *enrolled* with *used at
boot*. The header proves a token exists; it says nothing about whether anything
attempts it. Retracted, and the question reopened as genuinely undetermined.

**3. The gap that made it undeterminable.** `systemd-cryptsetup-generator(8)`:
an unprefixed `rd.luks.options=` applies "to any UUIDs not specified elsewhere,
and without an entry in /etc/crypttab" — so the answer depended on a file the
collector never captured. Three collector additions followed.

**4. A sharper gap found on the way.** `systemd-cryptenroll(1)`: binding to no
PCRs at all "is also the default, if this option is not used." So a token's
*presence* proves nothing about its binding, and an unbound enrollment is
invisible in inventory output. This is why the enrollment postcondition reads
the PCR mask rather than checking that a token exists.

**5. The crypttab hunt, ending in a negative finding across three locations.**
`/etc/crypttab`: empty. `/etc/crypttab.initramfs`: empty. The initramfs itself,
extracted from the signed UKI: no crypttab anywhere in the image. Combined with
a signed `.cmdline` carrying no `tpm2-device=`, that eliminated the only
remaining candidate. libcryptsetup's external token plugin is doing it unaided.

A negative finding took more work than a positive one would have and is worth
the same amount of writing down — the next person to ask "where is this
configured?" should not repeat the search.

**6. The mechanism is still not explained.** `cryptsetup-open(8)` documents
automatic token use for tokens *"unprotected by PIN"*, and this token is
PIN-protected. The man pages we have do not forbid the observed behaviour; they
just fail to account for it. The spec asserts that unlock succeeds — `measured`
— and does not assert why.

## Dead ends and near-misses

**`objcopy` would have rewritten the signed UKI in place.** Offering it as an
alternative to `ukify`, the synopsis check turned up: "If you do not specify
_outfile_, **objcopy** creates a temporary file and destructively renames the
result with the name of _infile_." A missing trailing argument on
`/boot/EFI/Linux/arch-linux.efi` would have produced a modified binary whose
sbctl signature no longer matched — discovered at the next boot, with Secure
Boot enabled. Caught by reading the man page before recommending the command,
not by running it. The command shape that reached the notes copies to `/tmp`
first, so no invocation can reach the original.

**`ukify inspect --section=… FILE` was wrong and I hadn't checked the order.**
The `--section=NAME:text|binary[@PATH]` syntax was correct; its position was
not. The synopsis is `ukify [OPTIONS...] inspect FILE...` — options precede the
verb. Failed with "unrecognized arguments". Reading a flag's description is not
the same as reading the synopsis.

**A `\<` awk escape that silently matched nothing.** The first RFC1918 stripping
filter used `\<` for a word boundary — a gawk extension. The target runs gawk,
so it would have worked there. This environment runs mawk, where it matched
nothing at all: the test passed addresses through unchanged and reported
success. The bug was not the escape; the bug was that the verification was
testing nothing while appearing to test something. Anchoring each pattern to the
start of the field with `^` needs no boundary escape, works in both, and rules
out the `110.x` prefix collision structurally rather than relying on a boundary
to catch it.

**Adding a collector section before checking what was already there.** The
per-UKI `ukify` loop went in before noticing that `bootctl status` already
prints the default entry's signed `.cmdline`, and that `state/boot.md` had
carried it all along at line 98 — identical to line 102. The loop still earns
its place, but only for the three non-default UKIs, and the cost question was
smaller than it looked. Check what the existing output already answers before
adding a source.

## The glyph progression

Four versions of one postcondition, worth preserving because each failure mode
was different:

1. **Count non-`✓` lines.** Broken by the header line `Verifying file database
   and EFI images in /boot...`, which carries no marker — permanently red on a
   healthy machine. A false positive, so it failed safe, but a permanently-red
   check is one nobody reads.
2. **Count `✗` lines.** Introduced a dependency on a glyph never observed —
   only the success case had been seen.
3. **Match the phrases, not the glyphs.** Upstream renders the failure marker
   both `✘` (U+2718) and `✗` (U+2717), the line format changed historically from
   `-> WARNING: X is not signed`, and there is a third state, `‼ … does not
   exist`. Phrases survive all three; `is not signed` does not contain
   `is signed` as a substring, so they discriminate cleanly.
4. **`--json`.** Retired the whole problem. No header to exclude, no glyph to
   guess. Two hours of increasingly careful text parsing replaced by a flag
   documented in `sbctl.8` the entire time.

The lesson is ordering, not cleverness: ask whether the tool has machine-
readable output *before* investing in parsing its human-readable output.

## Conclusions revised

| Claimed | Corrected to | Why |
|---|---|---|
| §12.2's cmdline edit is unnecessary; the cycle dissolves | Undetermined | Conflated *enrolled* with *used at boot* |
| (then) Undetermined | §12.2 touches nothing | Measured: no crypttab in three locations |
| An unbound enrollment hands over the key | It costs the Secure-Boot-state binding; the PIN still gates it | `--tpm2-with-pin=yes` was already in force |
| A command-less step is one a human must satisfy | One the executor cannot make true, only check | The standing-invariant case fits neither "firmware menu" nor "lookup gap" |
| PCR 2 is near-constant — no discrete GPU | Measured non-empty, `ec3a64e9…`, unexplained | False premise: integrated Radeon, and something extends it |
| PCR 0 buys evil-maid-with-a-flash-programmer | It buys defence against a root shell | SPI write protection `Disabled`, `action-contact-oem` |
| `spec/` does not exist yet | `spec/` exists; `spec/boot` does not | — |

The PCR 2 and evil-maid entries are the two that mattered. Both were reasoning
from a premise that felt obvious enough not to check, and both pointed the
recommendation the same wrong way.

## Two bugs the worked examples surfaced

Writing step records forced a question prose had let slide: *when does this
run?*

`cmdline-write`'s precondition consulted `swapon --show`, which reports the path
recorded at `swapon` time — `/mnt/swap/swapfile` — so it would have failed
inside `arch-chroot` for a reason unrelated to the hazard. `map-swapfile`
verifies suitability itself, so its success is the better gate.

The btrfs default-subvolume step conflated a build action with a standing
invariant, which forced a fragile derivation. Splitting them removed it: at
build time the top level is still mounted and `@` is directly nameable.

Prose can carry `swapon --show` indefinitely without anyone noticing which
namespace it runs in. A precondition cannot.

## Still open

- The token-plugin mechanism is `inferred`. Behaviour is measured; the man pages
  do not account for a PIN-protected token being tried automatically.
- PCR 2's contents are unexplained and its stability across peripheral
  attachment is untested — the revisit condition on ADR-0013.
- `spec/boot` and the remaining chapters. The worked examples are a review
  artifact and dissolve into them; `runbooks/tpm2-reenroll` duplicates a
  verification block until they land.
- The enrollment is decided but not applied — the machine is still on `[7]`.
