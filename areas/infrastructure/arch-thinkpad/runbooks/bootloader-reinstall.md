---
title: "Reinstall the bootloader without unsigning it"
type: runbook
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-24
commit: pending
created: 2026-08-24
updated: 2026-08-24
tags: [runbook, secure-boot, systemd-boot, sbctl, efi, recovery]
---

# Reinstall the bootloader without unsigning it

**Unlike [[tpm2-reenroll|the TPM2 runbook]], this is not expected behaviour.**
Something removed the firmware's boot entry, or something replaced a signed
loader with an unsigned one. Those are two different faults with two different
repairs, and **the repair for one makes the other worse**, which is why the gate
below comes before the fix.

`bootctl install` copies fresh binaries into the ESP (*measured*, `bootctl(1)`,
read 2026-08-24: install "Installs **systemd-boot** into the EFI system
partition. A copy of **systemd-boot** will be stored as the EFI default/fallback
loader at *ESP*/EFI/BOOT/BOOT\*.EFI"). Fresh binaries are unsigned. On a machine
with Secure Boot enabled, running it alone turns a missing menu entry into a
machine that will not boot at all.

> **The one rule to carry out of this page.** If you run `bootctl install`, you
> run `sbctl sign-all` after it. Every time, no exceptions. The gate below only
> decides whether you need `bootctl install` at all.

**PRD §9.6 gets this wrong.** Its recovery row for "Firmware update bricked boot
entries" is `bootctl install` from the ISO chroot, then `mkinitcpio -P`, with no
signing step. Followed literally on this machine that procedure leaves the
firmware rejecting its own loader. Use this page instead.

## How it presents

Two faults, and they look nothing alike.

**A — the machine boots, but the named entry is gone.** The firmware boot menu
no longer lists "Linux Boot Manager", or it boots straight through without
offering it. Everything works. The machine is reaching the ESP through the
removable-media fallback at `EFI/BOOT/BOOT*.EFI`, which Lenovo firmware accepts.
Typical causes: a firmware update, an NVRAM clear, a CMOS battery event.
**Urgency: none.** You have lost a named entry and a boot-order position.

**B — the machine does not boot.** The firmware refuses the loader and returns
to its own setup screen or an error, with no systemd-boot menu. With Secure Boot
enabled and an unsigned loader on the ESP, this is what that looks like
(*inferred* — the mechanism is documented, this machine's firmware has not been
observed in this state). Typical cause: something ran `bootctl install` after
the loader was signed. Following §9.6 is the most likely way to arrive here.
**Urgency: high, and you need the Arch ISO.**

**A then B** is the common trajectory: fault A appears, §9.6 is followed, fault
B results.

## Confirm which one you have

Skip to [From the Arch ISO](#from-the-arch-iso) if the machine will not boot —
fault B is established by the fact that you are reading this on another device.

Otherwise, both readings, before touching anything:

```bash
# Is anything on the ESP unsigned? Empty output means everything is signed.
sudo sbctl verify --json | jq -r '.[] | select(.is_signed != 1) | .file_name'

# Is the firmware's own list still carrying the entry?
sudo bootctl status | grep -E '^[[:space:]]*Title:'
```

The second command's expected output includes a line reading exactly `Title:
Linux Boot Manager`. **`Fallback Linux Boot Manager` is a different entry and
does not count** — it contains the same words, so read the whole line rather
than scanning for the phrase.

| First command | Second command | Fault | Do |
|---|---|---|---|
| empty | no `Linux Boot Manager` | **A** | `bootctl install`, then `sbctl sign-all` |
| lists the loader binaries | has `Linux Boot Manager` | **B** | `sbctl sign-all` **only** |
| lists the loader binaries | no `Linux Boot Manager` | **A + B** | `bootctl install`, then `sbctl sign-all` |
| empty | has `Linux Boot Manager` | neither | **Stop.** Nothing here is broken; whatever sent you to this page is something else |

That last row matters. If both readings are clean, running `bootctl install`
*creates* fault B on a machine that did not have it.

## Repair, from the booted system

```bash
# Only if the gate said you need it. Never on its own.
sudo bootctl install

# Re-signs everything already in sbctl's database, including the two loader
# binaries bootctl just replaced.
sudo sbctl sign-all
```

`sign-all` "Signs all enrolled EFI binaries" (*measured*, `sbctl(8)`, read
2026-08-24). It works because [[../spec/boot#uki-sign|`uki-sign`]] signs with
`-s`/`--save` — "Save file to the database" — so every file the build signed is
already enrolled and does not have to be named again under pressure. That is the
payoff of the `-s` in the spec, and the reason this page is four commands rather
than eight paths.

**`sign-all` on an empty database signs nothing and does not complain.** Check
first if there is any chance the key store or database was lost:

```bash
sudo sbctl list-files    # must not be empty
```

## From the Arch ISO

The fault-B path, and the one §9.6 sends you down. Boot the ISO, then:

```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
# No -o subvol=: this mounts the btrfs default subvolume, which is what the
# machine actually boots. PRD §9.5 says `-o subvol=@`, which mounts the
# pre-rollback root instead — a different filesystem with a different
# /var/lib/sbctl. See spec/disk#btrfs-default-matches-root.
mount /dev/mapper/cryptroot /mnt
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt

bootctl install
sbctl sign-all
```

**`bootctl install` tells you nothing about whether it worked here.**
`--graceful` "Ignore[s] failure when the EFI System Partition cannot be found,
when EFI variables cannot be written, or a different or newer boot loader is
already installed… **Since version v258, it is implicitly enabled when running
inside a chroot**" (*measured*, `bootctl(1)`, read 2026-08-24), and this system
runs systemd 261.2. So it exits 0 having done nothing, if `/mnt/boot` was not
mounted or was the wrong partition. Verify the files, not the exit status:

```bash
ls -l /boot/EFI/systemd/systemd-boot*.efi /boot/EFI/BOOT/BOOT*.EFI
sbctl verify --json | jq -r '.[] | select(.is_signed != 1) | .file_name'
```

Then leave and reboot:

```bash
exit
umount -R /mnt
reboot
```

**The firmware entry may still be missing after this.** The same implicit
`--graceful` swallows a failed EFI variable write, so the chroot repair fixes
the ESP reliably and the NVRAM entry only maybe. Once booted, run the booted-
system repair above if the gate still reports it absent. The machine will boot
either way, through the fallback path.

## Verify

Two assertions, duplicated from
[[../spec/boot#uki-sign|spec/boot#uki-sign]] and
[[../spec/boot#bootloader-nvram-entry|spec/boot#bootloader-nvram-entry]].
**All copies must move together.**

```bash
# Everything on the ESP is signed. sbctl verify exits 0 even when it reports an
# unsigned file, so never test $?.
out=$(sudo sbctl verify --json)
total=$(jq 'length' <<<"$out")
signed=$(jq '[.[] | select(.is_signed == 1)] | length' <<<"$out")
if [ "$total" -gt 0 ] && [ "$signed" -eq "$total" ]; then
  echo "RESULT=PASS ($signed/$total signed)"
else
  echo "RESULT=FAIL ($signed/$total signed)"
  jq -r '.[] | select(.is_signed != 1) | .file_name' <<<"$out"
fi

# The firmware's list carries the entry. Anchored: `Fallback Linux Boot Manager`
# contains `Linux Boot Manager`.
sudo bootctl status | grep -qE '^[[:space:]]*Title:[[:space:]]+Linux Boot Manager$' \
  && echo RESULT=PASS || echo "RESULT=FAIL (no 'Linux Boot Manager' entry)"
```

**Why these are duplicated rather than transcluded.** The general limit is in
[[../spec/_index|spec/_index.md]]: a step record is one fenced block, so
embedding a section brings its `command:` along with its `postcondition:`.

The reason differs from [[tpm2-reenroll|tpm2-reenroll]]'s, and the difference is
worth knowing. There, the spec's command was actively dangerous to run from the
runbook — it lacked `--wipe-slot=tpm2` and would have destroyed a working
enrollment. Here both spec commands are *correct*, and transcluding them would
have been safe. What makes duplication right anyway is narrower: this page must
present the two repairs in a fixed order, gate them on a diagnosis, and offer a
chroot variant that neither step record covers. Transcluding two sections whose
prose argues for a booted-system context, into a page read at a firmware error
screen, trades the wrong thing.

So: not a blanket rule against transclusion. The question is always what each
option costs *when it fails*.

## Afterwards

If Secure Boot is enabled and the loader was ever unsigned, re-run the collector
so `state/` records the repaired state, and commit — the diff is the record that
this happened.

```bash
cd ~/dev/infra-docs/arch-thinkpad && ./collect/collect.sh
```

If this fired because of a firmware update, [[tpm2-reenroll|the TPM2
enrollment]] is almost certainly invalid too — PCR 0 moves on every firmware
update by design. Expect a passphrase prompt on the next boot and run that
runbook as well.
