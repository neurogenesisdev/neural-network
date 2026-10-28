---
title: "Disk layout — partitions, LUKS2, btrfs subvolumes, swapfile, ESP"
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-23
updated: 2026-08-24
tags: [spec, disk, luks, btrfs, swapfile, esp, hibernation]
---

# Disk layout

Ten steps, from an unpartitioned drive to a mounted subvolume tree with a
hibernation-sized swapfile. Read [[_schema|_schema.md]] first.

Replaces PRD §1.1–§1.6, and rejects part of both §1.2 and §1.4: the machine runs
continuous TRIM at the dm-crypt and btrfs layers, which the PRD decided against
at each. That is settled in
[[../decisions/20260823-continuous-trim-both-layers|ADR-0014]], which this
chapter's findings produced — the divergence surfaced while grounding these
steps against `state/`, and the reasoning did not belong in a step record.

**This is the first chapter written from scratch.** `spec/boot` promoted three
worked examples that had already been argued through; here only two steps —
`btrfs-default-subvol` and `btrfs-default-matches-root` — arrive that way. The
other eight were written against man pages and `state/`, and per the accepted
limitation recorded in [[../sessions/20260823-spec-boot-promotion|the second
session log]], none of them has been executed by whoever wrote them.

## Where this chapter sits

```
disk-partition ─→ luks-format ─→ btrfs-mkfs ─→ btrfs-subvols-create
                                                        │
                                                        ▼
                                               btrfs-default-subvol
                                                        │
                                                        ▼
                                       ┌──── btrfs-subvol-mount ────┐
                                       ▼                            ▼
                                  esp-mount                  swapfile-create

first-boot ──┬─→ btrfs-default-matches-root
             └─→ resume-offset-matches-swapfile
```

Everything above the gap runs **in the live installer, before `pacstrap`** —
not merely before `first-boot` but before the chroot exists at all. The
[[areas/infrastructure/arch-thinkpad/spec/_index#The DAG is not a reading order|derived-context rule]] puts installer
and chroot on the same side of the boundary, which is right for the question it
answers (`sudo` or not: none of these steps uses it) and hides the one that
bites here: **paths.** Every path in this chapter is `/mnt`-prefixed, and the
same file is `/swap/swapfile` to `cmdline-write` one chapter over. The two names
denote one file; a check that hardcodes either is correct in exactly one
execution context.

The two steps below the gap are standing invariants, and neither takes an edge
to the build step it watches. That is deliberate and argued in
`resume-offset-matches-swapfile`.

---

## disk-partition

**The most destructive step in the spec, and the only one with a second disk
attached while it runs.** `sgdisk -Z` "Zap (destroy) the GPT and MBR data
structures and then exit" (*measured*, `sgdisk(8)`, read 2026-08-23) — pointed
at the wrong node it destroys whatever is there, with no confirmation prompt and
no undo. The wrong node is not hypothetical: `state/hardware.md`'s `## usb`
section shows a `Samsung Electronics Co., Ltd Portable SSD T5` attached, and
`state/storage.md`'s `## block devices` shows it as `sda` with a 465 GB ext4
partition — the unencrypted rescue store of [[../decisions/20260815-t5-rescue-partition-unencrypted|ADR-0009]],
which exists precisely to be readable when this machine is not.

So the target is a parameter, not a literal. `/dev/nvme0n1` is exactly the kind
of value the vault forbids in a step record: it is stable on this machine and
assigned by enumeration order, which is to say stable until it is not. The
precondition does the work a literal would only have appeared to do:

- **`$DISK` must be set**, and `${DISK:?}` makes an unset variable a halt rather
  than an empty string concatenated into `sgdisk -Z ""`.
- **Nothing on it may be mounted.** `lsblk -no MOUNTPOINT` is the column the
  collector rejected for inventory, because it "displays only one mount point
  (usually the last mounted instance of the filesystem)" (*measured*,
  `lsblk(8)`, quoted in `collect.sh`). Asserting *emptiness* is the one question
  where "one of them" and "all of them" cannot differ, so the defect that made
  it wrong there makes no difference here.
- **The transport must be `nvme`.** This is what distinguishes the internal
  drive from the T5, which is `usb`. `TRAN` is a documented column (*measured*,
  `lsblk --list-columns`, run 2026-08-23).

That last one narrows the spec to NVMe-attached targets. Same narrowing, and the
same justification, as the microcode ordering in `mkinitcpio-config`: these
records build *this* machine, and a check that pins the assumption is better
than an assumption nobody wrote down.

**Partition device names are derived, never concatenated.** `${DISK}1` is right
for `sda` and wrong for `nvme0n1`, which needs `p1`. The postcondition asks
`lsblk` for the children by `PARTN`, "partition number as read from the
partition table" (*measured*, `lsblk --list-columns`), rather than trusting
listing order.

The same derivation is what `luks-format` and `esp-mount` check their own
parameters against, so those two steps cannot be pointed at a partition of
another disk. That matters most for `luks-format`: `sda2` on the rescue drive is
466 GB of unformatted space, and a `luksFormat` aimed there would succeed. Three
steps here take a parameter, and how one reaches a step is a convention rather
than a schema field — [[_schema#Conventions|`_schema.md`]] states it, including
why validating a parameter's *type* is not enough.

**The type codes are asserted by name, not by GUID, and that is a compromise.**
The authoritative values are `c12a7328-…` for the ESP and `4f68bce3-…` for the
x86-64 root partition (*measured*, `systemd-gpt-auto-generator(8)` Table 1, read
twice 2026-08-23). Neither can be written here: the pre-commit hook blocks any
staged line matching the UUID shape, `CLAUDE.md` says redact rather than
`--no-verify`, and a *standardised type* GUID has no redaction token because it
is not an identifier of anything. So the check greps `sgdisk -i`, which prints
"the translation of sgdisk's internal partition type code to a plain type name"
(*measured*, `sgdisk(8)`). **The name strings themselves are `inferred`** — they
come from sgdisk's type table, which was not read on this machine; the lookup
that closes it is `sgdisk -L | grep -iE 'ef00|8304'`, one command in the live
environment. The match is case-insensitive and deliberately short for that
reason, and a mismatch reports the string it actually saw.

The stakes are lower than that paragraph suggests: a wrong ESP type is caught
again downstream, because `bootctl` will not adopt a partition that is not an
ESP, and `state/boot.md`'s `## bootctl` shows it reporting `ESP: /boot` today.

The ESP size check is a floor, not an equality. `sgdisk -n 1:0:+1G` positions the
end "measured in … gibibytes (G)" relative to the start (*measured*,
`sgdisk(8)`), but partition alignment is the tool's business and an equality
would fail for alignment rather than for the hazard. The hazard is a
habit-sized 256 or 512 MiB ESP, which four UKIs do not fit in; 900 MiB
discriminates that cleanly.

### Two defects, found by re-testing rather than by reading

**The partition count aborted on zero.** `nparts=$(… | grep -cx part)` — `grep
-c` prints `0` and exits 1 when it counts nothing (*measured*, run 2026-08-24),
and under `set -e` an assignment from a failing command substitution ends the
script. So the one case this step most needs to report — `sgdisk` did nothing,
in the most destructive step in the spec — produced no `RESULT=` line and no
diagnostic, **failing the same way a broken check would**, which is the
convention `_schema.md` states and this chapter quotes twice. `awk` counts
instead: it exits 0 and prints `0` for no matches. Same defect as
[[base#fstab-generate|`fstab-generate`]]'s, which is where the class was first
seen; it is now the seventh convention in
[[_schema#A postcondition must reach its verdict line on every input it is meant to judge|`_schema.md`]].

**The type-name parse binds to the wrong parenthesis.** `sed -nE
's/^Partition GUID code:.*\((.*)\)$/\1/p'` is greedy, so it captures from the
*last* `(` in the line. Against a plain name it is correct; against a type name
that itself contains parentheses it returns the tail of that name — for a name
of the shape `Linux x86-64 root (/)`, the two characters `/)`. The step would
then report `p2 type is '/)', want x86-64 root` **on a correctly partitioned
disk**: a permanently-red check, the failure `luks-format` avoids by not
counting keyslots. `[^(]*` binds the capture to the first parenthesis and is
identical on a name without any.

*Whether that defect is live here is `inferred`, and it is a third ungrounded
thing in the same three lines.* `sgdisk(8)` documents `-i` as printing "the
translation of `sgdisk`'s internal partition type code to a plain type name"
and **gives no example of its output** (*measured*, read 2026-08-24) — so the
`Partition GUID code:` label and the parenthesised format are as unsourced as
the type strings the prose above already marks `inferred`. The set below was
therefore run twice, against a type name with nested parentheses and one
without; the fixed block behaves identically on both, and the unfixed block
passes its baseline only on the second. `sgdisk -i 1 /dev/nvme0n1` on the
machine settles which is real, and a collector section recording the partition
types would ground the label, the format and the strings at once — which is
what the chapter's open list already asks for, now on three counts rather than
one.

**The mutation set, written down this time.** Extracted from this document
rather than retyped, with `lsblk` and `sgdisk` stubbed; per the standing
limitation, this is synthesis, not execution.

| # | Mutation | Fixed block | Unfixed block |
|---|---|---|---|
| 1 | baseline — 2 partitions, correct types, 1 GiB ESP | PASS | PASS *(only with a paren-free type name)* |
| 2 | **zero partitions — `sgdisk` did nothing** | FAIL, `expected 2 partitions, found 0` | **no `RESULT=` line, exit 1** |
| 3 | one partition only | FAIL, `found 1` + empty p2 type | same |
| 4 | a third partition | FAIL, `found 3` | same |
| 5 | ESP 512 MiB | FAIL, `ESP is 536870912 bytes, want >= 900 MiB` | same |
| 6 | ESP exactly 900 MiB — the floor | PASS | PASS |
| 7 | types swapped between p1 and p2 | FAIL, both type diagnostics | same |
| 8 | p1 present but no `PARTN` | FAIL, `no partition 1 on …` | same |
| 9 | `lsblk` itself fails | no `RESULT=` line, exit 1 | same |

Case 9 is a residual and is left as one: a failing `lsblk` is a broken check,
not the condition being detected, and it now reads differently from case 2 —
which is all the convention asks. Case 6 exists because the check is `-ge`.

```yaml
id: disk-partition
requires: []
precondition: |
  set -euo pipefail
  # DISK is a parameter, not a literal: /dev/nvme0n1 is assigned by enumeration
  # order. ${DISK:?} halts on unset rather than zapping "".
  test -b "${DISK:?DISK must name the target disk, e.g. /dev/nvme0n1}"
  # Nothing mounted from it. The singular MOUNTPOINT column is unfit for
  # inventory (it shows only one of several) and perfectly fit here, where the
  # expected answer is none.
  [ -z "$(lsblk -nro MOUNTPOINT "$DISK" | tr -d '[:space:]')" ] \
    || { echo "$DISK has mounted filesystems"; exit 1; }
  # The rescue drive (ADR-0009) is a USB-attached SSD and is attached during
  # the build. Transport is what tells the two apart.
  [ "$(lsblk -dno TRAN "$DISK")" = nvme ] \
    || { echo "$DISK is not an NVMe device"; exit 1; }
command: |
  set -euo pipefail
  : "${DISK:?DISK must name the target disk}"
  sgdisk -Z "$DISK"
  sgdisk -n 1:0:+1G -t 1:ef00 -c 1:EFI "$DISK"
  sgdisk -N 2 -t 2:8304 -c 2:cryptroot "$DISK"
  # "informs the operating system kernel of partition table changes"
  partprobe "$DISK"
postcondition: |
  set -euo pipefail
  : "${DISK:?DISK must name the target disk}"
  fail=0
  note() { echo "  $1"; fail=1; }
  # Derived, never concatenated: ${DISK}1 is right for sda and wrong for
  # nvme0n1. PARTN is the partition number from the table, so this does not
  # depend on lsblk's listing order either.
  part_of() { lsblk -lnpo NAME,PARTN "$DISK" | awk -v n="$1" '$2==n {print $1}'; }
  # awk, not `grep -c`: grep prints 0 and exits 1 when it counts nothing, and
  # under set -e an assignment from a failing substitution ends this
  # postcondition with no RESULT= line. Zero partitions is the "sgdisk did
  # nothing" case, which is the one thing this step most needs to report.
  nparts=$(lsblk -lno TYPE "$DISK" | awk '$1 == "part" { n++ } END { print n + 0 }')
  [ "$nparts" -eq 2 ] || note "expected 2 partitions, found $nparts"
  # Type by plain name, not by GUID: the pre-commit hook blocks UUID-shaped
  # strings and a standard type GUID has no redaction token. The name strings
  # are inferred from sgdisk's type table — see the prose.
  # [^(] binds the capture to the FIRST parenthesis. A greedy .* binds it to
  # the last, which returns '/)' for a name that contains parentheses.
  t1=$(sgdisk -i 1 "$DISK" | sed -nE 's/^Partition GUID code:[^(]*\((.*)\)$/\1/p')
  t2=$(sgdisk -i 2 "$DISK" | sed -nE 's/^Partition GUID code:[^(]*\((.*)\)$/\1/p')
  printf '%s' "$t1" | grep -qi 'EFI system'   || note "p1 type is '$t1', want EFI system"
  printf '%s' "$t2" | grep -qi 'x86-64 root'  || note "p2 type is '$t2', want x86-64 root"
  # Floor, not equality: alignment is the tool's business. 900 MiB rejects a
  # habit-sized 256/512 MiB ESP, which four UKIs do not fit in.
  esp=$(part_of 1)
  if [ -n "$esp" ]; then
    sz=$(lsblk -bdno SIZE "$esp")
    [ "$sz" -ge $((900 * 1024 * 1024)) ] || note "ESP is $sz bytes, want >= 900 MiB"
  else
    note "no partition 1 on $DISK"
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## luks-format

`cryptsetup-luksFormat(8)` states the cost of getting the target wrong in one
sentence: "Doing a luksFormat on an existing LUKS container will regenerate the
volume key. Unless you have a header backup, all old encrypted data in the
container will be permanently irretrievable." (*measured*, read 2026-08-23.)
The precondition therefore refuses to run against a device that is already a
LUKS container — `cryptsetup isLuks` "Returns true if <device> is a LUKS device,
false otherwise", one of the few exit statuses in this chapter that is
documented rather than observed, so it is used as an exit status rather than
parsed.

**This step is interactive and stays a `command` step anyway.** The synopsis is
`cryptsetup luksFormat [<options>] <device> [<key file>]`; with no key file the
passphrase is read from the terminal (`inferred` from the synopsis — the page
does not spell it out for this action). That does not make it a command-less
step under `_schema.md`'s rule, which is about what an executor *cannot make
true*, not about what it can do unattended. `tpm2-enroll` sets the precedent:
`--tpm2-with-pin=yes` prompts too, and it keeps its command.

**The postcondition asserts format parameters, never keyslot inventory.** This
is the trap the step exists to avoid. `state/storage.md`'s `## luks` section
shows two keyslots today — slot 0 from this step, slot 2 written by
`systemd-cryptenroll` during `tpm2-enroll`, with `Iterations: 1000` where slot 0
has none — and [[../backlog/luks-second-keyslot|the second-keyslot backlog item]]
will add a third. A check asserting "exactly one keyslot" would be true for
about an hour of the machine's life and then permanently red, and a permanently
red check is one nobody reads. What luksFormat actually fixes is the container:
the segment cipher, the sector size and the volume key length. Those are read
per keyslot in the dump — every slot reports the same `Cipher key: 512 bits`,
since it is one volume key — so the assertion iterates rather than indexing, and
is slot-count independent by construction.

The parse targets are the labels in `state/storage.md`'s `## luks`, which is
`cryptsetup luksDump` output (*measured*, collector run 2026-08-23):
`Version:`, and under `Data segments:` the indented `cipher:` and `sector:`.

**`--pbkdf argon2id` is set and not asserted, and the gap is the collector's.**
The flag is documented — "The PBKDF can be: *pbkdf2*, *argon2i*, or *argon2id*"
(*measured*, `cryptsetup-luksFormat(8)`) — but `collect.sh` deletes lines
matching `PBKDF|Time cost|Memory|Threads|Salt` from the dump, so `state/` has no
record of the label's spelling or position and nothing here can be grounded in
it. Writing the check anyway would mean asserting against a label seen only
inside a `sed` deletion expression. The lookup that closes it is one
`cryptsetup luksDump` on the machine, or a collector change that keeps the
`PBKDF:` line while still dropping the salt beneath it. Worth doing: argon2id
versus pbkdf2 is the memory-hardness of the passphrase, and nothing in `state/`
currently records which one this container uses.

**Discard is not persisted here, deliberately, and this is where the PRD and the
machine part company.** PRD §1.2 rejects continuous TRIM outright — "Online TRIM
through dm-crypt leaks block-allocation patterns to the SSD controller" — and
the built machine does it anyway, via `rd.luks.options=discard` in the cmdline
(*measured*, `state/boot.md`). What it does *not* do is write the flag into the
header: `--persistent` means "the specified activation flags are persistently
written into metadata and used next time automatically" (*measured*,
`cryptsetup-open(8)`), and the header reports `Flags: (no flags)` (*measured*,
`state/storage.md`). So the grant is per-activation and its sole source is one
unprefixed token on the kernel command line — which is exactly what
[[boot#cmdline-write|`cmdline-write`'s negative postcondition]] protects, and
why that assertion is load-bearing rather than pedantic.

Granting discards at all rejects §1.2, and
[[../decisions/20260823-continuous-trim-both-layers|ADR-0014]] is where that is
argued: the leak `cryptsetup-open(8)` warns about is real and accepted, because
the weekly `fstrim.timer` §1.2 offers instead discloses the same thing through
the same grant — and cannot run without it. Not persisting the flag is the half
of §1.2 this step keeps.

```yaml
id: luks-format
requires: [disk-partition]
precondition: |
  set -euo pipefail
  : "${DISK:?DISK must name the target disk}"
  test -b "${LUKSDEV:?LUKSDEV must name the container partition, e.g. /dev/nvme0n1p2}"
  # Not merely a block device: partition 2 of the disk this chapter partitioned.
  # sda2 on the rescue drive is 466 GB of unformatted space, and luksFormat
  # aimed there would succeed.
  want=$(lsblk -lnpo NAME,PARTN "$DISK" | awk '$2==2 {print $1}')
  [ "$LUKSDEV" = "$want" ] \
    || { echo "LUKSDEV=$LUKSDEV is not partition 2 of $DISK ($want)"; exit 1; }
  # Refuse an existing container: luksFormat regenerates the volume key and,
  # without a header backup, all old data is permanently irretrievable.
  # isLuks' exit status is documented, so it is used rather than parsed.
  if cryptsetup isLuks "$LUKSDEV"; then
    echo "$LUKSDEV is already a LUKS device"; exit 1
  fi
command: |
  set -euo pipefail
  : "${LUKSDEV:?LUKSDEV must name the container partition}"
  # Reads the passphrase from the terminal: no key file argument is given.
  # --sector-size is deliberately omitted (PRD §1.2): this drive offers a
  # 512-byte LBA format only, and forcing 4096 creates a mismatch, not a win.
  cryptsetup luksFormat \
    --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --pbkdf argon2id \
    "$LUKSDEV"
  cryptsetup open "$LUKSDEV" cryptroot
postcondition: |
  set -euo pipefail
  : "${LUKSDEV:?LUKSDEV must name the container partition}"
  fail=0
  note() { echo "  $1"; fail=1; }
  cryptsetup isLuks --type luks2 "$LUKSDEV" || note "not a LUKS2 device"
  dump=$(cryptsetup luksDump "$LUKSDEV")
  # Data segment: the container's own parameters, unaffected by keyslot churn.
  seg_cipher=$(sed -nE 's/^[[:space:]]+cipher:[[:space:]]+(.*)$/\1/p' <<<"$dump")
  [ "$seg_cipher" = "aes-xts-plain64" ] || note "segment cipher is '$seg_cipher'"
  # Volume key length, read across every keyslot rather than indexing slot 0.
  # Slot count changes over the machine's life: tpm2-enroll adds one and the
  # second-passphrase backlog item adds another. Slot *inventory* is not this
  # step's business; the volume key it established is.
  keybits=$(sed -nE 's/^[[:space:]]+Cipher key:[[:space:]]+([0-9]+) bits$/\1/p' <<<"$dump")
  [ -n "$keybits" ] || note "no 'Cipher key:' line in luksDump"
  for b in $keybits; do
    [ "$b" -eq 512 ] || note "a keyslot reports a $b-bit volume key, want 512"
  done
  # The mapping exists and backs the partition this chapter created.
  test -b /dev/mapper/cryptroot || note "/dev/mapper/cryptroot is absent"
  backing=$(cryptsetup status cryptroot | awk '/device:/ {print $2}')
  [ "$backing" = "$LUKSDEV" ] || note "cryptroot backs '$backing', want $LUKSDEV"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## btrfs-mkfs

Separate from `btrfs-subvols-create` because the failure modes are unrelated:
one is "the filesystem is not there", the other is "the filesystem is there and
the subvolume layout is wrong", and a step that halts should say which.

`mkfs.btrfs` refuses a device that already holds a filesystem — `-f/--force`
exists to "Forcibly overwrite the block devices when an existing filesystem is
detected. By default, mkfs.btrfs will utilize libblkid to check for any known
filesystem on the devices" (*measured*, `mkfs.btrfs(8)`, read 2026-08-23). That
default is a safety property this spec accepts rather than overrides: `-f` is
not passed, so a re-run against a populated container fails loudly instead of
erasing it. The `-L` label is set for legibility only; nothing in the build
resolves the filesystem by label, and asserting it would be asserting a
decoration.

```yaml
id: btrfs-mkfs
requires: [luks-format]
precondition: |
  test -b /dev/mapper/cryptroot
command: |
  set -euo pipefail
  # No -f: mkfs.btrfs refuses a device with a known filesystem on it, and that
  # refusal is the guard against a re-run erasing a populated container.
  mkfs.btrfs -L system /dev/mapper/cryptroot
postcondition: |
  set -euo pipefail
  t=$(blkid -s TYPE -o value /dev/mapper/cryptroot)
  [ "$t" = btrfs ] && echo RESULT=PASS || echo "RESULT=FAIL (TYPE=$t)"
```

---

## btrfs-subvols-create

Six subvolumes, matching `state/storage.md`'s `## subvolumes` — `@`, `@home`,
`@swap`, `@snapshots`, `@log`, `@cache`. The four nested ones that section also
lists (`@/var/lib/portables`, `@/var/lib/machines`, `@home/.snapshots`,
`@home/luiswg/.local/share/containers`) are created later and by other things:
systemd makes the first two, snapper the third, and
[[containers#containers-subvol|`spec/containers`]]
the fourth. Asserting them here would be asserting other chapters' work.

The step mounts the top level at `/mnt` and leaves it mounted. **The unmount PRD
§1.3 puts at the end of this section belongs to `btrfs-subvol-mount` instead**,
and moving it is not tidying — `btrfs-default-subvol` needs an accessible path
to `@`, which only exists while the top level is mounted. Written the PRD's way,
the two steps cannot both run.

`subvolume list` prints "ID *ID* gen *generation* top level *parent_ID* path
*path*" (*measured*, `btrfs-subvolume(8)`, read 2026-08-23), and the collector
strips the `gen` field because it churns. The postcondition compares whole path
fields rather than grepping: `grep -q ' path @'` matches `@home` as happily as
`@`, so six greps would pass with only `@home` present. Same defect class as the
`\<` awk escape recorded in
[[../sessions/20260823-build-spec-schema|the first session log]] — a test that
passes while checking nothing.

```yaml
id: btrfs-subvols-create
requires: [btrfs-mkfs]
precondition: |
  set -euo pipefail
  # The top level, not a subvolume: subvolume creation needs the toplevel
  # mounted, and so does btrfs-default-subvol immediately after.
  test -d /mnt
  ! mountpoint -q /mnt
command: |
  set -euo pipefail
  mount /dev/mapper/cryptroot /mnt
  for s in @ @home @swap @snapshots @log @cache; do
    btrfs subvolume create "/mnt/$s"
  done
  # Deliberately no umount here. btrfs-default-subvol needs /mnt/@ to be an
  # accessible path; the unmount is the first action of btrfs-subvol-mount.
postcondition: |
  set -euo pipefail
  # Whole-field comparison, not grep: ' path @' matches @home too, so a grep
  # per name would pass with only @home present.
  paths=$(btrfs subvolume list /mnt | sed -E 's/.* path //')
  fail=0
  for s in @ @home @swap @snapshots @log @cache; do
    printf '%s\n' "$paths" | grep -qx "$s" || { echo "  missing subvolume: $s"; fail=1; }
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## btrfs-default-subvol

*Promoted from `_worked-examples.md` 2026-08-23, unchanged. That file was
deleted 2026-08-24 when its last two examples left for `spec/containers`.*

This one has teeth on the running machine. The measured cmdline carries **no**
`rootflags=subvol=`, so the btrfs default subvolume is the sole selector for
`/` — and it is baked into a signed UKI, which cannot be edited from the boot
menu. `btrfs subvolume get-default /` currently reports
`ID 517 top level 259 path @snapshots/184/snapshot` (*measured*, collector run
2026-08-23), consistent with `findmnt`. The system is running from a rollback
snapshot, exactly as intended.

PRD §3.2 still specifies `rootflags=subvol=@`. Carrying that forward would take
the default subvolume out of the boot path and make `snapper rollback` a no-op
that reports success — which is the whole hazard. Hence the negative
precondition on the invariant below, and the matching negative postcondition in
[[boot#cmdline-write|`cmdline-write`]].

`btrfs-subvolume(8)` on `set-default`: "This will hide the top-level subvolume …
**Takes action on next mount.**" (*measured*, read 2026-08-22.) That sentence is
the reason this is a silent failure class: nothing observable changes until a
reboot.

This is two steps, not one. Writing it as one conflated a build action with a
standing invariant and forced a fragile derivation that a simpler split removes.
`set-default [<subvolume>|<id> <path>]` takes an accessible path — so at build
time, while the top level is still mounted and before the subvolume mounts hide
it, `@` is directly nameable. `@` is a name this spec chooses, not a per-build
value, so it is a legitimate constant.

```yaml
id: btrfs-default-subvol
requires: [btrfs-subvols-create]
precondition: |
  # The top level must still be mounted: set-default needs an accessible path
  # to @, which the subvolume mounts in btrfs-subvol-mount will hide.
  test -d /mnt/@
command: |
  btrfs subvolume set-default /mnt/@
postcondition: |
  [ "$(btrfs subvolume get-default /mnt | sed -E 's/.* path //')" = "@" ] \
    && echo RESULT=PASS || echo RESULT=FAIL
```

---

## btrfs-subvol-mount

**The edge to `btrfs-default-subvol` is a lifetime constraint, not a data
dependency.** Mounting does not consume anything `set-default` produces; it
destroys the condition `set-default` requires, by unmounting the top level whose
`/mnt/@` path is the only accessible name for the target. Without the edge a
topological sort can legally order the mount first, and the build then halts at
a precondition that is doing its job. Edges in this spec order executions, and
"A must happen while B's world still exists" is as real an ordering as "A
consumes B's output".

PRD §1.3 warns that skipping the unmount makes "each subvolume appear twice in
`/proc/mounts`". That is the assertion this postcondition can make precisely:
after the step there must be exactly **six** btrfs mounts under `/mnt` and the
top level must be gone. If the top level is still mounted the count is seven and
`/mnt` reports `subvol=/` — so one count and one option check cover both the
duplicate hazard and its cause.

**Only `/mnt`'s compression is asserted, and the others are decoration.**
`btrfs(5)`: "Most mount options apply to the whole filesystem and only options in
the first mounted subvolume will take effect… you can't set per-subvolume
nodatacow, nodatasum, or compress using mount options" (*measured*, read
2026-08-23). So `compress=zstd:1` on the five later mounts is inert, and
`@swap`'s omission of it is equally inert — the swapfile's CoW exemption comes
from the file attribute `btrfs filesystem mkswapfile` sets, not from a mount
option. The measured `## mounts` output does show `compress=zstd:1` on `/swap`,
but that is *not* independent evidence of the fs-wide behaviour: `## fstab`
carries the option on that line explicitly, so the reading is circular. The man
page is the source; the live output is consistent with it and proves nothing on
its own.

The live options also carry `ssd`, `space_cache=v2` and `discard=async` that no
mount command here passes. The first two are documented defaults — "By default,
BTRFS will enable or disable SSD optimizations depending on status of a device
with respect to rotational or non-rotational type" and "(default:
space_cache=v2)" — so PRD §1.4 was right to omit them and `## fstab`'s explicit
copies are redundant. The third is not: `discard=async` is "(default: async when
devices support it since 6.2)" and this kernel is 7.1.7, so §1.4's *deliberate
omission* of it stopped meaning "off" without anyone changing anything —
`nodiscard` would now be required to get what §1.4 asked for. Nothing in this
chapter writes it; [[../decisions/20260823-continuous-trim-both-layers|ADR-0014]]
is why.

`findmnt -M` is used rather than `-T`. `--mountpoint <dir>` requires the path to
*be* a mountpoint, where `--target` resolves to the filesystem containing it —
so with the mount missing, `-T /mnt/home` cheerfully answers about `/mnt` and the
check passes. `_schema.md` asks that a check not fail the same way as the thing
it detects; this is the mirrored case, a check that would *pass* the same way.

### The mount count aborted on zero, and counted a prefix

One line carried both defects. `n=$(findmnt -rno TARGET -t btrfs | grep -c
'^/mnt')` **aborts on the zero case twice over**: `findmnt` exits 1 when it
finds nothing, and `grep -c` exits 1 when it counts nothing, so under `set -euo
pipefail` "no btrfs mounts at all" ended the postcondition with no `RESULT=`
line — while the five other checks in the block had already produced their
diagnostics and were discarded with it.

`findmnt(8)` is what makes the repair cheap rather than a judgment call: "The
exit value is 0 if there is something to display, or 1 on any error (for example
if no filesystem is found based on the user's filter specification…)"
(*measured*, read 2026-08-24). The tool returns the same 1 for "nothing
matched" and for "something went wrong", so `|| mounts=` discards no distinction
`findmnt` was ever willing to make. That is the opposite of the reasoning in
[[boot#uki-sign|`uki-sign`]], where an exit status is parsed around *because* it
carries information; here it is collapsed because, documentedly, it does not.

The second defect is quieter. `'^/mnt'` is a prefix, not a path: a btrfs mount
at `/mnt2` counted toward the six. Nothing in this build creates one, so the
consequence is narrow — but it is the `' path @'`-matches-`@home` defect from
[[#btrfs-subvols-create|`btrfs-subvols-create`]] four steps up, in a chapter
that already names that class. `$1 == "/mnt" || index($1, "/mnt/") == 1` is the
boundary, written without a regex because `awk` here is `mawk` on one side and
`gawk` on the other.

**The mutation set**, same method: the block extracted from this document,
`findmnt` stubbed, mount options taken from `state/storage.md`'s `## mounts`
with the `/mnt` prefix restored.

| # | Mutation | Fixed block | Unfixed block |
|---|---|---|---|
| 1 | baseline — six mounts, correct subvolumes | PASS | PASS |
| 2 | **nothing mounted at all** | FAIL, six `not a mountpoint` + `found 0` | **no `RESULT=` line, exit 1** |
| 3 | top level never unmounted — 7 mounts, `/mnt` at `subvol=/` | FAIL, `/mnt has subvol=/, want subvol=/@` + `found 7` | same |
| 4 | `/mnt/home` not mounted | FAIL, `not a mountpoint: /mnt/home` + `found 5` | same |
| 5 | `/mnt/var/log` mounted at `subvol=/@cache` | FAIL, names both | same |
| 6 | `/mnt` without `compress=zstd:1` | FAIL, `/mnt lacks compress=zstd:1` | same |
| 7 | **a stray btrfs mount at `/mnt2`** | PASS | **FAIL, `found 7` — a false alarm** |
| 8 | `findmnt` errors rather than finding nothing | FAIL, `found 0` | no `RESULT=` line, exit 1 |

Cases 2 and 8 are deliberately indistinguishable in the output, because
`findmnt` makes them indistinguishable in its exit status. Case 7 is the only
row where the two columns disagree other than on the abort, and it disagrees in
the direction that matters: the unfixed block reports a failure that is not
there.

*One correction from running the set:* case 3 was first recorded as gaining a
diagnostic from the fix. It does not — both columns produce both lines. The
difference was in a `grep` filtering the harness output, not in the block, and
it is noted because it is the same shape as the reading the chapter already
retracted about `compress=zstd:1` on `/swap`: a comparison that looked like
evidence and was an artifact of how it was read.

```yaml
id: btrfs-subvol-mount
requires: [btrfs-default-subvol]
precondition: |
  set -euo pipefail
  # The top level is still mounted here — this step is what unmounts it.
  mountpoint -q /mnt
  test -d /mnt/@
command: |
  set -euo pipefail
  # Unmount the top level first, or every subvolume appears twice in
  # /proc/mounts (PRD §1.3). This is the unmount §1.3 puts one step earlier;
  # btrfs-default-subvol needs /mnt/@ to survive until now.
  umount /mnt
  mount -o rw,noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
  mkdir -p /mnt/boot /mnt/home /mnt/swap /mnt/.snapshots /mnt/var/log /mnt/var/cache
  # compress= on these is inert (btrfs(5): options are filesystem-wide and only
  # the first mounted subvolume's take effect). Passed for legibility.
  mount -o rw,noatime,compress=zstd:1,subvol=@home      /dev/mapper/cryptroot /mnt/home
  mount -o rw,noatime,subvol=@swap                      /dev/mapper/cryptroot /mnt/swap
  mount -o rw,noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
  mount -o rw,noatime,compress=zstd:1,subvol=@log       /dev/mapper/cryptroot /mnt/var/log
  mount -o rw,noatime,compress=zstd:1,subvol=@cache     /dev/mapper/cryptroot /mnt/var/cache
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # -M, not -T: --target resolves to the filesystem *containing* the path, so a
  # missing /mnt/home mount would be answered for by /mnt and pass.
  check() { # <mountpoint> <expected subvol>
    local opts
    opts=$(findmnt -no OPTIONS -M "$1" 2>/dev/null) \
      || { note "not a mountpoint: $1"; return; }
    case ",$opts," in
      *",subvol=$2,"*) ;;
      *) note "$1 has $(grep -o 'subvol=[^,]*' <<<"$opts"), want subvol=$2" ;;
    esac
  }
  check /mnt            /@
  check /mnt/home       /@home
  check /mnt/swap       /@swap
  check /mnt/.snapshots /@snapshots
  check /mnt/var/log    /@log
  check /mnt/var/cache  /@cache
  # Exactly six, or the top level was never unmounted and every subvolume is
  # mounted twice (PRD §1.3). Seven-with-/mnt-at-subvol=/ is that failure.
  # Two aborts were in one line here. findmnt exits 1 when it finds nothing and
  # grep -c exits 1 when it counts nothing, so under set -e the zero case —
  # nothing mounted at all — ended the postcondition with no RESULT= line.
  # findmnt(8) returns that same 1 "on any error", so `|| mounts=` discards no
  # distinction the tool was willing to make; awk then makes zero a number.
  mounts=$(findmnt -rno TARGET -t btrfs) || mounts=
  # Anchored on a path boundary rather than a prefix: '^/mnt' also counted a
  # btrfs mount at /mnt2.
  n=$(printf '%s\n' "$mounts" \
        | awk '$1 == "/mnt" || index($1, "/mnt/") == 1 { n++ } END { print n + 0 }')
  [ "$n" -eq 6 ] || note "expected 6 btrfs mounts under /mnt, found $n"
  # compress is filesystem-wide, so it is asserted once, on the first mount.
  findmnt -no OPTIONS -M /mnt | grep -q 'compress=zstd:1' \
    || note "/mnt lacks compress=zstd:1"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## esp-mount

`mkfs.fat` on the ESP is destructive to whatever was there, and on this machine
something was: `state/boot.md` lists a `Windows Boot Manager` EFI variable
pointing at `EFI/Microsoft/Boot/bootmgfw.efi` while the ESP itself contains only
the two systemd-boot binaries. [[boot#uki-sign|`uki-sign`]] reads that as a
stale NVRAM entry (`inferred`); this step is the plausible reason the file it
names is gone (`inferred`). Nothing here needs the inference to be right — it is
recorded so the two halves are not re-derived separately.

**The mask is the point of this step.** `fmask`/`dmask` "Set the umask applied to
regular files only" / "to directories only", defaulting to "the umask of the
current process" (*measured*, `mount(8)`, read 2026-08-23). Left at the default
the ESP is world-readable, including the random seed systemd-boot writes, which
produces a warning on every `bootctl` run. `0077` is what
`state/storage.md`'s `## mounts` shows on `/boot` today, and it reaches the
installed system only because `genfstab` copies the options it finds — so
getting it wrong here is not a mount-time inconvenience, it is written into
`/etc/fstab` and persists.

PRD §1.4 adds that vfat applies these at mount time only, so a later correction
needs a full umount/mount cycle rather than a remount. That claim is
**`inherited-unverified`**: `mount(8)` documents the options but says nothing
about their behaviour under remount. The lookup that would close it is the
`vfat` section of the kernel's own filesystem documentation. It does not affect
this step, which mounts fresh; it affects anyone repairing the mask later.

`-F 32` is explicit because "If nothing is specified, mkfs.fat will
automatically select between 12, 16 and 32 bit, whatever fits better for the
filesystem size" (*measured*, `mkfs.fat(8)`) — and a 1 GiB partition is
comfortably within the range where FAT16 "fits better", while UEFI wants FAT32.

```yaml
id: esp-mount
requires: [btrfs-subvol-mount]
precondition: |
  set -euo pipefail
  : "${DISK:?DISK must name the target disk}"
  test -b "${ESPDEV:?ESPDEV must name the EFI system partition, e.g. /dev/nvme0n1p1}"
  # Partition 1 of the disk this chapter partitioned, not merely a vfat-capable
  # block device: mkfs.fat here destroys whatever it is pointed at.
  want=$(lsblk -lnpo NAME,PARTN "$DISK" | awk '$2==1 {print $1}')
  [ "$ESPDEV" = "$want" ] \
    || { echo "ESPDEV=$ESPDEV is not partition 1 of $DISK ($want)"; exit 1; }
  # The mount point is a directory on @, created by btrfs-subvol-mount.
  test -d /mnt/boot
  ! mountpoint -q /mnt/boot
command: |
  set -euo pipefail
  : "${ESPDEV:?ESPDEV must name the EFI system partition}"
  # -F 32 explicitly: at 1 GiB, mkfs.fat's automatic choice would be FAT16.
  mkfs.fat -F 32 -n EFI "$ESPDEV"
  # The masks apply at mount time and genfstab copies them into /etc/fstab.
  # Left at the default the systemd-boot random seed is world-readable.
  mount -o fmask=0077,dmask=0077 "$ESPDEV" /mnt/boot
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  line=$(findmnt -no FSTYPE,OPTIONS -M /mnt/boot 2>/dev/null) \
    || { echo "RESULT=FAIL (/mnt/boot is not a mountpoint)"; exit 1; }
  fstype=${line%% *}
  opts=${line#* }
  [ "$fstype" = vfat ] || note "FSTYPE is $fstype, want vfat"
  for m in fmask=0077 dmask=0077; do
    case ",$opts," in *",$m,"*) ;; *) note "missing mount option: $m" ;; esac
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## swapfile-create

`btrfs filesystem mkswapfile` creates "a new file that's suitable and formatted
as a swapfile", and the page states the requirement rather than enumerating what
the command sets: "A swapfile must be created in a specific way: NOCOW and
preallocated. Subvolume containing a swapfile cannot be snapshotted and blocks
of an activated swapfile cannot be balanced." (*measured*,
`btrfs-filesystem(8)`, read 2026-08-23.) That second sentence is the whole
reason `@swap` is its own subvolume: with the swapfile on `@`, the root snapshot
this machine boots from could not exist.

Because the page describes the requirement and not the implementation, the
postcondition does not go looking for the NOCOW attribute. It asks the tool that
checks: per `btrfs-inspect-internal(8)`, `map-swapfile` "will also verify that
the *file* is suitable as a swapfile" (*measured*, quoted in
[[boot#cmdline-write|`cmdline-write`]]), so a successful run is the assertion.
That is the same reasoning that made `map-swapfile` rather than `swapon` the
gate one chapter over.

**The size is asserted against live RAM, not against 36 GiB.** The command asks
for `--size 36g` because 36 is a number a human chose for a 32 GB machine; the
requirement is "swap ≥ RAM as a floor, not a target" (PRD §1.6). `MemTotal`
resolves at run time, so the check keeps working after a RAM upgrade — and,
usefully, *stops* working the moment RAM exceeds the swapfile, which is one of
the two ways this file legitimately gets recreated.

**`swapon --show` is the right gate here and the wrong one in the next chapter.**
Identical command, opposite verdict, decided entirely by execution context: in
the installer the kernel records the path this step passes it, `/mnt/swap/swapfile`,
and that is the file the step means. Inside `arch-chroot` the same command
reports that same recorded string while the file is reachable as
`/swap/swapfile`, which is why `cmdline-write`'s precondition does not consult
it. The bug that taught this was in prose for a long time before a precondition
had to name its namespace.

The `swapon` is not decoration: `genfstab` in `spec/base` detects the active swap
and emits an fstab line for it, which §2.2 then edits to add `pri=10`. Skip the
activation and the line is absent, and the machine boots with no disk swap and
no error.

```yaml
id: swapfile-create
requires: [btrfs-subvol-mount]
precondition: |
  set -euo pipefail
  # The @swap subvolume, not a directory on @: a subvolume holding an active
  # swapfile cannot be snapshotted, which is why it is mounted separately.
  findmnt -no OPTIONS -M /mnt/swap | grep -q 'subvol=/@swap'
  test ! -e /mnt/swap/swapfile
command: |
  set -euo pipefail
  # 36 GiB for 32 GB RAM: hibernation needs swap >= RAM as a floor, with margin
  # for session state and re-expanded zram pages. --uuid clear leaves no
  # identifier in the file.
  btrfs filesystem mkswapfile --size 36g --uuid clear /mnt/swap/swapfile
  swapon /mnt/swap/swapfile
postcondition: |
  set -euo pipefail
  f=/mnt/swap/swapfile
  fail=0
  note() { echo "  $1"; fail=1; }
  test -f "$f" || { echo "RESULT=FAIL ($f absent)"; exit 1; }
  # map-swapfile "will also verify that the file is suitable as a swapfile", so
  # a successful run is the NOCOW-and-preallocated assertion.
  btrfs inspect-internal map-swapfile -r "$f" >/dev/null \
    || note "$f is not usable as a swapfile"
  # Floor resolved against live RAM, not against the 36 GiB the command asks
  # for: a RAM upgrade must fail this check, not silently outgrow it.
  ram=$(awk '/^MemTotal:/ {print $2 * 1024}' /proc/meminfo)
  size=$(stat -c %s "$f")
  [ "$size" -ge "$ram" ] || note "swapfile is $size bytes, RAM is $ram"
  # Correct here and wrong inside the chroot: in the installer the path the
  # kernel recorded is the path this step means.
  swapon --show=NAME --noheadings | grep -qx "$f" || note "$f is not active"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## btrfs-default-matches-root

*Promoted from `_worked-examples.md` 2026-08-23, unchanged. That file was
deleted 2026-08-24 when its last two examples left for `spec/containers`.*

The invariant is what catches the hazard on a running system, and it has no
command because nothing can make it true — a rollback moves both readings
together, and if they have diverged, the next boot lands somewhere other than
what is currently running. Re-checkable forever, and correct after any number of
rollbacks, because it compares two live readings rather than a stored id.

It also catches a defect this chapter cannot assert directly. `genfstab` copies
the mount options the kernel reports, which always include `subvol=`, so the
root line it writes pins the subvolume that was mounted at install time; PRD
§2.2 strips it, and `state/storage.md`'s `## fstab` shows the root line today
carrying neither `subvol=` nor `subvolid=`. That edit belongs to whichever step
in `spec/base` writes the fstab. If it is ever missed, the mounted root stops
following the default subvolume and this invariant is what says so.

```yaml
id: btrfs-default-matches-root
requires: [first-boot]
precondition: |
  # If rootflags=subvol= is present the default subvolume does not select root,
  # and this invariant is asserting something that no longer governs.
  ! grep -q 'rootflags=subvol=' /etc/kernel/cmdline
postcondition: |
  set -euo pipefail
  src=$(findmnt -no SOURCE /)
  if ! grep -q '\[' <<<"$src"; then
    echo "RESULT=FAIL (root SOURCE carries no subvolume: $src)"; exit 1
  fi
  mounted=$(sed -E 's/.*\[\/?([^]]*)\]/\1/' <<<"$src")
  default_path=$(btrfs subvolume get-default / | sed -E 's/.* path //')
  [ "$default_path" = "$mounted" ] && echo RESULT=PASS \
    || echo "RESULT=FAIL (default=$default_path mounted=$mounted)"
```

---

## resume-offset-matches-swapfile

**The edge is not enough, and the reason generalises.** `cmdline-write` requires
`swapfile-create` because the resume offset is only valid for the swapfile as
allocated. That edge orders one execution. It says nothing about the machine six
months later, where recreating the swapfile — to resize it after a RAM upgrade,
or because a restore put a different file at that path — invalidates an offset
baked into four signed UKIs, and nothing reports it. Hibernation still appears to
work: the image is written, the machine powers off, and the failure surfaces one
boot later as a cold start with the session gone.

That is the same shape as `btrfs-default-subvol`, and the two cases together
give the test. A standing invariant is warranted when a property is
**established once at build time**, **invalidatable afterwards by a legitimate
action**, and **silent when it breaks**. Both cases satisfy all three; a build
step that fails loudly on its own postcondition satisfies only the first. PRD
§10.4 already knew this one — "Re-verify the offset any time the swapfile is
recreated" — which is a prose instruction addressed to whoever remembers to read
it, and is the form of check this spec exists to replace.

**What it compares matters more than that it exists.** There are three readings
of the offset and they disagree in exactly the cases that matter:

| Scenario | `/etc/kernel/cmdline` | signed default entry | `/proc/cmdline` |
|---|---|---|---|
| Swapfile recreated, nothing else | FAIL | FAIL | FAIL |
| Recreated, `cmdline-write` re-run, no `mkinitcpio -P` | **PASS** | FAIL | FAIL |
| Fully repaired, not yet rebooted | PASS | PASS | **FAIL** |

Only the middle column is right in all three. `/etc/kernel/cmdline` is intent
and passes while the UKIs still carry the old value; `/proc/cmdline` is what
booted, and hibernation is resumed by the *next* kernel, not the running one, so
it raises a false alarm on a machine that has just been correctly repaired. The
invariant therefore reads `bootctl status`, whose `options:` line reports the
default entry's embedded command line (*measured*, `state/boot.md` `## bootctl`),
and which needs `sudo` because `/boot` is `dmask=0077`.

*Known limit, same shape as `uki-generate`'s:* `bootctl status` describes the
default entry only, so a stale offset in one of the three non-default UKIs — the
fallback entries, which are what you boot when the default fails — is not
covered. The collector covers it, printing every UKI's `.cmdline` under
`## cmdline (signed, per UKI)`; all four hash identically today. Closing the gap
in the spec means `bootctl list --json=`, which `bootctl(1)` documents as
existing without naming its fields; that is the lookup, and it was not run here.

The `resume=` device is checked alongside the offset because an offset is
meaningless without the device it is relative to —
`btrfs-inspect-internal(8)`: "The hibernation expects offset relative to the
physical block device."

### Two more of the same family, in the step that named the hazard

This block was not on the list of things to fix. It was tested because the other
two were, and it carried two instances of the same defect — which is the
argument for testing a block you have no reason to suspect.

**The `options:` line count aborted on zero**, `n=$(printf '%s\n' "$opts" |
grep -c .)`, directly under the comment that says "Zero and many are distinct
diagnostics: an empty parse means the label moved, which must not read the same
as a mismatched offset." Zero read as *nothing at all* — no verdict, no
diagnostic, and no way to tell an executor's silence from a check that had not
run. The prose was right about what the check was for and the check could not do
it.

**The offset extraction aborted whenever there was no offset to extract.**
`signed_offset=$(grep -oE 'resume_offset=[0-9]+' <<<"$opts" | cut -d= -f2)` —
`grep -oE` exits 1 on no match, and `pipefail` carries that status out past
`cut`, which exits 0. So a signed entry with no `resume_offset=` ended the
postcondition three lines *above* `[ -n "$signed_offset" ] || { echo "  no
resume_offset= in the signed entry"; … }`. **That diagnostic was unreachable
from the day it was written.** Reading the block does not show this; running it
against an entry missing the token does.

These two, the two above and one in [[boot#cmdline-write|`spec/boot`]] are one
shape with four tools in it — `grep -c`, `grep -oE` inside a pipe under
`pipefail`, `blkid`, and `findmnt`'s find-nothing exit. The general form is the
**seventh convention** in [[_schema#A postcondition must reach its verdict line on every input it is meant to judge|`_schema.md`]],
where it belongs: it is a rule about how checks get written, not a fact about
this chapter.

**The mutation set**, with `sudo`, `bootctl`, `btrfs` and `findmnt` stubbed and
the `options:` fixture shaped after `state/boot.md`'s `## bootctl`:

| # | Mutation | Fixed block | Unfixed block |
|---|---|---|---|
| 1 | baseline — signed offset and device match live | PASS | PASS |
| 2 | **no `options:` line — the label moved** | `RESULT=FAIL (bootctl status yielded 0 options: lines)` | **no `RESULT=` line, exit 1** |
| 3 | two `options:` lines | `RESULT=FAIL (… yielded 2 options: lines)` | same |
| 4 | live offset differs from signed | FAIL, `offset: signed=… live=…` | same |
| 5 | live device differs from signed | FAIL, `device: signed=… live=…` | same |
| 6 | **signed entry has `resume=` but no `resume_offset=`** | FAIL, `no resume_offset= in the signed entry` | **no `RESULT=` line, exit 1** |
| 7 | signed entry has neither token | FAIL, both diagnostics | **no `RESULT=` line, exit 1** |

Rows 2, 6 and 7 are the three ways this invariant can find something wrong with
the *signed entry itself* rather than with the offset — which is to say, the
three cases where it is most useful — and it reported none of them.

```yaml
id: resume-offset-matches-swapfile
requires: [first-boot]
precondition: |
  # If the cmdline does not ask for hibernation there is no offset to be stale,
  # and this invariant is asserting something that no longer governs.
  grep -q 'resume_offset=' /etc/kernel/cmdline
postcondition: |
  set -euo pipefail
  # What will boot next, not what booted: hibernation is resumed by the next
  # kernel. /boot is dmask=0077, hence sudo.
  opts=$(sudo bootctl status | sed -nE 's/^[[:space:]]*options:[[:space:]]+(.*)$/\1/p')
  # awk, not `grep -c`, for the reason the next comment gives: zero is the
  # case this count exists to separate, and grep -c exits 1 on zero, which
  # under set -e ended the postcondition with no RESULT= line at all.
  n=$(printf '%s\n' "$opts" | awk 'length($0) > 0 { n++ } END { print n + 0 }')
  # Zero and many are distinct diagnostics: an empty parse means the label
  # moved, which must not read the same as a mismatched offset.
  [ "$n" -eq 1 ] || { echo "RESULT=FAIL (bootctl status yielded $n options: lines)"; exit 1; }
  # Same family as the count above, one tool along: grep -oE exits 1 when it
  # matches nothing, and pipefail carries that out of the pipe past cut. The
  # `no resume_offset=` diagnostic three lines down was unreachable until this
  # was written, because the assignment that feeds it aborted first.
  signed_offset=$(grep -oE 'resume_offset=[0-9]+' <<<"$opts" | cut -d= -f2) || signed_offset=
  signed_dev=$(grep -oE 'resume=[^ ]+' <<<"$opts" | cut -d= -f2) || signed_dev=
  live_offset=$(sudo btrfs inspect-internal map-swapfile -r /swap/swapfile)
  # findmnt reports SOURCE as /dev/mapper/cryptroot[/@swap]; the offset is
  # relative to the block device, so compare the part before the bracket.
  live_dev=$(findmnt -no SOURCE -M /swap | sed -E 's/\[.*//')
  fail=0
  [ -n "$signed_offset" ] || { echo "  no resume_offset= in the signed entry"; fail=1; }
  [ "$signed_offset" = "$live_offset" ] \
    || { echo "  offset: signed=$signed_offset live=$live_offset"; fail=1; }
  [ "$signed_dev" = "$live_dev" ] \
    || { echo "  device: signed=$signed_dev live=$live_dev"; fail=1; }
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## Open against this chapter

- **`--pbkdf argon2id` is set and unasserted.** `collect.sh` strips the `PBKDF:`
  line from the dump, so `state/` cannot ground the parse. One `cryptsetup
  luksDump` on the machine closes it, or a collector change that keeps the label
  and drops only the salt beneath it.
- **The partition type check is ungrounded in three places, not one.** The type
  *strings* were already marked `inferred`. Re-testing added two more: `sgdisk(8)`
  gives no example of `-i` output, so the **`Partition GUID code:` label** and
  the **parenthesised format** the `sed` parses are unsourced too. The GUIDs
  themselves are sourced and cannot be committed — the pre-commit hook blocks
  UUID-shaped strings, `CLAUDE.md` forbids `--no-verify`, and a standardised
  type GUID is not an identifier so it has no redaction token. **The redaction
  rule has reached a case it was not designed for.** Either the hook needs an
  allow-list for standard type GUIDs, or the collector should record the
  partition types — the second is cheaper, matches how the rest of this chapter
  grounds its parses, and now closes three gaps rather than one.
- **`cryptsetup status`'s `device:` parse now appears in four steps across two
  chapters.** `spec/boot` already flags it as measured from observation rather
  than documentation and suggests the collector record the backing device.
  `luks-format` is the fourth call site, and the argument is stronger for it.
- **The root fstab line's missing `subvol=` is asserted only indirectly.** The
  strip belongs to `spec/base`, and `btrfs-default-matches-root` is what fails
  if it is skipped. Once that chapter exists, decide whether the edit deserves
  its own postcondition or whether the invariant is the right place for it to
  surface.
- **`resume-offset-matches-swapfile` covers the default boot entry only.** The
  fallback UKIs are covered by the collector, not by the spec. `bootctl list
  --json=` would close it; its field names are undocumented in `bootctl(1)` and
  were not run here.
- **Three of the ten postconditions now have written-down mutation sets; seven
  do not.** `disk-partition`, `btrfs-subvol-mount` and
  `resume-offset-matches-swapfile` were re-tested 2026-08-24 against 24 cases,
  recorded above. The other seven were not, and **`luks-format` is the one that
  matters**: it is the other step whose failure mode is destroyed data, and its
  postcondition parses `luksDump` across a keyslot count that changes over the
  machine's life.
- **Nothing in this chapter has been executed.** Two of the ten steps arrive
  from worked examples that were argued through in a prior round; the other
  eight rest on man pages and on `state/`. That is the standing limitation
  recorded in the second session log, and it is sharper here than in `spec/boot`
  because `disk-partition` and `luks-format` are the two steps in this vault
  whose failure mode is destroyed data. **A mutation set narrows that and does
  not close it:** `lsblk`, `findmnt`, `sgdisk`, `bootctl` and `btrfs` were all
  stubbed, so what was tested is shell logic against output shapes taken from
  `state/` and from man pages.
- **The harness is not the target's toolchain, in two ways, and one of them has
  already cost a bug.** `awk` in the devbox is mawk 1.3.4 where the machine runs
  gawk — the divergence behind the `\<` escape that matched nothing and let a
  check pass while testing nothing. `grep` there is **ugrep 7.5.0, not GNU
  grep** (*measured*, 2026-08-24), which is new information and not previously
  recorded anywhere. The `grep -c` behaviour these fixes turn on is
  POSIX-specified and holds for both; anything resting on a GNU extension would
  test clean in the container and fail on the machine.
