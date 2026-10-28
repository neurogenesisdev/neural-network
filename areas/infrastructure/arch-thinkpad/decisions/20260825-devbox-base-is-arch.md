---
title: The devbox base image is Arch, matching the host
type: decision
id: ADR-0015
aliases: [ADR-0015]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
updated: 2026-08-26
tags: [decision, containers, devbox, toolchain, parity, pacman]
constrains: spec/containers, spec/devenv
---

<!-- provenance was `inherited-unverified` in draft, on three findings taken from
     a host reading of files this vault does not hold. It is `measured` now
     because the record no longer rests on any of them: the toolchain parity
     table, the bind-mount enumeration and the UID requirement were all re-taken
     from inside the running container (/proc/self/mountinfo, id -u, the
     environment), and the two claims that could not be re-taken — the wrapper's
     short-name reference and the `useradd` literal — are no longer load-bearing
     anywhere in this record or in ADR-0016. The recipe repo is still unmirrored;
     that is a gap in the vault, not a weakness in this claim. -->

# ADR-0015 — The devbox base image is Arch, matching the host

- **Status:** accepted
- **Date:** 2026-08-25
- **Provenance:** measured

## Context

The devbox base has been Debian trixie since 2026-08-17. That was deliberate:
`_source/Block-C-Dev-Environment.md` §2 records "The host stays Arch; the box does
not have to match." The reasoning was that a rolling base defeats a reproducible
artifact, and matching the host bought nothing.

It bought something, and the price was paid before anyone noticed. The container's
`awk` was **mawk 1.3.4** where the machine runs **gawk**. A `\<` word-boundary
escape — a gawk extension — matched nothing under mawk, so a collector verification
**passed while exercising nothing at all**.
[[../backlog/devbox-contents-audit|backlog/devbox-contents-audit]] calls that "the
worst failure shape available — a check that fails the same way as a pass." A second
instance was found on 2026-08-24: `grep` in the container was **ugrep 7.5.0**, not
GNU grep, and nothing had been resting on a GNU extension only by luck.

The container's purpose has also drifted. It was built to run coding agents against
Node/Python projects; it is now also where this vault is written and where spec
postconditions are reasoned about — postconditions for a machine whose toolchain the
container did not share.

## Decision

Base the devbox image on **Arch**, pinned by digest:

    FROM docker.io/library/archlinux@sha256:c9dc8b5d…

Mount the host's pacman database read-only at `/host/pacman`, and answer every host
package question through `--dbpath /host/pacman` —
[[../spec/_schema#Conventions|the host-question rule]] in `_schema.md`.

**The container user's UID must equal the UID owning the host-side state.** That
resolves to 1000 today, so the `Containerfile` creates a user with `--uid 1000` — but
the postcondition asserts the *requirement*, not the number, so a renumbered host user
is caught rather than silently breaking the mount. The user's **name and home stay as
they are** (`dev`, `/home/dev`); renaming them to match the host is parity of a
different kind and is deferred with the rest of it
([[20260825-harness-state-and-identity|ADR-0017]]).

**Reverses Block-C §2's "the box does not have to match."** That rationale is in
`_source/` and no decision record superseded it; this one does, by name.

## Rationale

Parity is **measured**, not assumed (2026-08-25):

| | Host | `archlinux:base` |
|---|---|---|
| awk | gawk 5.4.1, `/usr/bin/gawk` | gawk 5.4.1, `/usr/bin/gawk` |
| grep | GNU grep 3.12-modified | GNU grep 3.12-modified |
| pacman | 7.1.0 / libalpm 16.0.1 | 7.1.0 |

The base switch **closes** the mawk class rather than mitigating it. No lint, no
discipline, no remembering: the divergence cannot recur because there is no second
implementation to diverge to.

**The UID requirement is what makes the swap non-destructive, and it is a requirement
rather than a literal copied from the old recipe.** Measured from inside the running
container 2026-08-25: `id -u` is 1000, and all three bind mounts — `/work`,
`$STATE/claude`, `$STATE/codex-state`, whose host sources `/proc/self/mountinfo` gives
as `/@home/luiswg/…` — are owned by uid 1000 and writable. So the rule the replacement
must satisfy is *the container user's UID equals the UID that owns the host-side
state*, and 1000 is what that resolves to today. Stated that way it survives a host
user being renumbered, which a literal copied out of the old Containerfile would not.

The postcondition asserts the requirement, not the number: it reads the owner of the
mounted state and compares it to `id -u`, so a mismatch is caught whichever side
moved. (Host-side ownership is `inferred` from the mount source path naming `luiswg`;
the lookup is one `stat -c %U ~/.local/state/devbox/claude` on the host.)

## Consequences

### Accepted negative: container and host are now hard to tell apart

This is the real cost and it is worth stating without softening. Before, a `pacman`
command typed in the wrong place failed with `command not found`, and a Debian prompt
was a standing reminder of which side you were on. **The base's difference was itself
a guard rail, and this decision removes it.** From now on a host command run in the
container succeeds, against the wrong package database, and reports a plausible
answer.

Three things narrow it, none of which closes it:

- [[../spec/_schema#Conventions|The host-question rule]] routes every host package
  question through `--dbpath /host/pacman`, so the *correct* invocation is visibly
  different from the incorrect one rather than identical to it.
- The mount is named `/host/pacman`, not `/var/lib/pacman`. The path is the reminder.
- The image emits its identity as OCI labels
  ([[20260825-devbox-pinning-four-layers|ADR-0016]]), so "which side am I on" has an
  answer that does not depend on noticing.

Residual, and named so it is not rediscovered: **an agent or human who runs `pacman -Q`
inside the container gets the container's 892-package answer with no error and no
warning.** Nothing in this design prevents that. It is the price of parity, and parity
was bought to close a failure mode that had already fired.

**This decision removes one tell of four, and the other three survive.** The container
announced itself in four ways: a different distro in `/etc/os-release`, a different
user (`dev`), a different home (`/home/dev`), and a project mounted at `/work` rather
than at its host path. **This record removes the first only.** `whoami` still returns
`dev`, `$HOME` is still `/home/dev`, and `pwd` in the project is still `/work`.

[[20260825-harness-state-and-identity|ADR-0017]] changes the project *identity* and
adds markers, and **defers** path, user and home parity to a separate conditional
decision. So the guard-rail loss is **partial here and full there**, and the full
version is one of the stated reasons parity is deferred: it deepens this cost, and the
mitigation for it does not exist yet.

What replaces the tell this record does remove, and the split matters:

- **For a check, a live reading.** `/run/.containerenv` exists inside and cannot exist
  outside (*measured 2026-08-25*), and `systemd-detect-virt -c` answers the same
  question on an Arch base. Neither is a marker this system sets, so neither can drift
  out of sync with reality — the same reason [[../spec/base#first-boot|`spec/base`]]
  uses `systemd-detect-virt -r` for the chroot boundary rather than a sentinel file.
- **For a human, three markers**, each asserted by a postcondition in
  [[../spec/devenv#devbox-identity-markers|`devbox-identity-markers`]]:
  `--hostname devbox…`, `DEVBOX=1`, and a shell prompt that renders the hostname.
- **The prompt is an open item, not a solved one.** `archlinux:base` ships no shell
  configuration at all, so today there would be no prompt to put a marker in. Until
  that is written, the human-facing half of this mitigation does not exist and the
  machine-readable half is carrying it alone.

### Other consequences

- **`/var/lib/pacman:ro` is a fifth enumerated host path**, and the rule it excepts is
  stated here for the first time because it was never written down:

  > **Host paths are enumerated and justified individually.** There is no blanket
  > "none except `/work`" — there never was. Adding one is a decision, not a
  > configuration change.

  **Measured 2026-08-25 from `/proc/self/mountinfo`**, on the wrapper entry point:
  three bind mounts, `/work` ← `/@home/luiswg/dev/infra-docs`, `/home/dev/.claude` ←
  `/@home/luiswg/.local/state/devbox/claude`, `/home/dev/.codex-state` ←
  `/@home/luiswg/.local/state/devbox/codex-state`. `$SSH_AUTH_SOCK` is the fourth and
  is opt-in (`--with-agent`); it was absent from that session, which is how the entry
  point was identified. `/var/lib/pacman` read-only is the fifth.

  **There are two enumerations, not one, and only one of them is measurable from
  inside.** The wrapper's set is above. The attached devcontainer's is a different
  list in a different file, and Block-C records that it additionally receives
  `$SSH_AUTH_SOCK` **unconditionally**. So "the mount set" is not a property of the
  image; it is a property of the entry point, and the rule has to be applied twice.

  Two further mounts are already proposed and unresolved —
  `/usr/share/doc/arch-wiki/text` plus `/usr/bin/wiki-search`
  ([[../backlog/devbox-wiki-mount|backlog/devbox-wiki-mount]]) and `/usr/share/man`
  ([[../backlog/devbox-contents-audit|backlog/devbox-contents-audit]]). Both now have
  a rule to be measured against.

- **The accidental cross-implementation testing is gone, and that was an argument
  against this change.** `backlog/devbox-contents-audit` asked for the gawk question
  to be decided deliberately rather than defaulted, precisely because the container
  being *stricter* than the target is what caught the `\<` bug. This decision answers
  it: parity wins, because a divergence that catches one bug by accident and hides the
  next by the same mechanism is not a testing strategy. **The audit's condition comes
  due with it** — the collector now needs a POSIX-only lint from somewhere else, or
  the next gawk extension goes in unnoticed. Recorded as an open item, not closed
  here.

- **`archlinux:base` ships no `jq`, `git` or `ripgrep`**, all of which the current
  image has and spec postconditions call. The build installs them explicitly.
- **`archlinux:base` ships no sync databases**, so the build must `-Sy` before
  installing anything — against the pinned snapshot (ADR-0016), never against a live
  mirror.
- **`archlinux:base` ships nftables 1.1.6**, matching the host. Relevant to the
  deferred egress work; nothing in this record depends on it.
- **Man pages become useful.** `backlog/devbox-contents-audit` correctly rejected
  installing `man-db` on a Debian base, because `man cryptsetup` there would report
  Debian's version. On Arch that objection dissolves, and the `/usr/share/man` mount
  it proposed becomes the version-accurate channel 2 the root `CLAUDE.md` wants. Not
  adopted here; the obstacle it names has moved and the item should be re-read.

### What this record does not touch

**[[20260817-rootless-podman-over-docker|ADR-0001]] is unaffected by this decision and
is affected by the next one.** Its measured claim — rootless podman writes *no*
nftables rules, `inet filter` hash `4bc77135…` — remains true of everything here. The
egress work will write nftables rules, in a container netns rather than the host
table, and the ADR's headline sentence does not carry that distinction. **That is the
egress ADR's problem to solve, and it is not solved here.** Flagged so the next round
does not inherit it silently.

## Revisit when

The container gains a fourth use, or an Arch-specific breakage in the image costs more
than the mawk class did. A rolling base *is* less reproducible than a stable one —
ADR-0016 is what makes that survivable, and if its snapshot pin ever proves unworkable
this decision has to be re-argued rather than patched.
