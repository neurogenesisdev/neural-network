---
title: Restore from restic — a file, a tree, or a machine that is gone
type: runbook
provenance: inferred
device: arch-thinkpad
stale: false
last_verified: 2026-08-30
commit: pending
created: 2026-08-30
tags: [runbook, backups, restic, b2, recovery]
---

# Restore from restic

**Fires when:** a file or directory is lost and snapper cannot supply it — the
snapshot has aged out, the whole subvolume is gone, or the NVMe is. Also the
drill: this procedure should be run deliberately once, on nothing important, so
that its first execution is not during an incident.

**Provenance warning, read this first.** This runbook is `inferred`. **No restore
from either repository has been performed.** The commands are written from
restic's own help output; what they have not done is run against these
repositories on this machine. The first person to use this is also testing it.

## 0. Which repository, and can you open it

Two repositories, two different passwords, both in KeePassXC.

| Repository | Where | Password |
|---|---|---|
| B2 (remote, daily) | `s3:…/…` — read it from `/etc/restic/b2.env` | KeePassXC; also `/etc/restic/repo-password` on the laptop |
| T5 (local, manual) | `/run/media/veracrypt1/backups/restic/arch-thinkpad` | KeePassXC; also `/root/.restic-password` on the laptop |

**If the laptop is the thing that is gone, both on-machine copies are gone with
it.** The KeePassXC database is the only remaining path to either, which is why it
is synced off-device. Establish that you can open KeePassXC *before* provisioning
a replacement machine.

## 1. Set up access on whatever machine you are using

On the laptop, the credentials are already in place and every command below can
use `--password-file`. Anywhere else, export them by hand — from KeePassXC, into
a shell you will close:

```bash
export AWS_ACCESS_KEY_ID=<keyID>
export AWS_SECRET_ACCESS_KEY=<applicationKey>
export RESTIC_REPOSITORY=s3:<endpoint>/<bucket>
restic snapshots          # prompts for the repository password
```

The endpoint and bucket are in `/etc/restic/b2.env` if you still have the machine,
and in the Backblaze console if you do not. Deliberately not written here: a
constant that decays, and an identifier this vault redacts.

If `restic snapshots` lists snapshots, everything downstream will work. If it
fails, stop and diagnose here rather than trying restore commands — the failure
modes (wrong password, wrong bucket, revoked key) all look like an unhelpful error
at the next step.

## 2. Find what you want

```bash
restic snapshots --host arch-thinkpad
restic find --host arch-thinkpad '<pattern>'
restic ls <snapshot-id> /home/luiswg/<path>
```

`restic find` takes a shell pattern and searches every snapshot, which is the
command to reach for when the question is *when did this last exist* rather than
*give me the latest*.

## 3. Restore

**One file, into a scratch directory.** Never restore over a live file as a first
move — you cannot compare afterwards if you have overwritten the thing you would
compare against.

```bash
restic restore latest --target /tmp/restic-test --include /home/luiswg/<path>
```

The restored file appears at `/tmp/restic-test/home/luiswg/<path>` — the target is
a *root*, and the original absolute path is recreated underneath it. Then compare
before moving anything back:

```bash
cmp /tmp/restic-test/home/luiswg/<path> /home/luiswg/<path>
```

**A tree**, same shape: `--include /home/luiswg/Documents`.

**Browsing instead of extracting**, when you do not know what you want yet:

```bash
mkdir -p /tmp/restic-mount
restic mount /tmp/restic-mount
```

Snapshots appear as directories; `Ctrl-C` unmounts. This needs FUSE and is
read-only. Over B2 it fetches on demand, so it is fine for looking around and slow
for copying gigabytes — use `restore` for those.

## 4. Rebuilding a lost machine

The repository holds `/`, `/home` and `/var/log`. **It does not hold the ESP, the
LUKS header, or anything excluded by
[[../decisions/20260818-restic-exclusions|ADR-0008]].** So this is the data half of
a rebuild, not the whole of one:

1. Install Arch and re-create the disk layout — [[../spec/disk|spec/disk]],
   [[../spec/base|spec/base]], [[../spec/boot|spec/boot]].
2. Restore `/home` from the newest snapshot into the new `@home`.
3. Restore `/etc` selectively rather than wholesale. Machine-specific files —
   `fstab`, `crypttab`, anything holding a UUID — belong to the new disk, not the
   old one. Restoring them over a fresh install produces a machine that will not
   boot, and the error arrives one reboot later.
4. Re-authenticate the agent harnesses. `.credentials.json`, `.claude.json` and
   `auth.json` are excluded on purpose; one login each.
5. Re-enroll TPM2 — [[tpm2-reenroll|tpm2-reenroll]]. The old enrollment was bound
   to a TPM that no longer exists.

**Treat a restore of `projects/` and `paste-cache` as containing secrets.** Agent
transcripts are a verbatim record of what was typed and pasted into a session, and
`paste-cache` is currently *in* both repositories — see ADR-0008's 2026-08-30
amendment, which recommends excluding it again and explains why it is not.

## 5. Integrity, when you suspect the repository rather than the machine

```bash
restic check                          # structure only, cheap
restic check --read-data-subset=5%    # downloads and verifies a sample
```

The second is the monthly habit; nothing schedules it
([[../spec/backups#What this chapter does not cover|spec/backups]]).

## Why this duplicates the spec's verification

[[../spec/backups#restic-restore-test|`restic-restore-test`]] contains the same
compare, and the duplication is deliberate under the rule in
[[areas/infrastructure/arch-thinkpad/runbooks/_index#Single-sourcing|runbooks/_index]]. Transclusion would carry the step's
`command:` onto a page read during an incident, and the adjacent commands in that
chapter write credential files and initialise repositories. A wrong neighbouring
command is expensive here.
