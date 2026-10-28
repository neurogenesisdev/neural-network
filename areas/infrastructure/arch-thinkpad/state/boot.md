## bootctl
System:
      Firmware: UEFI 2.70 (Lenovo 0.5472)
 Firmware Arch: x64
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Measured UKI: yes
   Measured OS: yes
  Boot into FW: supported
 Platform Lang: en_US.UTF-8

Current Boot Loader:
        Product: systemd-boot 261.2-1-arch
       Features: + Boot counting
                 + Menu timeout control
                 + One-shot menu timeout control
                 + Default entry control
                 + One-shot entry control
                 + Support for XBOOTLDR partition
                 + Support for passing random seed to OS
                 + Load drop-in drivers
                 + Support Type #1 sort-key field
                 + Support @saved pseudo-entry
                 + Support Type #1 devicetree field
                 + Enroll SecureBoot keys
                 + Retain SHIM protocols
                 + Menu can be disabled
                 + Multi-Profile UKIs are supported
                 + Loader reports network boot URL
                 + Support Type #1 uki field
                 + Support Type #1 uki-url field
                 + Loader reports active TPM2 PCR banks
                 + Loader reports firmware keyboard layout
                 + Loader measures SMBIOS information
      Partition: /dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>
         Loader: `-/boot//EFI/systemd/systemd-bootx64.efi
Keyboard Layout: n/a (not reported by firmware)
  Current Entry: arch-linux.efi

Current Stub:
        Product: systemd-stub 262-1-arch
       Features: + Stub reports loader partition information
                 + Stub reports stub partition information
                 + Stub reports network boot URL
                 + Picks up credentials from boot partition
                 + Picks up system extension images from boot partition
                 + Picks up configuration extension images from boot partition
                 + Measures kernel+command line+sysexts
                 + Support for passing random seed to OS
                 + Pick up .cmdline from addons
                 + Pick up .cmdline from SMBIOS Type 11
                 + Pick up .dtb from addons
                 + Stub understands profile selector
                 + Stub measures SMBIOS information
      Partition: /dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>
           Stub: `-/EFI/Linux/arch-linux.efi

 System Token: set
       Exists: yes

Available Boot Loaders on ESP:
          ESP: /boot (/dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>)
         File: |-/boot//EFI/systemd/systemd-bootx64.efi (systemd-boot 261.2-1-arch)
               `-/boot//EFI/BOOT/BOOTX64.EFI (systemd-boot 261.2-1-arch)

Boot Loaders Listed in EFI Variables:
        Title: Linux Boot Manager
           ID: 0x0001
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>
         File: `-/boot//EFI/systemd/systemd-bootx64.efi

        Title: Fallback Linux Boot Manager
           ID: 0x0002
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>
         File: `-/boot//EFI/systemd/systemd-boot-fallbackx64.efi

        Title: Windows Boot Manager
           ID: 0x0000
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>
         File: `-/boot//EFI/Microsoft/Boot/bootmgfw.efi

Boot Loader Entry Locations:
          ESP: /boot (/dev/disk/by-partuuid/<PARTUUID_nvme0n1p1>, $BOOT)
       config: /boot//loader/loader.conf

Default Boot Loader Entry:
         type: Boot Loader Specification Type #2 (UKI, .efi)
        title: Arch Linux (7.2.8-arch1-2)
           id: arch-linux.efi
       source: /boot//EFI/Linux/arch-linux.efi (on the EFI System Partition)
     sort-key: arch
      version: 7.2.8-arch1-2
        uname: 7.2.8-arch1-2
        linux: /boot//EFI/Linux/arch-linux.efi
      profile: 0
      options: rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
        extra: /boot//loader/credentials/nvpcr-anchor.<MACHINE_ID>.cred

## cmdline (configured)
rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none

## cmdline (signed, per UKI)
--- arch-linux-fallback.efi
.cmdline:
  size: 318 bytes
  sha256: a7563415864141c7c6dc5c6dcae234b97f0c9291a00daf98a8c6c4952a9aa99f
  text:
    rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
.uname:
  size: 13 bytes
  sha256: b95efac35e2fd23d899f39f4d60553ed1335040748911d49a518790419df12a8
  text:
    7.2.8-arch1-2
--- arch-linux-lts-fallback.efi
.cmdline:
  size: 318 bytes
  sha256: a7563415864141c7c6dc5c6dcae234b97f0c9291a00daf98a8c6c4952a9aa99f
  text:
    rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
.uname:
  size: 13 bytes
  sha256: c02e2c3707a1905df6462b8e26a5a40b5022a2e1bba6546516ce7afd91fdd6c6
  text:
    6.18.54-2-lts
--- arch-linux-lts.efi
.cmdline:
  size: 318 bytes
  sha256: a7563415864141c7c6dc5c6dcae234b97f0c9291a00daf98a8c6c4952a9aa99f
  text:
    rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
.uname:
  size: 13 bytes
  sha256: c02e2c3707a1905df6462b8e26a5a40b5022a2e1bba6546516ce7afd91fdd6c6
  text:
    6.18.54-2-lts
--- arch-linux.efi
.cmdline:
  size: 318 bytes
  sha256: a7563415864141c7c6dc5c6dcae234b97f0c9291a00daf98a8c6c4952a9aa99f
  text:
    rd.luks.name=<FSUUID_nvme0n1p2>=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw quiet loglevel=3 resume=/dev/mapper/cryptroot resume_offset=533760 lsm=landlock,lockdown,yama,integrity,apparmor,bpf slab_nomerge init_on_alloc=1 init_on_free=1 randomize_kstack_offset=on vsyscall=none
.uname:
  size: 13 bytes
  sha256: b95efac35e2fd23d899f39f4d60553ed1335040748911d49a518790419df12a8
  text:
    7.2.8-arch1-2

## UKIs
arch-linux-fallback.efi
arch-linux-lts-fallback.efi
arch-linux-lts.efi
arch-linux.efi

## loader.conf
default arch-linux.efi
timeout 3
console-mode max
editor no

## mkinitcpio HOOKS/MODULES
MODULES=(btrfs)
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)

## presets
ALL_config="/etc/mkinitcpio.conf"
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
ALL_kver="/boot/vmlinuz-linux-lts"
PRESETS=('default' 'fallback')
PRESETS=('default' 'fallback')
default_uki="/boot/EFI/Linux/arch-linux-lts.efi"
default_uki="/boot/EFI/Linux/arch-linux.efi"
fallback_options="-S autodetect"
fallback_options="-S autodetect"
fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
fallback_uki="/boot/EFI/Linux/arch-linux-lts-fallback.efi"

## secure boot
Installed:	✓ sbctl is installed
Setup Mode:	✓ Disabled
Secure Boot:	✓ Enabled
Vendor Keys:	microsoft

## tpm2 pcrs (sealed-against set)
0 platform-code 3ff07cdf16de0c6476b6b6360ba0579d1349bf7bf2b1b4663df2bec18e0e3fc9
2 external-code b5006fc9cb8965a4ef6592871fb50e7a7b4c8e3d78489d7916d7c5f6d6c2df2b
7 secure-boot-policy bc0be39a94b760d86b79b0a78e8c77e91a3922ab2c7bd1254d742a7c287e5655

## kernels
linux 7.2.8.arch1-2
linux-lts 6.18.54-2
linux-firmware 20260916-1
amd-ucode 20260916-1
