---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [boot]
tags: [backlog, shutdown, systemd, luks, plasma, diagnostics]
created: 2026-09-03T00:00:00Z
updated: 2026-09-08T00:00:00Z
commit: pending
---

# Shutdown hangs at a black screen with a blinking cursor

> [!NOTE] Summary
> Twice now (2026-09-01 00:12 and 2026-09-03 01:08) a Plasma shutdown ended at a black screen with a blinking cursor and needed the power button held. Both times the journal ends normally at the point journald itself stops, so the hang is *after* the logged part of shutdown and `quiet loglevel=3` hides the rest. Cause unknown; a shutdown-time debug hook is installed to get evidence on the next occurrence.

## Measured 2026-09-03

- Both journals end cleanly: user slice removed, `tailscaled` stopped (its
  `ExecStopPost` cleanup run is the "Program starting" line that looked like a
  restart — it is not one), `Finished Generate shutdown-ramfs`. Nothing after,
  because journald has stopped by then. No stop-job timeout, no
  `A stop job is running` line in either.
- Both times Plasma's logout dialog first reported an application refusing to
  close (kitty on 2026-09-03). Both times something with a device-mapper or
  container footprint was still up at logout: the VeraCrypt GUI
  (`Failed to kill control group`) on 2026-09-03, a podman container on
  2026-09-01. Coincidence or cause is not established.
- `HOOKS` has no explicit `shutdown` hook; the systemd initramfs uses
  `mkinitcpio-generate-shutdown-ramfs.service` (static, ran) to tear down
  `cryptroot` after the pivot. A hang there would look exactly like this and
  would never be logged.
- Plasma settings are not involved in the hang. The "programs came back on
  workspace 4" observation is **session restore** — Plasma's default *restore
  previous session* on login, with `~/.config/session/` holding the saved
  windows — not hibernation: `uptime -s` shows a fresh boot and there is no
  hibernation entry in the previous journal.

**2026-09-08:** no shutdown has happened since the hook was installed —
same boot since 2026-09-03 01:09, suspends only. `/shutdown-log.txt` does not
exist yet, which is expected.

## Instrumentation

`/usr/lib/systemd/system-shutdown/debug-shutdown.sh` writes `/shutdown-log.txt`
(mounts, device-mapper table, processes, `dmesg` tail) at the last moment
systemd still runs, before the ramfs pivot. After the next shutdown, whether the
file exists partitions the problem:

| `/shutdown-log.txt` | Meaning |
|---|---|
| present, complete | hang is in the ramfs teardown or the kernel power-off — read the mounts and DM table it captured |
| absent | a stop job never finished; raise `systemd.log_target=kmsg` next |

**Remove the hook when the item closes** — it remounts `/` read-write at
shutdown, which is harmless on btrfs but not something to leave behind.

## Next steps

1. Next shutdown: eject the T5 and stop any devbox container first, so the two
   suspects are out of the picture; read `/shutdown-log.txt` after boot.
2. If the file is present and the DM table still lists `veracrypt1` or a podman
   overlay, that is the finding.
3. If nothing conclusive, add `systemd.log_level=debug systemd.log_target=kmsg`
   to the cmdline for one boot (UKI rebuild and sign; PCR 7 is unaffected by
   cmdline, so no TPM re-enrollment).
