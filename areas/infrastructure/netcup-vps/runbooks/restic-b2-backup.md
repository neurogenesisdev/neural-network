---
title: "restic → B2 backup for netcup-vps: install, verify, restore"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-11
commit: pending
created: 2026-09-11
updated: 2026-09-11
tags: [runbook, backups, restic, b2, credentials, systemd]
---

# restic → B2 backup for netcup-vps

> [!NOTE] Summary
> Plan step 0.2. Daily restic backup to a dedicated Backblaze B2 bucket over the
> S3 endpoint (same shape as arch-thinkpad, [[../../arch-thinkpad/decisions/20260830-remote-repository-is-b2|ADR-0028]]),
> secrets via `LoadCredential=` ([[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]]),
> a monthly integrity check, a staleness marker, and a restore test **from the
> laptop**. Deletion-resistance is attempted with a write-only key; the runbook
> says how to find out whether restic tolerates it and what to record either way.
> **Executed 2026-09-11** (restic 0.17.x from trixie; record the exact version
> from `restic version` in the session log). Outcome of §4a: the write-only key
> *can* hide snapshots and packs (restic's `forget` is a soft delete on B2's S3
> endpoint, needing only `writeFiles`) but cannot destroy them — no `deleteFiles`,
> no `writeBucketLifecycleRules`. Everything hidden is recoverable for 30 days
> with the laptop's admin key; measured with `b2 ls --versions`.

Lane: **admin** (your sudo user, `sudo -i`). Time: 2–3 h including the wait for
the first upload. Stopping points are marked ▶.

## 0 — Human steps in the B2 console (15 min)

1. Create bucket `<bucket>` — private, **Object Lock off** for now (stage 3),
   lifecycle rule **"keep prior versions for 30 days"** (not "keep only last
   version" as on the laptop: here prior versions *are* the deletion
   resistance).
2. Create application key **`netcup-vps-restic`**, restricted to that bucket,
   capabilities exactly: `listBuckets, listFiles, readFiles, writeFiles`.
   Note the key ID and key in KeePassXC (`KeePassXC: netcup-vps-restic-b2`).
3. Create a second key **`laptop-vps-restic-admin`**, same bucket, full
   capabilities. This one lives only in KeePassXC and is used from the laptop
   for restore tests and `forget --prune`.
4. Generate the repository password on the laptop: `openssl rand -base64 32`
   → KeePassXC (`KeePassXC: netcup-vps-restic-repo`). The VPS gets a copy in
   step 2; the laptop copy is the one that matters for recovery.

▶ Stop here if interrupted; nothing on the host has changed.

## 1 — Install (5 min)

```bash
sudo -i
apt update && apt install -y restic
restic version                     # record it in the session log
```

## 2 — Credentials via LoadCredential (10 min)

Three root-owned files, one line each, no trailing newline issues (a second line
changes the password — the laptop spec learned this).

```bash
install -d -m 700 /etc/credstore
printf '%s' '<repo password>'   > /etc/credstore/restic-password
printf '%s' '<B2 key id>'       > /etc/credstore/restic-b2-key-id
printf '%s' '<B2 app key>'      > /etc/credstore/restic-b2-key
chmod 600 /etc/credstore/restic-*
for f in /etc/credstore/restic-*; do printf '%s: %s lines\n' "$f" "$(wc -l < "$f")"; done   # want 0 lines each (no newline)
history -c                          # the values were on the command line
```

Wrapper so the same command works under systemd (reads `$CREDENTIALS_DIRECTORY`)
and interactively as root (reads `/etc/credstore`):

```bash
cat > /usr/local/bin/restic-b2 <<'EOF'
#!/bin/sh
set -eu
d="${CREDENTIALS_DIRECTORY:-/etc/credstore}"
export AWS_ACCESS_KEY_ID="$(cat "$d/restic-b2-key-id")"
export AWS_SECRET_ACCESS_KEY="$(cat "$d/restic-b2-key")"
export RESTIC_PASSWORD_FILE="$d/restic-password"
export RESTIC_REPOSITORY="s3:s3.eu-central-003.backblazeb2.com/<bucket>"
exec /usr/bin/restic --cache-dir /var/cache/restic "$@"
EOF
chmod 755 /usr/local/bin/restic-b2
```

Replace `<bucket>` and the region host with what the B2 console shows for your
bucket (the laptop's is `eu-central-003`; verify yours).

## 3 — Decide what is backed up (15 min)

The subvolume layout ([[../decisions/20260830-subvolume-layout|ADR-0026]]) makes
this a mount question. List them:

```bash
findmnt -t btrfs -o TARGET,SOURCE,OPTIONS
```

`--one-file-system` treats each btrfs subvolume as its own filesystem, so every
subvolume you want must be named on the command line. Expected set (adjust to
`findmnt`): `/ /home /var/log` plus any subvolume holding `/etc`, `/root`,
`/srv` or `/opt` if separate. **Excluded by omission:** the containers subvolume
(rebuildable) and `/var/cache`.

```bash
install -d -m 755 /etc/restic
cat > /etc/restic/excludes.txt <<'EOF'
# correctness
/proc
/sys
/dev
/run
/tmp
/var/tmp
/.snapshots
# rebuildable
/var/cache
/var/lib/containers
/home/*/.cache
/home/*/.local/share/containers
/home/*/.npm
**/node_modules
# credentials — ADR-0034 / ADR-0017 mirror; add every row of the ADR-0034 table that is a file
/etc/credstore
/home/*/.claude/.credentials.json
/home/*/.codex/auth.json
/home/*/.local/state/devbox/opencode/auth.json
/home/*/.config/secrets
/home/*/.ssh/gh-*
EOF
```

`/etc/credstore` is excluded on purpose: the restic password backed up inside
the repository it protects is useless, and the B2 key in a snapshot survives its
own revocation ([[../../arch-thinkpad/decisions/20260825-harness-state-and-identity|ADR-0017]]'s revocation argument).

## 4 — Init and first backup (5 min + upload time)

```bash
restic-b2 init
restic-b2 backup --one-file-system --exclude-caches --exclude-file /etc/restic/excludes.txt \
  --tag auto --host netcup-vps / /home /var/log        # + the subvolumes from step 3
restic-b2 snapshots
```

▶ Stop point. A repository with one snapshot exists.

### 4a — The lock test (decides the key's capabilities)

restic writes a lock file at the start of every operation and **deletes** it at
the end. A key without `deleteFiles` may leave locks behind, and the next run may
refuse to start. Find out:

```bash
restic-b2 list locks                 # want: empty after the backup above
restic-b2 backup --one-file-system --exclude-file /etc/restic/excludes.txt --tag auto --host netcup-vps / /home /var/log
restic-b2 list locks
```

- **Both empty:** the write-only key works — locks are hidden, not deleted.
  **Measured 2026-09-11: both empty.** Kept. But note the second finding below.
- **Locks remain or the second run errors on locks:** add `deleteFiles` to
  `netcup-vps-restic` in the B2 console (no other capability). Deletion
  resistance then rests on the 30-day prior-version lifecycle alone, and hard
  deletion of versions by a key holder is the accepted residual. Record that
  instead. Object Lock stays stage 3 (restic/restic#3491 says `forget` and
  Object Lock do not mix yet — verify before then).

**Second finding (measured 2026-09-11):** `restic-b2 forget <id> --prune` from
the VPS *succeeds* — the same hide fallback removes snapshot and pack files
from restic's view. `b2 ls --versions b2://<bucket>/snapshots/` from the laptop
then shows the hidden snapshot as a prior version under its hide marker. So the
key gives deletion *resistance*, not deletion *proofing*: the 30-day
prior-version lifecycle is load-bearing, and the VPS key cannot shorten it
(no `writeBucketLifecycleRules`) or hard-delete versions (no `deleteFiles`).
Recovery after a malicious hide = laptop admin key, un-hide or copy the
versions back, `restic check`. A drill for that is stage 3.

Either way, **`forget --prune` for retention runs from the laptop** — see §7.

Test gotchas: `--keep-last 0` is rejected ("no policy was specified"); forget a
throwaway snapshot **by ID** to force a real delete attempt.

## 5 — Units (20 min)

```bash
cat > /etc/systemd/system/restic-b2.service <<'EOF'
[Unit]
Description=Daily restic backup to B2
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
Nice=10
IOSchedulingClass=idle
CacheDirectory=restic
StateDirectory=restic
LoadCredential=restic-password:/etc/credstore/restic-password
LoadCredential=restic-b2-key-id:/etc/credstore/restic-b2-key-id
LoadCredential=restic-b2-key:/etc/credstore/restic-b2-key
ExecStart=/usr/local/bin/restic-b2 backup --one-file-system --exclude-caches --exclude-file /etc/restic/excludes.txt --tag auto --host netcup-vps / /home /var/log
ExecStartPost=/usr/bin/touch /var/lib/restic/last-success
EOF

cat > /etc/systemd/system/restic-b2.timer <<'EOF'
[Unit]
Description=Daily restic backup to B2

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
EOF

cat > /etc/systemd/system/restic-b2-check.service <<'EOF'
[Unit]
Description=Monthly restic integrity check
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
Nice=15
IOSchedulingClass=idle
CacheDirectory=restic
LoadCredential=restic-password:/etc/credstore/restic-password
LoadCredential=restic-b2-key-id:/etc/credstore/restic-b2-key-id
LoadCredential=restic-b2-key:/etc/credstore/restic-b2-key
ExecStart=/usr/local/bin/restic-b2 check --read-data-subset=5%%
EOF

cat > /etc/systemd/system/restic-b2-check.timer <<'EOF'
[Unit]
Description=Monthly restic integrity check

[Timer]
OnCalendar=monthly
RandomizedDelaySec=6h
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemd-analyze verify /etc/systemd/system/restic-b2*.service   # want: silent
systemctl daemon-reload
systemctl enable --now restic-b2.timer restic-b2-check.timer
systemctl start restic-b2.service && systemctl status restic-b2.service --no-pager
ls -l /var/lib/restic/last-success
```

Adjust the subvolume list in both `ExecStart=` lines to match step 3. Note
`network-online.target` on this host depends on `systemd-networkd-wait-online`
or the ifupdown equivalent being enabled — check `systemctl is-enabled
systemd-networkd-wait-online.service`; the laptop learned that the target is
only worth what its wait-online unit is.

▶ Stop point. The timer owns backups from here.

## 6 — Staleness alarm (10 min)

No desktop here, so the alarm is a failing unit plus, optionally, an ntfy push
(pulls stage 2.1 forward by ten minutes — recommended).

```bash
cat > /usr/local/bin/restic-b2-stale-check <<'EOF'
#!/bin/sh
# exit 1 when the marker is missing/old or the monthly check failed
set -u
days="${RESTIC_STALE_DAYS:-3}"
m=/var/lib/restic/last-success
if [ ! -f "$m" ]; then echo "no successful backup recorded"; exit 1; fi
if [ -n "$(find "$m" -mtime +"$days")" ]; then echo "last success older than $days days"; exit 1; fi
if systemctl is-failed --quiet restic-b2-check.service; then echo "integrity check failed"; exit 1; fi
echo "ok: $(stat -c %y "$m")"
EOF
chmod 755 /usr/local/bin/restic-b2-stale-check

# optional ntfy. NOTE (measured 2026-09-12 via `systemctl cat`): the unit actually deployed is the
# topic-only form — `LoadCredential=ntfy-topic:/etc/credstore/ntfy-topic`, no token, no account,
# `-m 20`, Priority high. The token variant below is what this runbook proposed; treat it as inferred.
cat > /etc/systemd/system/ntfy-alert@.service <<'EOF'
[Unit]
Description=ntfy alert for %i

[Service]
Type=oneshot
LoadCredential=ntfy-token:/etc/credstore/ntfy-token
ExecStart=/bin/sh -c 'curl -fsS -H "Authorization: Bearer $(cat $CREDENTIALS_DIRECTORY/ntfy-token)" -H "Title: netcup-vps: %i failed" -d "$(systemctl status %i --no-pager -n 5)" https://ntfy.sh/<topic>'
EOF

cat > /etc/systemd/system/restic-b2-stale.service <<'EOF'
[Unit]
Description=restic staleness check
OnFailure=ntfy-alert@%n.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/restic-b2-stale-check
EOF

cat > /etc/systemd/system/restic-b2-stale.timer <<'EOF'
[Unit]
Description=Daily restic staleness check

[Timer]
OnCalendar=*-*-* 09:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

# same OnFailure= on the backup and check units:
for u in restic-b2 restic-b2-check; do
  mkdir -p /etc/systemd/system/$u.service.d
  printf '[Unit]\nOnFailure=ntfy-alert@%%n.service\n' > /etc/systemd/system/$u.service.d/alert.conf
done
systemctl daemon-reload
systemctl enable --now restic-b2-stale.timer
systemctl start restic-b2-stale.service && echo PASS
```

If you skip ntfy, delete the `OnFailure=` lines; the failing unit still shows in
`systemctl --failed`, which you check when you log in. The ntfy topic name is a
secret-ish value (anyone who knows it can subscribe); use a random one.

## 7 — Retention runs from the laptop (10 min, then monthly)

The VPS key may not delete (§4a), and even if it can, keeping prune authority off
the host is the point. On arch-thinkpad, with the `laptop-vps-restic-admin` key
and the repo password from KeePassXC:

Start a shell that records no history — **run this line alone, then paste
the rest**; pasted as one block, the new shell swallows the following lines
(cost two attempts on 2026-09-11):

```bash
HISTFILE= zsh
```

```bash
read -rs 'AWS_ACCESS_KEY_ID?admin key ID: '; echo        # zsh syntax; bash would be read -rsp
read -rs 'AWS_SECRET_ACCESS_KEY?admin app key: '; echo
read -rs 'RESTIC_PASSWORD?repo password: '; echo
export AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY RESTIC_PASSWORD
export RESTIC_REPOSITORY=s3:s3.eu-central-003.backblazeb2.com/netcup-vps-restic-k7q2
restic snapshots
restic forget --tag auto --host netcup-vps --keep-daily 7 --keep-weekly 5 --keep-monthly 12 --prune
exit
```

(`AWS_*` because restic's S3 backend reads Amazon's variable names; B2's S3
endpoint speaks the same protocol. No Amazon account involved.)

Monthly, by hand for now; becomes a laptop timer if it is forgotten twice.

## 8 — Restore test from the laptop (15 min) — the acceptance criterion

```bash
# same environment as §7
restic restore latest --target /tmp/vps-restore --include /etc/hostname --include /var/log/dpkg.log
diff <(ssh netcup-tailnet cat /etc/hostname) /tmp/vps-restore/etc/hostname && echo RESTORE=PASS
rm -rf /tmp/vps-restore
```

Then the negative test, on the VPS, which must **fail**:

```bash
restic-b2 backup --tag test /etc/hostname
restic-b2 forget <that snapshot id> --prune      # 2026-09-11: SUCCEEDS (hide, not delete) — see §4a second finding
```
Then from the laptop, admin key: `b2 ls --versions b2://netcup-vps-restic-k7q2/snapshots/`
must still list the forgotten snapshot's file — that is the recoverability proof.

## Done when

- `restic-b2 snapshots` lists ≥ 2 snapshots, taken by the timer, not by hand.
- `/var/lib/restic/last-success` is fresher than 24 h; `restic-b2-stale.service` passes.
- Restore from the laptop verified byte-identical (§8).
- §4a outcome recorded in ADR-0034's table and in `backlog/restic-client.md` (close it). **Done 2026-09-11.**
- `systemctl --failed` is empty.

## Related

- [[../backlog/restic-client|restic-client]] — the backlog item this closes; note its target was the Storage Box, which ADR-0028 found does not exist
- [[../../arch-thinkpad/spec/backups|arch-thinkpad spec/backups]] — the laptop's version, with the cache-dir and wait-online lessons
- [[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — where the key's capabilities are recorded
- [[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]] — why `LoadCredential=`
