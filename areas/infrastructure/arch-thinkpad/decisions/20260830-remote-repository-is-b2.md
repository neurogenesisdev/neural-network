---
title: The remote restic repository is Backblaze B2, reached over the S3 API
type: decision
id: ADR-0028
aliases: [ADR-0028]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-30
commit: pending
impact: high
created: 2026-08-30
tags: [decision, backups, restic, b2, netcup, storage-box]
constrains: spec/backups
---

# ADR-0028 — The remote restic repository is Backblaze B2, reached over the S3 API

- **Status:** accepted
- **Date:** 2026-08-30
- **Provenance:** measured (the repository exists and holds a snapshot), measured
  (the provider comparison, from vendor documentation fetched 2026-08-30)

## Context

[[../backlog/remote-restic-repository|remote-restic-repository]] had been blocked
since July on a target: the Hetzner VPS was offline, and the item named three
options — B2, the Hetzner Storage Box, or resuming the VPS.

**Two of the three stopped existing.** The VPS is gone with its data. The Storage
Box the item calls "already owned" is also gone — established by the operator on
2026-08-30, not by this vault, which still described it as available. That is the
`inherited-unverified` failure the root `CLAUDE.md` warns about, arriving in a
backlog item rather than a spec: a resource recorded as owned, carried forward
unchecked for nine days, and load-bearing for the one decision it blocked.

netcup's storage products entered the comparison because the machine's other host
is a netcup root server and buying storage beside it looked like the cheap answer.

## Decision

**The remote repository is Backblaze B2, addressed through restic's S3 backend**
at `s3:s3.eu-central-003.backblazeb2.com/<bucket>`, with a bucket-scoped
application key and a repository password held only in KeePassXC.

Local block storage on the netcup server is **not** ruled out as a future second
remote. It is ruled out as *the* remote.

## The comparison

Sizes matter more than rates here: the repository is 12.809 GiB stored (measured,
first snapshot, below). Every per-terabyte argument is being applied to about one
eightieth of a terabyte.

| Target | Cost at this size | How restic reaches it | Independence |
|---|---|---|---|
| **Backblaze B2** | first 10 GB free, then $6.95/TB/mo — **≈$0.03/mo** | native S3 backend | separate company, separate account |
| Hetzner Storage Box BX11 | ~€3.20/mo flat, 1 TB, unlimited traffic | SFTP over SSH | separate company; 100 sub-accounts |
| netcup Storage Space 250 GB | €6/mo | **NFS only** | same provider; contractually tied to the server |
| netcup Local Block Storage | €0.0119/GB/mo → 250 GB ≈ €2.98 | nothing — it is a block device | same provider, **same machine** |

### Why netcup Storage Space is rejected

It is **NFS only** — measured from netcup's own help centre, 2026-08-30, which
also states that access control is by IP allowlist and that exceeding the monthly
traffic allowance freezes the storage until month end. Three consequences, any one
of which decides it:

1. **NFS over the public internet is unencrypted and authenticated by source
   address.** The client here is a laptop that roams between networks and has no
   static address to allowlist.
2. **The storage is contractually attached to the server.** Cancelling the vServer
   deletes the Storage Space. A backup that dies with the machine it is beside is
   not an independent copy.
3. It is the most expensive option in the table.

### Why local block storage is rejected as *the* remote, and on which grounds

The operator's prior reading — from a general web search — was that block storage
is *unreachable* and therefore unusable for restic. **That is not the objection,
and getting the reason right matters for the future second-remote question.** Block
storage attaches to the RS 1000 as an ordinary device; running `rest-server` or
plain SFTP on that host, reached over the Tailscale mesh, is a working restic
target and a cheap one.

The objection is **correlation**. The backup would live on the same machine, at the
same provider, as one of the systems it is meant to survive. One netcup account
problem, one compromised host, and both copies are in the blast radius —
[[20260830-no-luks-on-always-online-vps|ADR-0021]]'s reasoning about that machine's
disk applies to anything stored on it. The netcup server's own backup item
([[../../netcup-vps/backlog/restic-client|restic-client]]) makes the same argument
in the other direction: it is looking for an *off-provider* target too.

### Why not the Storage Box, if it came back

It wins on flat-rate capacity and would be the better answer above roughly 450 GB
of stored data. At 12.809 GiB it is paying €3.20/mo for a property — unlimited
traffic on a terabyte — that this repository has no use for. If the netcup server's
restic client lands and both hosts want one target, that arithmetic changes and
this record should be re-read rather than assumed.

## Consequences

- **The repository password is the whole game.** It exists in KeePassXC and in
  `/etc/restic/repo-password` on the machine being backed up. The operator
  confirmed on 2026-08-30 that both the B2 and T5 passwords are now in KeePassXC
  and synced off-device, which is what makes the remote copy openable after the
  event it insures against. Before that confirmation this was a backup that could
  not be restored, and it was in that state for the duration of the first upload.
- **The application key is scoped to the one bucket** (read and write). A laptop
  compromise reaches this bucket's contents and nothing else in the account.
  Append-only was considered and **not** adopted: it needs a second, differently
  scoped key for `prune`, and the threat it defends against — an attacker with root
  on the laptop deliberately destroying backups — is above this machine's threat
  model.
- **The S3 backend is used rather than restic's native `b2:` backend.** Both
  address the same service. This is `inferred`, not measured: the S3 path is the
  more heavily exercised one and handles B2's rate limiting better. Nothing here
  tested the alternative.
- **Cost is not a constraint and should not become an argument.** At $0.03/mo,
  excluding data to save storage is not a defensible reason for an exclusion —
  see [[20260818-restic-exclusions|ADR-0008]]'s 2026-08-30 amendment, which had to
  be written because this round nearly made that mistake.
- **A restore has not been performed.** See *Revisit when*.

## Revisit when

- **A restore is tested.** [[../runbooks/restic-restore|runbooks/restic-restore]]
  exists and has not been executed. Until it has, this record documents an upload,
  not a backup.
- **The netcup server's restic client is set up.** Two clients wanting one target
  is the condition under which the Storage Box arithmetic changes.
- **Stored data approaches ~450 GB**, where flat-rate capacity overtakes
  per-gigabyte billing.
