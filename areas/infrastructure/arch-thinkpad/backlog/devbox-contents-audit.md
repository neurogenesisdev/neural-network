---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [devenv, vault]
tags: [devbox, containers, documentation, lookup, tooling]
created: 2026-08-23T00:00:00Z
updated: 2026-08-23T00:00:00Z
commit: pending
---

# Audit what the devbox contains against what it is now used for

> [!NOTE] Summary
> The container's contents were decided in one session for one purpose — running coding agents against a Node/Python project. The purpose has drifted: it is now also where the infra vault is written, where documentation lookups happen, and where spec postconditions get reasoned about. Nothing has been re-examined against the new purpose. [[devbox-wiki-mount]] is one instance; this is the general question.

## What is actually installed

Measured from inside the container, 2026-08-23. The Containerfile itself is not
reachable from here — it lives on the host — so this is `apt-mark showmanual`,
which is the closest available proxy for what the build explicitly asked for.

**Base:** Debian 13 (trixie). **13 manual packages, 555 installed in total:**

```
build-essential  ca-certificates  curl  git  jq  less  nodejs  npm
openssh-client   python3  python3-pip  python3-venv  ripgrep
```

**To enumerate the real intent,** read the Containerfile on the host and compare
against that list — anything manual but absent from the Containerfile arrived by
some other route, which is the same class of finding as the Copilot CLI agent in
[[devbox-cleanup-next-rebuild]].

## Findings against current use

**`jq` is present — candidate closed.** It was flagged as possibly missing
because spec postconditions depend on it (`uki-sign`, `tpm2-enroll`). It is in
the manual set. No action.

**`awk` is mawk, and the target runs gawk.** `mawk 1.3.4 20250131` here;
`state/notes.md` and the 2026-08-23 session record the host as gawk. This
divergence has already caused one silent bug: a `\<` word-boundary escape in a
collector filter is a gawk extension, so it matched nothing under mawk and the
verification **passed while testing nothing at all**. That is the worst failure
shape available — a check that fails the same way as a pass.

This is the highest-value item here, and the fix is not obvious. Installing
`gawk` makes the container match the target and stops the class of bug. But it
also removes the accidental cross-implementation testing that *caught* this one:
the bug was found precisely because the container was stricter than the target.
Worth deciding deliberately rather than defaulting. If gawk is installed, the
collector needs a POSIX-only lint from somewhere else, or the next extension
goes in unnoticed.

**Man pages: the premise needs correcting before acting.** The case for
installing `man-db` + `man-pages` was that a local man page reflects the
*installed version*, where `man.archlinux.org` is upstream-current. That
reasoning does not hold for this container, because **the container is Debian
and the target is Arch.** `man cryptsetup` here would report Debian trixie's
cryptsetup, which is neither the version on the ThinkPad nor necessarily
current — strictly worse than the man.archlinux.org fetch it would replace. And
the Arch-specific tools that prompted the question, `mkinitcpio` and `sbctl`,
are not in Debian's archive at all.

Two further obstacles, both measured:

- **`man` is not installed** — no `man`, `mandb`, `apropos` or `whatis`.
- **Man pages are excluded at unpack time.** `/etc/dpkg/dpkg.cfg.d/docker`
  carries `path-exclude /usr/share/man/*`. `/usr/share/man/man{1,5,7,8}` exist
  and are **empty**. So installing `man-db` populates nothing; the exclusion has
  to be overridden *and* every package already installed reinstalled.

**The version-accurate fix is a mount, not a package** — the same shape as
[[devbox-wiki-mount]]. Arch does not strip man pages, so the host's
`/usr/share/man` holds exactly the installed-version pages the lookup rule wants:

```
--volume /usr/share/man:/usr/share/man:ro
```

plus `man-db` in the image for a reader. That makes `CLAUDE.md`'s channel 2
local and *more* accurate than the web fetch, rather than less. Confirm
`man-db`'s index behaves against a read-only tree — `mandb` wants to write a
cache, and `man` may fall back to slower uncached lookup or complain.

**`ripgrep` is present, `grep` is GNU.** Consistent with the host, where
`alias grep='rg'` is live and interactive-only. No action; noted because
`state/notes.md` records that alias as a source of silent empty results, and
scripts in this repo already use `command grep` for it.

## What this is really asking

Not "add packages" — the question is whether the container is still the right
shape for what it is used for. Three uses now share it, and only the first was
designed for:

1. Running coding agents against Node/Python projects — the original purpose.
2. Writing and reasoning about `arch-thinkpad/` — needs documentation access,
   which is what [[devbox-wiki-mount]] and the man page item above are about.
3. Reasoning about spec postconditions for a machine whose tooling the container
   does not have. `btrfs`, `cryptsetup`, `sbctl`, `systemctl`, `podman`,
   `mkinitcpio`, `ukify` and `bootctl` are all absent (measured) — so a
   postcondition can be written here but never dry-run.

Use 3 may be fine as-is: those tools need a real machine with a real LUKS
volume, and installing Arch binaries into a Debian container to run them against
nothing would be worse than useless. But it should be an accepted limitation
with a named consequence — postconditions are reviewed here and only ever
executed there — rather than something nobody has said out loud.

## Related

- [[devbox-wiki-mount]] — the instance that prompted this. Same mount pattern as
  the man page proposal, and both want the same rebuild.
- [[devbox-cleanup-next-rebuild]] — where the concrete changes land.
- [[devcontainer-ipc-bridge-hardening]], [[20260818-devcontainer-agent-exposure|ADR-0003]]
  — the container's *security* boundary has been examined; its contents have not.
- [[20260818-base-image-pinned-by-digest|ADR-0007]] — record versions in the
  commit message on any rebuild.
