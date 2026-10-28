---
title: "Boot chain — cmdline, UKI, Secure Boot, TPM2"
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-23
updated: 2026-08-24
tags: [spec, boot, secure-boot, tpm2, uki, mkinitcpio]
---

# Boot chain

Ten steps, from the kernel command line to the TPM2 enrollment that unlocks the
root volume. Read [[_schema|_schema.md]] first; the executor contract it defines
is what makes several of these steps safe to fail.

Replaces PRD §3.2, §3.4 and §12.2, and rejects part of §3.2 and §12.2 — see
`cmdline-write` and `tpm2-enroll`.

## Where this chapter sits

```
cmdline-write ──┐
                ├─→ uki-generate ──→ bootloader-install ──┐
mkinitcpio-config ┘                                       │
                                                          ▼
                                                     first-boot
                                                     (spec/base)
                                                     │        │
                              firmware-setup-mode ←──┘        └──→ bootloader-nvram-entry
                                      │                                        │
                              sbctl-enroll-keys ──→ uki-sign ←─────────────────┘
                                                        │
                                     tpm2-enroll ←── secureboot-enable
```

**It crosses the execution-context boundary twice, and `bootctl install` is the
step that does it both times.** `cmdline-write`, `mkinitcpio-config`,
`uki-generate` and `bootloader-install` run under `arch-chroot`. `first-boot` —
defined in [[base|spec/base]], not here — is the boundary, and everything from
`firmware-setup-mode` onward runs on the booted system, including
`bootloader-nvram-entry`, which is the second half of the same PRD command. The
chapter cannot be executed top to bottom, and the `sudo` in the later commands
is the visible tell. [[areas/infrastructure/arch-thinkpad/spec/_index#The DAG is not a reading order|spec/_index]] has
the general rule and [[base#What the boundary actually is|spec/base]] has the
observable one.

Two of the later steps have no `command`, for two different reasons: one is a
firmware menu, the other is a firmware menu *whose ordering matters*. Neither is
a lookup gap.

**Five postconditions here were changed on 2026-08-24 by a sweep for
[[_schema#A postcondition must reach its verdict line on every input it is meant to judge|`_schema.md`'s seventh convention]],
and each carries its mutation set below.** The convention came out of
`spec/disk`, and the sweep was the check that it was not a `spec/disk` problem.
It was not: `cmdline-write` had the worst instance in the vault — a written
diagnostic that no input could reach — and `bootloader-nvram-entry` had a
separate defect of the older don't-fail-the-same-way kind. `mkinitcpio-config`,
`bootloader-install`, `firmware-setup-mode`, `sbctl-enroll-keys` and
`secureboot-enable` were checked and are clean.

---

## cmdline-write

The old PRD §3.2 writes `/etc/kernel/cmdline` with two values baked in: the LUKS
container UUID and `resume_offset=533760`. Both are wrong by construction. A
rebuild formats a new LUKS container with a new UUID, and a new swapfile lands
at a new physical offset. The measured cmdline still carries 533760
(`state/boot.md`, `## cmdline (configured)`), which is exactly the trap — it
looks like a constant because it has not changed yet.

`btrfs-inspect-internal(8)` is worth quoting, because the obvious alternative is
wrong: "Do not use **filefrag** or *FIEMAP* ioctl values reported as physical,
this is different due to internal filesystem mappings. The hibernation expects
offset relative to the physical block device." The `-r` flag "print only the
value suitable as resume offset" (*measured*, read 2026-08-23).

**The negative postcondition encodes a rejection.** PRD §12.2 tells you to add
`rd.luks.options=<UUID>=tpm2-device=auto`. Per
`systemd-cryptsetup-generator(8)`, a UUID-prefixed list "will override the
options for the given UUID," while an unprefixed list applies "to any UUIDs not
specified elsewhere." Adding the prefixed form would therefore stop the existing
unprefixed `discard` from applying to the root volume — silently losing discard
on a QLC SSD, inside a signed UKI, to gain an option that measurement shows is
unnecessary (see `tpm2-enroll`). The assertion makes that regression fail the
build rather than sit there being ignorable, which is how `_schema.md` says a
rejected configuration is recorded when `spec/` has no supersession pointers.

**No `rootflags=subvol=` is written, and that is load-bearing elsewhere.** It
makes the btrfs default subvolume the sole selector for `/`, which is what
`btrfs-default-matches-root` in `spec/disk` exists to assert. PRD §3.2 specifies
`rootflags=subvol=@`; carrying that forward would take the default subvolume out
of the boot path and make `snapper rollback` a no-op that reports success.

The `device:` parse is `measured`: `cryptsetup status cryptroot` prints
`device:  /dev/nvme0n1p2`, field 2 (run 2026-08-23). `cryptsetup-status(8)`
documents the verb but not its output labels, so this is grounded in observation
rather than documentation — if the label changes, the variable empties and the
postcondition reports FAIL rather than writing a wrong UUID. It appears in three
steps in this chapter; if it is wrong it is wrong in all three.

The precondition deliberately does not consult `swapon`. This step runs inside
`arch-chroot`, where the kernel still reports the path recorded at `swapon` time
(`/mnt/swap/swapfile`), so a path match would fail for a reason having nothing
to do with the hazard. `map-swapfile` is the better gate anyway: per
`btrfs-inspect-internal(8)` it will "also verify that the *file* is suitable as
a swapfile," so its success *is* the precondition.

### A check that no input could reach

The postcondition read four values and compared three of them, and **all four
readings could end the script before any comparison happened**. `blkid` is the
sharpest: it exits non-zero when it resolves nothing, so `live_uuid=$(blkid …)`
aborted — and the very next line was `[ -n "$live_uuid" ] || fail=1`, a
diagnostic written for exactly that case and **unreachable from the day it was
written**. Identical shape to
[[disk#resume-offset-matches-swapfile|`resume-offset-matches-swapfile`]]'s, found
by the same sweep, and the reason this is now
[[_schema#A postcondition must reach its verdict line on every input it is meant to judge|the seventh convention]]
rather than a note in one chapter.

Each reading now gets its own diagnostic instead of a shared `fail=1`. That is
not tidying: with `live_uuid` empty, `grep -q "rd.luks.name==cryptroot"` reports
as *the cmdline is wrong*, sending the reader after a signed UKI when the truth
is that there is no LUKS mapping to compare against.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — UUID and offset match the cmdline | PASS | PASS |
| 2 | `/etc/kernel/cmdline` unreadable | `RESULT=FAIL (/etc/kernel/cmdline is unreadable)` | **no `RESULT=`** |
| 3 | no `cryptroot` mapping | FAIL, names the missing device *and* the missing UUID | **no `RESULT=`** |
| 4 | **`blkid` resolves nothing** | FAIL, `no UUID for … — nothing to compare` | **no `RESULT=`, exit 2** |
| 5 | `map-swapfile` fails | FAIL, `map-swapfile returned no offset` | **no `RESULT=`** |
| 6 | live UUID differs from the cmdline's | FAIL, quotes the live UUID | same |
| 7 | live offset differs | FAIL, quotes the live offset | same |
| 8 | PRD §12.2's UUID-prefixed `rd.luks.options=` present | FAIL, names §12.2 | FAIL, no diagnostic |
| 9 | PRD §3.2's `rootflags=subvol=` present | FAIL, names §3.2 | FAIL, no diagnostic |

Rows 8 and 9 are the two negative postconditions, and the unfixed column is why
they gained messages: both set `fail=1` silently, so the step reported `FAIL`
with no indication which rejected configuration had come back.

```yaml
id: cmdline-write
requires: [swapfile-create, btrfs-subvol-mount, base-install]
precondition: |
  set -euo pipefail
  # map-swapfile verifies suitability as well as reporting the offset, so its
  # success is the gate. Deliberately not swapon: this runs inside the chroot.
  btrfs inspect-internal map-swapfile -r /swap/swapfile >/dev/null
command: |
  set -euo pipefail
  luksdev=$(cryptsetup status cryptroot | awk '/device:/ {print $2}')
  luksuuid=$(blkid -s UUID -o value "$luksdev")
  offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)
  test -n "$luksuuid" && test -n "$offset"
  printf 'rd.luks.name=%s=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=%s lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none\n' \
    "$luksuuid" "$offset" > /etc/kernel/cmdline
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Every reading below is allowed to come back empty, per _schema.md's seventh
  # convention. blkid exits non-zero when it resolves nothing, and that
  # assignment aborting is what made the `-n "$live_uuid"` check below
  # unreachable from the day it was written.
  cmdline=$(cat /etc/kernel/cmdline) \
    || { echo "RESULT=FAIL (/etc/kernel/cmdline is unreadable)"; exit 1; }
  luksdev=$(cryptsetup status cryptroot | awk '/device:/ {print $2}') || luksdev=
  live_uuid=$(blkid -s UUID -o value "$luksdev") || live_uuid=
  live_offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile) || live_offset=
  # Each empty reading gets its own diagnostic, or a comparison against an
  # empty string reports as a mismatch and sends the reader after the cmdline.
  [ -n "$luksdev" ]    || note "cryptsetup status named no backing device"
  [ -n "$live_uuid" ]  || note "no UUID for '$luksdev' — nothing to compare"
  [ -n "$live_offset" ] || note "map-swapfile returned no offset"
  if [ -n "$live_uuid" ]; then
    grep -q "rd.luks.name=$live_uuid=cryptroot" <<<"$cmdline" \
      || note "cmdline does not carry rd.luks.name=$live_uuid=cryptroot"
  fi
  if [ -n "$live_offset" ]; then
    grep -qE "resume_offset=$live_offset( |$)" <<<"$cmdline" \
      || note "cmdline does not carry resume_offset=$live_offset"
  fi
  # Negative: a UUID-prefixed rd.luks.options overrides rather than merges.
  if grep -qE 'rd\.luks\.options=[0-9a-f-]{36}=' <<<"$cmdline"; then
    note "cmdline carries a UUID-prefixed rd.luks.options= (PRD §12.2) — it overrides the unprefixed discard"
  fi
  # Negative: rootflags=subvol= would take the default subvolume out of the
  # boot path and silently defeat snapper rollback. See spec/disk.
  if grep -q 'rootflags=subvol=' <<<"$cmdline"; then
    note "cmdline carries rootflags=subvol= (PRD §3.2) — the default subvolume stops selecting /"
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## mkinitcpio-config

The initramfs must be systemd-based, because the TPM2 unlock depends on it. The
measured configuration (`state/boot.md`, `## mkinitcpio HOOKS/MODULES`) is:

```
MODULES=(btrfs)
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

`mkinitcpio.conf(5)` states that "Order is important, as it defines the order in
which hooks will be run during bootup" but does not document the individual
hooks. Those are documented by the tool itself, via `mkinitcpio -H <hook>`; all
twelve were read on the machine 2026-08-23 and everything below is quoted from
that output (*measured*).

**Membership.** Three of the checks are sourced directly. The `systemd` hook "is
meant to replace the 'base', 'usr', 'udev' and 'resume' hooks" — which is what
makes the negative check on `udev` an assertion rather than a preference; the
same help adds that "Other hooks with runtime components will need to be ported,
and will not work as intended." It also asks that `base` be kept: "You also may
wish to still include the 'base' hook (before this hook) to ensure that a rescue
shell exists on your initramfs" — hence `base` first, and an ordering assertion
rather than mere presence.

`sd-encrypt` is the hook that makes the rest of this chapter work: it "allows
systemd to unlock encrypted LUKS1 volumes with a password, and encrypted LUKS2
volumes with a password, FIDO2 token or TPM2 key." The plain `encrypt` hook
carries no such capability, and swapping it in is a silent failure — the LUKS2
token stays valid, `tpm2-enroll`'s postcondition still passes, and the machine
asks for a passphrase every boot with nothing logged.

**Ordering, and the one hazard that needed an assertion.** `autodetect` "must be
run before other subsystem hooks in order to take advantage of auto-detection.
Any hooks placed before 'autodetect' will be installed in full." That is a
correctness requirement for `kms`, `keyboard`, `block`, `filesystems` and `fsck`,
each of whose help text repeats the advice from its own side, so it is asserted.

The sharper one is `microcode`: "If the autodetect hook runs before this hook, it
will only add early microcode update files for the processor of the system the
image is built on." In the measured array `autodetect` *does* precede
`microcode`, so the generated images carry AMD microcode only.

**That is the deliberate choice, and its cost is a silent failure on foreign
hardware.** Rebuilding this spec on a different CPU — or restoring these images
to one — produces a UKI with the wrong microcode, no error at any stage, and no
diff in `state/` to show it. The ordering is therefore asserted rather than left
incidental, so that reversing it is a decision someone makes on purpose. The
assertion does not prevent the hazard; it makes the position visible and pins it
against silent drift. The narrower reading is what matters: these UKIs are valid
for the machine that built them, which is already true of the LUKS UUID and the
resume offset in `cmdline-write`.

The membership test iterates and compares exactly rather than matching a word
boundary. That is deliberate: a `\<` boundary escape in an earlier collector
filter was a gawk extension that matched *nothing* under mawk, and the test
passed while checking nothing at all. Exact comparison has no such failure mode
and needs no assumption about which awk is present.

```yaml
id: mkinitcpio-config
requires: [base-install]
command: |
  set -euo pipefail
  sed -i -E 's/^MODULES=.*/MODULES=(btrfs)/' /etc/mkinitcpio.conf
  sed -i -E 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)/' \
    /etc/mkinitcpio.conf
postcondition: |
  set -euo pipefail
  hooks=$(sed -nE 's/^HOOKS=\((.*)\)[[:space:]]*$/\1/p' /etc/mkinitcpio.conf)
  mods=$(sed -nE 's/^MODULES=\((.*)\)[[:space:]]*$/\1/p' /etc/mkinitcpio.conf)
  [ -n "$hooks" ] || { echo "RESULT=FAIL (no HOOKS array)"; exit 1; }
  # Position of a hook in the array, or 0 if absent. Exact comparison per
  # element: no word-boundary escape, because \< is a gawk extension that
  # matched nothing under mawk and passed while checking nothing at all.
  idx() { local n=$1 i=0 h; for h in $hooks; do i=$((i+1));
            [ "$h" = "$n" ] && { echo "$i"; return; }; done; echo 0; }
  has() { [ "$(idx "$1")" -ne 0 ]; }
  before() { local a b; a=$(idx "$1"); b=$(idx "$2")
             [ "$a" -ne 0 ] && [ "$b" -ne 0 ] && [ "$a" -lt "$b" ]; }
  fail=0
  note() { echo "  $1"; fail=1; }
  for h in base systemd autodetect sd-encrypt; do
    if ! has "$h"; then note "missing hook: $h"; fi
  done
  case " $mods " in *" btrfs "*) ;; *) note "MODULES lacks btrfs" ;; esac
  # Negative: `systemd` "is meant to replace the 'base', 'usr', 'udev' and
  # 'resume' hooks", and `encrypt` cannot unlock a LUKS2 volume with a TPM2
  # key — it would boot to a passphrase prompt with nothing logged.
  for h in udev resume encrypt; do
    if has "$h"; then note "hook must be absent: $h"; fi
  done
  # base before systemd, so a rescue shell survives in the initramfs.
  if ! before base systemd; then note "base must precede systemd"; fi
  # autodetect before every subsystem hook whose help says it needs it. For
  # microcode this pins the deliberate choice to build machine-specific images.
  for h in microcode kms keyboard block filesystems fsck; do
    if has "$h" && ! before autodetect "$h"; then
      note "autodetect must precede $h"
    fi
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## uki-generate

`mkinitcpio(8)`: `-P`/`--allpresets` "Process all presets contained in
*/etc/mkinitcpio.d*", and a preset's `<preset>_uki` is the "Output file path for
a unified kernel image (passed via **-U** to mkinitcpio)", which "combines the
initramfs, the kernel, and the kernel cmdline into one executable" (*measured*,
read 2026-08-23). The cmdline it combines in is `/etc/kernel/cmdline`, per: "If
none is specified, try one of the files */etc/kernel/cmdline*, …".

That last quote is why this step requires `cmdline-write`. The cmdline is not
referenced at boot; it is *baked in* at generation, and later signed. Editing
`/etc/kernel/cmdline` without regenerating changes nothing about how the machine
boots, and nothing reports that.

**The postcondition targets exactly that divergence.** It resolves the UKI paths
out of the preset files rather than listing them — four exist today
(`state/boot.md`, `## UKIs`), but the set changes when a kernel is added or
removed, so a literal list is the constant this vault forbids. Each resolved UKI
must exist and be newer than both inputs.

*Known limit, stated rather than hidden:* an mtime comparison detects a stale
UKI, not a wrong one. The content-level check — that every UKI's embedded
`.cmdline` matches — is real and is done, but post-boot by the collector, whose
`## cmdline (signed, per UKI)` section prints the `.cmdline` and `.uname` of
each. All four currently hash identically (`a7563415…`, measured 2026-08-23),
which is what closes PRD §3.2's warning that "a hibernation image is silently
discarded if you boot an entry lacking them."

The content check is not done *here* because it needs `ukify`, which ships in
the separate `systemd-ukify` package (`state/packages.md`) rather than with
systemd — so at chroot time its availability is an assumption, and a
postcondition that errors when a package is missing fails the same way as one
that detects a bad UKI. `_schema.md`: a check must not fail the same way as the
thing it detects.

**The glob was the hazard, not the `sed`.** With no preset files present the
shell passes the literal `/etc/mkinitcpio.d/*.preset` to `sed`, which errors,
and `pipefail` ended the postcondition before the `[ -n "$ukis" ]` guard below
it could speak — so "no presets at all" and "presets that declare no UKI" were a
reported failure and a silence respectively, when they are the same finding.
They now share one message.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — presets resolve to UKIs newer than both inputs | PASS | PASS |
| 2 | **no preset files at all** | `RESULT=FAIL (no preset declares a _uki= path)` | **no `RESULT=`** |
| 3 | presets present, none declaring `_uki=` | same message | same message |
| 4 | a preset names a UKI that is absent | FAIL, `missing: <path>` | same |
| 5 | UKI older than `cmdline` and `mkinitcpio.conf` | FAIL, both `stale vs` lines | same |

```yaml
id: uki-generate
requires: [cmdline-write, mkinitcpio-config, esp-mount]
precondition: |
  set -euo pipefail
  # Both inputs must exist before anything is baked in.
  test -s /etc/kernel/cmdline
  test -s /etc/mkinitcpio.conf
command: |
  set -euo pipefail
  mkinitcpio -P
postcondition: |
  set -euo pipefail
  # Resolve UKI paths from the presets: the set changes when a kernel is added
  # or removed, so a literal list would be a constant that decays.
  # The glob is the hazard, not the sed: with no preset files present the shell
  # passes the literal pattern to sed, sed errors, and pipefail ends the
  # postcondition before the guard below can speak (_schema.md, seventh).
  ukis=$(sed -nE 's/^[A-Za-z_]+_uki="([^"]+)".*/\1/p' /etc/mkinitcpio.d/*.preset 2>/dev/null \
           | sort -u) || ukis=
  [ -n "$ukis" ] || { echo "RESULT=FAIL (no preset declares a _uki= path)"; exit 1; }
  fail=0
  for u in $ukis; do
    if [ ! -f "$u" ]; then echo "missing: $u"; fail=1; continue; fi
    # Newer than both inputs, or the UKI predates the config it should carry.
    [ "$u" -nt /etc/kernel/cmdline ]   || { echo "stale vs cmdline: $u"; fail=1; }
    [ "$u" -nt /etc/mkinitcpio.conf ]  || { echo "stale vs conf: $u";    fail=1; }
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## bootloader-install

**PRD §4.2 says this step's chroot half cannot write EFI variables and "only
places the loader at the removable-media fallback path". Both halves of that are
wrong, and what is actually true is a worse problem.**

`bootctl install` "Installs **systemd-boot** into the EFI system partition. A
copy of **systemd-boot** will be stored as the EFI default/fallback loader at
*ESP*/EFI/BOOT/BOOT\*.EFI. The boot loader is then added to the top of the
firmware's boot loader list." (*measured*, `bootctl(1)`, read 2026-08-24.) It
writes **both** locations, not only the fallback — which `state/boot.md`'s
`## bootctl` confirms, listing `EFI/systemd/systemd-bootx64.efi` and
`EFI/BOOT/BOOTX64.EFI` under "Available Boot Loaders on ESP".

And it does attempt the EFI variable write in a chroot. The suppression is
narrower than §4.2 assumes: `--variables=` "Controls whether to touch the
firmware's boot loader list stored in EFI variables… If not specified or set to
**auto**, EFI variables will not be modified when execution in a **container
runtime** is detected" (*measured*, read 2026-08-24). A chroot is not a
container runtime, and the option distinguishes them deliberately. *(There is no
`--no-variables`; the flag is `--variables=yes|no|auto`, added in version 258.)*

**What is true is that the failure is silent by design.** `--graceful`: "Ignore
failure when the EFI System Partition cannot be found, when EFI variables cannot
be written, or a different or newer boot loader is already installed. Currently
only applies to **is-installed**, **update**, **random-seed** and **install**
verbs. **Since version v258, it is implicitly enabled when running inside a
chroot.**" (*measured*, read 2026-08-24.) The machine runs systemd 261.2
(*measured*, `state/boot.md`, `## bootctl`), so this is in force.

So inside the chroot `bootctl install` exits 0 whether or not the NVRAM entry
was written, **and** whether or not it found an ESP at all. The exit status
carries no information here. That is why this step's postcondition asserts the
ESP contents and says nothing about EFI variables, and why the NVRAM half is
`bootloader-nvram-entry` rather than a clause in this one. It is not that the
chroot cannot do it — it is that nothing can tell you whether it did.

**`bootctl is-installed` is the right verb and this environment disarms it.**
The man page draws exactly the line these two steps need: it "Checks whether
**systemd-boot** is installed in the ESP… this hence checks whether
**systemd-boot** is one (of possibly many) installed boot loaders — and neither
whether it is the default nor whether it is registered in any EFI variables"
(*measured*, read 2026-08-24). But `is-installed` is one of the four verbs
`--graceful` covers, so in the chroot it too returns success when the ESP cannot
be found. Asking the tool produced the correct verb and the man page is what
revealed the environment had switched it off; both channels were needed.

The paths are globbed rather than spelled. `bootctl(1)` writes the fallback as
`EFI/BOOT/BOOT*.EFI`, and the architecture suffix is not a constant this spec
should pin.

**`loader.conf` is written here and asserted here, and its authorship is a
documented gap.** Neither `bootctl(1)`'s `install` entry nor `systemd-boot(7)`
says whether `install` creates a default `loader.conf`; systemd-boot(7) only
says the loader "reads runtime configuration such as the boot timeout and
default entry from /loader/loader.conf on the ESP" (*measured*, read
2026-08-24). Writing it *after* `bootctl install` is correct under either
answer, which is why the command is ordered that way rather than because the
order is known to matter.

`editor no` is the line that earns the assertion. It is what stops a cmdline
being edited at the boot menu, which is the only reason a signed UKI's embedded
cmdline is tamper-evident rather than merely inconvenient to change —
[[disk#btrfs-default-subvol|`btrfs-default-subvol`]] and
[[areas/infrastructure/arch-thinkpad/state-notes|notes.md]] both rest on it, and until now nothing checked it
existed. It gets its own diagnostic. The measured content is
`state/boot.md`'s `## loader.conf`.

```yaml
id: bootloader-install
requires: [uki-generate, base-install]
precondition: |
  set -euo pipefail
  # Runs under arch-chroot, so the ESP is /boot rather than /mnt/boot. bootctl
  # is graceful about a missing ESP in a chroot and would report success, so
  # this has to be established before rather than inferred after.
  findmnt -no FSTYPE -M /boot | grep -qx vfat
command: |
  set -euo pipefail
  # Exits 0 in a chroot even when it finds no ESP and even when the EFI
  # variable write fails: --graceful is implicit here since systemd 258. The
  # postcondition, not the exit status, is what says whether this worked.
  bootctl install
  # After the install, not before: whether `bootctl install` writes a default
  # loader.conf is undocumented, and this order is correct either way.
  # printf rather than a heredoc, so the block survives being copied out of
  # this document with its indentation — a heredoc terminator must be at
  # column zero, and <<- strips tabs, not the spaces a transclusion carries.
  printf '%s\n' \
    'default arch-linux.efi' \
    'timeout 3' \
    'console-mode max' \
    'editor no' \
    > /boot/loader/loader.conf
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Globbed, not spelled: bootctl(1) documents the fallback as
  # EFI/BOOT/BOOT*.EFI and the architecture suffix is not this spec's constant.
  # `bootctl is-installed` would be the right verb and is disarmed by the same
  # implicit --graceful that makes the command's exit status uninformative.
  ls /boot/EFI/systemd/systemd-boot*.efi >/dev/null 2>&1 \
    || note "no systemd-boot binary in /boot/EFI/systemd/"
  ls /boot/EFI/BOOT/BOOT*.EFI >/dev/null 2>&1 \
    || note "no fallback loader in /boot/EFI/BOOT/"
  # Deliberately no assertion about EFI variables. In a chroot the write is
  # attempted and its failure is ignored, so there is nothing here to observe.
  # bootloader-nvram-entry asserts it, on the far side of first-boot.
  conf=/boot/loader/loader.conf
  test -f "$conf" || { echo "RESULT=FAIL ($conf absent)"; exit 1; }
  # editor no has its own diagnostic: it is what makes a signed UKI's embedded
  # cmdline tamper-evident, and two other chapters rest on it.
  grep -qE '^editor[[:space:]]+no$' "$conf" \
    || note "loader.conf lacks 'editor no' — the boot menu can edit the cmdline"
  grep -qE '^default[[:space:]]+\S+\.efi$' "$conf" || note "loader.conf has no default entry"
  grep -qE '^timeout[[:space:]]+[0-9]+$'   "$conf" || note "loader.conf has no timeout"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## firmware-setup-mode

A real node with no `command`, because nobody can shell out to a BIOS menu and
the menu path is not the durable fact — the observable result is. Per
`_schema.md`, absence of `command` carries this without a schema field.

It requires `first-boot` as well as `uki-generate`: this is the first step on
the far side of the execution-context boundary.

The `sbctl status` output shape is grounded in `state/boot.md`'s `## secure
boot` section, which shows `Setup Mode:` and `Secure Boot:` labels with a tick
and a word (*measured*). `Disabled` does not contain the substring `Enabled`, so
the two match patterns discriminate cleanly.

```yaml
id: firmware-setup-mode
requires: [uki-generate, first-boot]
postcondition: |
  sbctl status | grep -q 'Setup Mode:.*Enabled' \
    && echo RESULT=PASS || echo RESULT=FAIL
```

---

## sbctl-enroll-keys

```yaml
id: sbctl-enroll-keys
requires: [firmware-setup-mode]
precondition: |
  sbctl status | grep -q 'Setup Mode:.*Enabled'
command: |
  set -euo pipefail
  sbctl create-keys
  # -m keeps Microsoft's certificates: Lenovo firmware updates delivered
  # through fwupd are signed by them, and dropping it loses that path.
  sbctl enroll-keys -m
postcondition: |
  sbctl status | grep -q 'Setup Mode:.*Disabled' \
    && echo RESULT=PASS || echo RESULT=FAIL
```

Enrolling keys takes the firmware *out* of setup mode, which is why the pre- and
postconditions look inverted relative to each other. `Vendor Keys: microsoft` in
`state/boot.md` is the measured confirmation that `-m` took effect.

---

## bootloader-nvram-entry

The other half of PRD §3.4's `bootctl install`, on the far side of the boundary,
and the reason it is a second step is not §4.2's: the chroot invocation *tries*
to write the EFI variable and is documented to ignore its own failure. Success
and silent failure are indistinguishable from inside, so the entry has to be
asserted where it can be read — from firmware, on a booted system.

**It is a step and not a standing invariant**, which is worth saying because it
satisfies the three-part test in [[disk#resume-offset-matches-swapfile|`spec/disk`]]:
established once at build time, invalidatable afterwards by a legitimate action
(PRD §9.6 lists "Firmware update bricked boot entries" as a recovery scenario),
and silent when it breaks — the machine still boots, from the removable-media
fallback at `EFI/BOOT/BOOT*.EFI`, so the loss is a named entry and a boot-order
position rather than an outage. That is three for three. It is a step anyway,
because `_schema.md`'s rule for a command-less step is that the executor *cannot
make it true*, and here it plainly can: the command is this one.

**Its position relative to `uki-sign` is the load-bearing part.** `bootctl
install` copies binaries into the ESP, so running it after signing replaces
`systemd-bootx64.efi` and `BOOTX64.EFI` with unsigned copies — the same silent
class `uki-sign`'s own prose warns about for regenerated UKIs, and with a worse
ending, since a firmware that rejects the loader gives no menu to recover from.
The edge from `uki-sign` to here is what orders it. `secureboot-enable`'s
precondition, which re-verifies every file at run time rather than trusting the
edges, is the backstop if a human arrives out of order.

The step is safe against `--graceful`'s third clause — "a different or newer
boot loader is already installed" — because `bootloader-install` put the same
version there from the same package one boundary earlier.

The parse target is `bootctl status`, whose "Boot Loaders Listed in EFI
Variables" section is measured in `state/boot.md`'s `## bootctl`. **The title
string is `measured` from that output rather than from documentation** —
`bootctl(1)` describes the section without naming what systemd-boot registers
itself as — so if it changes, the match fails and this reports FAIL rather than
passing on something else. Same standing as the `cryptsetup status` `device:`
parse elsewhere in this chapter.

**The match is anchored, and the reason is on the same machine.**
`state/boot.md` shows *two* Linux entries: `Linux Boot Manager` at `0x0001` and
`Fallback Linux Boot Manager` at `0x0002`. An unanchored `grep 'Title: Linux
Boot Manager'` matches both, so a build with only the fallback entry would pass.
Identical discrimination problem to `is signed` versus `is not signed` in
`uki-sign`, and solved the same way.

*Unexplained, recorded rather than resolved:* that `0x0002` entry points at
`EFI/systemd/systemd-boot-fallbackx64.efi`, which appears neither in "Available
Boot Loaders on ESP" nor among the eight files `sbctl verify` reports. It is a
dangling NVRAM entry, the second on this machine after `Windows Boot Manager`,
and under Secure Boot selecting it would fail. Neither `bootctl(1)` nor
`systemd-boot(7)` mentions such a binary (*measured*, both read 2026-08-24).

**The sweep found a different defect here, of the older kind.** The check was
`if sudo bootctl status | grep -qE 'Title: …'`, and under `pipefail` a `bootctl`
that failed outright takes the `else` branch — reporting *no `Linux Boot
Manager` entry in the firmware's list* when nothing was read from the firmware
at all. That is `_schema.md`'s original don't-fail-the-same-way rule rather than
the seventh convention: a verdict is emitted, and it is wrong about why. On this
page the misdirection is expensive — the reader is being told the NVRAM entry is
gone, which is [[../runbooks/bootloader-reinstall|a runbook]] away. Reading
first and matching second separates them.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — both `Linux Boot Manager` and the fallback listed | PASS | PASS |
| 2 | only `Fallback Linux Boot Manager` | FAIL, no entry — the anchoring case | same |
| 3 | the section is present and empty | FAIL, no entry | same |
| 4 | **`bootctl status` fails outright** | `RESULT=FAIL (bootctl status did not run — this says nothing about the entry)` | **FAIL, `no 'Linux Boot Manager' entry`** |

```yaml
id: bootloader-nvram-entry
requires: [first-boot]
precondition: |
  set -euo pipefail
  # The binaries must already be on the ESP. If they are not, this command
  # would install them here instead — on the far side of the boundary, where
  # bootloader-install's postcondition can no longer speak for them.
  sudo ls /boot/EFI/systemd/systemd-boot*.efi >/dev/null
command: |
  set -euo pipefail
  # Re-runs the whole install, which is why nothing signed may exist yet: this
  # replaces the ESP binaries with fresh unsigned copies. uki-sign requires
  # this step so that ordering is an edge rather than a hope.
  sudo bootctl install
postcondition: |
  set -euo pipefail
  # Anchored: `Fallback Linux Boot Manager` contains `Linux Boot Manager`, and
  # an unanchored match would pass on a machine that has only the fallback
  # entry. The title string is measured from state/boot.md, not documented.
  # Read first, match second. Piped straight into grep, a bootctl that failed
  # outright takes the else branch and reports the entry as missing — the check
  # failing the same way as the thing it detects.
  status=$(sudo bootctl status) \
    || { echo "RESULT=FAIL (bootctl status did not run — this says nothing about the entry)"; exit 1; }
  if grep -qE '^[[:space:]]*Title:[[:space:]]+Linux Boot Manager$' <<<"$status"
  then echo RESULT=PASS
  else echo "RESULT=FAIL (no 'Linux Boot Manager' entry in the firmware's list)"
  fi
```

---

## uki-sign

The real hazard here is not signing-before-keys, which fails loudly — the PRD
records the message: `Secureboot key directory doesn't exist, not signing!` It
is **regenerating UKIs after signing**, which leaves unsigned binaries in place
with no error at all. That is why this postcondition must be re-runnable and
must resolve through `sbctl` rather than against a stored path.

**`bootctl install` is the second member of that class, and the edge to
`bootloader-nvram-entry` is what keeps it on the right side.** It copies fresh
binaries into the ESP, so running it after this step replaces
`systemd-bootx64.efi` and `BOOTX64.EFI` with unsigned ones. PRD §9.6 walks
straight into it: its recovery row for "Firmware update bricked boot entries"
reads `bootctl install` from the ISO chroot, then `mkinitcpio -P`, and says
nothing about re-signing. On a Secure Boot machine that procedure leaves the
firmware rejecting its own loader — the repair for a broken boot entry breaking
the boot outright. It is also why
`sbctl` is worth preferring over a hand-rolled `sbsign` loop: the signing
database is what makes the check re-derivable.

`sbctl verify` is `measured` (run 2026-08-23): it requires sudo, prints
`✓ <path> is signed` per file, and **exits 0 regardless**. So the postcondition
must not test `$?` — an exit 0 with an unsigned file listed would pass.

That measurement also settled what this step signs. `sbctl verify` reports eight
files on this machine: the four UKIs, `BOOTX64.EFI`, `systemd-bootx64.efi`, and
both `vmlinuz-*` kernels. The kernels are EFI binaries in the ESP, so `verify`
scans them whether or not this spec signs them — and an unsigned file in that
list is a permanent FAIL. A check with a known-acceptable failure is a check
nobody reads, so the step signs them. They are never loaded directly under this
design (`editor no`, Type #2 UKI entries only — `state/boot.md`,
`## loader.conf`); they are signed to keep the assertion clean, not because the
boot path needs it.

The postcondition parses **JSON, not text**. `sbctl verify --json` emits an
array of `{"file_name": …, "is_signed": 1}` (*measured*, 2026-08-23), which
removes both text-parsing hazards at once: there is no header line to exclude,
and no dependence on which glyph upstream renders the failure marker as —
upstream writes it both `✘` (U+2718) and `✗` (U+2717), and the line format
changed historically from `-> WARNING: X is not signed`. Four versions of this
check were written before the flag was noticed; it had been in `sbctl.8` the
whole time. Ask whether a tool has machine-readable output before investing in
parsing the human-readable kind.

`is_signed` is the **number 1, not a boolean**, so the comparison is `== 1`
rather than a truthiness test. Written that way it also fails on `null`, on a
missing field, and on `true` — a schema change becomes a loud failure instead of
a silent pass.

Two residual risks, recorded so they are not rediscovered:

- **A Windows bootloader on the ESP would make this permanently red.** sbctl
  issue #260 reports `verify` marking Microsoft binaries unsigned even with
  Microsoft keys enrolled. `state/boot.md` lists a `Windows Boot Manager` EFI
  variable pointing at `bootmgfw.efi`, but the ESP's own listing contains only
  the two systemd-boot binaries — a stale NVRAM entry, consistent with the eight
  files `verify` actually reports (`inferred`).
- **Permission errors present as unsigned.** sbctl issue #46: without sufficient
  access `verify` reports signed files as unsigned. That is why the command uses
  `sudo` rather than relying on `/boot` being readable.

**"Exits 0 regardless" is measured; "emits JSON regardless" is not.** The
postcondition leaned on the first to justify never testing `$?`, then fed the
output straight to `jq`, which exits non-zero on anything that is not JSON. So
an `sbctl` that printed an error message instead of an array ended the
postcondition silently — the seventh convention, reached from the one direction
this step's prose had already reasoned carefully about.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — every file signed | `RESULT=PASS (n/n signed)` | same |
| 2 | one file unsigned | FAIL, `(1/2 signed)` + the file name | same |
| 3 | empty array | `RESULT=FAIL (0/0 signed)` — the `total -gt 0` guard | same |
| 4 | `is_signed` is boolean `true` rather than `1` | FAIL, names the file | same |
| 5 | `is_signed` field missing | FAIL, names the file | same |
| 6 | **output is an error message, not JSON** | `RESULT=FAIL (… returned nothing parseable)` | **no `RESULT=`** |
| 7 | **`sbctl` fails outright** | same message | **no `RESULT=`** |

Rows 4 and 5 are the `== 1` comparison earning its keep. They were already
correct and are in the set anyway, because a mutation set that exercises only
the lines you changed cannot tell you whether you broke the ones you did not.

```yaml
id: uki-sign
requires: [sbctl-enroll-keys, uki-generate, bootloader-nvram-entry]
precondition: |
  sbctl status | grep -q 'Setup Mode:.*Disabled'
command: |
  set -euo pipefail
  # Enumerate rather than list literally: the set of UKIs and kernels changes
  # when one is added or removed. /boot is dmask=0077, hence the privileged
  # enumeration.
  sudo ls -1 /boot/EFI/Linux/ | while read -r u; do
    sudo sbctl sign -s "/boot/EFI/Linux/$u"
  done
  sudo ls -1 /boot/ | grep '^vmlinuz-' | while read -r k; do
    sudo sbctl sign -s "/boot/$k"
  done
  sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
  sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
postcondition: |
  set -euo pipefail
  # sbctl verify exits 0 even when it reports an unsigned file, so never test $?.
  # --json removes the text parsing entirely: no header line to exclude and no
  # dependence on which glyph upstream renders the failure marker as.
  out=$(sudo sbctl verify --json) || out=
  # jq exits non-zero on input that is not JSON. sbctl's exit-0-regardless is
  # measured; that its output is *always* a JSON array is not, so a parse
  # failure has to report rather than abort (_schema.md, seventh).
  total=$(jq 'length' <<<"$out" 2>/dev/null) || total=
  # is_signed is the number 1, not a boolean — compare, never test truthiness.
  # != 1 also catches a missing or null field rather than letting it pass.
  signed=$(jq '[.[] | select(.is_signed == 1)] | length' <<<"$out" 2>/dev/null) || signed=
  if [ -z "$total" ] || [ -z "$signed" ]; then
    echo "RESULT=FAIL (sbctl verify --json returned nothing parseable)"; exit 1
  fi
  # total -gt 0 guards the empty case: an empty array satisfies "all signed".
  if [ "$total" -gt 0 ] && [ "$signed" -eq "$total" ]; then
    echo "RESULT=PASS ($signed/$total signed)"
  else
    echo "RESULT=FAIL ($signed/$total signed)"
    jq -r '.[] | select(.is_signed != 1) | .file_name' <<<"$out"
  fi
```

---

## secureboot-enable

The second command-less step, and the one where **the ordering is the whole
point**. Enabling Secure Boot in firmware before `uki-sign` has succeeded leaves
a machine that will not boot: the firmware rejects the unsigned UKI, and the
recovery is a trip back into the firmware menu to turn it off again. The DAG
edge from `uki-sign` is what encodes that, and the precondition re-checks it at
run time rather than trusting the edge — because the executor contract permits a
human to arrive at this step out of order, and the cost of being wrong here is
paid at the next power cycle rather than immediately.

The precondition is the full `uki-sign` assertion, not a weaker proxy such as
"keys are enrolled". Enrolled keys with an unsigned UKI is precisely the state
that bricks the boot.

There is no `command` because this is a firmware menu. What matters is the
observable result, which `sbctl status` reports from the booted system.

```yaml
id: secureboot-enable
requires: [uki-sign]
precondition: |
  set -euo pipefail
  # Not a proxy: enabling Secure Boot with any unsigned EFI binary in place
  # produces an unbootable machine. Same assertion as uki-sign's postcondition.
  out=$(sudo sbctl verify --json)
  total=$(jq 'length' <<<"$out")
  signed=$(jq '[.[] | select(.is_signed == 1)] | length' <<<"$out")
  [ "$total" -gt 0 ] && [ "$signed" -eq "$total" ]
postcondition: |
  set -euo pipefail
  sbctl status | grep -q 'Secure Boot:.*Enabled' \
    && sbctl status | grep -q 'Setup Mode:.*Disabled' \
    && echo RESULT=PASS || echo RESULT=FAIL
```

---

## tpm2-enroll

**A failing precondition here is usually correct behaviour.** A firmware update
legitimately changes the PCRs, which invalidates the enrollment by design. The
executor contract in `_schema.md` covers this without a schema field: it halts
and reports, and never re-runs a step to make a check pass. Re-running
`systemd-cryptenroll` to clear a "failure" here destroys a working enrollment.
The repair is a different step — [[../runbooks/tpm2-reenroll|the re-enrollment
runbook]] — reached deliberately, not by an executor retrying.

`--tpm2-pcrs=` is the field that matters, and the reason the postcondition reads
the header rather than checking that a token exists. Per
`systemd-cryptenroll(1)`: binding to no PCRs at all "is also the default, if
this option is not used" (*measured*, read 2026-08-22). An enrollment with no
PCR binding and one bound to PCR 7 are indistinguishable by presence. With
`--tpm2-with-pin=yes` also in force the practical cost of an unbound enrollment
is the loss of the Secure-Boot-state binding, not open access — the PIN still
gates it — but that is precisely the property this step exists to establish.

**The PCR set is decided in [[../decisions/20260823-tpm2-pcr-selection|ADR-0013]],
not here:** `0+2+7`, because `org.fwupd.hsi.Amd.SpiWriteProtection` is
`Disabled` and `action-contact-oem`, so firmware modification on this machine is
reachable from a root shell. PCR 7 alone does not move when firmware is modified
with the enrolled keys left intact.

**Status: satisfied.** The machine was re-enrolled against `[0,2,7]` with a PIN
on 2026-08-23 and rebooted; the PIN prompt returns and this postcondition passes
(*measured*, `state/storage.md`, `## luks tokens`). Between the writing of this
step and that re-enrollment the spec deliberately asserted a property the system
did not have, and reported FAIL — which was the correct behaviour, not a defect
to be softened.

**This step touches nothing else.** PRD §12.2 instructs you to add
`rd.luks.options=<UUID>=tpm2-device=auto` and rebuild the UKIs. Measurement says
otherwise: there is no `/etc/crypttab`, no `/etc/crypttab.initramfs`, no
crypttab anywhere in the extracted initramfs, and no `tpm2-device=` in the
signed `.cmdline` — and unlock works. `tpm2-enroll` is a DAG leaf. Nothing
requires it, and it requires nothing but a settled Secure Boot state. The cycle
implied by the PRD does not exist. That negative finding cost more to establish
than a positive one would have; it is written down so the next person asking
"where is this configured?" does not repeat the search across three locations.

`sd-encrypt`'s help confirms it from the other side, which is the stronger form
of the finding: "Alternatively, if the file */etc/crypttab.initramfs* exists, it
will be added to the initramfs as */etc/crypttab*." That names the exact
mechanism by which a crypttab would reach the initrd — and that file does not
exist on this machine (`state/storage.md`, `## crypttab`). The search was not
merely exhaustive; the one documented path is confirmed absent.

**Gap, narrowed 2026-08-23.** This was recorded as "the man pages do not account
for the observed behaviour," which was too strong. `mkinitcpio -H sd-encrypt`
states the capability outright: the hook "allows systemd to unlock encrypted
LUKS1 volumes with a password, and encrypted LUKS2 volumes with a password,
FIDO2 token or TPM2 key" (*measured*, read 2026-08-23). So the TPM2 unlock path
is documented, and it is documented at exactly the layer that
`mkinitcpio-config` puts in place — which is also why that step asserts
`sd-encrypt` rather than treating the hook list as cosmetic.

What remains unexplained is narrower: the **PIN interaction specifically**.
`cryptsetup-open(8)` documents automatic use of LUKS2 tokens "unprotected by
PIN," and this token is PIN-protected, so that page still does not account for
it being tried automatically. `sd-encrypt`'s help does not spell the PIN case
out either way. The likely mechanism is the external token plugin reached by
`systemd-cryptsetup`, whose policy differs from the `cryptsetup open` CLI
(`inferred`). The spec asserts the behaviour, which is measured; it does not
assert the mechanism.

**This postcondition is duplicated in
[[../runbooks/tpm2-reenroll|runbooks/tpm2-reenroll]] and both copies must move
together.** Not an oversight: transcluding this section into the runbook would
carry `command:` with it, and that command lacks `--wipe-slot=tpm2`. The runbook
records why the duplication is the safer trade.

**The 2026-08-24 sweep moved both copies, and the rule earned itself.** Two
changes: the `cryptsetup status` reading is allowed to come back empty and gets
its own verdict, because an empty device makes `luksDump` fail and read as *no
TPM2 token* — a different fact, on the page someone opens at an unexpected
passphrase prompt. And `tpm2-pcrs` gained `// []`, because a token bound to no
PCRs at all made `jq` print `null (null) cannot be sorted` above the verdict.
The verdict was already right; the message was a stack trace on **precisely the
case this step exists to detect**, which the prose above spends a paragraph
explaining. A check whose answer is buried is the failure `_schema.md` names,
and a jq trace buries it as effectively as 400 lines would.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — one token, `[0,2,7]`, PIN | PASS | PASS |
| 2 | **no `cryptroot` mapping** | `RESULT=FAIL (cryptsetup status named no backing device)` | **no `RESULT=`** |
| 3 | **token bound to no PCRs** | FAIL | FAIL, under a `jq` stack trace |
| 4 | bound to PCR 7 only | FAIL | same |
| 5 | bound to `[0,2,7]` without a PIN | FAIL | same |
| 6 | no `systemd-tpm2` token at all | FAIL | same |

Row 3 is the case the whole step is written around: `--tpm2-pcrs=` defaults to
binding to nothing, so an unbound enrollment and a correct one are
indistinguishable by presence.

```yaml
id: tpm2-enroll
requires: [secureboot-enable]
precondition: |
  set -euo pipefail
  # Secure Boot must be in its final state: enrolling against PCR 7 before
  # that means re-enrolling afterward.
  sbctl status | grep -q 'Secure Boot:.*Enabled'
  sbctl status | grep -q 'Setup Mode:.*Disabled'
command: |
  set -euo pipefail
  luksdev=$(cryptsetup status cryptroot | awk '/device:/ {print $2}')
  # 0+2+7 per ADR-0013. Not 7: SPI write protection is disabled on this
  # hardware, so firmware modification does not move PCR 7.
  systemd-cryptenroll --tpm2-device=auto \
    --tpm2-pcrs=0+2+7 \
    --tpm2-with-pin=yes \
    "$luksdev"
postcondition: |
  set -euo pipefail
  luksdev=$(cryptsetup status cryptroot | awk '/device:/ {print $2}') || luksdev=
  # An empty device would make luksDump fail and read as "no TPM2 token", which
  # is a different fact (_schema.md, seventh).
  [ -n "$luksdev" ] \
    || { echo "RESULT=FAIL (cryptsetup status named no backing device)"; exit 1; }
  if cryptsetup luksDump --dump-json-metadata "$luksdev" \
     | jq -e '[.tokens[] | select(.type == "systemd-tpm2")] as $t
              | ($t | length) == 1
                and ((($t[0]["tpm2-pcrs"] // []) | sort) == [0, 2, 7])
                and ($t[0]["tpm2-pin"] == true)' >/dev/null
  then echo RESULT=PASS; else echo RESULT=FAIL; fi
```

---

## Open against this chapter

- **The UKIs carry AMD microcode only,** because `autodetect` precedes
  `microcode`. Asserted rather than fixed: rebuilding on a different CPU
  produces a wrong-microcode image with no error and no `state/` diff. Fine
  while this spec targets one machine; revisit if it is ever used to build for
  another.
- **`mkinitcpio-config` asserts membership and relative position, not the exact
  array.** `modconf` and `sd-vconsole` are present on the machine and
  unasserted — neither has a documented ordering constraint and neither is
  load-bearing for this chapter.
- **The `cryptsetup status` `device:` parse appears in three steps here.** It is
  measured, but from observation rather than documentation. Worth considering
  whether the collector should record the backing device so `state/` grounds it
  in one place instead of three.
- **`uki-generate` asserts freshness, not content.** The content check lives in
  the collector rather than the spec, for the `systemd-ukify` availability
  reason given above. If `systemd-ukify` becomes part of the `base-install` set,
  fold the check back in here and drop the split.
- **PRD §9.6's recovery procedure is unsafe under Secure Boot and is not a spec
  problem.** `bootctl install` then `mkinitcpio -P`, with no re-signing step,
  replaces the signed loader with an unsigned one. `uki-sign`'s prose records
  it; the fix belongs in a runbook, since §9.6 is a table of things you do when
  something is already broken and this chapter has no step for that state.
- **`bootloader-nvram-entry`'s title match is grounded in `state/` alone.**
  `bootctl(1)` documents the "Boot Loaders Listed in EFI Variables" section
  without naming the title systemd-boot registers. `bootctl list --json=` might
  make the whole parse structural, but its field names are undocumented — the
  same lookup `resume-offset-matches-swapfile` in `spec/disk` declined to guess
  at, and the same one that would close both.
- **The `Fallback Linux Boot Manager` entry points at a binary that is not on
  the ESP.** Measured, unexplained, and undocumented in both `bootctl(1)` and
  `systemd-boot(7)`. Harmless while the default entry works; under Secure Boot
  it is an entry that cannot boot, sitting in the boot order.
- **Five of the ten postconditions here now have written-down mutation sets;
  five do not.** `cmdline-write`, `uki-generate`, `uki-sign`,
  `bootloader-nvram-entry` and `tpm2-enroll` were re-tested 2026-08-24 against
  31 cases. `mkinitcpio-config` was tested in a previous round against seven
  mutations that were **not** written down — the same defect that deferred the
  `spec/disk` fixes for a round — and `firmware-setup-mode`, `sbctl-enroll-keys`,
  `bootloader-install` and `secureboot-enable` have none. They were swept for
  the seventh convention and are clean; that is a narrower claim than tested.
- **`tpm2-enroll` carries no `sudo`, and its runbook copy now does.** The step
  runs on the booted system, but its `cryptsetup status`, `cryptsetup luksDump`
  and `systemd-cryptenroll` are all unprefixed. As a normal user `cryptsetup
  status cryptroot` prints `Cannot initialize device-mapper, running as non-root
  user.` and returns nothing (*measured* 2026-10-03, through the copy in
  [[../runbooks/tpm2-reenroll|runbooks/tpm2-reenroll]]), so the postcondition
  reports `cryptsetup status named no backing device` on a correct enrollment.
  The runbook's two `luksdev=` lines gained `sudo` that day; this step was left
  alone because its mutation set was not re-run. The copies have diverged until
  it is. Related: since systemd 262 `--tpm2-with-pin=yes` enrolls an Argon2id
  PIN slot (*measured*, same run) — the postcondition does not look at that
  field and still passes.
- **The PIN interaction with automatic token use is `inferred`.** Narrowed from
  "the mechanism is undocumented" — `sd-encrypt` documents TPM2 unlock; what no
  page covers is a *PIN-protected* token being tried automatically. PCR 2
  remains measured non-empty and unexplained. Both are revisit conditions on
  ADR-0013.
