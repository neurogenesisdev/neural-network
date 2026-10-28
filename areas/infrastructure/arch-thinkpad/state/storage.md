## block devices
NAME            SIZE TYPE  FSTYPE
zram0           7.6G disk  swap
nvme0n1       953.9G disk  
|-nvme0n1p1       1G part  vfat
`-nvme0n1p2   952.9G part  crypto_LUKS
  `-cryptroot 952.9G crypt btrfs

## luks
LUKS header information
Version:       	2
Epoch:         	16
Metadata area: 	16384 [bytes]
Keyslots area: 	16744448 [bytes]
UUID:          	<FSUUID_nvme0n1p2>
Label:         	(no label)
Subsystem:     	(no subsystem)
Flags:       	(no flags)

Data segments:
  0: crypt
	offset: 16777216 [bytes]
	length: (whole device)
	cipher: aes-xts-plain64
	sector: 512 [bytes]

Keyslots:
  0: luks2
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	AF hash:    sha512
	Area offset:32768 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
  1: luks2
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	AF hash:    sha256
	Area offset:290816 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
  3: luks2
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	Hash:       sha512
	Iterations: 1000
	AF hash:    sha512
	Area offset:806912 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
Tokens:
  0: systemd-tpm2

## luks tokens
{
  "0": {
    "tpm2-pcr-bank": "sha256",
    "tpm2-pcrs": [
      0,
      2,
      7
    ],
    "tpm2-pin": true,
    "type": "systemd-tpm2"
  }
}

## crypttab

## subvolumes
ID 256 top level 5 path @
ID 257 top level 5 path @home
ID 258 top level 5 path @swap
ID 259 top level 5 path @snapshots
ID 260 top level 5 path @log
ID 261 top level 5 path @cache
ID 262 top level 256 path @/var/lib/portables
ID 263 top level 256 path @/var/lib/machines
ID 264 top level 257 path @home/.snapshots
ID 537 top level 257 path @home/luiswg/.local/share/containers

## default subvolume
ID 517 top level 259 path @snapshots/184/snapshot

## mounts
/           /dev/mapper/cryptroot[/@snapshots/184/snapshot] rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=517,subvol=/@snapshots/184/snapshot
/.snapshots /dev/mapper/cryptroot[/@snapshots]              rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=259,subvol=/@snapshots
/boot       /dev/nvme0n1p1                                  rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro
/home       /dev/mapper/cryptroot[/@home]                   rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=257,subvol=/@home
/swap       /dev/mapper/cryptroot[/@swap]                   rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=258,subvol=/@swap
/var/cache  /dev/mapper/cryptroot[/@cache]                  rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=261,subvol=/@cache
/var/log    /dev/mapper/cryptroot[/@log]                    rw,noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=260,subvol=/@log

## fstab
/swap/swapfile      	none      	swap      	defaults,pri=10  	0 0
UUID=<PARTUUID_cryptroot>	/         	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2	0 0
UUID=<PARTUUID_cryptroot>	/.snapshots	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2,subvol=/@snapshots	0 0
UUID=<PARTUUID_cryptroot>	/home     	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2,subvol=/@home	0 0
UUID=<PARTUUID_cryptroot>	/swap     	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2,subvol=/@swap	0 0
UUID=<PARTUUID_cryptroot>	/var/cache	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2,subvol=/@cache	0 0
UUID=<PARTUUID_cryptroot>	/var/log  	btrfs     	rw,noatime,compress=zstd:1,ssd,space_cache=v2,subvol=/@log	0 0
UUID=<FSUUID_nvme0n1p1>      	/boot     	vfat      	rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro	0 2

## swap
/dev/zram0     partition 7.6G 0B 100
/swap/swapfile file       36G 0B  10

## snapper configs
Config | Subvolume
-------+----------
home   | /home
root   | /

## snapper retention (configured limits)
home NUMBER_LIMIT 10
home NUMBER_LIMIT_IMPORTANT 5
home TIMELINE_CREATE yes
home TIMELINE_LIMIT_DAILY 7
home TIMELINE_LIMIT_HOURLY 5
home TIMELINE_LIMIT_MONTHLY 2
home TIMELINE_LIMIT_QUARTERLY 0
home TIMELINE_LIMIT_WEEKLY 0
home TIMELINE_LIMIT_YEARLY 0
root NUMBER_LIMIT 10
root NUMBER_LIMIT_IMPORTANT 5
root TIMELINE_CREATE yes
root TIMELINE_LIMIT_DAILY 7
root TIMELINE_LIMIT_HOURLY 5
root TIMELINE_LIMIT_MONTHLY 2
root TIMELINE_LIMIT_QUARTERLY 0
root TIMELINE_LIMIT_WEEKLY 0
root TIMELINE_LIMIT_YEARLY 0

## nvme health
critical_warning			: 0
available_spare				: 100%
available_spare_threshold		: 5%
percentage_used				: 1%
power_cycles				: 9481
unsafe_shutdowns			: 121
media_errors				: 0
