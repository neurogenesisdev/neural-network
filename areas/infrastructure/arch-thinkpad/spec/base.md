---
title: "Base installation — pacstrap, fstab, the user, and the boundary"
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-24
updated: 2026-08-24
tags: [spec, base, pacstrap, fstab, users, sudo, first-boot]
---

# Base installation

Four steps, from a mounted subvolume tree to a machine running its own kernel.
Read [[_schema|_schema.md]] first.

Replaces PRD §2.1–§2.3 and §3.5, and absorbs §3.9 and the boundary §4 opens
with. It diverges from §2.2 in one place, argued in `fstab-generate`: the PRD
strips `subvolid=` from the generated fstab and leaves `subvol=`, and its own
reason for stripping the first applies unchanged to the second.

**The chapter defines `first-boot`, so it is where the execution-context
boundary stops being a convention and becomes an assertion.** Four steps in
[[boot|spec/boot]] and [[disk|spec/disk]] already depend on that node.

## Where this chapter sits

```
esp-mount ─→ base-install ─┬─→ fstab-generate ──┐
              (spec/disk)  │                    │
                           ├─→ user-create ─────┤
                           │                    │
                           ├─→ cmdline-write    │   uki-generate (spec/boot)
                           └─→ mkinitcpio-config│        │
                                (spec/boot)     ▼        ▼
                                            ┌── first-boot ──┐
   bootloader-install ──────────────────────┘    │  │        │
        (spec/boot)                              │  ▼        ▼
                                                 │ firmware-setup-mode   btrfs-default-matches-root
                                                 │   (spec/boot)         resume-offset-matches-swapfile
                                                 │                             (spec/disk)
                                                 ▼
                                          collector-run ─┬─→ collect-tool-versions
                                                         └─→ collect-harness-config
```

`base-install` and `fstab-generate` run in the **live installer**, on
`/mnt`-prefixed paths, alongside all ten steps of `spec/disk`. `user-create`
runs under `arch-chroot`. Everything downstream of `first-boot` runs on the
booted system.

`bootloader-install` was referenced here before it existed anywhere. It is now
defined in [[boot#bootloader-install|spec/boot]], along with
`bootloader-nvram-entry` — the second half of the same PRD command, on the far
side of `first-boot`. See the last section for how it was found.

**The three collector steps were added 2026-08-26 and are a different kind of
node.** They do not build the machine; they establish that the vault's record of
it is complete. `collector-run` is the only one with a `command:` — the other
two assert one section each of its output, and giving them their own `command:`
would have been two DAG nodes for one action, entitling a topological sort to
run the collector twice. [[devenv#toolchain-parity|`toolchain-parity`]] is
related in subject and sits on a different chain entirely, in
[[devenv|spec/devenv]], because it reads the host live and never opens `state/`.

## What the boundary actually is

[[areas/infrastructure/arch-thinkpad/spec/_index#The DAG is not a reading order|spec/_index]] derives a step's execution
context from the DAG — chroot-side if `first-boot` is absent from its transitive
`requires` closure, booted-system if present — and notes that the corroborating
tell, `sudo`, **computes privilege rather than namespace**. `spec/disk` showed
the cost of the gap from one side: its ten steps run in the installer, which the
privilege tell cannot distinguish from chroot-side, and the difference that
bites is that `/mnt/swap/swapfile` and `/swap/swapfile` are one file.

This is the chapter where the namespace actually changes, so it is where that
can be said concretely rather than as a caveat. Four namespaces are in play, not
two:

| | `systemd-detect-virt -r` | `/proc/cmdline` carries `rd.luks.name=` | root of `/` |
|---|---|---|---|
| Live installer | not a chroot | no — the ISO's | the ISO's |
| Under `arch-chroot /mnt` | **chroot** | no — still the ISO's | `cryptroot[/@]` |
| Booted system | not a chroot | **yes** | `cryptroot[/@…]` |
| Rescue chroot from the booted system | **chroot** | yes | `cryptroot[/@]` |

`systemd-detect-virt -r` "Detect[s] whether invoked in a chroot(2) environment.
In this mode, no output is written, but the return value indicates whether the
process was invoked in a chroot() environment or not" (*measured*,
`systemd-detect-virt(1)`, read 2026-08-24). It is the direct namespace question
and it is **not sufficient on its own**: the live installer is not a chroot
either, so the first and third rows are identical under it.

`/proc/cmdline` is not sufficient on its own either — the fourth row is the
counterexample, and it is not hypothetical, since PRD §9.5's rollback procedure
and §9.6's `bootctl install` repair are both `arch-chroot` entered from the Arch
ISO after the machine has been built.

**The two together are, and each covers exactly the other's blind spot.** That
is why `first-boot`'s postcondition asserts both rather than picking the
cleverer one. The rows that need separating are row 1 from row 3 (which
`/proc/cmdline` does) and row 4 from row 3 (which `systemd-detect-virt -r`
does).

**The privilege regime changes at the same node, and is a consequence rather
than a definition.** Before `first-boot` every command runs as root, in the
installer or in a chroot entered as root. After it there is no root shell in
normal use: the build ends with a `wheel` user reaching root through `sudo`, and
`/boot` is `dmask=0077` (`state/storage.md`, `## mounts`), so even reading the
ESP needs it. That is the tell `spec/_index` names, and it is downstream of the
namespace change, not the same fact. `first-boot`'s own postcondition
deliberately uses no privileged command: it is the step that establishes the
privilege regime, so it must not depend on it.

---

## base-install

`pacstrap -K /mnt <packages>`. `-K` "Initialize an empty pacman keyring in the
target (implies `-G`)", where `-G` is "Avoid copying the host's pacman keyring
to the target" (*measured*, `pacstrap(8)`, read 2026-08-24). Without it the
target inherits the ISO's keyring, which is a snapshot of whenever that ISO was
built.

**The ESP must be mounted before this runs, and nothing downstream says so
loudly.** The `linux` package writes `/boot/vmlinuz-linux` — measured twice
over: `state/boot.md`'s `## presets` shows `ALL_kver="/boot/vmlinuz-linux"`, and
`state/boot.md`'s `## bootctl` shows `ESP: /boot`, so the kernels live on the
ESP. With `/mnt/boot` unmounted, pacstrap writes them onto `@` instead, the ESP
stays empty, and every later step still succeeds: `mkinitcpio -P` finds its
kernel, `uki-generate`'s postcondition resolves UKI paths from the presets and
finds the files it expects, and `uki-sign` signs them. The whole boot chain is
built correctly on the wrong filesystem, and the firmware finds nothing. The
edge to `esp-mount` orders it; the precondition re-checks it, on the
`secureboot-enable` principle that an edge is not worth trusting when the cost
of it being wrong is paid several steps later.

**The postcondition asserts that packages are *installed*, never that they are
*explicit*, and `state/` is what forces the distinction.** `state/packages.md`'s
`## explicit` is `pacman -Qqe` (*measured*, `collect.sh` line 11), and five of
the thirty-four packages below are absent from it: `linux-headers`,
`linux-lts-headers`, `nftables`, `sudo`, `pciutils`. At least one of those is
demonstrably present — `nftables.service` is in `state/services.md`'s
`## system enabled` and `state/network.md`'s `## nftables` shows a live
default-drop ruleset. So **absence from `## explicit` does not mean absence from
the machine**; the install reason moved, which `pacman -Qqe` reports and
`pacman -Qq` does not. (*Inferred*: that the reason is `dependency` for all five
is reasoning, not observation. `pacman -Qi nftables | grep 'Install Reason'` on
the machine settles it, and nothing here depends on the answer.)

A check written against `-Qqe` would therefore be red on a correctly built
machine for five entries, which is the permanently-red failure `luks-format`
avoids by not counting keyslots. The same argument, reached from the other
direction.

**`--sysroot`, not `-r`.** `pacman(8)` on `-r/--root`: "**NOTE**: This option is
not suitable for performing operations on a mounted guest system. See
*--sysroot* instead." And on `--sysroot`: "This path will be prepended to all
other configuration directories … This allows mounted guest systems to be
properly operated on." (*measured*, read 2026-08-24.) `-r` is the flag reached
for by habit and the man page rules it out for exactly this case.

The membership test compares whole lines rather than grepping, for the reason
`btrfs-subvols-create` gives: `grep -q linux` matches `linux-firmware`, so a
grep per name would pass with `linux` itself missing.

```yaml
id: base-install
requires: [esp-mount]
precondition: |
  set -euo pipefail
  # The subvolume tree and the ESP must both be in place. The ESP one is the
  # hazard: with /mnt/boot unmounted, the kernels land on @ and every later
  # step still succeeds against the wrong filesystem.
  findmnt -no FSTYPE -M /mnt      | grep -qx btrfs
  findmnt -no FSTYPE -M /mnt/boot | grep -qx vfat
command: |
  set -euo pipefail
  # -K initialises an empty keyring in the target rather than inheriting the
  # ISO's, which is a snapshot of whenever the ISO was built.
  pacstrap -K /mnt \
    base base-devel \
    linux linux-headers \
    linux-lts linux-lts-headers \
    linux-firmware linux-firmware-realtek amd-ucode \
    btrfs-progs dosfstools gptfdisk efibootmgr mtools \
    dmidecode \
    cryptsetup \
    networkmanager openssh nftables \
    git neovim sudo \
    man-db man-pages terminus-font \
    pacman-contrib reflector \
    smartmontools nvme-cli usbutils pciutils \
    fwupd \
    zram-generator \
    sbctl
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # --sysroot, not -r: pacman(8) says -r "is not suitable for performing
  # operations on a mounted guest system". `|| true` so that a missing database
  # reports the line below rather than aborting under set -e with no verdict.
  installed=$(pacman -Qq --sysroot /mnt || true)
  [ -n "$installed" ] || { echo "RESULT=FAIL (no package database under /mnt)"; exit 1; }
  # Installed, not explicit. pacman -Qqe reports the install reason, which
  # moves over the machine's life: five of these are absent from state/'s
  # ## explicit today and at least one of them is demonstrably running.
  for p in base base-devel linux linux-headers linux-lts linux-lts-headers \
           linux-firmware linux-firmware-realtek amd-ucode \
           btrfs-progs dosfstools gptfdisk efibootmgr mtools dmidecode \
           cryptsetup networkmanager openssh nftables git neovim sudo \
           man-db man-pages terminus-font pacman-contrib reflector \
           smartmontools nvme-cli usbutils pciutils fwupd zram-generator sbctl; do
    # Whole-line comparison: `grep -q linux` matches linux-firmware.
    printf '%s\n' "$installed" | grep -qx "$p" || note "not installed: $p"
  done
  # The direct observation that the ESP was mounted when pacstrap ran. Stronger
  # than the precondition, which only established it beforehand.
  for k in vmlinuz-linux vmlinuz-linux-lts; do
    test -f "/mnt/boot/$k" || note "/mnt/boot/$k is absent — was the ESP mounted?"
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## fstab-generate

`genfstab` "helps fill in an fstab file by autodetecting all the current mounts
below a given mountpoint and printing them in fstab-compatible format to
standard output", and `-U` is "Use UUIDs for source identifiers (shortcut for
*-t UUID*)" (*measured*, `genfstab(8)`, read 2026-08-24).

**Where the options it writes come from is not in the man page, and it matters
for three separate claims in this vault**, so it is grounded in `state/`
instead. `state/storage.md`'s `## fstab` carries `ssd` and `space_cache=v2` on
every btrfs line and `codepage=437,iocharset=ascii,shortname=mixed,utf8` on the
vfat line. No mount command in `spec/disk` passes any of them, and `btrfs(5)`
documents the first two as defaults. So genfstab wrote the options the *kernel*
reported, not the options the mount command was given (*measured* by that
comparison; the man page does not state it).

That is what makes the two edits below necessary rather than cosmetic: the
kernel always reports `subvolid=` and `subvol=`, so the generated root line
pins the subvolume that happened to be mounted at install time.

### The divergence: §2.2 strips one of the two

PRD §2.2 strips `subvolid=` and gives the reason: "it breaks `snapper rollback`,
which creates a new subvolume ID that a pinned fstab will not follow." The
reason is right and **it applies verbatim to `subvol=`** — a rollback creates a
new snapshot at a new *path* as well as a new id, and a line pinning
`subvol=/@` does not follow that either. `btrfs(5)` treats them as one
mechanism: `subvol` "Mount subvolume from *path* rather than the toplevel
subvolume… **This mount option overrides the default subvolume set for the given
filesystem**", and `subvolid` says the same sentence about a number (*measured*,
read 2026-08-24). Overriding the default subvolume is the whole hazard, and the
option name is not the part that does it.

§2.2 strips only the first. Its own "expected final shape" block shows the root
line as `…,compress=zstd:1,subvol=/@`, and both verification blocks that check
this — §11 and Appendix A — grep for `subvolid` alone. So the document is
internally consistent and consistently half-right.

**The machine is not.** `state/storage.md`'s `## fstab` shows the root line
today as `rw,noatime,compress=zstd:1,ssd,space_cache=v2` — neither option
present, while the other five btrfs lines keep their `subvol=`. Whoever built
this machine made the edit the PRD does not describe. The built system is more
correct than the document, and nothing recorded why, which is the same shape as
the TRIM divergence [[../decisions/20260823-continuous-trim-both-layers|ADR-0014]]
settled one chapter ago.

### The edit gets its own postcondition, and the reason is not redundancy

[[disk#btrfs-default-matches-root|`btrfs-default-matches-root`]] was until now
the only thing that would notice a missed strip. Its own prose says so, and says
the mounted root would stop following the default subvolume. **Two things are
wrong with leaving it there.**

First, the invariant lives on the far side of `first-boot`, so the earliest it
can speak is after a reboot — one step after the fstab is the only artifact that
could still be corrected cheaply.

Second, and worse, it is not clear it fires at all.
`systemd-remount-fs.service(8)`: the service "applies mount options listed in
fstab(5) … to the root file system, the /usr/ file system, and the kernel API
file systems. This is required so that the mount options of these file systems —
which are pre-mounted by the kernel, the initrd, container environments or
system manager code — are **updated** to those configured in /etc/fstab"
(*measured*, read 2026-08-24). The root filesystem is therefore **remounted from
fstab, not mounted from it**, and the subvolume was already chosen by the
initramfs from `root=/dev/mapper/cryptroot` with no `rootflags=`. Whether a
remount carrying a mismatched `subvol=` changes the mounted subvolume, errors,
or is ignored is **not answered by `btrfs(5)`** — its `subvol` entry describes
mounting. (*Inferred*: it cannot change it, since the subvolume is which tree
the mount is rooted at rather than a tunable. The lookup that would settle it is
`btrfs(5)`'s remount discussion read in full on the machine, or a
`mount -o remount,subvol=…` on a scratch filesystem.)

An invariant that *might* not fire is worse than no coverage, because it reads
as coverage. So the edit is asserted here, where it is made.

**Nothing about that argument depends on it, though, because the edit has a
failure mode the invariant could not express even in principle.** The obvious
way to write the strip is the PRD's `sed`, generalised — and a global
`s/,?subvol=[^,]*//` takes `subvol=` off all six lines, not one. The other five
mounts then resolve to the top level: six directories holding the wrong tree, no
error at any stage, and a root that is still correct so
`btrfs-default-matches-root` passes. The postcondition therefore asserts in both
directions — the root line has neither option, and each of the other five still
carries its own `subvol=`.

### Two more hazards §2.2 names and does not check

**`genfstab` appends.** `>>` into a file that already has entries duplicates
every mount. §2.2 says nothing about re-running; the precondition refuses if the
file already carries `UUID=` lines.

**"Edit the existing line rather than appending a second one"** is §2.2's
instruction for the swapfile priority, in bold, addressed to a human. Two
`/swap/swapfile` lines is the failure it warns about, and it is a line of prose
where a count belongs. Both are asserted.

### `set -e` is deliberately absent from this postcondition

The counting checks legitimately count zero, and `grep -c` exits non-zero when
it counts nothing. Under `set -e` the "no swap line at all" case — which §2.2
explicitly anticipates — would abort the postcondition mid-run with no
`RESULT=` line, instead of reporting the one diagnostic it exists to produce.
`_schema.md`: a check must not fail the same way as the thing it detects.
`set -uo pipefail` keeps the rest.

This postcondition was run against a synthesised `genfstab -U` output for this
machine's mount set and eight mutations — the naive global strip, §2.2 as
written, a doubled `genfstab`, an appended second swap line, a missing `pri=10`,
default ESP masks, no swap line at all, and `genfstab` without `-U`. PASS on the
first and a distinct diagnostic on each of the other eight. Per the accepted
limitation in [[../sessions/20260823-spec-boot-promotion|the second session
log]], that is synthesis rather than execution: no real `genfstab` was run.

```yaml
id: fstab-generate
requires: [base-install, swapfile-create]
precondition: |
  set -euo pipefail
  test -f /mnt/etc/fstab
  # genfstab appends. A second run duplicates every mount, and the duplicate is
  # the entry that wins.
  ! grep -q '^UUID=' /mnt/etc/fstab
  # The swapfile must be active or genfstab emits no swap line at all, and the
  # machine boots with no disk swap and no error.
  swapon --show=NAME --noheadings | grep -qx /mnt/swap/swapfile
command: |
  set -euo pipefail
  genfstab -U /mnt >> /mnt/etc/fstab
  # subvolid= everywhere: a rollback creates a new id and a pinned fstab does
  # not follow it (PRD §2.2).
  sed -i -E 's/,?subvolid=[0-9]+//g' /mnt/etc/fstab
  # subvol= on the root line ONLY. The address matches a line whose second
  # field is exactly `/`. Stripping it globally is the obvious generalisation
  # and leaves the other five mounts resolving to the top level.
  sed -i -E '\%^[^#[:space:]]+[[:space:]]+/[[:space:]]%s/,?subvol=[^,[:space:]]+//' /mnt/etc/fstab
  # Edit the swap line, never append a second one (PRD §2.2). zram is created
  # at pri=100, so the disk swapfile must be explicitly lower or the kernel may
  # prefer QLC over compressed RAM.
  sed -i -E '\%^/swap/swapfile[[:space:]]%s/[[:space:]]defaults([[:space:]])/ defaults,pri=10\1/' /mnt/etc/fstab
postcondition: |
  # No `set -e`: the counting checks legitimately count zero, and grep -c exits
  # non-zero when it does. Aborting there would suppress the diagnostic for the
  # exact case PRD §2.2 anticipates.
  set -uo pipefail
  f=/mnt/etc/fstab
  fail=0
  note() { echo "  $1"; fail=1; }
  tab=$(grep -vE '^[[:space:]]*#|^[[:space:]]*$' "$f")
  row()  { printf '%s\n' "$tab" | awk -v m="$2" -v c="$1" '$2==m {print $c}'; }
  rows() { printf '%s\n' "$tab" | awk -v m="$1" '$2==m' | grep -c .; }
  # Exactly one line per mount point. genfstab appends, so a second run
  # duplicates all of them.
  ok=""
  for m in / /home /swap /.snapshots /var/log /var/cache /boot; do
    n=$(rows "$m")
    if [ "$n" -eq 1 ]; then ok="$ok $m"
    else note "expected 1 fstab line for $m, found $n"; fi
  done
  have() { case " $ok " in *" $1 "*) return 0 ;; *) return 1 ;; esac; }
  # -U was passed: a /dev/mapper/ or /dev/nvme path here is the enumeration-order
  # literal this vault forbids, written into the installed system.
  for m in / /home /swap /.snapshots /var/log /var/cache /boot; do
    have "$m" || continue
    src=$(row 1 "$m")
    case "$src" in UUID=*) ;; *) note "$m is sourced as '$src', want UUID=" ;; esac
  done
  # Root: neither option. btrfs(5) says each "overrides the default subvolume
  # set for the given filesystem", and the default subvolume is the sole
  # selector for / on this machine. `subvolid=` does not contain `subvol=`, so
  # the two patterns discriminate cleanly.
  if have /; then
    [ "$(row 3 /)" = btrfs ] || note "root line fstype is '$(row 3 /)', want btrfs"
    ro=$(row 4 /)
    case ",$ro," in *,subvol=*)   note "root line carries subvol=: $ro" ;; esac
    case ",$ro," in *,subvolid=*) note "root line carries subvolid=: $ro" ;; esac
  fi
  # The other five must still carry theirs. A global strip of subvol= passes
  # every check above and leaves five mounts on the top level.
  sub() {
    have "$1" || return 0
    o=$(row 4 "$1")
    case ",$o," in
      *",subvol=$2,"*) ;;
      *) note "$1 has $(printf '%s' "$o" | grep -o 'subvol=[^,]*' || echo 'no subvol='), want subvol=$2" ;;
    esac
    case ",$o," in *,subvolid=*) note "$1 carries subvolid=" ;; esac
  }
  sub /home       /@home
  sub /swap       /@swap
  sub /.snapshots /@snapshots
  sub /var/log    /@log
  sub /var/cache  /@cache
  # The ESP masks reach the installed system only because genfstab copied what
  # esp-mount passed. Getting them wrong is persistent, not mount-time.
  if have /boot; then
    [ "$(row 3 /boot)" = vfat ] || note "/boot fstype is '$(row 3 /boot)', want vfat"
    bo=$(row 4 /boot)
    for m in fmask=0077 dmask=0077; do
      case ",$bo," in *",$m,"*) ;; *) note "/boot lacks $m" ;; esac
    done
  fi
  # Matched on field 1: the swapfile's mount point is `none`. One line, not two
  # — PRD §2.2 puts "edit rather than append" in bold and checks it nowhere.
  sw=$(printf '%s\n' "$tab" | awk '$1=="/swap/swapfile"')
  n=$(printf '%s\n' "$sw" | grep -c .)
  if [ "$n" -ne 1 ]; then
    note "expected 1 /swap/swapfile line, found $n"
  else
    [ "$(printf '%s' "$sw" | awk '{print $3}')" = swap ] || note "swapfile line fstype is not 'swap'"
    case ",$(printf '%s' "$sw" | awk '{print $4}')," in
      *,pri=10,*) ;;
      *) note "swapfile line lacks pri=10: $(printf '%s' "$sw" | awk '{print $4}')" ;;
    esac
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## user-create

The first step that runs under `arch-chroot`, which "wraps the chroot(1) command
while ensuring that important functionality is available, e.g. mounting /dev/,
/proc/ and other API filesystems, or exposing resolv.conf(5) to the chroot"
(*measured*, `arch-chroot(8)`, read 2026-08-24). The `/proc` it mounts is why
`/proc/cmdline` still reports the ISO's kernel inside it — the second row of the
table above.

**The hazard PRD §3.5 names is real and now sourced.** `useradd(8)` on
`-p/--password`: "Without this option, the new account will be locked and with
no password defined, i.e. a single exclamation mark in the respective field of
/etc/shadow" (*measured*, read 2026-08-24). `useradd` here is deliberately run
without `-p` — an encrypted hash on a command line is not a thing to write into
a spec — so the account is created locked, and `passwd` is what unlocks it.
`passwd` run interactively does nothing at all if the two entries disagree.
Combined with a locked root that is a system with no way in.

`passwd -S` is therefore the assertion: it "Display[s] account status
information… The first field is the user's login name. The second field
indicates if the user account has a locked password (L), has no password (NP),
or has a usable password (P)" (*measured*, `passwd(1)`, read 2026-08-24). Field
two must be `P`. That is the check that separates "the command ran" from "the
command worked", and it is the one §3.5 puts in prose.

**The sudo grant is resolved, not grepped.** §3.5 checks
`grep -c '^%wheel ALL=(ALL:ALL) ALL' /etc/sudoers`, which is asserting against a
constant: it misses a grant arriving through `/etc/sudoers.d`, it is sensitive
to whitespace the parser does not care about, and it says nothing about whether
the file parses at all. Two live readings replace it:

- `visudo -c` — "The existing *sudoers* file (and any other files it includes)
  will be checked for syntax errors… If the check completes successfully,
  `visudo` will exit with a value of 0. If an error is encountered, `visudo`
  will exit with a value of 1." (*measured*, `visudo(8)`, read 2026-08-24.) One
  of the few documented exit statuses in this vault, so it is used as one.
- `sudo -l -U luiswg <command>` — "Used in conjunction with the `-l` option to
  list the privileges for user instead of for the invoking user", and with a
  command given, "the exit value will only be 0 if the command is permitted by
  the security policy, otherwise it will be 1" (*measured*, `sudo(8)`, read
  2026-08-24). `-U` "is restricted to the root user and users with either the
  'list' privilege…", which is satisfied: inside the chroot this runs as root.
  `-n` — "Avoid prompting the user for input of any kind" — guarantees it
  cannot hang an executor waiting on a password prompt.

**Only `wheel` is asserted, and the other four groups are `inferred`.** §3.5
adds `audio,video,storage,input`. Nothing in `state/` records group membership —
there is no collector section for `id` or `getent group` — so there is no
grounding for them, and `wheel` is the only one this chapter can tie to an
observable, via the sudo grant. The lookup that closes it is one `id luiswg` on
the machine, or a collector section; and the sharper question it would let
someone ask is whether the four are load-bearing at all under seat management,
which is a decision rather than a measurement.

**`passwd -l root` is not a step here, deliberately.** §3.5 sets it out with the
costs on both sides and ends "Either choice is defensible; make it deliberately
rather than by copying this line." That is an undecided decision, and `state/`
cannot even report which way this machine went — nothing collects `passwd -S
root`. Writing it into the spec would be recording a choice nobody made, in the
one place that is supposed to record intent accurately. Last section.

`/bin/bash` is the login shell this step sets. The machine runs zsh today
(`state/packages.md`, and `chsh` in `state/notes.md`), which arrives in a later
chapter; the build-time shell is not the final one and is not asserted as if it
were.

**Changed 2026-08-24 by the sweep for
[[_schema#A postcondition must reach its verdict line on every input it is meant to judge|`_schema.md`'s seventh convention]]**:
`st=$(passwd -S luiswg | awk …)` aborted the postcondition when `passwd -S`
failed, discarding the three checks below it. It now falls to empty and reports
`<unreadable>`, which is deliberately not the same string as any status
`passwd -S` can print — `P`, `L` and `NP` all mean the command worked.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — account exists, `P`, sudoers parses, grant present, in `wheel` | PASS | PASS |
| 2 | no `luiswg` account | `RESULT=FAIL (no luiswg account)` | same |
| 3 | **`passwd -S` fails** | FAIL, `status is '<unreadable>', want P` | **no `RESULT=`** |
| 4 | account still locked (`L`) | FAIL, quotes `L` | same |
| 5 | no password set (`NP`) | FAIL, quotes `NP` | same |
| 6 | `/etc/sudoers` does not parse | FAIL, `does not parse` | same |
| 7 | no sudo grant for `pacman` | FAIL, names the grant | same |
| 8 | not in `wheel` | FAIL, `not in wheel` | same |

Rows 4 and 5 are the reason this check exists at all — `useradd` without `-p`
creates the account locked, and `passwd` run interactively does nothing when the
two entries disagree, so both are states a build reaches by doing everything
right except the one thing.

```yaml
id: user-create
requires: [base-install]
precondition: |
  set -euo pipefail
  # Runs under arch-chroot: these are the target's files, unprefixed.
  test -f /etc/passwd
  # useradd fails on an existing account, and a re-run must not look like a
  # fresh one.
  ! getent passwd luiswg >/dev/null
command: |
  set -euo pipefail
  # No -p: an encrypted hash does not belong on a command line or in a spec.
  # useradd(8) therefore leaves the account locked with no password, and the
  # interactive passwd below is what makes it usable.
  useradd -m -G wheel,audio,video,storage,input -s /bin/bash luiswg
  passwd luiswg
  # Interactive. Uncomment: %wheel ALL=(ALL:ALL) ALL
  EDITOR=nvim visudo
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  getent passwd luiswg >/dev/null || { echo "RESULT=FAIL (no luiswg account)"; exit 1; }
  # Field 2 of `passwd -S`: P usable, L locked, NP no password. useradd creates
  # the account locked, and passwd silently does nothing when the two entries
  # disagree — so this separates "the command ran" from "the command worked".
  st=$(passwd -S luiswg | awk '{print $2}') || st=
  [ "$st" = P ] || note "luiswg password status is '${st:-<unreadable>}', want P"
  # Syntax and includes. A sudoers file that does not parse costs every path to
  # root at once, and visudo's exit status is documented.
  visudo -c >/dev/null || note "/etc/sudoers does not parse"
  # The grant resolved rather than grepped: this covers /etc/sudoers.d and says
  # what the policy actually permits. -n so it cannot wait on a prompt.
  sudo -l -n -U luiswg /usr/bin/pacman >/dev/null 2>&1 \
    || note "luiswg is not permitted to run pacman via sudo"
  # audio, video, storage and input are set by the command and asserted by
  # nothing: state/ records no group membership. See the prose.
  id -nG luiswg | tr ' ' '\n' | grep -qx wheel || note "luiswg is not in wheel"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## first-boot

**The boundary node, and a command-less step for a reason `_schema.md` covers
but had not yet met.** The rule is "a step with no `command` is one the executor
cannot make true. It can only check it," and the three shapes named against it
are a firmware menu, a lookup gap, and a standing invariant. This is a fourth:
an action that **destroys the executor that would perform it**. `reboot` does
not return, so no process that issues it can observe whether it worked. The rule
as written already covers it — which is the argument for having phrased it that
way rather than as "a human must satisfy this" — but the "known cost" in
`_schema.md`, that the shapes render identically and are told apart only by the
prose, now applies to four rather than three.

The teardown PRD §3.9 specifies belongs in prose for the same reason:

```
exit
swapoff /mnt/swap/swapfile
umount -R /mnt
reboot
```

§3.9 orders the `swapoff` after `umount -R /mnt` and tolerates its failure with
`2>/dev/null || true`. Swapping the two removes the need for that: `umount -R`
has no reason to release a file still in use as swap, and a `|| true` on the
step that is meant to quiesce the disk is a suppressed error rather than a
handled one. This is the same one-line reordering `spec/disk` made to §1.3's
`umount` — a PRD sequence whose two halves are individually right.

**What the postcondition asserts is argued in full above.**
`! systemd-detect-virt -r` and `rd.luks.name=` in `/proc/cmdline` are each
insufficient alone and together separate all four namespaces. The root source
check is corroborating rather than load-bearing; it is cheap and it names the
filesystem, which is what makes a failure readable.

**The swap path is the boundary made observable, and it is the exact reading
that was wrong before.** `swapon --show` reports the path recorded at `swapon`
time. `swapfile-create` activates `/mnt/swap/swapfile` in the installer, and the
first-round bug was `cmdline-write` consulting that string from inside the
chroot, where the file is `/swap/swapfile`. After a successful `first-boot` the
kernel's recorded path has lost the prefix, because swap was activated afresh
from the installed fstab. So one reading answers two questions: the namespace
changed, and `fstab-generate`'s swap line survived into a running system. The
second is worth having on its own — a missing swap line boots fine and silently
costs hibernation, and this is the earliest point at which anything can see it.

It requires `bootloader-install`. Without it the ESP holds UKIs and no loader:
`state/boot.md`'s `## bootctl` shows the firmware reaching `systemd-boot` at
`/boot/EFI/systemd/systemd-bootx64.efi` with `/boot/EFI/BOOT/BOOTX64.EFI` as the
fallback. That id was referenced from here while no chapter defined it, and
writing this step is what surfaced the gap; `spec/boot` now covers it. It does
**not** require `bootloader-nvram-entry`, which is downstream of this node — the
named firmware entry is what a build gains after first boot, not what makes the
first boot possible. The removable-media fallback path is why.

**Changed 2026-08-24, and the mutation set is the first thing that has exercised
the four-namespace table.** `src=$(findmnt -no SOURCE /)` aborted the whole
postcondition when `findmnt` returned nothing — the same documented conflation
of *found nothing* with *any error* that
[[disk#btrfs-subvol-mount|`btrfs-subvol-mount`]] hit, and the seventh convention
in `_schema.md` is where the general form now lives. The empty case gets its own
`case` arm, because `/ is on '', want /dev/mapper/cryptroot` reads as a wrong
filesystem rather than as no answer.

| # | Mutation | Fixed | Unfixed |
|---|---|---|---|
| 1 | baseline — booted system, row 3 of the table | PASS | PASS |
| 2 | live installer / under `arch-chroot` — row 1 and row 2 | FAIL via `detect-virt` or the cmdline | same |
| 3 | the ISO's kernel booted, no `rd.luks.name=` | FAIL, `did the UKI boot?` | same |
| 4 | **rescue chroot from the booted system — row 4** | FAIL, `running inside a chroot` | same |
| 5 | **`findmnt` reports nothing for `/`** | FAIL, `findmnt reported no source for /` | **no `RESULT=`** |
| 6 | `/` is not on `cryptroot` | FAIL, quotes the source | same |
| 7 | no swap active at all | FAIL, `no disk swap, and no hibernation` | same |
| 8 | swap active but still `/mnt/swap/swapfile` | same diagnostic | same |

Rows 2, 3 and 4 are the three the table above says need separating, and row 4 is
the one `/proc/cmdline` cannot see on its own. The set is **synthesised
readings**, not four executions: `systemd-detect-virt`, `findmnt`, `swapon` and
`/proc/cmdline` were all stubbed, so what has been exercised is the logic that
combines them, which is one step short of the thing the open list still asks
for. Rows 7 and 8 produce the same message, which is correct — both mean the
installed fstab's swap line did not reach a running system — and it is worth
noting that this check therefore cannot tell you *which*.

```yaml
id: first-boot
requires: [uki-generate, bootloader-install, fstab-generate, user-create]
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Not in a chroot. Necessary and not sufficient: the live installer is not a
  # chroot either, which is what the next check separates.
  if systemd-detect-virt -r; then note "running inside a chroot"; fi
  # This kernel booted with our cmdline, so it is not the ISO's. Necessary and
  # not sufficient: a rescue chroot entered from the booted system sees the
  # same /proc, which is what the check above separates.
  grep -q 'rd\.luks\.name=' /proc/cmdline \
    || note "/proc/cmdline is not this machine's — did the UKI boot?"
  # Corroborating, and it names the filesystem so a failure is readable.
  # findmnt returns 1 for "found nothing" and for "any error" alike, so the
  # assignment must be allowed to come back empty (_schema.md, seventh).
  src=$(findmnt -no SOURCE /) || src=
  case "$src" in
    /dev/mapper/cryptroot*) ;;
    "") note "findmnt reported no source for /" ;;
    *) note "/ is on '$src', want /dev/mapper/cryptroot" ;;
  esac
  # The prefix is gone because swap was activated afresh from the installed
  # fstab, not carried over from the installer. Two questions, one reading: the
  # namespace changed, and fstab-generate's swap line reached a running system.
  swapon --show=NAME --noheadings | grep -qx /swap/swapfile \
    || note "/swap/swapfile is not active — no disk swap, and no hibernation"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## collector-run

*Added 2026-08-26.* The collector is how `state/` learns anything, and until now
nothing in the spec asserted that it had run or that its output was complete. That gap
is what let the toolchain question sit unanswerable in the vault for two months while
both sides of it were one command away.

**This is the only node here with a `command:`, and that is the point.** An earlier
draft gave `collect-tool-versions` and `collect-harness-config` **identical `command:`
blocks**, which is two DAG nodes for one action: a topological sort would have been
entitled to run `./collect/collect.sh` twice, and neither node was really the action —
each was an assertion wearing one. Same category error as an edge that means "related
to", moved from `requires:` into `command:`. One action, one node; each section gets an
assertion that cannot run anything.

**Every file is asserted present and non-empty, by name.** A collector module that
silently writes nothing leaves a file that exists and is zero bytes, which is
indistinguishable from a subsystem with nothing to report unless the expectation is
written down. The list is policy — what this collector is built to emit — in the same
class as the tool list below, not a machine-varying value.

**What this postcondition does *not* catch, corrected 2026-08-26.** An earlier draft of
this paragraph claimed it earned its place by catching a partial run: adding the harness
block introduced a reference to an undefined variable under `set -u`, which would have
aborted the collector partway and left `services.md`, `hardware.md`, `boot.md`,
`storage.md` and `network.md` holding the **previous run's content**. `-s` does not
catch that — those files exist and are non-empty, so every check passes. The claim was
the vacuous-correctness shape, written into the justification for a check.

Three separate things are actually doing that work, and only one of them is here:

- **The abort itself is caught by the exit status.** `collect.sh` runs under
  `set -euo pipefail`, so a partial run exits non-zero, and the
  [[_schema#The executor contract|executor contract]] halts on it. The `command:`
  catches it, not the `postcondition:`.
- **Blast radius is limited by block order.** The harness block was moved to the end of
  the collector on 2026-08-26 for this reason: it is the newest and least-proven, and
  upstream of five others it would have taken them all down with it.
- **A stale file written by a previous run is detected by nothing at all.** If the
  collector aborts and the exit status is ignored, `state/` looks complete and the git
  diff reads as "nothing changed". Closing that wants a per-run marker written into
  every file and asserted to match across all of them —
  [[../backlog/collector-stale-file-detection|a backlog item]], not a change to this
  step.

What this postcondition *does* catch is a module that ran and wrote nothing: a file that
exists and is zero bytes, which reads as a subsystem with nothing to report unless the
expectation is written down.

```yaml
id: collector-run
requires: [first-boot]
precondition: |
  set -euo pipefail
  # The collector resolves paths relative to its own directory and writes
  # nothing useful from elsewhere.
  test -x ./collect/collect.sh \
    || { echo "run from arch-thinkpad/ — ./collect/collect.sh is not executable here"; exit 1; }
command: |
  set -euo pipefail
  ./collect/collect.sh
postcondition: |
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Present and non-empty, by name. A module that wrote nothing leaves a file
  # that exists and is empty, which reads as "nothing to report".
  for f in boot collector-run hardware harness network packages services storage; do
    p="state/$f.md"
    [ -s "$p" ] || note "$p is missing or empty"
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## collect-tool-versions

*Added 2026-08-26.* **No `command`** — the third shape
[[_schema#What is deliberately absent|`_schema.md`]] names: nothing an executor runs
makes the section correct. `collector-run` produces it; this asserts it.

**The section is asserted by name and by contents, not by existence.** A collector that
stops emitting it leaves [[devenv#toolchain-parity|`toolchain-parity`]] reading nothing
— and reading nothing is exactly what the mawk bug did. So the postcondition names
every tool it expects, and a tool dropped from the loop fails the build rather than
quietly narrowing what is grounded.

**Field 2 must be an absolute resolved path**, because a bare name there would mean the
collector recorded what the shell showed. Two measured failures make that the rule:
`.zshrc` aliased `grep` to `rg`, so the first reading of the host's toolchain recorded
*"ripgrep 15.2.0" as the host's grep*; and `command -v gawk` in the container returns
`/usr/sbin/gawk` where `readlink -f` gives `/usr/bin/gawk`. See
[[_schema#Conventions|the host-question rule]].

**`<unreadable>` is a legitimate terminal state and is deliberately not a failure.** If
a tool genuinely has no `--version`, that is the truthful record and it will not
change; asserting otherwise would make this step red on a correct machine forever. The
collector distinguishes it from `<absent>` by branching on **exit status, not
emptiness** — a tool that rejects `--version` prints usage to stdout, so its first line
is non-empty and perfectly plausible as a version string.

**Counting is `awk`, not `grep -c`**, per the
[[_schema#Conventions|must-reach-the-verdict-line]] rule: zero matches must print `0`
and exit 0, or the assignment aborts under `set -e` and the diagnostic below it never
fires.

```yaml
id: collect-tool-versions
requires: [collector-run]
postcondition: |
  # set -e is dropped deliberately: several counts below legitimately read zero,
  # and guarding each one would be noise (fstab-generate is the worked case).
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  f=state/packages.md
  [ -r "$f" ] || { echo "RESULT=FAIL ($f is unreadable)"; exit 1; }
  # Exactly one section header. Two means an appended run rather than a rewrite.
  n=$(awk '$0 == "## tool versions" { n++ } END { print n + 0 }' "$f")
  [ "$n" -eq 1 ] || note "found $n '## tool versions' headers in $f, want 1"
  for t in awk grep sed jq findmnt btrfs cryptsetup sbctl systemctl \
           podman pasta nft pacman bootctl mkinitcpio loginctl; do
    # Restricted to the section: the tool names also occur in prose elsewhere.
    n=$(awk -v t="$t" '
          /^## / { insec = ($0 == "## tool versions") }
          insec && $1 == t { n++ }
          END { print n + 0 }' "$f")
    [ "$n" -eq 1 ] || { note "$t: $n lines under ## tool versions, want 1"; continue; }
    # An absolute resolved path, or <absent>. A bare name means the collector
    # recorded what the shell showed, which is the defect this section prevents.
    p=$(awk -v t="$t" '
          /^## / { insec = ($0 == "## tool versions") }
          insec && $1 == t { print $2; exit }' "$f")
    case "$p" in
      /*|"<absent>") ;;
      *) note "$t: path field is '$p', want an absolute path or <absent>" ;;
    esac
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## collect-harness-config

*Added 2026-08-26.* The bounded extraction
[[../decisions/20260825-harness-state-and-identity|ADR-0017]] leans on. `.claude.json`
is 44 KB, carries the OAuth account, and is excluded from restic — so MCP servers,
per-project trust and `allowedTools` would otherwise have **no diff surface at all**,
while being the surface that grants tools.

**The `# source:` marker is asserted per section, not counted**, and the reason is that
this check's own subject nearly shipped broken twice.

The first draft of the *collector* read a top-level `.mcpServers`; there is no such key
— MCP configuration is per-project (*measured 2026-08-26, `jq keys`*). Every run would
have emitted an empty section, which is exactly what a machine with no MCP servers
looks like: `state/_index.md`'s `"tpm2-pcrs": null` failure on a section whose job is
making tool-granting surface visible.

The first draft of *this postcondition* then counted markers with `$4 == "present"`
across the whole file and required three. **That worked by field-position accident** —
two of the three markers carry a parenthetical, and `$4` happened to line up in all
three. Rewording any marker would have dropped the count silently, and the diagnostic
would have blamed a moved `jq` field. It is now matched by anchored prefix, inside each
section, so a failure names the section that did not report.

**The leak check is structural, and the content sniff it replaced would have fired on
correct output.** `allowedTools` rows are verbatim permission strings; a rule naming an
OAuth domain — `WebFetch(domain:oauth.example.com)` — trips a case-insensitive
`grep 'oauth'`, and so does any tool string containing `userid`. A leak check that
fires on a clean run gets disabled, which is worse than not having one. What is asserted
instead is the **shape the allowlist produces**: every data row begins with an absolute
project path. A top-level `oauthAccount`, `userID` or `machineID` rendered into this
file would not, because `jq` emits those as bare strings or objects with no leading `/`.
`harness.md` also passes through `redact.sh`, so anything already in `redact.map` is
tokenised regardless — belt, and separately braces.

**What `redact.sh` actually rewrites, since the shape check depends on it.** Measured
2026-08-26 from `gen-redact-map.sh`: the map holds `/etc/machine-id`, the NVMe, board
and system serials, both Tailscale addresses, a `PARTUUID`/`FSUUID` pair per block
device, every interface MAC and permanent address, and usbguard device serials.
**No paths and no username.** So a project key passes through `$R` unchanged today,
and would still do so if path parity ever made the key a host path.

The check allows a `<TOKEN>` prefix anyway. If the map ever gains a path or username
entry, every row would fail the shape assertion on a completely correct run — and it
would fail at precisely the moment this section is meant to be showing that the key
changed. One alternation now is cheaper than diagnosing that later.

*Residual, named:* a leaked field that happens to render as an absolute path followed by
words would pass. That is narrow, and narrowing it further means guessing at content,
which is how the previous version got it wrong.

**`.projects` is keyed by the project path — `/work` today — which makes this section
its own check on ADR-0017's stable literal.** When `CLAUDE_CODE_PROJECT_DIR_NAME`
lands, the key changes; if it does not change, the literal did not take effect. That is
a happy accident of what the file stores, and worth naming because it is the only
observable this vault has for whether the literal worked.

*Still `inferred`:* `.type` on a server entry. `mcpServers` is `{}` for every project on
this machine, so there is no shape to read; the collector emits `<type-unread>` rather
than defaulting to a transport nobody verified, which makes a wrong field name visible
as a gap instead of a plausible value.

```yaml
id: collect-harness-config
requires: [collector-run]
postcondition: |
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  f=state/harness.md
  [ -r "$f" ] || { echo "RESULT=FAIL ($f is unreadable)"; exit 1; }
  for s in "## project trust" "## mcp servers" "## allowed tools"; do
    n=$(awk -v s="$s" '$0 == s { n++ } END { print n + 0 }' "$f")
    [ "$n" -eq 1 ] || { note "found $n '$s' headers in $f, want 1"; continue; }
    # The marker, inside this section, matched by anchored prefix rather than by
    # field position — two of the three markers carry a parenthetical, and a
    # positional test passes on those by accident. A section with no rows is a
    # legitimate state and reads identically to a jq field name that has moved;
    # the marker is what separates them, and it has already been wrong once.
    ok=$(awk -v s="$s" '
           $0 == s { insec = 1; next }
           /^## / { insec = 0 }
           insec && /^# source: projects present/ { ok = 1 }
           END { print ok + 0 }' "$f")
    [ "$ok" -eq 1 ] \
      || note "'$s' does not report 'projects present' — the collector may be reading a field that has moved, which is not the same as there being nothing to report"
  done
  # Structural, not a content sniff. Every data row the allowlist produces starts
  # with an absolute project path; a top-level oauthAccount, userID or machineID
  # rendered here would not. Matching on those field *names* would fire on
  # allowedTools rows, which are verbatim permission strings and legitimately
  # contain substrings like "oauth".
  #
  # A <TOKEN> prefix is allowed as well as /. redact.map holds no path or
  # username entries today (measured 2026-08-26 from gen-redact-map.sh), so a
  # project key survives redact.sh intact — but if it ever gains one, every row
  # would fail this check on a correct run, at exactly the moment the section is
  # meant to show the key changing. One alternation is cheaper than that.
  bad=$(awk '
    /^##/ { next }
    /^#/  { next }
    NF == 0 { next }
    $1 !~ /^(\/|<[A-Z_]+[A-Za-z0-9_]*>)/ { print }' "$f")
  if [ -n "$bad" ]; then
    printf '%s\n' "$bad" | sed 's/^/  not a project-keyed row: /'
    note "$f contains rows outside the extraction allowlist's shape"
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## Open against this chapter

- **`bootloader-install` was referenced here and defined nowhere, and the
  unresolved-`requires` table could not have found it.** Closed the same day —
  `spec/boot` gained [[boot#bootloader-install|`bootloader-install`]] and
  [[boot#bootloader-nvram-entry|`bootloader-nvram-entry`]] — but the finding
  outlives the fix. `bootctl install` and `loader.conf` are PRD §3.4, `spec/boot`
  promoted only the `mkinitcpio -P` half, and `uki-sign` was signing two files
  that existed only because of a step nobody wrote. **The table in
  `spec/_index.md` lists ids someone thought to reference**, so a missing step
  that nothing names is invisible to it, and the count that has been the
  round-over-round progress measure cannot see this class at all.
  PRD §4.2's account of why it is two steps — that "`bootctl install` inside
  `arch-chroot` cannot write EFI variables" — turned out to be wrong in both
  halves, and the real reason is recorded in `spec/boot`.
- **PRD §3's ordering requirement is lost in `spec/boot` and has no edge.** §3
  opens with it in bold: §3.6 (locale, keymap, `/etc/vconsole.conf`) must run
  before §3.4 generates the UKIs, because "mkinitcpio embeds
  `/etc/vconsole.conf` into the initramfs at build time". `uki-generate` carries
  no such edge. That claim is **`inherited-unverified`** — §3.4 lists
  `sd-vconsole: /etc/vconsole.conf not found` among its expected warnings, which
  is consistent with it but is not the same statement — and the lookup that
  closes it is `mkinitcpio -H sd-vconsole` on the machine, the channel that
  closed the hook-ordering gap in the previous round. `spec/boot` was
  deliberately not edited on the strength of an unverified claim.
- **~~Two postconditions in `spec/disk` abort where they should diagnose.~~
  Closed 2026-08-24, and there were not two.** Re-testing found **four**
  instances of the class in `spec/disk` and **six more across this chapter and
  `spec/boot`**, one of which — `cmdline-write`'s — had a diagnostic no input
  could reach. The general form is now the seventh convention in
  [[_schema#A postcondition must reach its verdict line on every input it is meant to judge|`_schema.md`]],
  every instance carries a written-down mutation set, and this chapter's two
  changed blocks are `user-create` and `first-boot`. **The deferral was right
  and the count in it was wrong**, which is the third time a count in this spec
  has read plausibly while being incomplete — after the dangling-id measure,
  twice.
- **`state/` records no group membership, so four of the five groups
  `user-create` sets are unassertable.** One `id luiswg`, or a collector
  section, closes it. The better question it enables is whether `audio`,
  `video`, `storage` and `input` do anything under seat management.
- **Whether root stays unlocked is undecided and unrecorded.** PRD §3.5 sets out
  both sides and explicitly declines to choose; `state/` has no `passwd -S root`.
  Deliberately not written into `user-create`. It wants an ADR — it is a choice
  with a cost that will be noticed later by someone who did not see the
  reasoning, which is the test `decisions/_index.md` states — and the collector
  change that would let the ADR cite the current state is one line.
- **`state/`'s fstab carries no `discard=async` while `## mounts` does.** Every
  other kernel-reported default that argued the "genfstab copies what the kernel
  reports" case — `ssd`, `space_cache=v2` — is present in both. Unexplained:
  either the install-time kernel did not report it, or genfstab filters it. It
  does not affect this chapter, since `discard=async` is a documented default at
  this kernel version and nothing in the spec writes it, but it weakens any
  future claim that fstab pins the btrfs half of ADR-0014.
- **`sof-firmware` is installed and PRD §2.1 calls it "absent and genuinely out
  of scope".** Not in the pacstrap set here and not asserted; noted only because
  it is a third small place where the document and the machine disagree.
- **Nothing in this chapter has been executed.** `fstab-generate`'s
  postcondition was tested against a synthesised `genfstab` output and eight
  mutations; `user-create` and `first-boot` gained sets of eight each on
  2026-08-24. That is the strongest form available under the standing
  limitation and is not the same as having run any of them. `first-boot`'s
  postcondition has still **not been run in any of the four namespaces it
  distinguishes** — the set exercises the logic that combines four stubbed
  readings, which is one step short. `base-install` has no set at all.
