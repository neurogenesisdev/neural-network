---
title: "Entering and working in the rescue system"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-30
commit: pending
created: 2026-08-30
updated: 2026-08-30
tags: [runbook, rescue, grml, netcup, recovery, chroot]
---

# Entering and working in the rescue system

The netcup rescue system is a **Grml live environment** — Debian sid-based,
running from squashfs on an overlay, with `/dev/vda` completely unmounted. It is
the offline-repair environment and the one the machine was built from.

Everything below was measured on 2026-08-30 during the build.

## Activating it

1. SCP → **Media → Rescue System** → activate. **Record the password it shows
   you.**
2. SCP → **Control** → power cycle. Rescue only runs when the machine is
   *running*; activating it on a powered-off server changes what will boot, not
   what is booted.
3. If it does not come up in rescue, check **Settings → Boot Order** — netcup's
   own partitioning guidance mentions setting network boot first before
   activating rescue.

## The password trap

> **The rescue system has its own root password, displayed in the SCP at
> activation. It is not the panel's root password and not the `luiswg` console
> password.**

Setting a new root password in the control panel and trying it against rescue
will fail every time, no matter how many times it is reset — the panel's reset
targets the *installed* system's `/etc/shadow`, which rescue does not use. This
cost time on 2026-08-30 and is the single most likely thing to waste time again.

## The host key will have changed

Expected. Rescue generates its own SSH host keys, so `ssh root@…` warns about a
changed identification. The offered key may be a different *algorithm* from the
one in `known_hosts`, which is the tell that it is a different key set rather
than a rotation.

Verify against the fingerprint in the SCP, or once connected via the console:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Then clear the stale entry:

```bash
# arch-thinkpad
ssh-keygen -R <SERVER_ID>.ultrasrv.de
```

The installed system's fingerprints are in
[[ssh-lockout-recovery#B — host key changed|the lockout runbook]]; they will be
offered again once rescue is deactivated.

## Terminal quirks — read before typing

Two independent problems, both measured, both of which make a rescue session
feel broken.

**Backspace behaves as space.** Grml has no `xterm-kitty` terminfo entry, so
readline misinterprets the terminal. Connect with the terminfo overridden:

```bash
# arch-thinkpad
TERM=xterm-256color ssh root@<SERVER_ID>.ultrasrv.de
```

`kitten ssh` also works by copying terminfo across, but the override is more
predictable in an environment that may be degraded for other reasons.

**Multi-line paste does not execute.** In the rescue shell, pasted line endings
arrive as carriage returns; each line overwrites the previous one and nothing
runs. Do not fight it —

> **Never paste a multi-line block into the rescue shell. Write it to a file on
> arch-thinkpad and pipe it in.**

```bash
# arch-thinkpad
ssh root@<SERVER_ID>.ultrasrv.de 'bash -s' < ~/script.sh 2>&1 | tee ~/run.log
```

The password prompt reads from `/dev/tty`, not stdin, so it still works with the
redirect. This also produces a reviewable artifact and a log, which is why
[[../build/_index|build/]] exists in the form it does.

**The `sudo` collision does not apply in rescue** — you are already root. It
applies on the installed system, where `bash -s` and an interactive `sudo` prompt
compete for stdin and sudo silently consumes the next line of the script as a
password attempt. On the installed system, `scp` the script over and run it in an
interactive session instead.

## Orienting once inside

```bash
cat /etc/os-release          # expect Grml / Debian forky-sid
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
mount | grep -E ' / |/dev/vda'
```

Expected shape: `/` is an `overlay` on squashfs, and **no `vda` partition is
mounted**. If a `vda` partition *is* mounted, unmount before manipulating the
table.

`vdb` is a 128 KiB vfat device — netcup's cloud-init config drive from the
original image. The rebuilt system does not use cloud-init. Leave it alone.

The rescue kernel is far newer than the target's (6.19 versus 6.12 on
2026-08-30). Irrelevant for disk work; relevant if anything is being tested for
kernel behaviour, which it should not be from here.

## Mounting the installed system

Six subvolumes plus `/boot`. Order matters — root first.

```bash
B="noatime,compress=zstd:1,discard=async"
mount -o "$B,subvol=@" /dev/vda3 /mnt
mkdir -p /mnt/{boot,home,var/log,var/cache,var/lib/containers,.snapshots}
mount -o "$B,subvol=@home"       /dev/vda3 /mnt/home
mount -o "$B,subvol=@log"        /dev/vda3 /mnt/var/log
mount -o "$B,subvol=@cache"      /dev/vda3 /mnt/var/cache
mount -o "$B,subvol=@containers" /dev/vda3 /mnt/var/lib/containers
mount -o "$B,subvol=@snapshots"  /dev/vda3 /mnt/.snapshots
mount /dev/vda2 /mnt/boot
findmnt -R /mnt
```

To reach the btrfs **top level** — for subvolume creation or inspection — mount
`subvolid=5` at a path **outside `/mnt`**:

```bash
mkdir -p /mnt2 && mount -o subvolid=5 /dev/vda3 /mnt2
btrfs subvolume list /mnt2
umount /mnt2 && rmdir /mnt2
```

Mounting it at `/mnt/btr` nests the filesystem inside the system being repaired.
`mount` does not create mountpoints, so the first attempt fails with *"mount
point does not exist"* — which is a harmless error that conveniently prevents the
harmful version.

## Chrooting

```bash
for d in dev dev/pts proc sys run; do
  mount --rbind "/$d" "/mnt/$d"; mount --make-rslave "/mnt/$d"
done
chroot /mnt /bin/bash
```

Without the bind mounts, `systemctl` inside the chroot warns *"/proc/ is not
mounted"* and its answers are unreliable. `passwd` and `passwd -S` work without
them.

Common repairs from the chroot:

```bash
passwd luiswg                          # reset the console password
update-grub && update-initramfs -u -k all
grub-install --target=i386-pc /dev/vda
grep GRUB_CMDLINE_LINUX= /etc/default/grub    # must contain net.ifnames=0
```

## Leaving

```bash
exit                        # if chrooted
sync
umount -R /mnt
findmnt /mnt || echo unmounted
```

If `umount -R` reports busy, `fuser -m /mnt` names the process — usually a stray
shell. Close it rather than forcing; a dirty btrfs unmount is worth avoiding when
the alternative is closing a terminal.

Then SCP → Media → **deactivate** Rescue System, confirm Boot Order has Hard Disk
first, and power cycle. **Watch the Screen console during boot** rather than
waiting on SSH — a failure is visible there and invisible from a hanging `ssh`.

## Afterwards

The build scripts in [[../build/_index|build/]] assume this environment and
assert on it: stage 1 refuses to run unless `/` is an overlay mount, which is
what makes it safe to have a `sgdisk --zap-all` in a file. If a repair changes
anything structural, the affected ADR needs amending — this folder has no
collector to notice on its own
([[../backlog/netcup-collector|netcup-collector]]).
