---
title: The devbox is pinned at four layers, and the image declares its own identity
type: decision
id: ADR-0016
aliases: [ADR-0016]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
updated: 2026-08-26
tags: [decision, containers, devbox, reproducibility, pinning, supply-chain]
supersedes: ADR-0007
constrains: spec/devenv
---

# ADR-0016 — The devbox is pinned at four layers, and the image declares its own identity

- **Status:** accepted
- **Date:** 2026-08-25
- **Provenance:** measured
- **Supersedes:** [[20260818-base-image-pinned-by-digest|ADR-0007]]

## Context

ADR-0007 pinned the base image by digest and stopped there, because "the agent
installers cannot be pinned this way without vendoring their installation, which was
judged not worth the maintenance." The built image became the version pin, with the
agent versions recorded in the commit message.

That judgement was correct on the evidence available in August 2026. **The evidence
changed.** Measured 2026-08-25:

- Claude Code's `install.sh` takes a **positional version argument**, validated
  against `stable|latest|N.N.N`.
- Codex's installer takes **`CODEX_RELEASE`**, normalised through `normalize_version`.
- The Arch Linux Archive serves a dated repo snapshot that `pacman -Syy` and
  `pacman -S` both accept, with signature and integrity checks passing against the
  **stock** `archlinux-keyring`.
- `podman run localhost/devbox@sha256:…` works, so a locally built image can be
  referenced by digest rather than by a moving tag.

None of that required vendoring. **ADR-0007 was not wrong; it is out of date.**

ADR-0007 also recorded a failure of its own model: Copilot Chat installed a CLI agent
into the container unasked — "an agent with shell access that is not in the
Containerfile, will not survive a rebuild, and is not in git." Its *Revisit when*
named that as the trigger, because the image-is-the-pin position "assumes the image
contents are a function of the recipe."

## Decision

Pin at **four layers**, and make the image state its own identity.

1. **Base image — by digest.** `FROM docker.io/library/archlinux@sha256:c9dc8b5d…`
2. **Repo snapshot — by date.** One line rewriting `/etc/pacman.d/mirrorlist` to
   `https://archive.archlinux.org/repos/<YYYY>/<MM>/<DD>/$repo/os/$arch` before any
   `-Sy`. Nothing is installed from a live mirror.
3. **Agent installers — by version literal.** Claude Code positionally, Codex through
   `CODEX_RELEASE`.
4. **Built-image consumption — by digest.** `devcontainer.json` and the `devbox`
   wrapper reference `localhost/devbox@sha256:…`, never a tag.

The image carries its build facts as **OCI labels**:

    io.github.luis-gordon.devbox.base-digest
    io.github.luis-gordon.devbox.snapshot-date
    io.github.luis-gordon.devbox.claude-version
    io.github.luis-gordon.devbox.codex-version
    io.github.luis-gordon.devbox.awk
    io.github.luis-gordon.devbox.grep

**Labels record what was built. The session log records what actually ran** —
`podman image inspect --format '{{.Digest}}'` of the image the session used. These are
deliberately two records: they diverge the moment there are two images on disk, and
collapsing them would make the label answer a question it cannot.

## Rationale

**Layer 1 is unchanged from ADR-0007 and its open question is now closed.**
[[../spec/containers#podman-first-pull|`spec/containers`]] recorded a risk that a
multi-architecture pull would make `podman-first-pull` permanently red, because the
manifest-list digest and the platform digest differ and the page does not say which
`RepoDigests` reports. Measured: `.Digest` is the **amd64 instance** (`c9dc8b5d…`) and
`RepoDigests` holds **both** it and the index digest (`b860afd5…`). The postcondition
asserts **membership** in `RepoDigests`, not equality, so it is correct under either
pin — and it is correct by construction rather than by luck.

**Layer 3 is the one ADR-0007 declined, and the two installers need different
treatment.** Claude's argument is positionally validated against a pattern, so a
malformed version fails at parse. Codex's `CODEX_RELEASE` is normalised, so a typo is
**not rejected at parse time** and the build proceeds with something other than what
was asked for. The build therefore carries a postcondition asserting the *installed*
version equals the pinned literal — the observable, not the input. That is the same
reasoning [[../spec/containers#podman-install|`podman-install`]] uses for asserting the
rootless capability instead of the sysctl.

**Layer 4 rests on the same argument as layer 1.** ADR-0007's own sentence — "a tag is
a moving reference, a digest is content-addressed and immutable" — applies to the image
this repo *builds* exactly as it applies to the image it *consumes*, and ADR-0007
applied it to only one end. Measured 2026-08-25:
`podman run localhost/devbox@sha256:…` returns `RC=0`, so the digest form is available
at the consuming end and the pin costs nothing.

*Reported separately, and not load-bearing here:* the `devbox` wrapper's last line is
`devbox:latest`, an unqualified short name, which additionally exposes podman's
short-name resolution. That comes from a host reading on 2026-08-25 of a file this
vault holds no copy of, so it is `inherited-unverified` and this record does not depend
on it — layer 4 stands on the moving-tag argument alone. It is noted because the same
edit fixes it.

**The trust anchor stays out of the pin.** The stock `archlinux-keyring` validated an
August-24 snapshot without complaint, so pinning the keyring would add a moving part
for no measured benefit.

## Consequences

### Accepted negative: a pinned snapshot receives no security updates

A frozen repo snapshot is frozen in both directions. Between rebuilds the image's
userland does not receive Arch security updates, and nothing about that state is
visible from inside the container.

**The mitigation is a rebuild cadence, and the cadence is measured rather than
remembered.** A stated cadence nobody enforces decays to a policy quietly not
followed, which is worse than a looser policy honestly kept. So:

- **Monthly is a ceiling, not a schedule.** The snapshot is bumped opportunistically
  whenever an agent update forces a rebuild anyway — one rebuild when the timing lines
  up — and unconditionally when the ceiling is reached.
- **Staleness is a check, not a habit.** `snapshot-date` is a label, so a postcondition
  can read it with `podman image inspect` and go red past the window without starting a
  container. The step record is `devbox-snapshot-freshness` and belongs to
  `spec/devenv`; the assertion is below.

**The exposure here is small, and saying so is part of the decision.** Short-lived
sandbox, `--cap-drop=ALL`, no host home, no `~/.ssh`, no KDBX, egress allowlisted
(pending), and no credentials beyond the agent tokens that already have their own bind
mounts. A month-old userland in that position is not the risk worth losing sleep over.
This paragraph exists so that nobody reads the paragraph above it and over-corrects
into a weekly rebuild treadmill.

### Keyring/snapshot skew is a loud failure, and its fix is written down now

Measured working today. If a future snapshot is old enough that the stock keyring
rejects it, `pacman` fails with a signature error and the **build stops** — it does not
silently install unverified packages. The fix, stated so it is not re-derived under
pressure: **pin `archlinux-keyring` from the same snapshot** and install it first.
`SigLevel = Never` is not an available answer and should be treated as a defect if it
ever appears in the recipe.

### Other consequences

- Rebuilding remains a deliberate act, and the commit message remains the audit trail
  — but it is now a **convenience**, not the mechanism. ADR-0007's named failure ("if
  the message is skipped, the record is lost — there is no automated capture") is
  closed: the labels are the automated capture.
- Four pins mean four things to bump. A rebuild that moves only the base digest and
  leaves the snapshot date behind is a legal state and a misleading one; the freshness
  check reads `snapshot-date` for exactly that reason.
- **ADR-0007's uninvited-software problem is narrowed, not solved.** Labels describe
  the recipe's output; they cannot describe what an extension writes into a running
  container afterwards. The Copilot CLI agent would still not appear in any label.
  [[../spec/devenv#devbox-ephemeral-writes|`devbox-ephemeral-writes`]] closes the other
  half: `podman diff` lists exactly the writes to the container layer.
- `~/.config/containers/auth.json` still holds registry credentials in cleartext
  ([[20260816-container-storage-and-bind-mounts|ADR-0006]]). Unchanged, restated
  because this record adds a registry interaction.

### The freshness assertion

Written here because it is this record's mitigation; the step record lands with
`spec/devenv`.

    set -uo pipefail
    : "${DEVBOX_IMAGE:?DEVBOX_IMAGE must name the built image by digest, e.g. localhost/devbox@sha256:<64 hex>}"
    : "${DEVBOX_MAX_AGE_DAYS:?DEVBOX_MAX_AGE_DAYS is the rebuild ceiling in days, e.g. 31}"
    lbl=io.github.luis-gordon.devbox.snapshot-date
    d=$(podman image inspect --format "{{index .Labels \"$lbl\"}}" "$DEVBOX_IMAGE" 2>/dev/null) || d=
    case "$d" in
      [0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]) ;;
      "") echo "RESULT=FAIL (no $lbl label on $DEVBOX_IMAGE — pre-ADR-0016 image?)"; exit 1 ;;
      *)  echo "RESULT=FAIL ($lbl is '$d', not YYYY-MM-DD)"; exit 1 ;;
    esac
    age=$(( ( $(date +%s) - $(date -d "$d" +%s) ) / 86400 ))
    if [ "$age" -gt "$DEVBOX_MAX_AGE_DAYS" ]; then
      echo "RESULT=FAIL (snapshot $d is ${age}d old, ceiling ${DEVBOX_MAX_AGE_DAYS}d — rebuild)"
    else
      echo RESULT=PASS
    fi

An absent label gets its own diagnostic rather than reading as a stale image: a
pre-ADR-0016 build has no label at all, and "no label" must not look like "very old",
which is [[../spec/_schema#Conventions|`_schema.md`]]'s don't-fail-the-same-way rule.

## Revisit when

An installer stops accepting a version argument, or the ALA stops serving the dates
being pinned. Either breaks a layer, and a broken layer should be recorded as such
rather than quietly dropped back to `latest`.
