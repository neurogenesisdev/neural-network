---
title: Backups — restic to Backblaze B2, and the T5 as the second copy
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-31
commit: pending
created: 2026-08-30
updated: 2026-09-02
tags: [spec, backups, restic, b2, systemd, metered]
---

# Backups

Two repositories, one exclusion set, one timer. The remote is Backblaze B2
([[../decisions/20260830-remote-repository-is-b2|ADR-0028]]); the Samsung T5 is
the second copy and stays manual
([[../decisions/20260818-backup-cadence-open|ADR-0012]]). What is excluded, and on
what grounds, is [[../decisions/20260818-restic-exclusions|ADR-0008]] — this
chapter installs the list, it does not argue it.

Read [[_schema|_schema.md]] first. Every step here is booted-system: all of them
have `first-boot` in their `requires` closure, so all commands use `sudo`.

**No secret appears in a step record.** The repository password and the B2 key are
written by a human into root-owned files, and every postcondition below reads the
credentials back out of `/etc/restic/b2.env` at run time rather than naming a
bucket, an endpoint or a key. That is *assert against live state* applied to
credentials: an endpoint copied into this document is both a constant that decays
and an identifier the redaction rules would have to cover.

**Two invocations, one exclusion file.** The T5 and B2 repositories are backed up
with the same paths and the same excludes, and are interchangeable in what they
cover but not in their passwords. They diverged before 2026-08-30 — the T5 held
`/etc /home /var/log` where B2 holds `/ /home /var/log` — and the T5 was
re-pointed at the wider set the same evening.

## restic-install

The package. Nothing here pins a version; restic's repository format is stable and
the exclusion semantics used below are not version-specific. Measured 2026-08-30:
`restic 0.19.1-1`.

```yaml
id: restic-install
requires: [first-boot]
command: sudo pacman -S --needed restic
postcondition: |
  set -uo pipefail
  v=$(restic version 2>/dev/null) || v=
  test -n "$v" && echo "RESULT=PASS $v" || echo "RESULT=FAIL restic absent"
```

## restic-b2-credentials

**No `command`: a human must satisfy this.** Three things have to happen outside
any shell this executor could run — a private B2 bucket with its lifecycle rule
set to *keep only the last version of the file*, an application key scoped to that
one bucket with read and write access, and a repository password generated with
`openssl rand -base64 32` and stored in KeePassXC **before** anything is written
to the repository.

The lifecycle rule is not cosmetic. Without it, every pack file `prune` deletes is
retained as a hidden version and billed indefinitely.

The postcondition cannot check the bucket or the key — it checks what must be true
on this machine for the rest of the chapter to work, and does so without printing a
secret. A password file of two lines is the failure worth catching: a trailing blank
line changes the password, and the error surfaces later as a repository that cannot
be decrypted.

```yaml
id: restic-b2-credentials
requires: [restic-install]
postcondition: |
  set -uo pipefail
  envf=/etc/restic/b2.env
  pw=/etc/restic/repo-password
  fail=0
  for f in "$envf" "$pw"; do
    m=$(sudo stat -c '%a %U' "$f" 2>/dev/null) || m=
    test "$m" = "600 root" || { echo "bad or missing: $f ($m)"; fail=1; }
  done
  n=$(sudo awk 'END { print NR + 0 }' "$pw" 2>/dev/null) || n=0
  test "$n" = "1" || { echo "password file is $n lines, want 1"; fail=1; }
  k=$(sudo awk -F= '/^(AWS_ACCESS_KEY_ID|AWS_SECRET_ACCESS_KEY|RESTIC_REPOSITORY)=./ { c++ } END { print c + 0 }' "$envf" 2>/dev/null) || k=0
  test "$k" = "3" || { echo "b2.env defines $k of 3 required keys"; fail=1; }
  test "$fail" = "0" && echo "RESULT=PASS" || echo "RESULT=FAIL"
```

## restic-exclusions

The list from [[../decisions/20260818-restic-exclusions|ADR-0008]]'s 2026-08-30
amendment, written with `printf` rather than a heredoc so the command survives being
copied out of a rendered page with its indentation.

The postcondition asserts five load-bearing entries rather than the whole file.
Checking every line would fail on any future addition, which is the permanently-red
shape; checking these five fails only if the list has been truncated or replaced.
`/home/.snapshots` is the 73 GiB recursion, `/swap` is a 36 GB swapfile, and the
three agent-state paths are the revocation argument.

**`paste-cache` is asserted by name for a reason**, and it is the only entry here
whose absence has already happened: it was dropped during the 2026-08-30
simplification by a rule about storage cost, although it is a credentials-ground
exclusion. It is the mechanical half of ADR-0017's mitigation for pasted secrets
reaching the repository. A check that would not have caught the one failure this
list has actually suffered is not much of a check.

```yaml
id: restic-exclusions
requires: [restic-install]
command: |
  sudo install -d -m 0700 /etc/restic
  sudo sh -c 'printf "%s\n" \
    /proc /sys /dev /run /tmp /var/tmp /mnt /media /lost+found \
    /swap /.snapshots /home/.snapshots \
    /home/luiswg/.config/Claude/vm_bundles \
    /home/luiswg/.config/discord \
    /var/cache /home/luiswg/.cache /var/log/journal \
    /home/luiswg/.local/share/Trash \
    /home/luiswg/.local/share/containers \
    /home/luiswg/.local/share/baloo \
    /home/luiswg/.local/state/devbox/claude/.credentials.json \
    /home/luiswg/.local/state/devbox/claude/.claude.json \
    /home/luiswg/.local/state/devbox/claude/plugins \
    /home/luiswg/.local/state/devbox/claude/paste-cache \
    /home/luiswg/.local/state/devbox/claude/shell-snapshots \
    /home/luiswg/.local/state/devbox/claude/backups \
    /home/luiswg/.local/state/devbox/codex-state/auth.json \
    > /etc/restic/excludes.txt'
  sudo chmod 0644 /etc/restic/excludes.txt
postcondition: |
  set -uo pipefail
  f=/etc/restic/excludes.txt
  n=$(sudo awk '
    $0 == "/home/.snapshots" { a = 1 }
    $0 == "/swap" { b = 1 }
    $0 ~ /claude\/[.]credentials[.]json$/ { c = 1 }
    $0 ~ /claude\/paste-cache$/ { d = 1 }
    $0 ~ /codex-state\/auth[.]json$/ { e = 1 }
    END { print (a + b + c + d + e) + 0 }' "$f" 2>/dev/null) || n=0
  test "$n" = "5" && echo "RESULT=PASS" || echo "RESULT=FAIL $n of 5 required exclusions present"
```

*Measured 2026-08-31:* `RESULT=PASS`.

## restic-b2-init

Creates the repository. The precondition is what stops this being destructive on a
re-run: `restic init` against an initialised repository refuses on its own, but the
check makes the refusal legible rather than an error to interpret under pressure.

```yaml
id: restic-b2-init
requires: [restic-b2-credentials, restic-exclusions]
precondition: |
  set -uo pipefail
  sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password cat config' >/dev/null 2>&1 \
    && { echo "repository already initialised"; exit 1; }
  exit 0
command: |
  sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password init'
postcondition: |
  set -uo pipefail
  id=$(sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password cat config' 2>/dev/null \
    | awk -F'"' '/"id"/ { print $4 }') || id=
  test -n "$id" && echo "RESULT=PASS repository ${id}" || echo "RESULT=FAIL cannot open repository"
```

## restic-b2-backup

The invocation. `--one-file-system` keeps it out of removable media — the T5 mounts
under `/run/media`, which the exclusion list also covers, so the drive is out twice
over. The three paths are named explicitly because `@home`, `@log` and the root
subvolume are separate btrfs mounts, and `--one-file-system` would otherwise stop at
each boundary.

The postcondition asserts a *recent* snapshot rather than any snapshot: a repository
holding only August's snapshots passes an existence check, and that is exactly the
state this chapter exists to prevent.

*Measured 2026-08-30:* 20.015 GiB processed, 249,561 files, **12.809 GiB stored**,
1h18m33s, over a link that lost DNS resolution twice mid-run and recovered through
restic's own retry.

```yaml
id: restic-b2-backup
requires: [restic-b2-init]
command: |
  sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password backup \
      --one-file-system --exclude-caches \
      --exclude-file /etc/restic/excludes.txt \
      --tag auto --host arch-thinkpad \
      / /home /var/log'
postcondition: |
  set -uo pipefail
  t=$(sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password snapshots \
      --host arch-thinkpad --latest 1 --json' 2>/dev/null \
    | awk -F'"' '/"time"/ { print $4; exit }') || t=
  test -n "$t" || { echo "RESULT=FAIL no snapshot for this host"; exit 0; }
  age=$(( ( $(date +%s) - $(date -d "$t" +%s) ) / 3600 ))
  test "$age" -lt 48 \
    && echo "RESULT=PASS latest snapshot ${age}h old" \
    || echo "RESULT=FAIL latest snapshot ${age}h old"
```

## restic-metered-guard

**Amended 2026-09-08 — the guard also requires connectivity.** Three timer runs
(2026-09-04/05/06) got past the metered check and failed: each fired on resume
while the Wi-Fi was associated to a network with no route out —
NetworkManager `CONNECTED_LOCAL`, device state `100`, so `network-online.target`
was satisfied and the metered field was not `yes`. restic retried DNS for
fifteen minutes and exited 1. The script now exits 1 unless
`nmcli networking connectivity` reports `full`, NetworkManager's own
captive-portal probe (`/usr/lib/NetworkManager/conf.d/20-connectivity.conf`,
enabled by the package). The script body below is the current one.


The machine is often on a phone hotspot with a data allowance, so the backup must
not run over a metered link. `ExecCondition=` is the mechanism: a non-zero exit
makes systemd **skip** the unit rather than fail it, so a tethered day leaves an
`inactive` unit and one journal line instead of a red timer nobody trusts.

Two things this guard gets right that an inline check would not:

- **It ignores `tun` devices.** With Mullvad as a Tailscale exit node the default
  route is `tailscale0`, so asking whether *the route* is metered answers about the
  tunnel rather than about what carries it.
- **It is a file, not a shell fragment inside a unit**, so it can be run by hand.
  That is the only way to exercise the negative case deliberately rather than
  waiting to be tethered and hoping to notice.

**NetworkManager guesses `no` for a phone hotspot**, because a hotspot over wifi
looks like ordinary wifi. The connection must be marked explicitly — that mark is
per-connection state on the machine and nothing this script can establish, which is
why it is the precondition rather than part of the command.

The script is written with `printf` rather than a heredoc, for the reason
[[_schema|_schema.md]] gives: a heredoc terminator does not survive being copied
with indentation, and a step whose command silently writes a broken script is worse
than one that is tedious to read. Bracket forms — `[.]` rather than an escaped dot —
remove one layer of backslashes that could be lost in transit.

**Verify the installed file with `cat /usr/local/bin/restic-not-metered` after
running the command.** The postcondition checks that the guard *answers*, not that it
says what you meant.

```yaml
id: restic-metered-guard
requires: [restic-install]
precondition: |
  set -uo pipefail
  m=$(nmcli -t -f connection.metered connection show "${HOTSPOT:?name of the hotspot connection, e.g. the phone}" 2>/dev/null | awk -F: '{ print $2 }') || m=
  test "$m" = "yes" || { echo "connection ${HOTSPOT} has metered=${m:-unset}, want yes"; exit 1; }
  exit 0
command: |
  sudo sh -c 'cat > /usr/local/bin/restic-not-metered' < scripts/restic-not-metered
  sudo chmod 0755 /usr/local/bin/restic-not-metered
postcondition: |
  set -uo pipefail
  test -x /usr/local/bin/restic-not-metered || { echo "RESULT=FAIL guard absent"; exit 0; }
  /usr/local/bin/restic-not-metered && v=unmetered || v=metered
  echo "RESULT=PASS guard runs, current link ${v}"
```

**The `command:` reads the script from a file rather than embedding it**, and that
is a deliberate departure from every other step in this spec. A step record is one
fenced block, and a shell script inside a `printf` inside a single-quoted `sh -c`
needs three layers of quoting for an `awk` program that itself uses both quote
characters. The embedded version was written, and it was unreadable enough that a
transcription error in it would not have been visible on review — which is the
failure mode the convention exists to prevent, reached by obeying the convention.
The script's own text is below, and lives at `scripts/restic-not-metered` in this
vault.

```bash
#!/bin/bash
# exit 0 = unmetered AND online, proceed. exit 1 = metered or not actually online, skip.
#
# Two checks, because two things went wrong. Metered (2026-08-31): the phone
# hotspot must not carry a backup. Connectivity (added 2026-09-08): three runs
# fired on resume while wlp3s0 was associated to a network with no route out —
# NetworkManager state CONNECTED_LOCAL, device state 100 — and restic retried
# DNS for fifteen minutes before failing. network-online.target is satisfied by
# an associated link; only NM's captive-portal probe says whether the internet
# is reachable. "full" is the only answer that means it is.
set -uo pipefail
conn=$(nmcli -t networking connectivity 2>/dev/null || echo unknown)
[ "$conn" = "full" ] || { echo "restic-not-metered: connectivity is '$conn', not 'full' — skipping" >&2; exit 1; }
hit=$(nmcli -t -f GENERAL.TYPE,GENERAL.STATE,GENERAL.METERED device show 2>/dev/null | awk -F: '
  /^GENERAL\.TYPE:/    { type = $2 }
  /^GENERAL\.STATE:/   { state = $2 }
  /^GENERAL\.METERED:/ {
    if (type != "loopback" && type != "tun" && state ~ /^100/ && $2 ~ /^yes/) n = 1
  }
  END { print n + 0 }')
test "$hit" = "0"
```

**The postcondition asserts that the guard answers, not what it answers.** Either
verdict is correct depending on where the laptop is, so a check demanding
`unmetered` would be red on a train and would be measuring the operator's location
rather than the machine's configuration.

*Measured 2026-08-31:* with the phone's connection marked `connection.metered yes`,
the guard returned `METERED`, and a manual `systemctl start restic-b2.service`
logged `Skipped due to 'exec-condition'` with `systemctl is-failed` reporting
`inactive`. **The negative path is the one that has been exercised**; the unmetered
path has never been seen, and it is the path that actually takes a backup.

## restic-b2-timer

**Amended 2026-09-02 — the units need a cache directory.** System services run
with no `$HOME`, and `restic backup` refuses to start without a cache location
(`restic check` builds a temporary one and so never showed the gap). Both units
now carry `CacheDirectory=restic` and `--cache-dir /var/cache/restic` on every
`ExecStart=`; the step records below are amended in place. Found by running the
T5 backup as a transient unit, not by the B2 timer, which had never got past its
metered guard. See [[areas/infrastructure/arch-thinkpad/state-notes|notes]].


Daily, with an hour of jitter, and `Persistent=true` so a run missed while the
machine was asleep is taken after the next boot instead of skipped. That clause is
the whole of [[../decisions/20260818-backup-cadence-open|ADR-0012]]'s answer.

`Persistent=true` and the metered guard interact in a way worth naming: a catch-up
run fires shortly after boot, which is disproportionately often when tethered. The
guard turns that into a skip rather than a surprise upload, which is why it is a
dependency of this step and not an optional extra.

`forget --prune` runs as a second `ExecStart` in the same unit, so retention is
applied by the same thing that takes the snapshot and cannot drift from it.

**The password reaches the unit as `RESTIC_PASSWORD_FILE`, not as a value.** The
`netcup-vps` half of this work argues for `LoadCredential=` instead
([[../../netcup-vps/backlog/restic-client|restic-client]]); the argument there is
that the machine's disk is unencrypted, which is not true here. The two hosts
therefore diverge on purpose, and the divergence is the reason to say so.

```yaml
id: restic-b2-timer-install
requires: [restic-b2-backup, restic-metered-guard]
command: |
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=restic backup to Backblaze B2" \
    "After=network-online.target" "Wants=network-online.target" "" \
    "[Service]" "Type=oneshot" "Nice=10" "IOSchedulingClass=idle" "CacheDirectory=restic" \
    "ExecCondition=/usr/local/bin/restic-not-metered" \
    "EnvironmentFile=/etc/restic/b2.env" \
    "Environment=RESTIC_PASSWORD_FILE=/etc/restic/repo-password" \
    "ExecStart=/usr/bin/restic backup --one-file-system --exclude-caches --exclude-file /etc/restic/excludes.txt --tag auto --host arch-thinkpad / /home /var/log" \
    "ExecStart=/usr/bin/restic forget --tag auto --host arch-thinkpad --prune --keep-daily 7 --keep-weekly 5 --keep-monthly 12" \
    > /etc/systemd/system/restic-b2.service'
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=Daily restic backup to B2" "" \
    "[Timer]" "OnCalendar=daily" "RandomizedDelaySec=1h" "Persistent=true" "" \
    "[Install]" "WantedBy=timers.target" \
    > /etc/systemd/system/restic-b2.timer'
  sudo systemctl daemon-reload
  sudo systemctl enable --now restic-b2.timer
postcondition: |
  set -uo pipefail
  n=$(sudo systemd-analyze verify /etc/systemd/system/restic-b2.service 2>&1 | awk 'END { print NR + 0 }') || n=1
  test "$n" = "0" && echo "RESULT=PASS unit verifies" || echo "RESULT=FAIL unit has warnings"
```

```yaml
id: restic-b2-timer
requires: [restic-b2-timer-install]
postcondition: |
  set -uo pipefail
  s=$(systemctl is-enabled restic-b2.timer 2>/dev/null) || s=
  n=$(systemctl show -p NextElapseUSecRealtime --value restic-b2.timer 2>/dev/null) || n=
  test "$s" = "enabled" && test -n "$n" && test "$n" != "0" \
    && echo "RESULT=PASS next ${n}" \
    || echo "RESULT=FAIL is-enabled=${s} next=${n}"
```

*Measured 2026-08-31:* enabled, `systemd-analyze verify` silent, next elapse
`Tue 2026-09-01 00:10:28 CEST`.

**`Wants=network-online.target` is only worth what
`NetworkManager-wait-online.service` is.** If that unit is not enabled, the backup
can start before any interface is up, and the failure presents as name-resolution
errors rather than as a missing dependency — the same output the 2026-08-30 run
produced from a genuine wifi drop, which is what makes the two hard to tell apart.

*Measured 2026-08-31:* `enabled`. Asserted below rather than left as prose, because
it is a dependency of this unit that lives outside it and nothing else in the chapter
would notice it being disabled by a future change.

```yaml
id: network-online-is-real
requires: [restic-b2-timer-install]
postcondition: |
  set -uo pipefail
  s=$(systemctl is-enabled NetworkManager-wait-online.service 2>/dev/null) || s=
  test "$s" = "enabled" \
    && echo "RESULT=PASS network-online.target is backed by a waiter" \
    || echo "RESULT=FAIL NetworkManager-wait-online is ${s:-absent}; Wants=network-online.target is decorative"
```

## restic-t5-backup

The second copy. **No automation, by decision** — the drive is a VeraCrypt volume
mounted by hand, so the precondition is the mount and the trigger is a human
noticing the drive is attached.

Its password is at `/root/.restic-password` and is **not** the B2 password. Both are
in KeePassXC as of 2026-08-30; before that, the T5 password existed only in that
file, on the disk it was backing up.

*Measured 2026-08-30:* the same 20.015 GiB input stored in **8.819 GiB** here
against B2's 12.809 GiB — the difference is deduplication against the August
snapshots this repository already held, not a difference in compression.

```yaml
id: restic-t5-backup
requires: [restic-exclusions]
precondition: |
  set -uo pipefail
  d=/run/media/veracrypt1/backups/restic/arch-thinkpad
  test -d "$d" || { echo "T5 not mounted at $d"; exit 1; }
  exit 0
command: |
  sudo restic -r /run/media/veracrypt1/backups/restic/arch-thinkpad \
    --password-file /root/.restic-password backup \
    --one-file-system --exclude-caches \
    --exclude-file /etc/restic/excludes.txt \
    --tag auto --host arch-thinkpad \
    / /home /var/log
postcondition: |
  set -uo pipefail
  d=/run/media/veracrypt1/backups/restic/arch-thinkpad
  p=$(sudo restic -r "$d" --password-file /root/.restic-password snapshots \
    --host arch-thinkpad --latest 1 --json 2>/dev/null \
    | awk -F'"' '/\/var\/log/ { print "wide"; exit }') || p=
  test "$p" = "wide" \
    && echo "RESULT=PASS latest T5 snapshot covers the wide path set" \
    || echo "RESULT=FAIL latest T5 snapshot does not cover /var/log"
```

## restic-restore-test

**An untested backup is a hypothesis.** This step is what converts the chapter from
an upload into a backup, and as of 2026-08-31 **no restore from the B2 repository
has been performed** — see [[../runbooks/restic-restore|runbooks/restic-restore]],
which is the procedure a human follows under pressure and which duplicates this
verification deliberately.

A single small file is enough to prove decryption, index and transport together. It
does not prove a bare-metal restore, and nothing here claims it does.

```yaml
id: restic-restore-test
requires: [restic-b2-backup]
command: |
  sudo sh -c 'set -a; . /etc/restic/b2.env; set +a; \
    restic --password-file /etc/restic/repo-password restore latest \
      --target /tmp/restic-test --include /home/luiswg/.bashrc'
postcondition: |
  set -uo pipefail
  a=/tmp/restic-test/home/luiswg/.bashrc
  b=/home/luiswg/.bashrc
  test -f "$a" || { echo "RESULT=FAIL nothing restored"; exit 0; }
  sudo cmp -s "$a" "$b" \
    && echo "RESULT=PASS restored file matches live" \
    || echo "RESULT=FAIL restored file differs from live"
```

## restic-check-timer

Monthly integrity verification. `restic check` alone validates structure and
metadata; `--read-data-subset=5%` additionally downloads and re-hashes a sample of
pack files, which is the only thing that detects **bit rot at the remote** — damage
to data restic is not otherwise reading until a restore needs it.

At 12.809 GiB stored that is roughly 640 MB fetched per run. B2's free egress
allowance is three times average monthly storage, so the sample costs nothing;
**the metered guard is on this unit for the same reason it is on the backup**, and
for a sharper one — 640 MB of verification traffic over a phone's data allowance
buys no recovery capability at all.

**A failure here means something different from a failed backup.** A missed backup
is self-correcting: the next run takes one. A failed check means the repository is
damaged, does not improve on its own, and wants attention rather than a retry next
month. Nothing notifies, which is
[[../backlog/backup-staleness-alarm|backup-staleness-alarm]]'s second case.

`%%` in `ExecStart` is not a typo — systemd reads `%` as a specifier prefix, so a
literal percent must be doubled. `%5` parses as an unknown specifier, and the
failure is at unit-load time rather than where the argument is used.

```yaml
id: restic-check-timer
requires: [restic-b2-timer-install]
command: |
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=restic integrity check (5%% sample) against B2" \
    "After=network-online.target" "Wants=network-online.target" "" \
    "[Service]" "Type=oneshot" "Nice=15" "IOSchedulingClass=idle" "CacheDirectory=restic" \
    "ExecCondition=/usr/local/bin/restic-not-metered" \
    "EnvironmentFile=/etc/restic/b2.env" \
    "Environment=RESTIC_PASSWORD_FILE=/etc/restic/repo-password" \
    "ExecStart=/usr/bin/restic --cache-dir /var/cache/restic check --read-data-subset=5%%%%" \
    > /etc/systemd/system/restic-b2-check.service'
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=Monthly restic integrity check" "" \
    "[Timer]" "OnCalendar=monthly" "RandomizedDelaySec=6h" "Persistent=true" "" \
    "[Install]" "WantedBy=timers.target" \
    > /etc/systemd/system/restic-b2-check.timer'
  sudo systemctl daemon-reload
  sudo systemctl enable --now restic-b2-check.timer
postcondition: |
  set -uo pipefail
  s=$(systemctl is-enabled restic-b2-check.timer 2>/dev/null) || s=
  f=$(systemctl is-failed restic-b2-check.service 2>/dev/null) || f=
  test "$s" = "enabled" || { echo "RESULT=FAIL timer is ${s:-absent}"; exit 0; }
  test "$f" != "failed" \
    && echo "RESULT=PASS timer enabled, last check ${f}" \
    || echo "RESULT=FAIL repository check failed — investigate, this does not self-correct"
```

**The postcondition asserts two different things on purpose.** That the timer exists,
and that the last run did not fail. The second is the one that matters and the one an
existence check would miss: a scheduled verifier reporting damage every month while
nobody reads the journal is worse than no verifier, because it looks like coverage.

*Measured 2026-08-31:* enabled, `systemd-analyze verify` silent, next elapse
`Tue 2026-09-01 01:56:36 CEST`. No check has run.

`printf` doubles the percent again inside the step record — `%%%%` reaches the unit
file as `%%`, which systemd reads as `%`. Three layers, each with its own escaping
rule, and the only way to be sure is
`grep read-data-subset /etc/systemd/system/restic-b2-check.service`.

## restic-staleness-alarm

Closes [[../backlog/backup-staleness-alarm|backup-staleness-alarm]]. The metered
guard makes a skipped backup silent by design; this is the mechanism that notices
when "skipped" has gone on too long. It reads a **local marker**, not B2 — the
question is asked most often on a metered link, and a marker cannot be fooled by a
repository that is unreachable rather than stale.

Two halves. `restic-b2.service` gains `StateDirectory=restic` and
`ExecStartPost=/usr/bin/touch /var/lib/restic/last-success`; systemd runs
`ExecStartPost=` only after every `ExecStart=` succeeded, so the marker asserts
that backup **and** prune completed. A user-session timer then runs
`/usr/local/bin/restic-b2-stale-check` daily and three minutes after login. It
exits 1 — the user unit shows `failed` — and raises a critical desktop notification
when the marker is missing, older than `RESTIC_STALE_DAYS` (default **7**, assumed
rather than argued), or `restic-b2-check.service` is `failed`. That last clause is
how the integrity check reaches a human; it has no channel of its own.

It runs in the **user** session, not the system one, because `notify-send` needs
the session bus and the moment the laptop is open is exactly when the warning can
be read. The units live in `/etc/systemd/user/` and are enabled with
`--global`, so nothing under `~/.config` is touched and the two survive a
dotfiles decision either way.

```yaml
id: restic-staleness-alarm
requires: [restic-b2-timer-install]
command: |
  # 1. marker, written by the backup unit on full success
  sudo sed -i 's|^IOSchedulingClass=idle$|&\nStateDirectory=restic|' /etc/systemd/system/restic-b2.service
  sudo sh -c 'printf "%s\n" "ExecStartPost=/usr/bin/touch /var/lib/restic/last-success" >> /etc/systemd/system/restic-b2.service'
  # 2. checker — script body is the one installed 2026-09-02; see the item for its logic
  sudo install -m 755 restic-b2-stale-check /usr/local/bin/restic-b2-stale-check
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=Warn if the B2 backup is stale or its integrity check failed" "" \
    "[Service]" "Type=oneshot" "ExecStart=/usr/local/bin/restic-b2-stale-check" \
    > /etc/systemd/user/restic-b2-stale.service'
  sudo sh -c 'printf "%s\n" \
    "[Unit]" "Description=Daily check that the B2 backup is not stale" "" \
    "[Timer]" "OnStartupSec=3min" "OnCalendar=daily" "Persistent=true" "" \
    "[Install]" "WantedBy=timers.target" \
    > /etc/systemd/user/restic-b2-stale.timer'
  sudo chmod 644 /etc/systemd/user/restic-b2-stale.service /etc/systemd/user/restic-b2-stale.timer
  sudo systemctl daemon-reload
  sudo systemctl --global enable restic-b2-stale.timer
  systemctl --user daemon-reload && systemctl --user start restic-b2-stale.timer
postcondition: |
  set -uo pipefail
  grep -q '^ExecStartPost=/usr/bin/touch /var/lib/restic/last-success$' /etc/systemd/system/restic-b2.service || { echo "RESULT=FAIL marker not written by backup unit"; exit 0; }
  t=$(systemctl --user is-active restic-b2-stale.timer 2>/dev/null) || t=
  test "$t" = "active" || { echo "RESULT=FAIL user timer is ${t:-absent}"; exit 0; }
  if systemctl --user is-failed restic-b2-stale.service >/dev/null 2>&1; then
    echo "RESULT=WARN alarm is raised — $(journalctl --user -u restic-b2-stale.service -n1 -o cat)"
  else
    echo "RESULT=PASS marker written on success, timer active, no alarm"
  fi
```

**The unit files must be world-readable.** Written under root's `umask 077` they
load as `bad` in the user manager and `systemctl --user start` reports *Unit not
found* — nothing says *permission denied*. Measured 2026-09-02, the first thing
this step got wrong.

*Measured 2026-09-02:* installed; the first run reported `No successful B2 backup
has ever been recorded by restic-b2.service`, unit `failed`, notification shown.
True — the backup timer has been guard-skipped on every elapse so far.

## What this chapter does not cover

- **A bare-metal restore.** The repository holds `/`, `/home` and `/var/log`, not
  the ESP and not the LUKS header. Recovery of an unbootable machine goes through
  [[../runbooks/bootloader-reinstall|bootloader-reinstall]] and the header backup
  named in [[../backlog/luks-second-keyslot|luks-second-keyslot]]; restic is the
  data half only.
- **Being told on the phone.** `restic-staleness-alarm` reaches the desktop and
  the user manager's failed list, nothing further. A closed laptop sees neither,
  and that condition correlates with being tethered. The phone route is the
  netcup question.
- **The netcup server's repository.** A second client against a target still to be
  chosen — [[../../netcup-vps/backlog/restic-client|netcup-vps/restic-client]].
