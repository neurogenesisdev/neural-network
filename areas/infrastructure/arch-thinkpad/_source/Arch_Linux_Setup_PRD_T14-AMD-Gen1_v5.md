# Arch Linux Setup PRD — ThinkPad T14 Gen 1 AMD (20UD/20UE) v5

> **Companion document:** `T14-System-State-Reference.md` records what the machine actually looks like after the build — exact hardware, versions, services, config locations, and the open items from this PRD that were never applied. This document is the runbook; that one is the current state. Keep them separate.

**Target system:** Lenovo ThinkPad T14 Gen 1 AMD · Ryzen 5 PRO 4650U (Renoir, Zen 2, 6C/12T) · 32 GB RAM · integrated Radeon Vega 7 · single NVMe (model TBD — see §0.3)

**Scope of this document:** base operating system only. Bootable, encrypted, snapshotted, hibernating, hardened, backed up, and maintainable. It ends at a verified base system with no desktop environment.

**Explicitly out of scope (v4 change):** desktop environment, display manager, audio/Bluetooth stack, GPU driver installation, gaming stack, development tooling, AUR helper. Those were §4, §5, §10, and §11 of v3 and are deleted, not deferred. Add them afterward as separate decisions.

**Stated goals:** full-disk encryption with a verified boot chain, btrfs with snapshots, working hibernation, CLI-first operation, low telemetry, long-term maintainability.

---

## v4 → v5 changelog

Corrections found by running the build and then auditing the result (2026-08-13).

- **`gptfdisk` and `dmidecode` added to §2.1.** `sgdisk` is used in §1.1 but was never installed on the target system, so it is unavailable for later partition inspection. `dmidecode` is needed for hardware inventory.
- **§3.5 reordered.** `passwd` must be verified *before* locking root. See the new verification step — an unset user password plus a locked root leaves no way into the booted system.
- **§3.6 must run before §3.1–§3.4.** Already noted in v4; restated here because it was missed during execution.
- **§2.2 rewritten** — `genfstab` emits its own swap line when the swapfile is active, so the fstab entry must be *edited*, not appended.
- **ESP mask fix folded into §1.4.** genfstab writes `fmask=0022,dmask=0022`, which makes the systemd-boot random seed world-readable. Set `0077` at mount time rather than fixing it after the fact.
- **§5.3 `MAKEFLAGS` marked optional.** It was never applied on the built system and nothing broke.
- **New §4.2** covering the post-install verification pass, and a list of PRD sections most likely to be skipped.
- **§12.5 build record updated** with verified outcomes and the operational failures encountered.

## v3 → v4 changelog

### Hardware retarget (Dell XPS 9570 → ThinkPad T14 AMD Gen 1)

- **Entire NVIDIA layer deleted.** No dGPU, no Optimus, no `nvidia-580xx-dkms`, no DKMS-on-kernel-upgrade fragility, no `NVreg_PreserveVideoMemoryAllocations`, no `nvidia-suspend/hibernate/resume` services, no initramfs pacman hook, no `prime-run`. The cmdline loses `nvidia-drm.modeset=1 nvidia-drm.fbdev=1`; `MODULES` stays `(btrfs)`.
- **The Driver trust zone is gone.** v3 had four zones with the legacy NVIDIA stack named as the fragile one. This machine has no equivalent. §16.1's "Pascal exit ramp" is deleted outright.
- **`intel-ucode` → `amd-ucode`.** The `microcode` mkinitcpio hook and the `ALL_microcode=(/boot/*-ucode.img)` glob are unchanged.
- **BIOS table rewritten for Lenovo.** The XPS-specific SATA/AHCI mode and "Fastboot: Thorough" settings do not exist on this firmware. ThinkPads since ~2020 have no CSM at all, so there is no UEFI/Legacy toggle to set.
- **Swapfile 20 GiB → 36 GiB** for 32 GB RAM, same reasoning as v3.
- **`amd_pstate=active` was added, tested on the hardware, and removed.** This firmware does not expose the `_CPC` ACPI object, so the driver cannot bind and the kernel falls back to `acpi-cpufreq` regardless. See §3.2.
- **ThinkPad battery charge thresholds added** to the TLP config — natively supported via `thinkpad_acpi` on this generation.
- **restic repository path** `xps9570` → `t14`.

### Corrections to v3 as written

- **`dosfsutils` is not a package.** The correct name is `dosfstools`. As written, v3's pacstrap line aborts. This is an install-blocker that must have been fixed by hand during the XPS run and never made it back into the document.
- **`--sector-size 4096` is no longer hardcoded.** It is now conditional on what the drive actually reports (§0.3, §1.2). Forcing 4096 on a drive formatted 512n is a silent misalignment, not a win.
- **Multilib removed.** v3 enabled it in §2.1 solely to satisfy `lib32-*` graphics and Steam packages. With the gaming stack out of scope, multilib has no consumer. Enable it later if and when you install something 32-bit.
- **zram sizing reconsidered.** `ram / 2` on a 16 GB machine gave 8 GB of compressed swap. The same ratio on 32 GB gives 16 GB, which is well past the point of usefulness and inflates the hibernation image. Now `ram / 4`, capped.
- **MAKEFLAGS rationale rewritten.** v3 justified `-j$(nproc --ignore=1)` by the memory pressure of `nvidia-580xx-dkms` rebuilds on a 16 GB machine. That workload no longer exists and there is twice the RAM. The conservatism is now about thermals in a 15 W chassis, which is a weaker but still real argument.

### Structural change

- **New §0.3: hardware ground truth.** v3 hardcoded facts about known hardware. This machine is new to you and secondhand, so several values in this document are placeholders that must be replaced with measured output before use. They are marked `<LIKE_THIS>` throughout and collected in the §0.3 worksheet.

---

## §0 — Pre-Boot

### §0.1 — Firmware update (do this first)

Check the installed BIOS level before touching anything else. Lenovo firmware updates commonly reset BIOS settings to defaults, so updating *after* configuring §0.2 means doing §0.2 twice.

From Windows, if it is still installed:

```powershell
Get-CimInstance Win32_BIOS | Select SMBIOSBIOSVersion, ReleaseDate
Get-CimInstance Win32_ComputerSystemProduct | Select Name, Version
```

The BIOS string for 20UD/20UE is the **R1C** series (`R1CETxxW`). Lenovo bundles the Embedded Controller program with it, so one version number covers both. Compare against Lenovo Commercial Vantage, which resolves your machine type automatically.

From the Arch ISO instead, if Windows is already gone:

```bash
pacman -Sy fwupd
fwupdmgr refresh
fwupdmgr get-devices
fwupdmgr update
```

This model is on LVFS and the fwupd path is reliable. Run it on AC power — Lenovo firmware refuses to flash below roughly 25% battery. The update stages to the ESP and applies on the next reboot, with one or two automatic power cycles and a black screen with a progress bar. Do not interrupt it.

### §0.2 — BIOS settings

F1 at the ThinkPad logo for setup, F12 for the one-time boot menu.

| Setting | Value | Reason |
|---|---|---|
| Security → Secure Boot | Disabled | Arch ISO is unsigned; re-enabled in §15.1 with your own keys |
| Security → Security Chip | Enabled (fTPM) | Required for §15.2 TPM2+PIN unlock |
| Security → Virtualization → SVM | Enabled | Cheap now, annoying to discover missing later |
| Security → Virtualization → IOMMU | Enabled | Same |
| Config → Power → Sleep State | Linux, if present | Selects S3 deep sleep over s2idle; see note below |
| Config → Keyboard/Mouse → F1–F12 | Primary function (preference) | Quality of life |
| Security → I/O Port Access → SD card reader | Disabled if unused | Attack surface |

Not present on this firmware, do not go looking: UEFI/Legacy boot mode (CSM was removed from ThinkPads years ago — the machine is UEFI-only by design), SATA operation mode, and "Fast BIOS Mode." Lenovo's `Startup → Boot Mode → Quick/Diagnostic` only controls whether POST shows the splash screen; it is cosmetic.

The Sleep State option may be absent on the AMD Gen 1 depending on firmware level. If you do not see it, do not hunt for it — verify from the installed system instead:

```bash
cat /sys/power/mem_sleep
```

`[s2idle] deep` means S3 is available and selectable. A bare `[s2idle]` means the firmware only offers modern standby. Either way this does not affect hibernation, which is what this document actually configures.

Set a supervisor password last, if at all. Lenovo's recovery path for a forgotten one is genuinely painful.

### §0.3 — Establish hardware ground truth

**Do this before §1.** Several values below are unknown for your specific unit. Boot the Arch ISO and record the real answers; every `<PLACEHOLDER>` in this document maps to one of these.

```bash
# UEFI mode — must return a non-empty listing
ls /sys/firmware/efi/efivars | head

# Storage: device name, model, size
lsblk -o NAME,SIZE,MODEL,TYPE
nvme list

# Storage: LBA format — decides §1.2 sector-size
nvme id-ns -H /dev/nvme0n1 | grep -A1 "LBA Format"

# Network hardware — confirms the wireless driver
lspci -k | grep -A3 -i network

# CPU and memory sanity check
lscpu | grep -E "Model name|Core|Thread"
free -h

# Everything else worth knowing
lspci -nnk
```

**Worksheet — fill in before proceeding:**

| Placeholder | Where used | Your value |
|---|---|---|
| `/dev/nvme0n1` | §1.1 onward | **`/dev/nvme0n1`** — Micron 2210 MTFDHBA1T0QFD, 953.9 GiB, FW P6MA000, SN <NVME_SERIAL> |
| `<LBA_DECISION>` | §1.2 | **512 only — omit `--sector-size`.** Format 0 is the sole format offered and is in use |
| LUKS UUID | §3.2 cmdline | ✅ `<FSUUID_nvme0n1p2>` |
| resume offset | §3.2 cmdline | ✅ `533760` |
| `<HOSTNAME>` | §3.6 | |
| Wireless chipset | §0.4 | RTL8852AE → `rtw89_8852ae` ✓ |
| CPU / RAM | confirmation | Ryzen 5 PRO 4650U, 6C/12T, 30 GiB usable ✓ |

**Drive notes.** The Micron 2210 is a **QLC** drive, DRAM-less, PCIe Gen3 x4, rated 360 TB TBW at this capacity, with an SLC write-acceleration cache in front of the QLC array. Two consequences for this build:

- Sustained large writes fall off sharply once the SLC cache saturates. The hibernation image lands in exactly that pattern, so expect hibernate and resume to take noticeably longer than on a TLC drive. This is a performance characteristic, not a fault.
- Keeping zram as the primary swap tier (§6.3, `pri=100` versus the swapfile's `pri=10`) matters more here than it would on TLC — it keeps routine swap traffic off the QLC entirely.

360 TBW is roughly 200 GB/day sustained over five years, so hibernation write volume is not an endurance concern in itself. The open question is how much of that budget the previous owner already spent — see the SMART baseline below.

**Establish the SMART baseline before partitioning.** On secondhand hardware this is the last cheap moment to discover the drive is worn out:

```bash
nvme smart-log /dev/nvme0n1
```

Record `percentage_used`, `data_units_written` (×512 KB for bytes), `power_on_hours`, and `media_errors`. Anything above roughly 20% used, or a write total that is a large fraction of 360 TB, is grounds for budgeting a replacement drive now rather than discovering it in a year.

**On the LBA format.** The `nvme id-ns -H` output lists each supported format with its data size and a relative performance rating; the one currently in use is flagged `(in use)`. Three cases:

1. Drive is already using a 4096-byte format → pass `--sector-size 4096` in §1.2.
2. Drive supports 4096 but is using 512 → you may reformat to the 4 KiB format before partitioning. This is **destructive** and irreversible in the sense that it wipes the drive: `nvme format --lbaf=<N> /dev/nvme0n1`. Many OEM drives ship 512e despite supporting 4 KiB, so this is a real and free improvement. Verify the drive is not the one holding anything you want.
3. Drive only offers 512 → omit `--sector-size` entirely and take the default. Do not force a mismatch.

### §0.4 — Networking in the live environment

The Arch ISO ships `iwd`, not NetworkManager.

```bash
loadkeys de

iwctl
# Inside [iwd]#:
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YOUR_SSID
exit

ping -c 3 archlinux.org
timedatectl set-ntp true
```

**This unit has a Realtek RTL8852AE**, not the Intel AX200 that ships in most T14 AMD Gen 1 machines. Driver is `rtw89_8852ae`, in-tree since kernel 5.16, with firmware in the `linux-firmware-realtek` split package (§2.1). Wireless working in the live environment confirms both driver and firmware are present — but see §4.1, because the rtw89 stack has stability quirks that only surface after sustained use.

### §0.5 — Trust zones

Three zones, down from v3's four. The Driver zone is deleted along with the hardware that justified it.

| Zone | Components | Posture |
|---|---|---|
| **Base OS** | kernel, initramfs, systemd, btrfs, LUKS, NetworkManager, nftables, AppArmor, usbguard | Encrypted at rest, signed at boot, MAC-confined, recoverable from snapshot plus external backup |
| **Userland** | anything installed after §14 | Out of scope for this document; assess per-package when you add it |
| **Firmware** | UEFI, EC, fTPM, NVMe firmware | Updated via fwupd/LVFS; trusted but not verifiable by you |

---

## §1 — Disk Layout and Partitioning

### §1.1 — Partition table

| Partition | Size | Type | Purpose |
|---|---|---|---|
| `/dev/nvme0n1p1` | **1 GiB** | EFI System (ef00) | Unencrypted ESP, FAT32 — sized for four UKIs plus headroom |
| `/dev/nvme0n1p2` | Remainder | Linux root x86-64 (8304) | LUKS2 container |

```bash
sgdisk -Z /dev/nvme0n1
sgdisk -n 1:0:+1G -t 1:ef00 -c 1:EFI /dev/nvme0n1
sgdisk -N 2 -t 2:8304 -c 2:cryptroot /dev/nvme0n1
partprobe /dev/nvme0n1
lsblk /dev/nvme0n1
```

The ESP is mounted at `/boot`, not `/efi`. With UKIs the kernel images live on the ESP itself, so the simpler single-mount layout is correct here.

### §1.2 — LUKS2 with explicit parameters

The defaults are sensible; for a security-first install, state them.

```bash
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  /dev/nvme0n1p2

cryptsetup open /dev/nvme0n1p2 cryptroot
ls /dev/mapper    # confirm 'cryptroot' exists
```

**Resolved:** `--sector-size` is omitted. This drive offers LBA Format 0 (512-byte) only, so there is nothing to align to and forcing 4096 would create a mismatch rather than a performance win. Do not reintroduce the flag from the v3 document.

**Discard policy, decided now rather than later:** this document does not pass `--allow-discards` at format time and does not include `rd.luks.options=discard` in the cmdline. Online TRIM through dm-crypt leaks block-allocation patterns to the SSD controller. Combined with omitting `discard=async` at the btrfs layer (§1.4), this is a deliberate privacy-leaning trade; SSD wear is managed by weekly batch TRIM via `fstrim.timer` instead.

If you later prefer continuous TRIM and accept the metadata leak, `cryptsetup refresh --allow-discards --persistent` applies it post-install without reformatting.

### §1.3 — Btrfs filesystem and subvolumes

```bash
mkfs.btrfs -L system /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache

btrfs subvolume list /mnt    # verify all six

# CRITICAL: unmount the top-level before mounting subvolumes.
# Otherwise each subvolume appears twice in /proc/mounts.
umount /mnt
```

### §1.4 — Mount subvolumes

Btrfs mount options such as `compress`, `nodatacow`, and `nodatasum` are filesystem-wide, not per-subvolume — only the first-mounted subvolume's options take effect across the whole filesystem. Putting `nodatacow,nodatasum` on `@swap`'s mount line is decorative. The swapfile's CoW exemption comes from the `NODATACOW` **file attribute** set by `btrfs filesystem mkswapfile`, not from mount options.

Also omitted deliberately: `space_cache=v2` (default for new filesystems since kernel 5.15; the option is a no-op), `ssd` (auto-detected), `discard=async` (using `fstrim.timer` instead — pick one, not both).

```bash
# Root @
mount -o rw,noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt

mkdir -p /mnt/{boot,home,swap,.snapshots,var/log,var/cache}

mount -o rw,noatime,compress=zstd:1,subvol=@home      /dev/mapper/cryptroot /mnt/home
mount -o rw,noatime,subvol=@swap                       /dev/mapper/cryptroot /mnt/swap
mount -o rw,noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o rw,noatime,compress=zstd:1,subvol=@log       /dev/mapper/cryptroot /mnt/var/log
mount -o rw,noatime,compress=zstd:1,subvol=@cache     /dev/mapper/cryptroot /mnt/var/cache

# ESP
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
mount -o fmask=0077,dmask=0077 /dev/nvme0n1p1 /mnt/boot
```

**Mount the ESP with `fmask=0077,dmask=0077`.** The default `0022` makes `/boot` world-readable, including the random seed systemd-boot writes in §3.4, which produces a security warning on every `bootctl` run. Setting it here means genfstab picks it up and there is nothing to fix later. Note that vfat applies these at mount time only — changing them afterwards requires a full `umount`/`mount` cycle, not a remount.

`noatime` eliminates write amplification from access-time updates and implies `nodiratime`. `compress=zstd:1` is the compression/throughput balance point and applies filesystem-wide.

### §1.5 — Verify mounts

```bash
findmnt -nt btrfs -S /dev/mapper/cryptroot
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
```

Expect six btrfs entries with no duplicates. Duplicates mean the top-level mount was not unmounted before the subvolume mounts; fix with `umount -R /mnt` and redo from §1.3's closing `umount`.

### §1.6 — Swapfile sized for hibernation

**Sizing:** hibernation requires swap ≥ RAM as a floor, not a target. Real systems hibernate with RAM partly occupied by session state, browser and editor caches, page cache pressure, and zram-compressed pages that must be re-expanded onto disk. With 32 GB RAM, **use 36 GiB.** The margin is free on a modern NVMe and prevents resume failures under memory pressure.

```bash
btrfs filesystem mkswapfile --size 36g --uuid clear /mnt/swap/swapfile
swapon /mnt/swap/swapfile
swapon --show
```

`btrfs filesystem mkswapfile` (kernel ≥ 6.1) sets the `NODATACOW` attribute automatically, validates single-device single-profile placement, and is the canonical modern method. **That file attribute — not the parent subvolume's mount options — is what makes the swapfile usable at all.**

Record the resume offset for §2.5.2:

```bash
btrfs inspect-internal map-swapfile -r /mnt/swap/swapfile
# → 533760
```

And the LUKS container UUID:

```bash
blkid -s UUID -o value /dev/nvme0n1p2
# → <FSUUID_nvme0n1p2>
```

---

## §2 — Base Installation

### §2.1 — pacstrap

```bash
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
```

Notes on specific entries:

- `amd-ucode` — Renoir microcode. Bundled into the UKI by the `microcode` hook.
- `linux-firmware-realtek` — **required for this unit's RTL8852AE wireless.** `linux-firmware` has been split into per-vendor packages; the metapackage normally pulls the vendor subpackages in, but naming this one explicitly costs nothing and removes the failure mode where you reboot into a system with no network and no way to fetch the firmware that would give you one. It also carries the `rtl_bt` blobs if you add Bluetooth later.
- `linux-lts` alongside `linux` — a single-kernel Arch install is one regression away from a paperweight. Both kernels, both UKIs, both boot entries. Costs roughly 150 MB and one menu line. On the XPS this doubled as the NVIDIA safety valve; here it is purely regression insurance, which is still reason enough.
- `dosfstools` — provides `mkfs.fat`. v3 said `dosfsutils`, which does not exist.
- `mtools` — omitting it produces cryptic UKI generation errors.
- `pacman-contrib` — `pacdiff`, `paccache`, `checkupdates`.
- `smartmontools nvme-cli` — SSD health monitoring, and worth having on a secondhand machine specifically.
- `gptfdisk` — provides `sgdisk`, used in §1.1. The live ISO has it; the installed system does not unless you ask. Omitting it means you cannot inspect your own partition table later.
- `dmidecode` — board, BIOS, and per-DIMM detail. Needed for any hardware inventory.
- `sbctl` — Secure Boot key management for §12.1.
- `openssh` — the **client**, for reaching your Hetzner box. An earlier draft cut this on the grounds that `sshd` shouldn't be enabled reflexively; that argument applies to the daemon, not the package. Install it, leave the service disabled (§7.2).
- `nftables` — §7.1 writes the ruleset and enables the unit but assumed the package was present. It isn't in `base`.
- `terminus-font` — readable console before any desktop exists, including at the LUKS prompt.
- `linux-firmware` is a metapackage with hard dependencies on the per-vendor subpackages, so amdgpu firmware for Renoir arrives without being named.
- No multilib. Nothing here is 32-bit.

Still deliberately absent, installed in their own sections after first boot: `apparmor` (§7.4), `usbguard` (§7.5), `tlp` (§8.1), `snapper`/`snap-pac` (§9.1), `restic` (§9.4). Also absent and genuinely out of scope: `sof-firmware`, which you will want when audio enters scope — remember it exists rather than debugging silence later.

### §2.2 — genfstab and post-processing

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

**Strip `subvolid=`** — it breaks `snapper rollback`, which creates a new subvolume ID that a pinned fstab will not follow:

```bash
sed -i 's/subvolid=[0-9]*,//g' /mnt/etc/fstab
```

**Set the swapfile priority.** Because the swapfile was activated with `swapon` in §1.6, `genfstab` detects it and writes its own entry — but with bare `defaults` and no priority. Check first:

```bash
grep swap /mnt/etc/fstab
```

Two lines are expected: the `@swap` subvolume mount, and `/swap/swapfile none swap defaults 0 0`. **Edit the existing line rather than appending a second one:**

```bash
sed -i '/^\/swap\/swapfile/s/defaults/defaults,pri=10/' /mnt/etc/fstab
```

Only if `genfstab` produced no swap line at all — for instance if the swapfile was never activated — append one manually:

```bash
echo "/swap/swapfile none swap defaults,pri=10 0 0" >> /mnt/etc/fstab
```

zram is created at priority 100 (§6.3), so the kernel prefers it; the disk swapfile at `pri=10` serves hibernation and last-resort overflow. Without an explicit priority both devices land at the same default and the kernel may select the QLC disk over compressed RAM. Hibernation writes to the `resume=` device explicitly, so this ordering does not interfere with it.

Expected final shape — verify before chroot:

```
UUID=<uuid>  /            btrfs  rw,noatime,compress=zstd:1,subvol=/@           0 0
UUID=<uuid>  /home        btrfs  rw,noatime,compress=zstd:1,subvol=/@home       0 0
UUID=<uuid>  /swap        btrfs  rw,noatime,subvol=/@swap                       0 0
UUID=<uuid>  /.snapshots  btrfs  rw,noatime,compress=zstd:1,subvol=/@snapshots  0 0
UUID=<uuid>  /var/log     btrfs  rw,noatime,compress=zstd:1,subvol=/@log        0 0
UUID=<uuid>  /var/cache   btrfs  rw,noatime,compress=zstd:1,subvol=/@cache      0 0
UUID=<uuid>  /boot        vfat   rw,relatime,fmask=0022,dmask=0022,...          0 2
/swap/swapfile  none      swap   defaults,pri=10                                0 0
```

### §2.3 — Chroot

```bash
arch-chroot /mnt
```

---

## §3 — Boot Chain: UKI from Day One

> **Ordering requirement.** Complete §3.6 (locale, keymap, hostname) **before** §3.4 generates the UKIs. `mkinitcpio` embeds `/etc/vconsole.conf` into the initramfs at build time, so if that file does not yet exist the first UKI ships with no keymap and no console font. With `KEYMAP=us` the keymap half of that is harmless (US is the fallback anyway), but the Terminus font would be missing and any future keymap change would silently fail to reach the prompt. The sections are numbered in dependency order for reading, not execution order — run §3.6 first, then §3.1 through §3.4.

mkinitcpio generates a **Unified Kernel Image** — one `.efi` containing kernel, initramfs, microcode, and cmdline. systemd-boot auto-discovers UKIs in `EFI/Linux/`. The 1 GiB ESP is sized for this. When you sign for Secure Boot in §15.1, you sign exactly one file per kernel and the cmdline is embedded and tamper-evident.

### §3.1 — Configure mkinitcpio presets

Edit `/etc/mkinitcpio.d/linux.preset`:

```ini
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default' 'fallback')

#default_image="/boot/initramfs-linux.img"
default_uki="/boot/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Comment out `default_image` and `fallback_image`. Repeat for `/etc/mkinitcpio.d/linux-lts.preset` with `arch-linux-lts.efi` and `arch-linux-lts-fallback.efi`.

### §3.2 — Kernel cmdline

Create `/etc/kernel/cmdline` as a single line:

```
rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
```

Both values are already substituted above, measured on this machine at §1.2 and §1.6. This is embedded into the UKI at `mkinitcpio -P` time and cannot be edited from the boot menu, which is part of the security model. **Both the default and fallback UKIs receive the resume parameters** — a hibernation image is silently discarded if you boot an entry lacking them, producing a fresh boot with the previous session lost and dirty filesystem state.

Parameters worth explaining:

- **`amd_pstate=active` was tried on this machine and removed.** It is inert here: Renoir mobile uses the shared-memory CPPC path rather than MSR, and Lenovo's firmware does not expose the `_CPC` ACPI object, so the driver cannot bind and the kernel silently falls back to `acpi-cpufreq`. `lscpu | grep -i cppc` returns nothing on this unit; `dmesg` reports `the _CPC object is not present in SBIOS or ACPI disabled`. This is reported by other 4650U owners on both the T14 G1 and the EliteBook 845 G7, on current firmware — it is a hardware/firmware limitation, not a misconfiguration. The kernel additionally disables the shared-memory path by default because of measured regressions on those designs. `acpi-cpufreq` with the `schedutil` governor is the correct driver for this chip; the practical difference on a 15 W part is small. The parameter was dropped rather than left in place, because an inert flag in the cmdline is misleading a year later.
- `lsm=…apparmor…` — this is what actually loads AppArmor (§8.4).
- `slab_nomerge` — prevents same-size slab cache merging, mitigating use-after-free exploitation.
- `init_on_alloc=1 init_on_free=1` — zeros memory on allocation and free, reducing info-leak surface. Small performance cost, accepted.
- `randomize_kstack_offset=on` — per-syscall kernel stack randomization.
- `vsyscall=none` — removes the legacy fixed-address vsyscall page as ROP gadget surface.

### §3.3 — mkinitcpio HOOKS

Edit `/etc/mkinitcpio.conf`:

```
MODULES=(btrfs)
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

- `keyboard` loads input-device kernel modules and is **required** to type the LUKS passphrase at all. Do not remove it — without it, some firmware gives you no input whatsoever at the prompt. `sd-vconsole` applies the keymap and font from `/etc/vconsole.conf`.
- This machine uses `KEYMAP=us`: the keyboard is physically QWERTY (the German stickers are being removed), and AltGr behaves poorly in the early-boot console, which makes US the pragmatic choice. A consequence worth knowing: because US is also the fallback when `sd-vconsole` fails, a keymap misfire is now silent rather than locking you out. That is a real robustness gain over a QWERTZ setup.
- `block` explicitly adds block-device modules. With `autodetect` this is often a no-op, but the fallback initramfs needs it when hardware changes — including USB recovery boots of the installed system.
- `microcode` bundles `amd-ucode` into the initramfs (mkinitcpio 38+), replacing the legacy separate `initrd` line.

### §3.4 — Generate UKIs and install systemd-boot

```bash
bootctl install
mkinitcpio -P
```

Warnings you can expect and safely ignore during any `mkinitcpio` run on this machine:

- `Possibly missing firmware for module: 'xhci_pci_renesas'` — firmware for a Renesas USB controller this laptop does not have. Permanent and harmless.
- `sd-vconsole: /etc/vconsole.conf not found` — only appears before §3.6 has written that file. If it persists *after* §3.6, the German keymap will not reach the LUKS prompt and it needs fixing.
- `Secureboot key directory doesn't exist, not signing!` — sbctl's pacman hook running before §12.1 has created keys. Expected until then.

`/boot/loader/loader.conf`:

```
default arch-linux.efi
timeout 3
console-mode max
editor no
```

`editor no` prevents cmdline editing at the boot menu — important once the UKIs are signed.

```bash
ls /boot/EFI/Linux/    # four .efi files expected
bootctl status
```

### §3.5 — Users and sudo

```bash
useradd -m -G wheel,audio,video,storage,input -s /bin/bash luiswg
passwd luiswg
EDITOR=nvim visudo    # uncomment: %wheel ALL=(ALL:ALL) ALL
```

**Verify before locking root.** `passwd` silently does nothing if the two entries do not match, leaving the account in the locked state `useradd` created. Combined with a locked root, that produces a system with no way in.

```bash
passwd -S luiswg                          # must show P, not L
grep -c '^%wheel ALL=(ALL:ALL) ALL' /etc/sudoers   # must be 1
su - luiswg -c 'sudo -v && sudo whoami'   # must print root
```

If `sudo` rejects a password you are sure is correct, it is `pam_faillock`: three failures lock the account for ten minutes and then reject the *correct* password too. Clear it and retry:

```bash
faillock --user luiswg --reset
```

This will look identical at the login prompt after first boot, so it is worth recognising now.

Only once all three checks pass:

```bash
passwd -l root                            # see tradeoff below
```

**On locking root.** `passwd -l` makes the root hash unmatchable, so root cannot log in by password anywhere; you reach it through `sudo` from the wheel account. This removes root as a direct target.

The cost is that systemd's `emergency.target` and `rescue.target` prompt for the root password via `sulogin`, which refuses to run against a locked account. That recovery path is gone. What remains: the fallback UKI (§3.1), the `linux-lts` entry, and the Arch ISO with `arch-chroot` — all of which are already part of this design, and any of which can unlock the account again with `passwd -u root`.

Because the disk is fully encrypted, anyone with physical access already needs the LUKS passphrase before reaching a console at all, so the marginal protection is modest and the marginal cost is modest. Either choice is defensible; make it deliberately rather than by copying this line. To keep the recovery path instead, set a strong root password and omit the `-l`.

### §3.6 — Locale, time, hostname

**Run this section first**, immediately after entering the chroot — see the ordering note at the top of §3.

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc

nvim /etc/locale.gen    # uncomment en_US.UTF-8 and de_DE.UTF-8
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo "FONT=ter-v18n" >> /etc/vconsole.conf    # terminus, readable on a 14" panel

echo "<HOSTNAME>" > /etc/hostname
cat > /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   <HOSTNAME>.localdomain  <HOSTNAME>
EOF
```

### §3.7 — journald persistence

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
```

`/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=500M
MaxFileSec=1month
Compress=yes
ForwardToSyslog=no
```

Without both the directory and `Storage=persistent`, journald runs volatile out of `/run/log/journal` and you lose logs across exactly the reboots you most need to investigate.

### §3.8 — Enable essential services

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
systemctl enable fstrim.timer
systemctl enable fwupd-refresh.timer
```

### §3.9 — Exit and reboot

```bash
exit
umount -R /mnt
swapoff /mnt/swap/swapfile 2>/dev/null || true
reboot
```

---

## §4 — First Boot

Log in as `luiswg`:

```bash
sudo pacman -Syu
```

If the LUKS prompt accepted your passphrase, the `keyboard` hook is working. Since `KEYMAP=us` matches the console default, a `sd-vconsole` failure would not be visible here — confirm it separately with `localectl status` once booted.

### §4.2 — First-boot verification and commonly skipped steps

Run this immediately, before installing anything else. It catches the things that silently do not happen.

```bash
sudo bootctl install          # writes the NVRAM entry the chroot could not
systemctl --failed
findmnt /boot                 # confirm fmask=0077,dmask=0077
swapon --show                 # zram at pri=100 present? (§6.3)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
sudo aa-status                # AppArmor loaded? (§7.4)
resolvectl status             # systemd-resolved running? (§7.3)
```

`bootctl install` inside `arch-chroot` cannot write EFI variables — it only places the loader at the removable-media fallback path. Lenovo firmware boots that fine, so the omission is invisible until you want a named boot entry.

**Sections most likely to be skipped**, in the order the omission will bite:

| § | Item | Symptom if skipped |
|---|---|---|
| 7.4 | AppArmor package | `lsm=…apparmor…` is on the cmdline with nothing to use it — no confinement at all |
| 6.3 | zram config | all swap traffic goes to the QLC disk instead of compressed RAM |
| 6.1 | sysctl hardening | none of the documented kernel hardening is in effect |
| 9.3 | snapper retention | snapshots accumulate without bound |
| 6.4 | btrfs scrub timer | checksums exist but nothing ever verifies them |
| 7.3 | DNS over TLS | queries go plaintext to whatever DHCP hands you |

None of these fail loudly. Check them explicitly rather than assuming they happened.

### §4.1 — Wireless bring-up and rtw89 quirks

```bash
dmesg | grep -i rtw89
nmcli device wifi list
nmcli device wifi connect YOUR_SSID --ask
```

Expect firmware version lines and no error spam. If the interface is absent entirely, `linux-firmware-realtek` did not install — fix from a USB tether or ethernet dongle.

**Do not apply the following preemptively.** The rtw89 driver has two well-documented failure modes on 8852-series parts, and the workarounds carry their own costs (higher idle power from disabled ASPM, higher power from disabled power-save). Apply only the one matching a symptom you actually observe.

*Symptom: random disconnects, `firmware failed` or PCIe-related errors in `dmesg`, link dying under load.* Create `/etc/modprobe.d/rtw89.conf`:

```
options rtw89_pci disable_clkreq=y disable_aspm_l1=y disable_aspm_l1ss=y
```

*Symptom: high latency at idle, first packet after a pause is slow, connection drops during suspend/resume cycles.* Add to the same file:

```
options rtw89_core disable_ps_mode=y
```

Then `sudo mkinitcpio -P` and reboot. If disabling power-save fixes it, also consider `WIFI_PWR_ON_BAT=off` in `/etc/tlp.conf` (§8.1) rather than the module option, which is the less blunt instrument.

**Firmware regressions are a real risk on this chipset.** A `linux-firmware-realtek` update in August 2025 broke RTL8852 wireless for a number of users; the fix was to move the newer versioned blob aside so the driver fell back to the older one. If wifi breaks immediately after an update, check `dmesg | grep rtw89` for the firmware version, then roll back via snapper (§9.1) — which is exactly the scenario `snap-pac` exists for.

---

## §5 — pacman and Mirrors

### §5.1 — pacman.conf

```ini
Color
VerbosePkgLists
ParallelDownloads = 6
ILoveCandy
```

### §5.2 — Reflector via timer

`/etc/xdg/reflector/reflector.conf`:

```
--country Germany,Netherlands,Austria
--age 12
--protocol https
--sort rate
--save /etc/pacman.d/mirrorlist
```

```bash
sudo systemctl enable --now reflector.timer
```

The timer reads this config on each run. A one-shot `reflector` invocation gets overwritten on the first timer fire.

### §5.3 — makepkg parallelism

`/etc/makepkg.conf`:

```
MAKEFLAGS="-j$(nproc --ignore=1)"
COMPRESSZST=(zstd -c -z -q -1 -)
```

**Optional.** This was not applied on the built system and nothing broke; treat it as a preference rather than a step. With 32 GB RAM the memory-pressure argument from v3 no longer applies. The remaining reason is thermal: the 4650U is a 15 W part in a thin chassis, and sustained all-core builds throttle. Leaving one thread free keeps the system responsive during long builds. This is a preference, not a correctness requirement — raise it to `-j$(nproc)` if you would rather have the build finish sooner.

---

## §6 — Memory and I/O Tuning

### §6.1 — sysctl

`/etc/sysctl.d/99-hardening.conf`:

```ini
# Memory pressure
vm.swappiness = 180
vm.vfs_cache_pressure = 50
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10

# Information disclosure
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.printk = 3 3 3 3

# Privilege boundary
kernel.unprivileged_userns_clone = 0
kernel.yama.ptrace_scope = 1

# Filesystem hardening
fs.protected_regular = 2
fs.protected_fifos = 2
fs.protected_hardlinks = 1
fs.protected_symlinks = 1

# Network basics
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
```

`vm.swappiness=180` is the kernel-recommended value when zram is present: it pushes hot pages into compressed RAM (microsecond latency) rather than onto the disk swapfile.

**Caveat on `unprivileged_userns_clone=0`:** this breaks rootless Podman, some Flatpak configurations, and several Electron sandboxes. Since userland is out of scope in v4, you will not hit it during this install — but you will hit it the first time you install a container runtime. Set it to `1` then, deliberately, understanding you are accepting the privilege-escalation surface.

### §6.2 — UMask

In `/etc/login.defs`, change `UMASK 022` to `UMASK 077`. Files you intend to share get `chmod`-ed up rather than down — fail-secure.

### §6.3 — zram

`/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = min(ram / 4, 8192)
compression-algorithm = zstd
swap-priority = 100
```

v3 used `ram / 2`, which on 16 GB gave 8 GB. The same ratio here would give 16 GB of compressed swap on a machine with 32 GB of real RAM — well past useful, and it enlarges the hibernation image for no benefit. 8 GB is ample.

The disk swapfile sits at `pri=10`, so zram is used first. Hibernation writes to the `resume=` device named in the cmdline, so this priority ordering governs runtime swap selection only and does not break hibernation.

### §6.4 — Btrfs scrub timer

```bash
sudo systemctl enable btrfs-scrub@-.timer
```

Monthly scrub detects bitrot via btrfs's checksums. Without it the checksums are theatre — corruption sits silent until you happen to read the bad block.

---

## §7 — Security Hardening

### §7.1 — nftables

`/etc/nftables.conf`:

```
#!/usr/bin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state invalid drop
        ct state {established, related} accept

        iif lo accept

        ip protocol icmp limit rate 10/second accept
        ip6 nexthdr icmpv6 limit rate 10/second accept

        # SSH — uncomment only if inbound SSH is actually required
        # tcp dport 22 ct state new limit rate 5/minute accept

        # mDNS for local discovery (optional)
        # udp dport 5353 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

```bash
sudo systemctl enable --now nftables
sudo nft list ruleset
```

The `nftables` package is already present from §2.1.

Hand-written rather than UFW: UFW abstracts a layer (iptables-nft → nftables) that contradicts a CLI-first system, and these thirty lines are what you will actually want to read at 2 AM. Outbound SSH to your Hetzner box is already covered by the default-accept output policy.

### §7.2 — SSH

This laptop is an SSH *client*. `openssh` is installed (§2.1) because you need the client; the `sshd` service is **not** enabled and should stay that way unless you have a concrete need for inbound access.

Client config at `~/.ssh/config`, useful on hotel and conference wifi:

```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 10m
    HashKnownHosts yes
```

```bash
mkdir -p ~/.ssh/sockets
```

If you do need an inbound server later, harden before enabling — `/etc/ssh/sshd_config.d/10-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers luiswg
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

Verify a working key exists in `~/.ssh/authorized_keys` *before* `systemctl enable --now sshd`, and uncomment the nftables rule in §7.1.

### §7.3 — DNS

`/etc/systemd/resolved.conf`:

```ini
[Resolve]
DNS=2a07:a8c0::#YOUR_NEXTDNS_ID.dns.nextdns.io 45.90.28.0#YOUR_NEXTDNS_ID.dns.nextdns.io
FallbackDNS=2620:fe::fe 9.9.9.9
DNSSEC=allow-downgrade
DNSOverTLS=yes
MulticastDNS=no
LLMNR=no
```

`DNSSEC=allow-downgrade` rather than `yes`: hotel and corporate networks mangle DNS in ways strict validation rejects, locking you out of resolution entirely.

NetworkManager handoff — without this, NM keeps handling DNS itself and bypasses your DoT configuration completely:

```bash
sudo mkdir -p /etc/NetworkManager/conf.d
cat | sudo tee /etc/NetworkManager/conf.d/dns.conf <<'EOF'
[main]
dns=systemd-resolved
systemd-resolved=true
EOF

sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart NetworkManager
resolvectl status
```

### §7.4 — AppArmor

```bash
sudo pacman -S apparmor
sudo systemctl enable --now apparmor
sudo aa-status
```

The `lsm=…apparmor…` cmdline parameter from §3.2 is what loads the LSM at boot; the package alone does nothing without it.

### §7.5 — usbguard

```bash
sudo pacman -S usbguard
sudo systemctl enable --now usbguard
sudo usbguard generate-policy > /tmp/rules.conf
sudo install -m 0600 -o root -g root /tmp/rules.conf /etc/usbguard/rules.conf
sudo systemctl restart usbguard
```

This snapshots currently-connected devices as the allowlist. Generate the policy with your normal peripherals attached, or you will be re-authorizing your own keyboard. New devices at hotel chargers and conference kiosks then require explicit `usbguard allow-device <id>`.

### §7.6 — LUKS header backup

```bash
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file ~/luks-header.bin
gpg --symmetric --cipher-algo AES256 ~/luks-header.bin
shred -u ~/luks-header.bin
```

Move `~/luks-header.bin.gpg` to at least three places: external SSD, your Hetzner box, and one offline copy. A header is roughly 16 KB. Header corruption without a backup means zero data recovery — and note that a header backup is not a passphrase backup; it does not help if you forget the passphrase.

---

## §8 — Power Management

### §8.1 — TLP with ThinkPad specifics

```bash
sudo pacman -S tlp tlp-rdw
sudo systemctl enable --now tlp.service
sudo systemctl enable NetworkManager-dispatcher.service
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
```

Do **not** install `power-profiles-daemon` alongside TLP; they conflict. If you later want tighter desktop-environment integration and less manual tuning, PPD is a legitimate alternative — but pick one. Note that PPD's main advantage on AMD mobile comes from `amd_pstate` EPP support, which this machine cannot use (§3.2), so the case for switching is weaker here than the general advice suggests.

`/etc/tlp.conf`:

```
RUNTIME_PM_ON_AC=auto
RUNTIME_PM_ON_BAT=auto
CPU_BOOST_ON_AC=1
CPU_BOOST_ON_BAT=0
PLATFORM_PROFILE_ON_AC=performance
PLATFORM_PROFILE_ON_BAT=low-power

# ThinkPad battery longevity — supported natively via thinkpad_acpi
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
```

The charge thresholds are the meaningful ThinkPad-specific addition. If the machine lives mostly on AC, holding the battery near 80% rather than 100% substantially slows calendar aging. Widen the window or remove the lines on travel days when you want full capacity.

### §8.2 — Verify

```bash
sudo tlp-stat -s
sudo tlp-stat -b        # battery and charge thresholds
sudo tlp-stat -p        # processor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver    # acpi-cpufreq on this unit — see §3.2
cat /sys/power/mem_sleep
```

`scaling_driver` reports `acpi-cpufreq` on this machine and that is correct — see §3.2 for why `amd_pstate` cannot bind to this firmware. Do not chase it.

`mem_sleep` reports `s2idle [deep]`: S3 deep sleep is available and already selected by default, so the BIOS Sleep State option discussed in §0.2 was moot on this unit.

### §8.3 — Hibernation test

This is the goal that shaped §1.6 and §3.2. Test it before you trust it.

```bash
systemctl hibernate
```

On resume, verify the LUKS prompt appears and accepts the passphrase, and that the session restores rather than starting fresh. **If it boots fresh, `resume=` or `resume_offset=` is wrong** — correct `/etc/kernel/cmdline` and run `sudo mkinitcpio -P` to rebuild all four UKIs.

Re-verify the offset any time the swapfile is recreated:

```bash
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
```

---

## §9 — Snapshots, Backup, Recovery

### §9.1 — Snapper

```bash
sudo pacman -S snapper snap-pac
sudo snapper -c root create-config /
sudo snapper -c home create-config /home
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

`snap-pac` creates pre/post snapshots around every pacman transaction, giving rollback granularity at the package-update level.

### §9.2 — Disable btrfs qgroups

```bash
sudo btrfs quota disable /
sudo btrfs quota disable /home
```

Some snapper configurations enable qgroups. On active NVMe systems they cause periodic CPU spikes during balance and cleanup that freeze the desktop for seconds. The benefit — exact per-snapshot space accounting — is not worth the latency.

### §9.3 — Retention

`/etc/snapper/configs/root`:

```
TIMELINE_LIMIT_HOURLY=5
TIMELINE_LIMIT_DAILY=7
TIMELINE_LIMIT_WEEKLY=0
TIMELINE_LIMIT_MONTHLY=2
TIMELINE_LIMIT_YEARLY=0
NUMBER_LIMIT=10
NUMBER_LIMIT_IMPORTANT=5
SPACE_LIMIT="0.5"
```

`@swap` is never snapshotted: it is mounted separately and outside snapper's root scope, and the btrfs kernel additionally refuses to snapshot a subvolume containing an active swapfile.

### §9.4 — External backup with restic

Snapshots are not backups; they die with the SSD controller. This is the third leg of 3-2-1.

```bash
sudo pacman -S restic

RESTIC_REPOSITORY="sftp:luis@hive.neurogenesis.dev:/srv/backup/t14" \
  RESTIC_PASSWORD_FILE="$HOME/.restic-password" \
  restic init
```

`~/.config/systemd/user/restic-backup.service`:

```ini
[Unit]
Description=Restic backup to Hetzner
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
Environment=RESTIC_REPOSITORY=sftp:luis@hive.neurogenesis.dev:/srv/backup/t14
Environment=RESTIC_PASSWORD_FILE=%h/.restic-password
ExecStart=/usr/bin/restic backup --exclude-caches \
  --exclude '/home/luiswg/.cache' \
  --exclude '/home/luiswg/.var/app/*/cache' \
  /home /etc /var/log
ExecStartPost=/usr/bin/restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

`~/.config/systemd/user/restic-backup.timer`:

```ini
[Unit]
Description=Daily restic backup

[Timer]
OnCalendar=*-*-* 03:30:00
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl --user enable --now restic-backup.timer
loginctl enable-linger luiswg    # so the timer runs when not logged in
```

The Steam exclusions from v3 are removed along with the gaming stack. Revisit the exclude list when you add large-cache userland.

### §9.5 — Booting a snapshot

systemd-boot does not natively boot btrfs snapshots. From the Arch ISO:

```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
mount -o subvol=@ /dev/mapper/cryptroot /mnt
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt

snapper -c root list
snapper -c root rollback <number>
exit
umount -R /mnt
reboot
```

### §9.6 — Recovery scenarios

| Scenario | Procedure |
|---|---|
| Bad pacman update | Boot, snapper rollback to the latest `pre` snapshot |
| Kernel update broke boot | Boot `arch-linux-lts.efi` from the boot menu |
| Initramfs broken, system unbootable | Boot the fallback UKI of the same kernel |
| LUKS header corrupt | `cryptsetup luksHeaderRestore /dev/nvme0n1p2 --header-backup-file <path>` |
| SSD died entirely | New SSD → reinstall from this PRD → restore `/home` from restic |
| Forgot passphrase, header backup intact | Locked out. A header backup is not a passphrase backup |
| Hibernation wedged | Boot the fallback UKI of the same kernel, accept session loss |
| Firmware update bricked boot entries | `bootctl install` from the ISO chroot, then `mkinitcpio -P` |

Keep this table honest by rehearsing at least the snapper rollback path once (§11).

---

## §10 — Maintenance

### §10.1 — Weekly

```bash
checkupdates                            # preview
# Read https://archlinux.org/news IF any of: linux, systemd, pacman,
#   openssl, mkinitcpio, glibc, cryptsetup appear above
sudo pacman -Syu
sudo pacdiff                            # merge .pacnew configs
paccache -rk2                           # prune cache, keep last 2
sudo fwupdmgr refresh && sudo fwupdmgr update
sudo journalctl --vacuum-time=2months
```

Unmerged `.pacnew` files are one of the classic Arch long-term-rot mechanisms. `pacdiff` is not optional hygiene.

### §10.2 — Monthly

```bash
sudo smartctl -a /dev/nvme0n1
sudo nvme smart-log /dev/nvme0n1
sudo btrfs filesystem usage /
sudo btrfs scrub status /
restic check
```

On a secondhand machine, watch `percentage_used` and `data_units_written` in the NVMe SMART log for the first few months to establish a baseline you can trust.

### §10.3 — Quarterly

- Rotate the LUKS passphrase: `cryptsetup luksChangeKey /dev/nvme0n1p2`
- Rehearse a snapshot rollback — recovery procedures decay without practice
- Re-verify the LUKS header backup is still readable on offline media

---

## §11 — Verification Block

Run after the first successful boot and again after §15.

```bash
# Boot and storage
bootctl status
cat /proc/cmdline
findmnt -no OPTIONS /
findmnt -nt btrfs
grep subvolid /etc/fstab                 # must return nothing
cryptsetup status cryptroot
swapon --show                            # zram pri=100, swapfile pri=10
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile

# CPU and power
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/power/mem_sleep
sudo tlp-stat -b

# Logs, time, DNS
journalctl -b -p warning --no-pager | head -50
resolvectl status
systemctl status NetworkManager systemd-resolved fstrim.timer

# Security posture
sudo aa-status
sudo nft list ruleset
sudo usbguard list-devices

# Snapshots and backup
snapper list
systemctl --user list-timers restic-backup.timer

# The actual goal
systemctl hibernate
```

---

## §12 — Deferred

### §12.1 — Secure Boot with your own keys

Requires Setup Mode: in BIOS, Security → Secure Boot → Reset to Setup Mode / Clear All Keys.

```bash
sudo sbctl status                        # must report Setup Mode
sudo sbctl create-keys
sudo sbctl enroll-keys -m                # -m keeps Microsoft keys for firmware updates
sudo sbctl sign -s /boot/EFI/Linux/arch-linux.efi
sudo sbctl sign -s /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl sign -s /boot/EFI/Linux/arch-linux-lts.efi
sudo sbctl sign -s /boot/EFI/Linux/arch-linux-lts-fallback.efi
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl verify
```

`sbctl` ships a pacman hook for re-signing on update. Keep `-m` — Lenovo firmware updates delivered through fwupd are signed by Microsoft's key, and dropping it means losing the fwupd path you used in §0.1.

Then re-enable Secure Boot in BIOS.

### §12.2 — TPM2 + PIN

TPM2-alone unlock means anyone with physical access boots straight into your decrypted root. TPM2 + PIN preserves physical-presence security:

```bash
sudo systemd-cryptenroll --tpm2-device=auto \
  --tpm2-pcrs=0+2+7 \
  --tpm2-with-pin=yes \
  /dev/nvme0n1p2
```

PCRs 0, 2, and 7 cover firmware, option ROMs, and Secure Boot state; tampering with any invalidates the TPM unlock and falls back to the passphrase. Add `rd.luks.options=<FSUUID_nvme0n1p2>=tpm2-device=auto` to `/etc/kernel/cmdline` and rebuild the UKIs.

Do this only after §12.1 — enrolling against PCR 7 before Secure Boot is in its final state means re-enrolling afterward. Also note that BIOS updates change PCR 0 and will invalidate the enrollment, which is the correct behavior but will surprise you the first time. Keep the passphrase memorized.

### §12.3 — Userland

Deliberately out of scope. When you add a desktop environment, container runtime, or editor, treat each as its own decision with its own trust assessment rather than a checklist inherited from a different machine.

Two things to carry forward from v3 when you get there:

- `kernel.unprivileged_userns_clone=0` (§6.1) will break rootless Podman and some Flatpak sandboxes. Change it deliberately, not by trial and error.
- Multilib is not enabled. Anything 32-bit needs `[multilib]` uncommented in `/etc/pacman.conf` **before** the install, not after.

---

## §12.5 — Build record (2026-08-11)

Installed and verified on the target hardware. Everything below was confirmed by running it, not by reading the config.

| Item | Result |
|---|---|
| Firmware | Lenovo 1.53 → 1.56 before install; UEFI 2.70 (Lenovo 0.5472) |
| Disk | Micron 2210 MTFDHBA1T0QFD, 953.9 GiB, QLC, FW P6MA000 |
| Drive wear at install | 38 TB budget barely touched — 3.58 TB written, `percentage_used` 0% |
| Sector size | 512/512, `--sector-size` correctly omitted |
| LUKS2 | active, aes-xts-plain64, 512-bit, argon2id |
| Subvolumes | all six mounted, no duplicates, `subvolid` stripped from fstab |
| Swapfile | 36 GiB, `pri=10`, offset 533760 |
| Boot chain | systemd-boot 261.2, four UKIs, Type #2 entries, cmdline embedded |
| Kernels | `linux` 7.1.7, `linux-lts` 6.18.43 |
| Wireless | RTL8852AE, `rtw89_8852ae`, firmware 0.13.36.2, clean bring-up, no workarounds needed |
| Sleep states | `s2idle [deep]` — S3 available and selected by default |
| CPU scaling | `acpi-cpufreq`; `amd_pstate` removed (§3.2) |
| Battery | Celxpert 5B10W139, **38 cycles**, 46,710 / 51,000 mWh = 91.6% capacity |
| Charge thresholds | 75/80 applied via `natacpi` / `thinkpad_acpi` |
| **Hibernation** | **works — session restored across a full power-off** |

**Post-build audit, 2026-08-13.** Two days of use surfaced the following, all recorded in the state reference:

- `xdg-desktop-portal-hyprland.service` is in a failed state — the PipeWire ordering race at first boot recurred.
- §6.1, §6.3, §6.4, §7.3, §7.4, and §9.3 were never applied. See §4.2.
- Drive history is fuller than first assessed: 2,925 power-on hours and 4,925 power cycles alongside the 3.6 TB written. Wear remains under 1% of budget, but "barely used" overstated it.
- Battery reads 92.6% capacity at 38 cycles — slightly better than the first measurement.
- `sgdisk` was unavailable post-install because `gptfdisk` was never in the pacstrap list. Fixed in §2.1.

Notes for the next time this document is used:

- The `passwd` → `sudo` failure during install was `pam_faillock`, not a wrong password. Three failures lock the account for ten minutes and reject the correct password during that window. `faillock --user <name> --reset` from a root shell clears it. This will look identical at the login prompt after first boot.
- `bootctl install` inside `arch-chroot` cannot write EFI variables; it lands the loader at the removable-media path only. Re-run it after first boot from a real EFI environment to get the NVRAM entry.
- genfstab gives the ESP `fmask=0022,dmask=0022`, which makes systemd-boot's random seed world-readable and produces a security warning on every `bootctl` run. Set `0077` — and note that vfat ignores mask changes on remount, so it needs a full `umount` / `mount` cycle. §1.4 now mounts it correctly from the start.
- Hyprland 0.55+ uses a **Lua** config (`hyprland.lua`), not hyprlang. Most configs published online will not paste in. `hyprpaper` 0.8.4 failed to parse a valid config on this system; `swaybg` replaced it without incident.
- Fn media keys on this machine are handled by the EC and emit no input event at all — no OSD daemon can catch them. Waybar's volume module is the working answer.
- `kernel.dmesg_restrict = 1` (§6.1) means `dmesg` needs sudo. Expected, not a fault.

---

## §13 — Strategic Notes

### §13.1 — What is fragile on this machine

Graphics, which shaped the entire XPS document, are a non-issue: Renoir is supported by in-tree `amdgpu` with no out-of-tree driver, no DKMS, and no vendor EOL clock. But this unit is not risk-free, and the specific risk was not the one predicted.

- **Realtek RTL8852AE wireless is the fragile component** (§4.1). The `rtw89` driver is in-tree and works, but the 8852 series has a track record of ASPM and power-save instability, and a firmware-blob update in August 2025 broke wireless outright for a subset of users. This is materially worse than the Intel AX200 most of these machines shipped with. It is nowhere near as bad as the Pascal situation on the XPS — the driver is upstream and maintained, not a legacy AUR branch with a countdown — but it is the component most likely to break on a routine update. `linux-lts` and `snap-pac` are load-bearing here, not decorative.
- ~~**Secondhand hardware condition.**~~ Largely resolved: 3.58 TB written against a 360 TB budget, 0% used. §10.2's SMART monitoring remains worthwhile for trend data on a QLC drive, but the starting point is effectively a new disk.
- ~~**Battery age.**~~ Resolved at install: 38 cycles, 91.6% capacity. Not a concern.
- **Hibernation.** Verified working at install (§12.5), but still the most breakage-prone feature here and the one most likely to regress silently across a kernel update — particularly if the swapfile is ever recreated, which invalidates `resume_offset`. §11 ends with `systemctl hibernate` for that reason. On the XPS this was entangled with the NVIDIA VRAM preservation chain; here there is no such coupling, which is why it worked first try.

Note the pattern: two of these four were unknowable before the hardware was in front of you, which is the argument for §0.3 existing at all.

### §13.2 — Where this document is honest versus aspirational

- **Honest:** boot chain, storage, encryption, snapshots, backup, MAC, firewall, hardening sysctls. All verifiable by the §11 block.
- **Aspirational:** "low telemetry" is a property of the base OS only. It becomes a claim about the whole system only if the userland you add later is chosen with the same care, which is your decision to make and not this document's to assert.
- **Unverified:** firmware. You trust Lenovo's UEFI, the EC, and the fTPM because you have no practical alternative. Naming that is better than pretending the trust chain starts at the bootloader.

### §13.3 — Toward v5

The structural improvement remains making this executable: a shell harness or Ansible playbook that turns `arch-chroot ./` into `./install.sh`, kept in lockstep with this spec. Dropping the userland sections makes that substantially more tractable — what remains is almost entirely deterministic, and the only genuinely interactive steps are the passphrase, the §0.3 worksheet, and the hibernation test.

---

## Appendix A — Quick reference

| Purpose | Command |
|---|---|
| All btrfs mounts | `findmnt -nt btrfs -S /dev/mapper/cryptroot` |
| Mountpoints per device | `lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS` |
| Subvolumes | `sudo btrfs subvolume list /` |
| LUKS mapping | `ls /dev/mapper` |
| LUKS metadata | `sudo cryptsetup luksDump /dev/nvme0n1p2` |
| LUKS UUID | `sudo blkid -s UUID -o value /dev/nvme0n1p2` |
| Swap state | `swapon --show` |
| Resume offset | `sudo btrfs inspect-internal map-swapfile -r /swap/swapfile` |
| Boot chain | `bootctl status` |
| Active cmdline | `cat /proc/cmdline` |
| Wireless driver state | `dmesg \| grep -i rtw89` |
| Wireless firmware version | `dmesg \| grep -i "rtw89.*fw"` |
| CPU scaling driver | `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver` |
| Sleep states | `cat /sys/power/mem_sleep` |
| Battery health | `sudo tlp-stat -b` |
| AppArmor | `sudo aa-status` |
| Firewall | `sudo nft list ruleset` |
| USB authorization | `sudo usbguard list-devices` |
| Snapshots | `snapper list` |
| Backup status | `restic snapshots` |
| Pending updates | `checkupdates` |
| `.pacnew` files | `sudo pacdiff` |
| SSD health | `sudo smartctl -a /dev/nvme0n1` |
| NVMe SMART | `sudo nvme smart-log /dev/nvme0n1` |
| NVMe LBA formats | `sudo nvme id-ns -H /dev/nvme0n1` |
| Btrfs scrub | `sudo btrfs scrub status /` |
| Firmware versions | `fwupdmgr get-devices` |
| Verify no subvolid in fstab | `grep subvolid /etc/fstab` (empty) |

---

## Appendix B — Placeholders to resolve

Two rows resolved from §0.3 and substituted inline throughout the document. The rest are resolved as you reach them.

| Placeholder | Source | Status |
|---|---|---|
| disk path | `lsblk -o NAME,SIZE,MODEL,TYPE` | ✅ `/dev/nvme0n1` — substituted throughout |
| sector size | `nvme id-ns -H /dev/nvme0n1` | ✅ 512 only — flag omitted from §1.2 |
| LUKS UUID | `blkid -s UUID -o value /dev/nvme0n1p2` | ✅ `<FSUUID_nvme0n1p2>` |
| resume offset | `btrfs inspect-internal map-swapfile -r /mnt/swap/swapfile` | ✅ `533760` — **re-measure if the swapfile is ever recreated** |
| `<HOSTNAME>` | your choice | ⬜ |
| `YOUR_NEXTDNS_ID` | NextDNS account | ⬜ |
| `YOUR_SSID` | your network | ⬜ |

---

## Appendix C — Provenance

Sections carried forward from the XPS 9570 v3 PRD, whose review attribution (ChatGPT, Gemini, Claude across v2→v3) applies unchanged to the storage, boot chain, hardening, and backup architecture.

New in v4 and unreviewed by any second model: the hardware-retarget decisions (§0.1–§0.3, §8.1), the `dosfstools` correction, the sector-size conditional, the zram resize, and the scope reduction. These warrant the same multi-model review treatment the v3 document received before you treat them as settled.
