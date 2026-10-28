---
title: "Containers — rootless podman, the storage subvolume, the first pull"
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
created: 2026-08-24
updated: 2026-08-26
tags: [spec, containers, podman, btrfs, subvolume, snapshots, digest]
---

# Containers

Three steps, from a booted system to a digest-pinned base image whose layers sit
outside the snapshot perimeter on purpose. Read [[_schema|_schema.md]] first.

**This is the first chapter with no PRD section behind it, and the PRD's silence
is not an omission — it is a position.** §12 puts a container runtime under
"Deliberately out of scope": "When you add a desktop environment, container
runtime, or editor, treat each as its own decision with its own trust assessment
rather than a checklist inherited from a different machine." Every other chapter
so far has replaced PRD prose and diverged from it in places. This one has
nothing to replace, and its sources are three decision records and a build log:
[[../decisions/20260817-rootless-podman-over-docker|ADR-0001]] (rootless podman,
not Docker), [[../decisions/20260816-container-storage-and-bind-mounts|ADR-0006]]
(nested subvolume, bind mounts, never named volumes),
[[../decisions/20260818-base-image-pinned-by-digest|ADR-0007]] (the image is the
version pin — **superseded 2026-08-25 by
[[../decisions/20260825-devbox-pinning-four-layers|ADR-0016]]**, which keeps the digest
pin and adds three layers around it), and `_source/Block-C-Dev-Environment.md`, which
records what was actually built on 2026-08-17/18.

**The PRD does touch this chapter once, and it touches it destructively.** §6.1
prescribes `kernel.unprivileged_userns_clone=0` and then says so itself: "this
breaks rootless Podman… you will hit it the first time you install a container
runtime. Set it to `1` then, deliberately." No chapter has promoted §6.1 yet, and
`state/` records **no sysctls at all** — there is no collector section for them —
so nothing in this vault says which way this machine went. What `state/` does say
is that `podman.socket` is user-enabled (`state/services.md`, `## user enabled`)
and that podman, `podman-docker` and `podman-compose` are installed
(`state/packages.md`, `## explicit`). So rootless podman runs here, and the
sysctl is therefore `1`, absent, or unsupported on this kernel (`inferred` — the
lookup is one `sysctl kernel.unprivileged_userns_clone` on the machine, which
also answers whether the knob exists at all).

`podman-install`'s postcondition asserts the **capability** rather than the knob,
for that reason and for the vault's older one: a sysctl name is a constant that
may not exist, and `podman info`'s `rootless` boolean is a live reading that
cannot be wrong about what it reports.

## Where this chapter sits

```
first-boot ──→ containers-subvol ──→ podman-install ──→ podman-first-pull
 (spec/base)
```

A straight line, and the two edges in it are both doing non-obvious work.

**`containers-subvol requires first-boot`, and the worked example this chapter
promotes had that wrong.** It required `[btrfs-subvol-mount, user-create]`, both
of which sit chroot-side, so the
[[areas/infrastructure/arch-thinkpad/spec/_index#The DAG is not a reading order|derived-context rule]] classified the
step as chroot-side too. Run there, `$HOME` is `/root`, and the step creates a
subvolume at `/root/.local/share/containers` — **and its postcondition passes**,
because the check resolves the same `$HOME` the command did. Not a check that
fails the same way as the thing it detects; a check that *succeeds* the same way.
That is a new member of the family and the reason the precondition below asserts
`$HOME` is on `@home` rather than merely that the path is absent.

**`podman-install requires containers-subvol`**, which reads backwards — the
subvolume is for podman's storage, so podman looks like the prerequisite. It is
the other way round, and for the reason [[disk#btrfs-subvol-mount|`spec/disk`]]
named for `set-default`: the edge orders an execution against a window rather
than against a data dependency. Installing podman enables `podman.socket`, and
from that moment any Docker-API client — the Dev Containers extension is the one
this machine has — can start podman on demand. The first podman process to run
creates `graphRoot`, and if the subvolume is not there yet it creates a plain
directory on `@home` instead. ADR-0006 says "before pulling any images"; putting
the subvolume before the *install* satisfies that with margin and closes the
window entirely rather than narrowing it.

---

## containers-subvol

*Promoted from `_worked-examples.md` 2026-08-24, with its
`requires` list corrected, a `mkdir` added to the command, and its postcondition
extended.*

[[../decisions/20260816-container-storage-and-bind-mounts|ADR-0006]] is the whole
of this step: `~/.local/share/containers` is a **nested btrfs subvolume**, and
snapshots do not descend into nested subvolumes. Image layers are therefore
invisible to snapper in both directions — never pinned by `@home` retention,
never reverted by a rollback. The ADR's context names the failure that argument
comes from: 142 GB of deletions that freed no space because snapshots pinned the
extents.

If podman pulls before the subvolume exists, the images land on `@home` and
nothing errors. The cost surfaces later and in both directions: container storage
is meant to be invisible to snapper, and images sitting on `@home` are neither
excluded from snapshots nor recoverable through them.

**The nesting is asserted, not just the subvolume-ness.** `btrfs subvolume show`
prints `Subvolume ID:` and `Parent ID:` among its fields (*measured*,
`btrfs-subvolume(8)`, read 2026-08-24 — the page gives a worked example output
block naming both). `state/storage.md`'s `## subvolumes` grounds it from the
other side: `ID 537 top level 257 path @home/luiswg/.local/share/containers`
against `ID 257 top level 5 path @home`, where `list`'s documented format is
"ID *ID* gen *generation* top level *parent_ID* path *path*" — so `top level` in
`list` and `Parent ID` in `show` are the same number, and it is 257. Asserting
that the parent is whatever `/home` currently reports is the live-state form of
that: a rollback changes the ids and cannot break the check.

**`findmnt -T`, and this is the chapter where `-T` is right.**
[[disk#btrfs-subvol-mount|`btrfs-subvol-mount`]] insists on `-M` because
`--target` "checks the *path* elements in reverse order to get the mountpoint",
so a missing mount is answered for by its parent and the check passes. Here that
resolution is exactly what is wanted: `$HOME` is not a mountpoint and never will
be, and the question is which filesystem contains it. Same pair of flags, opposite
choice, decided by whether the path is supposed to *be* a mountpoint — which is
the same shape as `swapon --show` being the right gate in `spec/disk` and the
wrong one one chapter over.

That one reading does two jobs. It is the ADR-0006 nesting requirement, and it is
what catches the step running in the wrong namespace: under `arch-chroot` as
root, `$HOME` is `/root`, which resolves to `@` and not `@home`.

**The exit status is parsed around, and the reason recorded in the worked example
was slightly too strong.** That note said `btrfs-subvolume(8)` "does not specify
its exit status when the path is not a subvolume". The page *does* have an EXIT
STATUS section — "**btrfs subvolume** returns a zero exit status if it succeeds.
A non-zero value is returned in case of failure" (*measured*, read 2026-08-24).
What it does not do is say which conditions count as failure for `show`
specifically, so "not a subvolume" could reasonably be either. The output parse
stands; the justification is narrower than it was written.

```yaml
id: containers-subvol
requires: [first-boot]
precondition: |
  set -euo pipefail
  d="$HOME/.local/share/containers"
  # A subvolume cannot be created over an existing directory, and podman
  # creates this path on first use. Order matters here, not just intent.
  test ! -e "$d"
  # -T, not -M: $HOME is not a mountpoint, and --target resolving to the
  # filesystem that contains it is the question being asked. Two jobs in one
  # reading — nesting inside @home is the whole of ADR-0006, and it is also
  # what catches this step running under arch-chroot, where $HOME is /root.
  findmnt -no SOURCE -T "$HOME" | grep -q '\[/@home\]$' \
    || { echo "$HOME is not on @home — wrong user, or wrong namespace"; exit 1; }
command: |
  set -euo pipefail
  d="$HOME/.local/share/containers"
  # btrfs subvolume create does not create parent directories, and
  # ~/.local/share need not exist on a freshly created account.
  mkdir -p "$(dirname "$d")"
  btrfs subvolume create "$d"
postcondition: |
  set -euo pipefail
  d="$HOME/.local/share/containers"
  fail=0
  note() { echo "  $1"; fail=1; }
  # Output parsed rather than $? tested: btrfs-subvolume(8) has an EXIT STATUS
  # section but does not say which conditions count as failure for `show`.
  show=$(btrfs subvolume show "$d" 2>/dev/null) || show=
  id=$(sed -nE 's/^[[:space:]]*Subvolume ID:[[:space:]]+([0-9]+)$/\1/p' <<<"$show")
  [ -n "$id" ] || { echo "RESULT=FAIL ($d is not a btrfs subvolume)"; exit 1; }
  # Nested in @home, not merely a subvolume somewhere. The nesting is what makes
  # snapshots not descend into it, which is the entire point of the step.
  # Compared against whatever /home reports now, so a rollback moves both.
  parent=$(sed -nE 's/^[[:space:]]*Parent ID:[[:space:]]+([0-9]+)$/\1/p' <<<"$show")
  homeshow=$(btrfs subvolume show /home 2>/dev/null) || homeshow=
  home=$(sed -nE 's/^[[:space:]]*Subvolume ID:[[:space:]]+([0-9]+)$/\1/p' <<<"$homeshow")
  # An empty read means the label moved, which must not report as a mismatch.
  [ -n "$parent" ] || note "no 'Parent ID:' in btrfs subvolume show — label moved?"
  [ -n "$home" ]   || note "could not read /home's subvolume ID"
  if [ -n "$parent" ] && [ -n "$home" ] && [ "$parent" != "$home" ]; then
    note "$d has parent $parent, want /home's $home — it is not nested in @home"
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## podman-install

The id four chapters have referenced and none defined. `podman-first-pull` in
the now-deleted `_worked-examples.md` named it on 2026-08-23 and it has been
the last entry on [[areas/infrastructure/arch-thinkpad/spec/_index#Unresolved `requires`|the unresolved table]] ever
since.

**Three packages, and each is a decision rather than a dependency.** `podman` is
[[../decisions/20260817-rootless-podman-over-docker|ADR-0001]], which is not a
preference: Docker's daemon writes its own nftables rules on start and on
container creation, underneath a hand-written default-drop `input` chain, and the
ADR settled it by measurement — the `inet filter` ruleset hash was identical
before and after building images and running containers. `podman-docker` supplies
the `docker` command alias, and `podman.socket` the Docker-compatible API that
the VS Code Dev Containers extension requires. `podman-compose` is the
`docker-compose` equivalent. All three are in `state/packages.md`'s `## explicit`.

**Lingering is the part that is easy to skip and hard to diagnose.**
`loginctl(1)` on `enable-linger`: "If enabled for a specific user, a user manager
is spawned for the user at boot and kept around after logouts. This allows users
who are not logged in to run long-running services." (*measured*, read
2026-08-24.) Block-C records the consequence of omitting it, and the reason it is
worth a step rather than a footnote: "Without linger the socket dies at logout
and the extension fails to attach with an error pointing at Docker rather than
systemd." An error message pointing away from its cause is
[[areas/infrastructure/arch-thinkpad/state-notes|notes.md]]'s own test, and this one points at the wrong product.

**The `Linger` property name is `inferred`, and the check says so.**
`loginctl(1)` documents `enable-linger` and documents `show-user` as showing
"properties of one or more users", and `--property=` as limiting the display —
but **it does not name a `Linger` property anywhere** (*measured*, read
2026-08-24). Writing `--property=Linger` and testing for `yes` would therefore
produce an empty string on a wrong guess, which reads identically to lingering
being off: the `"tpm2-pcrs": null` failure `_schema.md` warns about, on a check
whose whole job is to catch a silent omission. The postcondition splits the empty
case out with its own diagnostic. The lookup that closes it is channel 1 — one
`loginctl show-user luiswg` on the machine.

**`systemctl is-enabled`'s exit status is documented and is still the wrong thing
to use.** The man page gives a table: "Returns an exit code of 0 if at least one
is enabled, non-zero otherwise", and the states returning 0 are `enabled`,
`enabled-runtime`, `alias`, `static`, `indirect`, `generated` and `transient`
(*measured*, `systemctl(1)`, read 2026-08-24). Five of those are not "this step
enabled it". So the word is parsed and the status ignored — the mirror image of
[[base#user-create|`user-create`]]'s `visudo -c`, where a documented exit status
*is* used because it means exactly one thing. Having a documented exit status is
not the same as having a useful one, which is also what `--graceful` taught
[[boot#bootloader-install|`bootloader-install`]].

**The rootless assertion is how PRD §6.1 gets caught without naming its knob.**
`podman info`'s JSON carries `store` and `host` sections, and "Host Information
includes… security settings (with a `rootless` boolean)"; the page also warns
that "The Go template struct fields start with upper case. When running podman
info or podman info --format=json, the same names start with lower case"
(*measured*, `podman-info(1)`, read 2026-08-24), which is why the `jq` path is
`.host.security.rootless` and not `.Host.Security.Rootless`. If §6.1's sysctl is
ever applied, rootless podman stops working and this reading is what says so —
without this spec ever asserting a sysctl name that may not exist on this kernel.

**Packages are asserted as installed, never as explicit**, for the reason
[[base#base-install|`base-install`]] establishes: `pacman -Qqe` reports the
install *reason*, which moves over a machine's life. Podman's own dependencies
make the point on this machine — `netavark`, `crun` and `passt` are all in use
per Block-C and none is in `## explicit`.

```yaml
id: podman-install
requires: [containers-subvol]
precondition: |
  set -euo pipefail
  # Rootless: this must not run as root, and the booted-system context is what
  # the containers-subvol edge establishes.
  [ "$(id -u)" -ne 0 ] \
    || { echo "podman-install runs as the unprivileged user, not root"; exit 1; }
  # The storage subvolume must already exist. Installing podman enables a
  # socket, and the first podman process any client triggers creates graphRoot
  # as a plain directory on @home if the subvolume is not there.
  btrfs subvolume show "$HOME/.local/share/containers" 2>/dev/null \
    | grep -q 'Subvolume ID:' \
    || { echo "container storage subvolume is absent — run containers-subvol first"; exit 1; }
command: |
  set -euo pipefail
  # podman-docker gives the `docker` alias, podman.socket the Docker-compatible
  # API the Dev Containers extension talks to (ADR-0001).
  sudo pacman -S --needed podman podman-docker podman-compose
  # Without this the user manager exits at logout and the socket goes with it;
  # the extension then reports a Docker error rather than a systemd one.
  loginctl enable-linger "$(id -un)"
  systemctl --user enable --now podman.socket
postcondition: |
  set -euo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  u=$(id -un)
  # Installed, not explicit: -Qqe reports the install reason, which moves.
  installed=$(pacman -Qq || true)
  [ -n "$installed" ] || { echo "RESULT=FAIL (no package database)"; exit 1; }
  for p in podman podman-docker podman-compose; do
    # Whole-line comparison: `grep -q podman` matches podman-docker.
    printf '%s\n' "$installed" | grep -qx "$p" || note "not installed: $p"
  done
  # The word, not the exit status. systemctl(1)'s is-enabled table returns 0 for
  # static, alias, indirect, generated and transient as well as for enabled —
  # five states that are not what this step established.
  en=$(systemctl --user is-enabled podman.socket 2>/dev/null) || en=
  [ "$en" = enabled ] || note "podman.socket is '${en:-<unreadable>}', want enabled"
  act=$(systemctl --user is-active podman.socket 2>/dev/null) || act=
  [ "$act" = active ] || note "podman.socket is '${act:-<unreadable>}', want active"
  # The Linger property name is not in loginctl(1) and is inferred. An empty
  # read must therefore not report as lingering being off.
  linger=$(loginctl show-user "$u" --property=Linger --value 2>/dev/null) || linger=
  case "$linger" in
    yes) ;;
    "")  note "loginctl returned no Linger property — the name may have moved, which is not the same as lingering being off" ;;
    *)   note "lingering is '$linger', want yes — podman.socket dies at logout" ;;
  esac
  # The capability, not the sysctl. PRD §6.1 prescribes
  # kernel.unprivileged_userns_clone=0 and its own text says that breaks
  # rootless podman; this is the observable, and it needs no knob name.
  rootless=$(podman info --format json 2>/dev/null | jq -r '.host.security.rootless') || rootless=
  [ "$rootless" = true ] \
    || note "podman reports rootless='${rootless:-<unreadable>}' — check PRD §6.1's userns sysctl"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## podman-first-pull

*Promoted from `_worked-examples.md` 2026-08-24, with a
postcondition that asserts the pull.*

**The example's postcondition did not check that anything had been pulled.** It
re-ran the precondition's `graphRoot` reading and reported on that. A
postcondition that repeats its own precondition verbatim asserts nothing about
the command in between — it establishes that the world is still as it was, which
is true whether or not the step did anything. `_schema.md`: a postcondition asks
whether the step achieved what it was for, and what this one is for is an image.

### The parameter, and a clause of the convention that does not apply

`$DEVBOX_IMAGE_DIGEST` is the base image reference, of the shape
`docker.io/library/archlinux@sha256:<64 hex>`. It is a parameter and not a literal for
the plainest possible reason:
[[../decisions/20260825-devbox-pinning-four-layers|ADR-0016]] keeps ADR-0007's rule that
the built image is the version pin, so the digest changes on every deliberate rebuild.
A literal here would be the per-build constant this vault forbids, and it would be stale
by design.

*The example distro changed 2026-08-26.* It read `debian:trixie-slim@sha256:…`, the form
Block-C records for the pin built on 2026-08-18;
[[../decisions/20260825-devbox-base-is-arch|ADR-0015]] moves the base to Arch. **The
shape check below is unchanged and was already correct**, because it anchors on
`@sha256:` and 64 hex digits and never on the registry or the name — which is the
property that let it survive a change of distro without being touched.

[[_schema#Conventions|`_schema.md`'s parameter convention]] has three clauses,
and **the third does not fit this parameter** — which is worth saying plainly
rather than pretending otherwise. Clause 3 asks that a parameter be validated
against something the DAG established, not against its type, and the worked case
is `luks-format`: `test -b` and `! isLuks` both pass for `sda2` on the rescue
drive, so no property of the value's *type* separates the right partition from a
catastrophic one. Nothing in this DAG establishes an image reference. It comes
from outside the machine entirely.

But the hazard is also a different one, and that is the part that matters.
`luks-format`'s hazard is **pointing at the wrong object**; ADR-0007's hazard is
**a reference that moves**: "A tag is a moving reference. A digest is
content-addressed and immutable." A shape check — `@sha256:` followed by 64 hex
digits — is not a weak substitute for provenance validation here. It *is* the
property ADR-0007 requires, because being pinned by digest is a fact about the
string's shape and nothing else. Clause 3 was written for destructive steps and
this step is not one; the general form belongs in `_schema.md` and is now there.

*One divergence between the sources, unresolved:* `podman-pull(1)`'s synopsis is
`podman pull [options] [transport]name[:tag|@digest]`, which presents tag and
digest as **alternatives**, and its worked example pulls
`alpine@sha256:d734…` with no tag (*measured*, read 2026-08-24). Block-C's
recorded pin carries both. Whether podman accepts `name:tag@digest` and what it
does with the tag if so is not answered by the page (`inferred`: it accepts it
and the digest wins, since that is the form the machine was built with). The
`grep` below anchors on the digest suffix and is correct under either reading;
the lookup that settles it is one `podman pull` on the machine.

### What the pull is checked against

`podman-image-inspect(1)` reports `"Digest"` as `sha256:<hex>` and
`"RepoDigests"` as a list of `registry/name@sha256:<hex>` strings (*measured*,
read 2026-08-24, from the page's example output). The check asks for the image
**by the reference that was pulled** — so podman itself resolves it — and then
requires the digest to appear in `RepoDigests`. Matching the digest suffix rather
than the whole reference is deliberate: `RepoDigests` entries carry no tag, so a
whole-string comparison against a `name:tag@digest` parameter would fail on a
correct pull.

*Known limit — **resolved 2026-08-25**.* For a multi-architecture image the digest a
pull resolves to and the digest of the manifest list are not the same value, and
`podman-image-inspect(1)` does not say which one `RepoDigests` reports. Measured 2026-08-25 on
`archlinux:base`: `.Digest` is the **amd64 instance** digest and `RepoDigests` carries
**both** it and the manifest-index digest. The check asks for **membership** in
`RepoDigests` rather than equality with `.Digest`, so it is correct whichever of the
two the parameter names.

The risk this paragraph recorded — a permanently-red check on a correct pull — does not
exist for this image. Worth being precise about why: it is a property of what
`RepoDigests` contains, not of the check being clever. Choosing the *weaker* assertion
while the answer was unknown is what made it survive, and that is the transferable part.
Re-measure if the base ever changes registry.

**The `graphRoot` reading is the one that closes the ordering hazard, and it is
strictly better than reading config files.** `podman info` reports the store
"after `/etc/containers/storage.conf`, `~/.config/containers/storage.conf` and
the environment have all been applied" — it is the resolved value, so one reading
covers every way the store can be pointed somewhere else. That is why it appears
in the precondition *and* the postcondition rather than only in the precondition:
per the parameter convention's second clause, a postcondition must stand alone,
and this is the reading a re-run checks.

```yaml
id: podman-first-pull
requires: [podman-install]
precondition: |
  set -euo pipefail
  : "${DEVBOX_IMAGE_DIGEST:?DEVBOX_IMAGE_DIGEST must name the base image by digest, e.g. docker.io/library/archlinux@sha256:<64 hex>}"
  # Pinned by digest, not by tag (ADR-0007). A shape check, and the right one:
  # the hazard here is a reference that moves, and immutability is a property of
  # the shape. See the prose on the parameter convention's third clause.
  grep -qE '@sha256:[0-9a-f]{64}$' <<<"$DEVBOX_IMAGE_DIGEST" \
    || { echo "DEVBOX_IMAGE_DIGEST is not pinned by digest: $DEVBOX_IMAGE_DIGEST"; exit 1; }
  d="$HOME/.local/share/containers"
  btrfs subvolume show "$d" 2>/dev/null | grep -q 'Subvolume ID:' \
    || { echo "$d is not a subvolume — pulling now would put images on @home"; exit 1; }
  # The resolved store, not the configured one: podman info applies both
  # storage.conf locations and the environment before reporting.
  graphroot=$(podman info --format json | jq -r '.store.graphRoot') || graphroot=
  case "$graphroot" in
    "$d"|"$d"/*) ;;
    *) echo "graphRoot is '$graphroot', outside $d"; exit 1 ;;
  esac
command: |
  set -euo pipefail
  : "${DEVBOX_IMAGE_DIGEST:?DEVBOX_IMAGE_DIGEST must name the base image by digest}"
  # podman-pull(1): "podman pull can also pull images using a digest
  # podman pull image@digest". The image is the version pin, not the
  # Containerfile (ADR-0007).
  podman pull "$DEVBOX_IMAGE_DIGEST"
postcondition: |
  set -euo pipefail
  : "${DEVBOX_IMAGE_DIGEST:?DEVBOX_IMAGE_DIGEST must name the base image by digest}"
  d="$HOME/.local/share/containers"
  want=${DEVBOX_IMAGE_DIGEST##*@}
  fail=0
  note() { echo "  $1"; fail=1; }
  # Asked for by the reference that was pulled, so podman resolves it. An
  # absent image and a wrong one get different diagnostics.
  repod=$(podman image inspect --format '{{json .RepoDigests}}' \
            "$DEVBOX_IMAGE_DIGEST" 2>/dev/null) || repod=
  if [ -z "$repod" ]; then
    note "no local image for $DEVBOX_IMAGE_DIGEST"
  else
    # Digest suffix, not the whole reference: RepoDigests entries carry no tag,
    # so a whole-string match would fail on a correct pull.
    grep -qF "$want" <<<"$repod" \
      || note "local image does not carry $want; RepoDigests=$repod"
  fi
  # Re-read rather than trusting the precondition: a postcondition must stand
  # alone, and this is the reading a re-run checks.
  graphroot=$(podman info --format json 2>/dev/null | jq -r '.store.graphRoot') || graphroot=
  case "$graphroot" in
    "$d"|"$d"/*) ;;
    ""|null) note "podman info reported no graphRoot" ;;
    *) note "graphRoot is '$graphroot', outside $d — image layers are on @home" ;;
  esac
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

---

## The open item this chapter was asked to close

`_worked-examples.md` carried one entry against `containers-subvol`: the
precondition tests that the path does not exist, which catches the ordering, but
not "the case where podman ran, was uninstalled, and left a populated `graphRoot`
on `@home` — the directory would be gone or not depending on cleanup, and the
subvolume would be created over nothing while the images stayed put. Worth a
check that `@home` holds no orphaned overlay store before this runs."

**Closable in part, and the closable part is not where the entry pointed.** It
splits into two scenarios that read as one:

**The scenario as written does not hold for the default configuration.** Rootless
podman's store *is* `~/.local/share/containers/storage` — the images live at the
path being tested. If the directory is gone, they went with it; there is no state
in which the directory is absent and the images "stayed put". For images to
survive the directory's removal they must have been somewhere else, which means a
`storage.conf` had pointed `graphroot` elsewhere.

**That version is real, and it is already asserted — one step later.** A store
configured somewhere else is exactly what `podman-first-pull`'s `graphRoot`
reading catches, and it catches it better than a directory scan could, because
`podman info` reports the value after every configuration source has been applied.
There is no need to know which file did it. The entry asked for a check on
`@home` before `containers-subvol`; the check that answers the question is on
`podman`, after `podman-install`, and it was in the example already.

**What genuinely remains is not detectable and is not this step's business.** A
*previous* podman installation with a different `graphroot` can leave a store
elsewhere under `@home`. That costs snapshot-pinned space — the 142 GB failure
ADR-0006's context cites — and it is real. It is also:

- **undetectable without an undocumented signature.** There is no documented
  marker for "this directory is an abandoned containers store". A check would
  have to `find` `@home` for directory names like `overlay/`, `overlay-images/`
  or `db.sql`, which is asserting against constants that upstream is free to
  change, in a check that would go stale silently — and a stale check that
  reports clean is worse than no check.
- **not something this step establishes.** `containers-subvol` creates a
  subvolume. Whether an unrelated tree elsewhere on `@home` is wasting space is a
  cleanup question, and `podman system reset` and `podman unshare` are its tools,
  not a postcondition.

So it is recorded as an **accepted limitation with a named consequence** —
`sessions/_index.md`'s own category — rather than left as an open check nobody
can write. The consequence, stated so it is not rediscovered: *building this
machine on top of a previous podman installation that used a non-default
`graphroot` leaves that store on `@home`, inside the snapshot and restic
perimeter, and nothing in this spec will mention it.*

**The entry also missed something bigger than it found.** Testing the reframing
is what surfaced that `containers-subvol` was on the wrong side of the
execution-context boundary — a defect with a silent failure mode and a
postcondition that passed on it, against which "podman might have run before" is
a narrow concern. That is the second time in three rounds that examining a
recorded open item has found a different and larger defect next to it, after
`spec/base` found `bootloader-install` while writing `first-boot`'s `requires`.

---

## Open against this chapter

- **`state/` records no sysctls, so PRD §6.1's `unprivileged_userns_clone` is
  unresolvable from the vault.** The chapter routes around it by asserting the
  capability instead of the knob, which is the better check anyway — but the
  question "was §6.1 applied on this machine, and does the knob exist on this
  kernel" has no answer here, and §6.1 has other clauses that will matter when
  someone promotes it. One `sysctl -a` section in the collector closes it.
- **`state/` records no lingering, and the `Linger` property name is
  `inferred`.** `loginctl(1)` documents `enable-linger` and `--property=` but
  names no property. The postcondition reports an empty read as an empty read
  rather than as lingering being off, which is the disambiguation `_schema.md`
  asks for and not a substitute for looking it up. One `loginctl show-user
  luiswg` settles the name; a collector section would ground it permanently.
- **The multi-architecture digest question is `inferred` and could make
  `podman-first-pull` permanently red.** If a pull by manifest-list digest yields
  `RepoDigests` entries carrying the platform digest instead, the check fails on
  a correct pull. `podman image inspect` on the machine after a pull answers it
  in one command.
- **Whether `podman pull` accepts `name:tag@digest` is unanswered by
  `podman-pull(1)`**, whose synopsis presents the two as alternatives while
  Block-C's recorded pin uses both. The precondition's shape check is correct
  under either reading, so nothing here depends on it — but the parameter's
  documented example does.
- **`/etc/containers/nodocker` is not written and not asserted.** Block-C notes
  that `sudo touch /etc/containers/nodocker` silences the `podman-docker` banner
  on stderr, and that "some tooling parses stderr". It is cosmetic, and asserting
  a decoration is what `btrfs-mkfs` declines to do with its `-L` label. Recorded
  because the stderr point is not purely cosmetic and may earn a step later.
- **The bind-mount half of ADR-0006 has no step here.** The ADR has two rules —
  container storage is a nested subvolume, *and* project data is bind-mounted
  from `@home` rather than held in named volumes — and this chapter implements
  only the first. The second belongs to a `spec/devenv` chapter that does not
  exist, alongside the `devbox` wrapper, `CODEX_HOME`, `CLAUDE_CONFIG_DIR` and
  the devcontainer template. ADR-0007's frontmatter already says
  `constrains: spec/devenv`, naming a chapter nothing else references.
- **No mutation sets, and no execution.** These three postconditions were written
  against man pages, `state/` and Block-C, and none has been run — `podman`,
  `btrfs`, `loginctl` and `systemctl` are all absent from the container this was
  written in, which is the standing limitation from the second session log. The
  three re-tested steps in `spec/disk` and the seven in `spec/boot` and
  `spec/base` now have written-down sets; this chapter has none, and
  `podman-first-pull`'s `RepoDigests` parse is the one most likely to be wrong.

  *Amended 2026-08-26 — half closed, and the half that closed is not the half this
  bullet worried about.* The **toolchain** limitation is gone:
  [[../decisions/20260825-devbox-base-is-arch|ADR-0015]] moves the container to an Arch
  base, and `awk` and `grep` there are now gawk 5.4.1 and GNU grep 3.12, byte-identical
  to the machine (*measured 2026-08-25*). What remains is the **tool** half — `podman`,
  `btrfs`, `loginctl` and `systemctl` are still absent from the container, so these
  postconditions can be reviewed here and executed only there. That is now an accepted
  limitation with a named consequence rather than an open problem, because installing
  Arch system binaries into a container with no LUKS volume and no TPM to run them
  against would be worse than useless.

  The sentence about `RepoDigests` being "most likely to be wrong" was a good guess and
  is now answered: it was right (`podman image inspect` on a multi-arch image reports
  two digests) and the check survived anyway, because it asserted membership rather than
  equality. See the resolved *Known limit* under
  [[#podman-first-pull|`podman-first-pull`]].
