# Replacement container setup — artifacts, round 1 (non-network)

## Context

The devbox agent container moves its base image **Debian trixie → Arch**, for parity
with the host it is used to reason about. Two changes ride along: **egress
allowlisting** and **extending build pinning** from the base image (ADR-0007) to the
whole userland.

The prompting failure: `awk` in the container was **mawk 1.3.4** where the target is
**gawk**, and a `\<` word-boundary escape — a gawk extension — matched nothing under
mawk, so a verification **passed while exercising nothing**. `grep` there was **ugrep
7.5.0**, not GNU grep.

Probes 1, 2, 3, 6, 7 ran on the host 2026-08-25 (`~/probes/20260825-1352/`). The
network round — 4, 4g, 5, 8 — has **not** run. The egress ADR and the pod runbook are
therefore **not written this round**, and every egress claim below is marked pending.

## What is now measured

| | Host | Container (`archlinux:base`) |
|---|---|---|
| awk | gawk 5.4.1, `/usr/bin/gawk` | gawk 5.4.1, `/usr/bin/gawk` |
| grep | GNU grep 3.12-modified | GNU grep 3.12-modified |
| sed | GNU sed 4.10 | — |
| pacman | 7.1.0 / libalpm 16.0.1 | 7.1.0 |
| podman | 6.0.2, rootless, crun | — |
| pasta | 2026_07_28.f8df3f1 | — |
| nftables | 1.1.6 | 1.1.6 (shipped in base) |

No mawk, no busybox, no npm on the host. Base ships no `jq`, `rg`, `git`, and **no
sync DBs**.

**P2 digests.** `.Digest` = `c9dc8b5d…`, which the manifest index shows is the
**amd64 instance**; `RepoDigests` holds that *and* `b860afd5…`, the index digest.
`/etc/pacman.d/mirrorlist` exists and is `Include`d at `pacman.conf:81` and `:87`,
with no bare `Server` lines.

**P3.** Read-only bind mount works. `Q_RC=0`, 892 packages, identical under
`--userns=keep-id`. No lock contention; no copied snapshot needed. `-Qi`, `-Ql`,
`-Si` all `RC=0`. `-Si` returned `podman 6.0.2-3` from `extra`. **`-Qo` resolves
against the host DB, not the container filesystem** — confirmed with `/usr/bin/sbctl`
and `/usr/bin/tailscale`, both absent from the image, both answered correctly.
`-Qu` returned `RC=1` on a fully-updated host: *nothing to list*, not failure.

**P6.** ALA `2026/08/24` returns HTTP 200. Mirrorlist rewrite, `pacman -Syy` `RC=0`,
`pacman -S gawk jq ripgrep` `RC=0` with keyring and integrity checks passing against
the **stock** `archlinux-keyring`. The trust anchor does not have to be part of the
pin.

**P7.** Both installers take a version. Claude's is **positional** (`$1`, validated
against `stable|latest|N.N.N`). Codex's is **`CODEX_RELEASE`**, normalised through
`normalize_version`, so a typo'd version is *not* rejected at parse time.
`localhost/devbox@sha256:…` runs, `RC=0`.

### Two divergence axes, not one

**Interactive shell vs non-interactive.** `.zshrc:36` aliased `grep` to `rg` and
`:37` `diff` to `difft`; both are now commented out. The first P1 reading recorded
**"ripgrep 15.2.0" as the host's grep** — a wrong answer produced by a correct-looking
command. This is a second axis on top of Debian-vs-Arch, and it is the reason the
collector must record `readlink -f "$(command -v …)"` and never the shell-visible
name.

**`command -v` vs the resolved path.** In the container `command -v gawk` returns
`/usr/sbin/gawk` where `readlink -f` gives `/usr/bin/gawk` — `/usr/sbin` is a
symlink. A parity check comparing `command -v` output would report a divergence that
does not exist.

Both cases are the same rule, and it is the eighth convention below.

### What is now measured from inside the container

Read from within the running container 2026-08-25, which answers most of what was
previously taken on trust. `/proc/self/mountinfo` names the host-side source of every
bind mount:

| Container path | Host source (on `/dev/mapper/cryptroot`) |
|---|---|
| `/work` | `/@home/luiswg/dev/infra-docs` |
| `/home/dev/.claude` | `/@home/luiswg/.local/state/devbox/claude` |
| `/home/dev/.codex-state` | `/@home/luiswg/.local/state/devbox/codex-state` |

That corroborates `STATE="$HOME/.local/state/devbox"` independently of the wrapper's
line 29, and enumerates the mount set for this entry point **exactly** — three, not an
estimate. It also corroborates ADR-0006 from the other side: all three sources are on
`@home`, none in the container storage subvolume.

**It also identifies which entry point this is.** `SSH_AUTH_SOCK` is unset and there
are **zero** `VSCODE_*`, `REMOTE_CONTAINERS_*` and `GIT_ASKPASS` variables. Block-C
§3.9/§3.10 record that the attached devcontainer has all of them, unconditionally and
unsuppressably. So this is the sealed wrapper path without `--with-agent` — the mount
table above is **the wrapper's**, and the devcontainer's remains unmeasured.

**The current wrapper mounts the project at `/work`, not at `$PWD`** (measured above,
`id -u` = 1000, `$HOME` = `/home/dev`, user `dev`). Your note that the wrapper mounts
`$PWD` at `$PWD`, and that the user is `luiswg` with home `/home/luiswg`, is
therefore read as a **design constraint on the replacement** — which is what this
round's scope note says it is. Recording either as a description of what runs today
would put an intention inside a measurement.

Other live tells, measured: `container=podman` is set in the environment and
`/run/.containerenv` exists. `systemd-detect-virt` is **absent** from the Debian base
(`archlinux:base` ships systemd, so the replacement should have it — `inferred`, and
the lookup is one `systemd-detect-virt -c` in the built image).

### What remains unread, and why it no longer blocks anything

`~/dev/devbox/devbox` and `~/dev/devbox/Containerfile` are **not reachable from
here** — four candidate paths checked, none exists, and no mount exposes the recipe
repo. They are one `cat` away from your side, not from this one. Two claims rested on
your reading alone, and neither is load-bearing any more:

- the `devbox:latest` short name — already removed as a dependency, ADR-0016 stands
  on the moving-tag argument plus a measured `RC=0`;
- `useradd --uid 1000` as a literal — regrounded below on a measurement that supports
  the decision better than the literal did.

So no artifact depends on an unread file, and ADR-0015's provenance is upgraded on
evidence rather than on trust. **Mirroring the recipe repo into the vault remains
worth doing** — the cwd guard's test, `git_args`, and the full flag list are still
unexamined, and the wrapper is the only component of this system with no copy under
version control here. That is a task for the next round, not a blocker for this one.

---

# Artifact A — ADR-0015: the devbox base image is Arch

```markdown
---
title: The devbox base image is Arch, matching the host
type: decision
id: ADR-0015
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
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
Block-C §2 records "The host stays Arch; the box does not have to match." The
reasoning was that a rolling base defeats a reproducible artifact, and matching
the host bought nothing.

It bought something, and the price was paid before anyone noticed. The container's
`awk` was **mawk 1.3.4** where the machine runs **gawk**. A `\<` word-boundary
escape — a gawk extension — matched nothing under mawk, so a collector
verification **passed while exercising nothing at all**. `backlog/devbox-contents-audit`
calls that "the worst failure shape available — a check that fails the same way as
a pass." A second instance was found on 2026-08-24: `grep` in the container was
**ugrep 7.5.0**, not GNU grep, and nothing had been resting on a GNU extension
only by luck.

The container's purpose has also drifted. It was built to run coding agents against
Node/Python projects; it is now also where this vault is written and where spec
postconditions are reasoned about — postconditions for a machine whose toolchain the
container did not share.

## Decision

Base the devbox image on **Arch**, pinned by digest:

    FROM docker.io/library/archlinux@sha256:c9dc8b5d…

Mount the host's pacman database read-only at `/host/pacman`, and answer every
host package question through `--dbpath /host/pacman`
(`_schema.md`, eighth convention).

Keep `useradd --uid 1000` as an explicit literal in the Arch `Containerfile`, with a
postcondition asserting it.

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

**The UID requirement is what makes the swap non-destructive, and it is a
requirement rather than a literal copied from the old recipe.** Measured from inside
the running container 2026-08-25: `id -u` is 1000, and all three bind mounts —
`/work`, `$STATE/claude`, `$STATE/codex-state`, whose host sources
`/proc/self/mountinfo` gives as `/@home/luiswg/…` — are owned by uid 1000 and
writable. So the rule the replacement must satisfy is *the container user's UID
equals the UID that owns the host-side state*, and 1000 is what that resolves to
today. Stated that way it survives a host user being renumbered, which a literal
copied out of the old Containerfile would not.

The postcondition asserts the requirement, not the number: it reads the owner of the
mounted state and compares it to `id -u`, so a mismatch is caught whichever side
moved. (Host-side ownership is `inferred` from the mount source path naming
`luiswg`; the lookup is one `stat -c %U ~/.local/state/devbox/claude` on the host.)

## Consequences

### Accepted negative: container and host are now hard to tell apart

This is the real cost and it is worth stating without softening. Before, a `pacman`
command typed in the wrong place failed with `command not found`, and a Debian
prompt was a standing reminder of which side you were on. **The base's difference
was itself a guard rail, and this decision removes it.** From now on a host command
run in the container succeeds, against the wrong package database, and reports a
plausible answer.

Three things narrow it, none of which closes it:

- The eighth convention routes every host package question through
  `--dbpath /host/pacman`, so the *correct* invocation is visibly different from the
  incorrect one rather than identical to it.
- The mount is named `/host/pacman`, not `/var/lib/pacman`. The path is the reminder.
- The image emits its identity as OCI labels (ADR-0016), so "which side am I on" has
  an answer that does not depend on noticing.

Residual, and named so it is not rediscovered: **an agent or human who runs
`pacman -Q` inside the container gets the container's 892-package answer with no
error and no warning.** Nothing in this design prevents that. It is the price of
parity, and parity was bought to close a failure mode that had already fired.

**Amended 2026-08-25 — ADR-0017 removes the last visual guard rail.** This record was
written when the container still announced itself: a different distro in
`/etc/os-release`, a different user (`dev`), a different home (`/home/dev`), and a
project mounted at `/work` rather than at its host path.
[[20260825-harness-state-and-identity|ADR-0017]] removes all four for reasons that
are good on their own — path parity is what stops harness memory being keyed to a
launch directory. The consequence is that **nothing about a shell prompt, a `pwd`, a
`whoami` or an `/etc/os-release` distinguishes the two sides any more.**

What replaces it, and the split matters:

- **For a check, a live reading.** `/run/.containerenv` exists inside and cannot
  exist outside (*measured 2026-08-25*), and `systemd-detect-virt -c` answers the
  same question on an Arch base. Neither is a marker this system sets, so neither can
  drift out of sync with reality — the same reason `spec/base` uses
  `systemd-detect-virt -r` for the chroot boundary rather than a sentinel file.
- **For a human, three markers**, each asserted by a postcondition in ADR-0017:
  `--hostname devbox…`, `DEVBOX=1`, and a shell prompt that renders the hostname.
- **The prompt is an open item, not a solved one.** `archlinux:base` ships no shell
  configuration at all, so today there would be no prompt to put a marker in. Until
  that is written, the human-facing half of this mitigation does not exist and the
  machine-readable half is carrying it alone.

### Other consequences

- **`/var/lib/pacman:ro` is a fifth enumerated host path**, and the rule it excepts
  is stated here for the first time because it was never written down:

  > **Host paths are enumerated and justified individually.** There is no blanket
  > "none except `/work`" — there never was. Adding one is a decision, not a
  > configuration change.

  **Measured 2026-08-25 from `/proc/self/mountinfo`**, on the wrapper entry point:
  three bind mounts, `/work` ← `/@home/luiswg/dev/infra-docs`, `/home/dev/.claude` ←
  `/@home/luiswg/.local/state/devbox/claude`, `/home/dev/.codex-state` ←
  `/@home/luiswg/.local/state/devbox/codex-state`. `$SSH_AUTH_SOCK` is the fourth and
  is opt-in (`--with-agent`); it was absent from this session, which is how the entry
  point was identified. `/var/lib/pacman` read-only is the fifth.

  **There are two enumerations, not one, and only one of them is measurable from
  inside.** The wrapper's set is above. The attached devcontainer's is a different
  list in a different file, and Block-C records that it additionally receives
  `$SSH_AUTH_SOCK` **unconditionally**. So "the mount set" is not a property of the
  image; it is a property of the entry point, and the rule has to be applied twice.
  That is a second reason the probe-8 retire/keep decision matters beyond egress.

  Two further mounts are already proposed and unresolved —
  `/usr/share/doc/arch-wiki/text` plus `/usr/bin/wiki-search`
  (`backlog/devbox-wiki-mount`) and `/usr/share/man`
  (`backlog/devbox-contents-audit`). Both now have a rule to be measured against.

- **The accidental cross-implementation testing is gone, and that was an argument
  against this change.** `backlog/devbox-contents-audit` asked for the gawk question
  to be decided deliberately rather than defaulted, precisely because the container
  being *stricter* than the target is what caught the `\<` bug. This decision answers
  it: parity wins, because a divergence that catches one bug by accident and hides
  the next by the same mechanism is not a testing strategy. **The audit's condition
  comes due with it** — the collector now needs a POSIX-only lint from somewhere
  else, or the next gawk extension goes in unnoticed. Recorded as an open item, not
  closed here.

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
  it proposed becomes the version-accurate channel 2 the root `CLAUDE.md` wants.
  Not adopted here; the obstacle it names has moved and the item should be re-read.

### What this record does not touch

**ADR-0001 is unaffected by this decision and is affected by the next one.** Its
measured claim — rootless podman writes *no* nftables rules, `inet filter` hash
`4bc77135…` — remains true of everything here. The egress work will write nftables
rules, in a container netns rather than the host table, and the ADR's headline
sentence does not carry that distinction. **That is the egress ADR's problem to
solve, and it is not solved here.** Flagged so the next round does not inherit it
silently.

## Revisit when

The container gains a fourth use, or an Arch-specific breakage in the image costs
more than the mawk class did. A rolling base *is* less reproducible than a stable
one — ADR-0016 is what makes that survivable, and if its snapshot pin ever proves
unworkable this decision has to be re-argued rather than patched.
```

---

# Artifact B — ADR-0016: pinning across four layers (supersedes ADR-0007)

```markdown
---
title: The devbox is pinned at four layers, and the image declares its own identity
type: decision
id: ADR-0016
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
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
- Codex's installer takes **`CODEX_RELEASE`**, normalised through
  `normalize_version`.
- The Arch Linux Archive serves a dated repo snapshot that `pacman -Syy` and
  `pacman -S` both accept, with signature and integrity checks passing against the
  **stock** `archlinux-keyring`.
- `podman run localhost/devbox@sha256:…` works, so a locally built image can be
  referenced by digest rather than by a moving tag.

None of that required vendoring. ADR-0007 was not wrong; it is out of date.

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
`podman image inspect --format '{{.Digest}}'` of the image the session used. These
are deliberately two records: they diverge the moment there are two images on disk,
and collapsing them would make the label answer a question it cannot.

## Rationale

**Layer 1 is unchanged from ADR-0007 and its open question is now closed.**
`spec/containers` recorded a risk that a multi-architecture pull would make
`podman-first-pull` permanently red, because the manifest-list digest and the
platform digest differ and the page does not say which `RepoDigests` reports.
Measured: `.Digest` is the **amd64 instance** (`c9dc8b5d…`) and `RepoDigests` holds
**both** it and the index digest (`b860afd5…`). The postcondition asserts
**membership** in `RepoDigests`, not equality, so it is correct under either pin —
and it is correct by construction rather than by luck.

**Layer 3 is the one ADR-0007 declined, and the two installers need different
treatment.** Claude's argument is positionally validated against a pattern, so a
malformed version fails at parse. Codex's `CODEX_RELEASE` is normalised, so a typo is
**not rejected at parse time** and the build proceeds with something other than what
was asked for. The build therefore carries a postcondition asserting the *installed*
version equals the pinned literal — the observable, not the input. That is the same
reasoning `podman-install` uses for asserting the rootless capability instead of the
sysctl.

**Layer 4 rests on the same argument as layer 1.** ADR-0007's own sentence — "a tag
is a moving reference, a digest is content-addressed and immutable" — applies to the
image this repo *builds* exactly as it applies to the image it *consumes*, and
ADR-0007 applied it to only one end. Measured 2026-08-25:
`podman run localhost/devbox@sha256:…` returns `RC=0`, so the digest form is
available at the consuming end and the pin costs nothing.

*Reported separately, and not load-bearing here:* the `devbox` wrapper's last line is
`devbox:latest`, an unqualified short name, which additionally exposes podman's
short-name resolution. That comes from a host reading on 2026-08-25 of a file this
vault holds no copy of, so it is `inherited-unverified` and this record does not
depend on it — layer 4 stands on the moving-tag argument alone. It is noted because
the same edit fixes it.

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
  whenever an agent update forces a rebuild anyway — one rebuild when the timing
  lines up — and unconditionally when the ceiling is reached.
- **Staleness is a check, not a habit.** `snapshot-date` is a label, so a
  postcondition can read it with `podman image inspect` and go red past the window
  without starting a container. The step record is `devbox-snapshot-freshness` and
  belongs to `spec/devenv`; the assertion is below.

**The exposure here is small, and saying so is part of the decision.** Short-lived
sandbox, `--cap-drop=ALL`, no host home, no `~/.ssh`, no KDBX, egress allowlisted
(pending), and no credentials beyond the agent tokens that already have their own
bind mounts. A month-old userland in that position is not the risk worth losing sleep
over. This paragraph exists so that nobody reads the paragraph above it and
over-corrects into a weekly rebuild treadmill.

### Keyring/snapshot skew is a loud failure, and its fix is written down now

Measured working today. If a future snapshot is old enough that the stock keyring
rejects it, `pacman` fails with a signature error and the **build stops** — it does
not silently install unverified packages. The fix, stated so it is not re-derived
under pressure: **pin `archlinux-keyring` from the same snapshot** and install it
first. `SigLevel = Never` is not an available answer and should be treated as a
defect if it ever appears in the recipe.

### Other consequences

- Rebuilding remains a deliberate act, and the commit message remains the audit
  trail — but it is now a **convenience**, not the mechanism. ADR-0007's named
  failure ("if the message is skipped, the record is lost — there is no automated
  capture") is closed: the labels are the automated capture.
- Four pins mean four things to bump. A rebuild that moves only the base digest and
  leaves the snapshot date behind is a legal state and a misleading one; the
  freshness check reads `snapshot-date` for exactly that reason.
- **ADR-0007's uninvited-software problem is narrowed, not solved.** Labels describe
  the recipe's output; they cannot describe what an extension writes into a running
  container afterwards. The Copilot CLI agent would still not appear in any label.
- `~/.config/containers/auth.json` still holds registry credentials in cleartext
  (ADR-0006). Unchanged, restated because this record adds a registry interaction.

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
which is `_schema.md`'s don't-fail-the-same-way rule.

## Revisit when

An installer stops accepting a version argument, or the ALA stops serving the dates
being pinned. Either breaks a layer, and a broken layer should be recorded as such
rather than quietly dropped back to `latest`.
```

---

# Artifact C — eighth convention for `spec/_schema.md`

To be appended to the **Conventions** section.

```markdown
**The container has a plausible wrong answer to every host question.** The devbox is
Arch and the machine is Arch ([[../decisions/20260825-devbox-base-is-arch|ADR-0015]]),
so `pacman`, `awk`, `grep` and `sed` all exist on both sides and all answer. Before,
a host command typed in the container failed loudly; now it succeeds against the
wrong data. Two mechanisms, and both are about making the correct invocation *look*
different from the incorrect one.

**1. Host package questions go through `--dbpath /host/pacman`.** The container's own
repositories build the container and answer nothing about the machine. The host's
database is bind-mounted read-only at `/host/pacman`, and every question about what
the machine has names that path explicitly. A bare `pacman -Q` in a step record is a
defect even when it returns something.

Measured 2026-08-25, against a read-only mount, identical under `--userns=keep-id`:

| Query | Result |
|---|---|
| `-Q` | works, 892 packages, no lock contention |
| `-Qi`, `-Ql` | work |
| `-Si` | works — returned `podman 6.0.2-3` from `extra` |
| `-Qo` | resolves against the **host** database, not the container filesystem |
| `-Qu` | exits **1** on a fully-updated host |

Three of those rows are traps:

- **`-Qo` is right for the wrong-feeling reason.** It answered correctly for
  `/usr/bin/sbctl` and `/usr/bin/tailscale`, neither of which exists in the image.
  It is resolving the path against the database, not the filesystem — so it is
  usable, and the reason it works is not the reason it looks like it works.
- **`-Si` reflects the last `pacman -Sy` the *host* ran.** A postcondition asserting
  availability is asserting the freshness of the host's sync database, not the state
  of the repositories. Say which one is meant.
- **`-Qu` exits 1 when there is nothing to list.** That is the
  [[#Conventions|verdict-line rule]] in its original form: no postcondition may
  assert that `-Qu` succeeded, because success and "fully updated" are different
  things and the exit status conflates them.

No lock is taken for read-only queries, so no copied snapshot of the database is
needed and the mount stays `:ro`.

**2. A tool is identified by its resolved path, never by the name the shell shows.**
Use `readlink -f "$(command -v <tool>)"`. Two measured failures, on two different
axes, on the same day:

- **The shell's name is not the binary.** `.zshrc` aliased `grep` to `rg` and `diff`
  to `difft`. The first reading of the host's toolchain recorded **"ripgrep 15.2.0"
  as the host's grep** — a wrong answer produced by a correct-looking command. The
  aliases are now commented out, which changes nothing about the rule: an alias is
  one edit away at all times, and `state/notes` already records that `alias
  grep='rg'` silently changes the *interface*, not just the output.
- **`command -v` is not the resolved path.** In the container `command -v gawk`
  returns `/usr/sbin/gawk` where `readlink -f` gives `/usr/bin/gawk`, because
  `/usr/sbin` is a symlink. A parity check comparing `command -v` output would report
  a divergence that does not exist — a check failing red on a correct system, which
  is the [[disk#luks-format|permanently-red shape]] `spec/disk` avoids.

The two mechanisms are one rule: **a question about the machine, asked from inside
the container, must name what makes it a host question.** `--dbpath /host/pacman` for
packages; a resolved path for tools.
```

## Convention nine — paths shown to a human

Also appended to **Conventions**. Separate from the eighth: that one is about
*questions asked*, this one is about *answers printed*.

```markdown
**A path shown to a human must name which side it is on, or be true on both.**

*Worked case, measured 2026-08-25.* The `/plan` command printed
`/home/dev/.claude/plans/…` — a container path, rendered into a terminal a human
reads on the host, where `/home/dev` does not exist at all. The file was real: it is
at `~/.local/state/devbox/claude/plans/…` host-side, which
`/proc/self/mountinfo` gives directly. Nothing errored, and nothing about the string
said which filesystem it belonged to. That is the eighth convention's `command -v`
trap again — **a correct-looking string answering about the wrong side** — moved from
what a check reads to what a person is told.

Two repairs, and which one applies is decided by whether the path is a mount:

1. **Make it true on both sides.** Where the path is bind-mounted, mounting it at its
   host-identical absolute path means the printed string resolves for both readers
   and needs no annotation. This is
   [[../decisions/20260825-harness-state-and-identity|ADR-0017]]'s path parity, and
   it is the preferred repair because it removes the failure rather than labelling it.
2. **Name the side.** Where a path is deliberately container-only, say so:
   `/host/pacman` is the worked example, and it was named that way before this
   convention existed — the prefix *is* the annotation.

The rule binds prose, diagnostics and postcondition output alike. A `RESULT=FAIL`
line quoting a container path to someone standing at a host shell has told them
something they cannot act on.

*Known limit:* this cannot be enforced mechanically, because a path is just a string
and both sides are plausible. It is a review rule, and the review question is one
sentence — *if this line were read on the other side, would it still be true?*
```

---

# Artifact D — `spec/containers.md` corrections

Three edits, all grounded in P2. The chapter's frontmatter moves to
`last_verified: 2026-08-25`.

**D1 — close the multi-architecture open item.** Replace the *Known limit* paragraph
under `podman-first-pull` and its bullet in *Open against this chapter* with:

```markdown
*Resolved 2026-08-25.* For `archlinux:base`, `.Digest` is the **amd64 instance**
digest and `RepoDigests` carries **both** it and the manifest-index digest
(*measured*, `podman image inspect` on the machine). The check asks for membership in
`RepoDigests` rather than equality with `.Digest`, so it is correct whichever of the
two the parameter names. The risk this paragraph recorded — a permanently-red check
on a correct pull — does not exist for this image. It is a property of what
`RepoDigests` contains, not of the check's cleverness, so it is worth re-measuring if
the base ever changes registry.
```

**D2 — the parameter's example value changes distro.** Every occurrence of
`docker.io/library/debian:trixie-slim@sha256:<64 hex>` in `podman-first-pull`'s prose
and in its three `${DEVBOX_IMAGE_DIGEST:?}` guards becomes
`docker.io/library/archlinux@sha256:<64 hex>`. The shape check is unchanged and was
already correct.

**D3 — the toolchain limitation is now half-closed.** In *Open against this chapter*,
the final bullet ("No mutation sets, and no execution") gains:

```markdown
*Amended 2026-08-25.* The toolchain half of this limitation is closed by
[[../decisions/20260825-devbox-base-is-arch|ADR-0015]]: `awk` and `grep` in the
container are now gawk 5.4.1 and GNU grep 3.12, byte-identical to the machine
(*measured*). What remains is the tool half — `podman`, `btrfs`, `loginctl` and
`systemctl` are still absent from the container, so these postconditions can be
reviewed here and executed only there. That is now an accepted limitation with a
named consequence rather than an open problem, because installing Arch system
binaries into a container with no LUKS volume and no TPM to run them against would be
worse than useless.
```

---

# Artifact E — the collector emits tool versions

## E1 — `collect/collect.sh`

A new bounded section appended to `state/packages.md`. **Bounded, not
`pacman -Q` with versions**: `_index.md`'s anti-patterns forbid recording what the
system manages on its own, and on a rolling distro a full version list makes every
run a diff and no run a signal. The section emits only the tools this spec's
postconditions actually call.

```bash
# --- tool versions -----------------------------------------------------------
# Bounded on purpose: only tools that spec postconditions call. A full version
# list churns on every -Syu and signals nothing.
#
# readlink -f, never `command -v` output and never the shell-visible name.
# Both have produced a wrong answer on this machine (2026-08-25): .zshrc aliased
# grep to rg, and /usr/sbin is a symlink so `command -v gawk` reports a path that
# is not where the binary lives. Format: <name> <resolved-path> <version>
printf '\n## tool versions\n'
for t in awk grep sed jq findmnt btrfs cryptsetup sbctl systemctl \
         podman pasta nft pacman bootctl mkinitcpio loginctl; do
  p=$(command -v "$t" 2>/dev/null) || p=
  if [ -z "$p" ]; then
    printf '%s <absent> <absent>\n' "$t"
    continue
  fi
  r=$(readlink -f "$p") || r=$p
  # Branch on exit status, not on emptiness. A tool that rejects --version and
  # prints usage produces a non-empty first line, and testing `${v:-…}` would
  # record that usage message as the tool's version.
  if out=$("$r" --version 2>/dev/null); then
    v=$(printf '%s\n' "$out" | head -1)
  else
    v='<unreadable>'
  fi
  printf '%s %s %s\n' "$t" "$r" "$v"
done
```

`<absent>` rather than a skipped line: a tool that disappears must show as absent, not
as a shorter section — a check counting lines would otherwise pass while the tool it
cares about is gone.

**Emptiness is not the failure signal, and testing it would have recorded a usage
message as a version.** `mkinitcpio` and `bootctl` are the likely tools to reject
`--version`; a tool that does so typically prints usage *to stdout* and exits
non-zero, so the first line is non-empty and perfectly plausible. `${v:-<unreadable>}`
would have taken it. The exit status is the only reading that separates "this is the
version" from "this is a complaint about being asked."

`<unreadable>` is a **legitimate terminal state**, not a defect to be fixed: if a tool
genuinely has no `--version`, that is the truthful record and it will not change.
`collect-tool-versions` therefore does not fail on it — asserting otherwise would make
the step red on a correct machine forever.

## E2 — `spec/base.md`, new step: `collect-tool-versions`

The collector is how `state/` learns anything, and until now nothing in the spec
asserted that it had run or that its output was complete. That gap is what let the
toolchain question sit unanswerable in the vault for two months while both sides of
it were one command away.

**The section is asserted by name and by contents, not by existence.** A collector
that stops emitting the section leaves `toolchain-parity` reading nothing — and
reading nothing is exactly what the mawk bug did. So the postcondition names every
tool it expects, and a tool dropped from the loop fails the build rather than
quietly narrowing what is grounded.

**Counting is done with `awk`, not `grep -c`**, per the seventh convention: zero
matches must print `0` and exit 0, or the assignment aborts under `set -e` and the
diagnostic below it never fires.

```yaml
id: collect-tool-versions
requires: [first-boot]
precondition: |
  set -euo pipefail
  # The collector resolves paths relative to its own directory and writes
  # nothing useful from elsewhere.
  test -x ./collect/collect.sh \
    || { echo "run from arch-thinkpad/ — ./collect/collect.sh is not executable here"; exit 1; }
command: |
  set -euo pipefail
  ./collect/collect.sh
postcondition: |
  # set -e is dropped deliberately: several counts below legitimately read zero,
  # and guarding each one would be noise (fstab-generate is the worked case).
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  f=state/packages.md
  [ -r "$f" ] || { echo "RESULT=FAIL ($f is unreadable)"; exit 1; }
  # Exactly one section header. Two means an appended run rather than a rewrite.
  n=$(awk '$0 == "## tool versions" { n++ } END { print n + 0 }' "$f")
  [ "$n" -eq 1 ] || note "found $n '## tool versions' headers in $f, want 1"
  for t in awk grep sed jq findmnt btrfs cryptsetup sbctl systemctl \
           podman pasta nft pacman bootctl mkinitcpio loginctl; do
    # Restricted to the section: the tool names also occur in prose elsewhere.
    n=$(awk -v t="$t" '
          /^## / { insec = ($0 == "## tool versions") }
          insec && $1 == t { n++ }
          END { print n + 0 }' "$f")
    [ "$n" -eq 1 ] || { note "$t: $n lines under ## tool versions, want 1"; continue; }
    # Field 2 must be an absolute resolved path. A bare name here means the
    # collector recorded what the shell showed, which is the defect this
    # section exists to prevent.
    p=$(awk -v t="$t" '
          /^## / { insec = ($0 == "## tool versions") }
          insec && $1 == t { print $2; exit }' "$f")
    case "$p" in
      /*|"<absent>") ;;
      *) note "$t: path field is '$p', want an absolute path or <absent>" ;;
    esac
  done
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

## E3 — `spec/base.md`, new step: `toolchain-parity`

**No `command`, and that is the point.** `_schema.md` records four shapes an absent
`command` covers, and this is the third: a standing invariant that no action
establishes. Nothing an executor runs makes the container's `awk` match the
machine's — the image build does, one chapter away — so this step can only check.

**It reads live on both sides, and therefore requires only the image.** `state/` is
the record; a postcondition is the assertion, and `_schema.md` is explicit that the
spec records intent while `state/` and executor output record observation. The host
side is read with `readlink -f`; the image side comes from ADR-0016's labels, so no
container has to start and the check is cheap enough to run often.

*An earlier draft carried `requires: [collect-tool-versions, devbox-image-build]`.
The first edge was wrong* — the check never opens `state/packages.md`, so the edge
encoded an intention rather than a dependency, and `_schema.md` is explicit that
`requires` is edges, not narrative. `collect-tool-versions` grounds the vault's
record; this step asserts the invariant. They are related in subject and independent
in execution, and a DAG that says otherwise is a DAG nobody can trust to reorder.

**This is the step that would have caught the original bug** — a `\<` escape that
matched nothing under mawk while the machine ran gawk, letting a verification pass
while exercising nothing.

### Two findings, two verdicts, two remedies

**Asserting the full version string would make this check red by default.** Today's
host line is `GNU Awk 5.4.1, API 4.1, PMA Avon 8-g1, (GNU MPFR 4.2.2, GNU MP 6.3.0)`
— four independently versioned components, any of which moves on a host `-Syu`. And
ADR-0016 pins the container to a repo snapshot **precisely so that its userland lags
the rolling host**. Under a monthly rebuild ceiling the two strings would differ most
of the time, on a completely correct system. That is the permanently-red shape
`spec/disk` avoids with `luks-format`, and a check that is red by default is a check
that stops being read — which would leave the substitution bug uncaught again, by a
different route.

So the step separates what it is for from what it can see:

| Finding | Verdict | Remedy |
|---|---|---|
| **Implementation substitution** — mawk for gawk, ugrep for GNU grep | `RESULT=FAIL` | the base image is wrong |
| **Version drift** — same implementation, different build | `DRIFT=<tools>` | a rebuild, or nothing |

Drift is expected under ADR-0016 and is reported, not judged. Substitution is the bug
this step exists for and is a hard failure.

**The implementation token is a policy literal, and that is allowed.** "Assert against
live state, never against constants" forbids values that vary per machine or per
rebuild; `GNU Awk` is neither. It is what the decision requires to be true, in the
same class as `tpm2-enroll` asserting the PCR set `[0, 2, 7]` as a literal — the
policy is the constant, and a check that resolved it at run time would assert nothing.

```yaml
id: toolchain-parity
requires: [devbox-image-build]
postcondition: |
  set -uo pipefail
  : "${DEVBOX_IMAGE:?DEVBOX_IMAGE must name the built agent image by digest, e.g. localhost/devbox@sha256:<64 hex>}"
  fail=0
  drift=
  note() { echo "  $1"; fail=1; }
  ns=io.github.luis-gordon.devbox
  # tool:implementation-token. A policy literal, not a machine-varying value —
  # same class as tpm2-enroll's [0, 2, 7]. The version numbers are deliberately
  # not part of it: ADR-0016 pins the image to a snapshot so that they diverge.
  for pair in "awk:GNU Awk" "grep:GNU grep"; do
    t=${pair%%:*}
    want=${pair#*:}
    # Host: the resolved path, never `command -v` output. /usr/sbin is a symlink
    # on the image side, and an alias can replace the name on this side.
    p=$(command -v "$t" 2>/dev/null) || p=
    [ -n "$p" ] || { note "$t is absent on the host"; continue; }
    r=$(readlink -f "$p") || r=$p
    # Exit status, not emptiness: a tool that rejects --version prints usage to
    # stdout, and a non-empty first line would be taken for a version.
    if out=$("$r" --version 2>/dev/null); then
      hv=$(printf '%s\n' "$out" | head -1)
    else
      note "$t on the host rejected --version"; continue
    fi
    # Image: from the label, so nothing has to start.
    iv=$(podman image inspect --format "{{index .Labels \"$ns.$t\"}}" \
           "$DEVBOX_IMAGE" 2>/dev/null) || iv=
    case "$iv" in
      ""|"<no value>")
        # An unlabelled image is a pre-ADR-0016 build. Not the same finding as a
        # mismatch, and it must not report as one.
        note "$DEVBOX_IMAGE carries no $ns.$t label — pre-ADR-0016 image?"
        continue ;;
    esac
    # Verdict 1 — substitution. Both sides must carry the implementation token.
    case "$hv" in *"$want"*) ;; *) note "host $t is not $want: $hv" ;; esac
    case "$iv" in *"$want"*) ;; *) note "image $t is not $want: $iv" ;; esac
    # Verdict 2 — drift. Same implementation, different build. Informational:
    # under ADR-0016's snapshot pin this is the expected steady state.
    [ "$hv" = "$iv" ] || drift="${drift:+$drift }$t"
  done
  echo "DRIFT=${drift:-none}"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

`DRIFT=` is emitted on every run, including `none`, so its absence means the check did
not reach its end rather than that nothing drifted — the same reason `RESULT=` is
unconditional.

## E4 — DAG position, and the unresolved table stops being empty

```
first-boot ──→ collect-tool-versions                 (grounds state/)

devbox-image-build (spec/devenv) ──→ toolchain-parity  (asserts the invariant)
```

**Two chains, not one.** They share a subject and nothing else: `toolchain-parity`
reads the host live and the image from its labels, and never opens `state/`. The edge
that once joined them documented an intention, and an intention in a `requires:` list
is worse than no edge at all — it constrains a topological sort for a reason that does
not exist.

`toolchain-parity` requires `devbox-image-build`, which **does not exist** — it
belongs to `spec/devenv`, the chapter ADR-0007 and ADR-0006 both name and nothing has
written. So `spec/_index.md`'s *Unresolved `requires`* table goes from empty to one
entry, and the entry is the right one.

That is worth stating rather than apologising for. `spec/_index.md` complained on
2026-08-24 that the table "measures resolution, not coverage," and gave as its worked
case that **`spec/devenv` is named by two ADRs and by no `requires:` line, so the
table cannot see it**. This is the first `requires:` line that names it. The measure
was not wrong about `spec/devenv`; it was blind to it, and it is not blind any more.

---

# Artifact F — `state/notes.md` amendment

`notes.md` is hand-written and currently asserts something false about the machine.
Its *Shell and tooling* entry says of `alias grep='rg'`: *"It was not removed — `type
grep` confirms it is still live."* Both aliases are now commented out
(`.zshrc:36`, `:37`). Amend rather than delete — the entry's finding is still the
useful part:

```markdown
**Amended 2026-08-25.** Both aliases are now commented out at `.zshrc:36` and `:37`.
The finding stands and the reason for keeping this entry is sharper than it was: the
alias produced **a wrong reading of the machine's own toolchain**. The first attempt
to record the host's `grep` version returned "ripgrep 15.2.0" — a correct-looking
command answering about the wrong binary. That is why the collector resolves tools
with `readlink -f` and why `_schema.md`'s eighth convention names the shell-visible
name as untrustworthy. An alias is one edit from returning; the rule does not depend
on this one staying commented out.
```

---

# Artifact G — ADR-0017: harness state, config and identity

The fifth design axis, peer to ADR-0015 (base image) and ADR-0016 (pinning).

```markdown
---
title: Harness state, config and identity in the devbox
type: decision
id: ADR-0017
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
tags: [decision, containers, devbox, agents, state, config, credentials, identity]
constrains: spec/devenv
---

# ADR-0017 — Harness state, config and identity in the devbox

- **Status:** accepted
- **Date:** 2026-08-25
- **Provenance:** measured — the layout, the orphaned project trees and the config
  drift were all read from inside the running container 2026-08-25

## Context

The container was built for one harness. It now hosts more than one, and OpenCode and
Cline are expected. Every one of them keeps configuration, derived state, credentials
and cache — and this system has never said which is which, where each belongs, or
what happens to it on a rebuild. The result is four measured defects, none of which
announced itself.

**Project memory is keyed to the launch path.** `$STATE/claude/projects/` holds two
trees, `-work` and `-tmp`, 9.7 MB between them — one per directory the harness has
ever been launched from. Neither is reachable from the other; the earlier one is
simply orphaned. The replacement changes the mount point, which would orphan the
current one too, a third time.

**Config and state are not separated on disk.** Measured in `$STATE/claude`:

| Entry | Category | Mode |
|---|---|---|
| `settings.json`, 328 B | config | 0644 |
| `.claude.json`, 44.7 KB | config **and** state, mixed | 0600 |
| `.credentials.json`, 508 B | credential | 0600 |
| `history.jsonl`, `sessions/`, `session-env/`, `file-history/`, `plans/`, `projects/` | derived state | mixed |
| `plugins/` 21 MB, `paste-cache/`, `shell-snapshots/`, `backups/`, `.last-cleanup` | cache | 0755 |

**The seed is asymmetric and has already drifted.** The wrapper seeds Codex's config
from the repo `if [[ ! -f ]]` — once, never re-synced — and gives the other harness
no equivalent. Measured today: the repo/image copy at `~/.codex-default/config.toml`
is **214 bytes**; the live copy at `$CODEX_HOME/config.toml` is **609 bytes**,
carrying a comment block explaining the two-sandbox design that exists in exactly one
place and is not the one under version control. The drift is not hypothetical and it
is invisible from both ends.

**The permission allowlist is a file the agent can edit.** Measured:
`/work/.claude/settings.local.json` is tracked in git and has commit history, so the
project-level allowlist leaves a trace. `$STATE/claude/settings.json` is in
bind-mounted state with no repository at all, so the user-level one does not. Both
are writable by the process they constrain.

## Decision

### 1. Path parity, plus a stable project identity

**Mount the project at its host-identical absolute path.** The repository at
`~/dev/infra-docs` on the host appears at the same absolute path inside, replacing
`/work`.

**And set `CLAUDE_CODE_PROJECT_DIR_NAME` to a stable literal.** Belt and braces,
because the two fix different failures: path parity makes the path the same for every
harness at once, and a stable identity makes project memory independent of the path
even when it does change — a different checkout, a symlink, a second worktree.

### 2. Four categories, three destinations

| Category | Destination | Rule |
|---|---|---|
| Config | git | drift shows as a diff |
| Derived state | gitignored, in the restic include set | survives a rebuild, not reviewed |
| Credentials | gitignored, in `redact.map` | never in a commit, never in a document |
| Cache | disposable | not backed up, not preserved |

**Enforced by a deny-by-default `.gitignore`**, not by directory and not by env var.

**Config mounts are not read-only.** Harnesses legitimately write into their config
directory — onboarding flags, first-run acknowledgements, theme changes made from
inside. A read-only mount converts that into an error at a moment nobody is watching
for one.

### 3. Seeding becomes a symmetric sync-and-diff

One manifest, one mechanism, every harness:

- **absent** → install from the repo.
- **present and identical** → nothing.
- **present and different** → **report the diff and change neither side.**

The third case is the whole point. Overwriting silently would have destroyed the
609-byte comment block; skipping silently is what let it drift for eight days
unnoticed. Reporting makes the human choose, which is the only correct answer when
both copies are plausibly right.

### 4. Identity markers

`--hostname devbox…`, `DEVBOX=1`, and a shell prompt that renders the hostname. Each
carries a postcondition.

## Rationale

**Path parity over per-harness levers, and the argument is about scaling.**
`CLAUDE_CODE_PROJECT_DIR_NAME` is documented as a per-harness lever (*reported
2026-08-25; the lookup is one launch with it set, watching what appears under
`projects/`*). It fixes one harness. Path parity fixes the key itself — the launch
path — so it fixes every harness that keys on a path, including the ones not chosen
yet, and it does so without knowing their environment variable names. With three
harnesses expected, per-harness levers are three lookups and three chances to miss
one.

Path parity also repairs a second thing: it makes convention nine's failure
impossible for project paths, because a printed path resolves on both sides.

**The stable literal is not redundant with it.** Path parity makes the path the same;
it does not make the path *stable*. A second worktree, a rename, a symlinked checkout
all move it again, and the failure mode is silent — a new empty memory tree, which
reads as a fresh project rather than as a lost one. The literal removes the
dependency entirely.

**The split cannot be done by environment variable, and this had to be measured to
know.** `CLAUDE_CONFIG_DIR` relocates the *whole* tree — config, state, credentials
and cache land in one directory together, which is exactly what
`$STATE/claude` shows. So the separation has to be imposed from outside the harness.

**Of the three ways to impose it, a `.gitignore` allowlist is the only durable one.**
By directory is not available: the harness chooses its own layout and will add to it.
By symlink fails silently and badly — a harness that rewrites a file by
create-temp-and-rename replaces the symlink with a regular file, and from then on the
arrangement is over with no error and no diff. A deny-by-default `.gitignore` — ignore
everything, then un-ignore the named config files — survives both, and has the
property that **a new file the harness invents is excluded until someone decides
otherwise**, which is the correct default for a directory this system does not own.

**`.claude.json` cannot be split and is therefore state.** It mixes MCP server
definitions and per-project trust (config) with the OAuth account and counters
(state) in one 44 KB file. ADR-0006 already records why it cannot simply be left
outside — `CLAUDE_CONFIG_DIR` is what moves it inside the mounted path, and without
that you are signed out every exit. It goes to the state destination, and the
consequence is named below.

**A file inside the container cannot constrain the container.** The permission
allowlist is advisory in the strict sense: the process it governs can rewrite it, and
the user-level copy leaves no trace when it does. This is the same class of problem
the egress work addresses and the opposite kind of answer — an nftables rule written
by a `NET_ADMIN` sibling into a netns the agent has no capability to touch is a
control the agent cannot edit. **The allowlist is a workflow aid; the egress rules are
a boundary.** Recording that distinction is the point, because they look alike from
inside and are nothing alike.

## Consequences

- **This removes the last visual guard rail.** Same distro (ADR-0015), same user,
  same home, same working directory. ADR-0015's consequences section carries the
  amendment and names what replaces it: `/run/.containerenv` for checks (*measured
  present*), three markers for humans, and an open item for the prompt.
- **The prompt does not exist yet.** `archlinux:base` ships no shell configuration,
  so there is no prompt for a marker to appear in. Until that is written the
  human-facing mitigation is absent and the machine-readable one is carrying it alone.
  Open item, named rather than assumed.
- **Enterprise policy at `/etc/claude-code/CLAUDE.md` is deliberately not
  persisted.** Measured: the path does not exist. It is container-only by design, so
  its absence is a decision and not an oversight — stated here so nobody adds a mount
  for it later on the assumption that something was forgotten.
- **`settings.json` is 0644 with content, and ADR-0006's own trigger has fired.**
  That record says "22 bytes (defaults) and mode 0644 while its neighbours are
  0600. Harmless while empty; tighten if it gains content." It is 328 bytes and still
  0644. The content is a model name, a theme and a plugin list — not secret — but the
  condition the ADR set has been met and nothing happened, which is the part worth
  recording.
- **MCP configuration and per-project trust are not diffable**, because they live in
  `.claude.json` with the OAuth account. A change to which MCP servers this project
  trusts leaves no reviewable trace. Accepted; the alternative is committing a file
  containing a live account reference.
- **The orphaned `-work` tree needs migrating, and `-tmp` needs a decision.** Doing
  nothing orphans 9.7 MB of project memory a third time. Migration is a one-time move
  of `projects/-work` onto the new key; `-tmp` is a launch from `/tmp` and is almost
  certainly disposable, but deleting it is a choice someone should make rather than a
  side effect. **The migration target depends on how the stable literal composes the
  key** — whether it replaces the whole directory name or one component of it — which
  is the lookup named above and must be answered before the move, not after.
- **The sync-and-diff will report on first run**, because the drift already exists.
  That is the mechanism working, not a fault, and the 609-byte version is the one to
  keep.
- **A rebuild no longer silently loses local config edits** — it reports them. That is
  the direct repair of the failure this record was written for.

## Revisit when

A harness arrives that keys project memory on something other than a path, or one
that cannot be configured from a file at all. Either breaks the manifest model, and
the manifest is the part designed to absorb new harnesses without new decisions.
```

---

# Artifact H — step record: `devbox-ephemeral-writes`

`podman diff` reports changes to a container's writable layer. **Bind mounts do not
appear in it** — they are separate mounts, not part of the layer — so its output is
exactly the set of writes that die on the next rebuild, and nothing else.

**This half-closes ADR-0016's named gap.** That record concedes that labels "describe
the recipe's output; they cannot describe what an extension writes into a running
container afterwards," with the Copilot CLI agent as the worked case. A layer diff
can. It does not prevent the write, and it does not see anything written into a bind
mount — hence *half*.

**The parameter's hazard is a wrong target, so clause 3 applies in its original
form.** Diffing the wrong container reports clean, and a clean report is exactly what
this step is looking for — the failure would be silent and reassuring. So
`$DEVBOX_CONTAINER` is validated against something the DAG established: the
container's image must be `$DEVBOX_IMAGE`. That is the same shape as `luks-format`
tying `$LUKSDEV` back to what `disk-partition` produced, and the opposite of
`$DEVBOX_IMAGE_DIGEST`, whose hazard is a moving reference and whose check is
therefore a shape.

**No `command`.** Nothing an executor runs makes this true; a session does, and the
step can only look at the result afterwards. Third shape again.

```yaml
id: devbox-ephemeral-writes
requires: [devbox-image-build]
postcondition: |
  set -uo pipefail
  : "${DEVBOX_CONTAINER:?DEVBOX_CONTAINER must name a container that has run a normal session}"
  : "${DEVBOX_IMAGE:?DEVBOX_IMAGE must name the built agent image by digest, e.g. localhost/devbox@sha256:<64 hex>}"
  fail=0
  note() { echo "  $1"; fail=1; }
  # Clause 3: validate against what the DAG established, not against the type.
  # Diffing the wrong container reports clean, and clean is the answer this step
  # is hoping for — the wrong target would be silent and reassuring.
  img=$(podman inspect --format '{{.ImageName}}' "$DEVBOX_CONTAINER" 2>/dev/null) || img=
  [ -n "$img" ] || { echo "RESULT=FAIL (no container named $DEVBOX_CONTAINER)"; exit 1; }
  [ "$img" = "$DEVBOX_IMAGE" ] \
    || { echo "RESULT=FAIL ($DEVBOX_CONTAINER runs $img, not $DEVBOX_IMAGE)"; exit 1; }
  # Bind mounts are not part of the writable layer, so everything here is a write
  # that a rebuild destroys. The allowlist is what may legitimately be lost.
  d=$(podman diff "$DEVBOX_CONTAINER" 2>/dev/null) || d=
  # An empty diff is a legitimate result and must not read as a broken check.
  n=$(printf '%s\n' "$d" | awk 'NF { n++ } END { print n + 0 }')
  echo "EPHEMERAL_PATHS=$n"
  # Every path must sit under an allowed prefix. Anything else is state that
  # will be lost on rebuild without anyone having decided it was disposable.
  # awk rather than a `while read` loop: the loop would need a heredoc or a
  # pipe, and a pipe puts the accumulator in a subshell where fail=1 is lost.
  bad=$(printf '%s\n' "$d" | awk '
    NF == 0 { next }
    {
      p = $2
      if (p ~ /^\/(tmp|var\/tmp|run)(\/|$)/)            next
      if (p ~ /^\/home\/[^\/]+\/\.(cache|npm)(\/|$)/)   next
      if (p ~ /^\/home\/[^\/]+\/\.local\/state\//)      next
      print $1, p
    }')
  if [ -n "$bad" ]; then
    printf '%s\n' "$bad" | sed 's/^/  outside allowlist: /'
    fail=1
  fi
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

**Why `awk` and not a `while read` loop**, since the loop is the obvious shape: it
needs its input from somewhere, and both routes are defects here. A heredoc breaks the
seventh convention outright — a step record is copied out of a rendered page with its
indentation, and a terminator that no longer sits at column zero either hangs waiting
for a delimiter or swallows the rest of the block. A pipe avoids that and puts the
loop in a subshell, where `fail=1` is set and then discarded when the subshell exits,
so the verdict line reports `PASS` over a list of violations it just printed. `awk`
carries the result out on stdout instead, which neither construct can lose.

*Unverified, with the lookup:* `podman diff`'s output columns and whether it offers
`--format json` — one `podman diff --help`, channel 1. The `awk` above assumes
`<op> <path>`; if a JSON form exists it is the better reader and the regexes go away.

*Also unverified:* that `podman inspect --format '{{.ImageName}}'` returns the digest
reference when the container was started from one, rather than a resolved tag. If it
resolves, the clause-3 check compares the wrong strings and is permanently red — the
lookup is one `podman inspect` against a container started by digest, and it should be
run before this step is committed.

---

# Artifact I — step record: `devbox-identity-markers`

ADR-0017's decision §4 promises three markers; this is what asserts them. It runs
**inside** the container, which is the only place the question makes sense.

**The hostname is asserted by prefix, not by equality, and that is a deliberate
weakening.** Measured today the hostname is `devbox-infra-docs` — podman derives it
from the container name, so it currently carries *more* information than a flat
`devbox` would: it says which project as well as which side. Asserting equality with
`devbox` would fail on that and would push the fix toward discarding the project
component. Asserting the prefix keeps both: the marker is present, and whatever
follows it is free to be useful.

**`/run/.containerenv` is checked alongside the markers, and it is the one that
matters.** `DEVBOX` and the hostname are values this system sets, so they can be set
and still be wrong — an unset marker in an otherwise correct container, or a marker
inherited into something that is not the devbox at all. `/run/.containerenv` is
written by podman and is a live reading of the world, in the same role
`systemd-detect-virt -r` plays for `first-boot`'s chroot boundary. If the markers and
the live reading disagree, the markers are what is wrong.

```yaml
id: devbox-identity-markers
requires: [devbox-image-build]
postcondition: |
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Live reading first: podman writes this, nothing here sets it. Measured
  # present 2026-08-25 in the current container.
  [ -e /run/.containerenv ] \
    || note "/run/.containerenv is absent — this is not a podman container, and the markers below cannot be trusted"
  # Prefix, not equality: podman derives the hostname from the container name,
  # so a project suffix is expected and is worth keeping.
  h=$(hostname 2>/dev/null) || h=
  case "$h" in
    devbox|devbox-*) ;;
    "") note "hostname is unreadable" ;;
    *)  note "hostname is '$h', want devbox or devbox-*" ;;
  esac
  # An unset marker and a marker set to something else are different findings.
  case "${DEVBOX-}" in
    1)  ;;
    "") note "DEVBOX is unset — a shell that did not inherit the container environment, or an image built before ADR-0017" ;;
    *)  note "DEVBOX is '${DEVBOX}', want 1" ;;
  esac
  # The human-facing marker. This is expected to fail until the container's
  # shell configuration exists — archlinux:base ships none — and it fails
  # loudly on purpose rather than being omitted and forgotten.
  case "${PS1-}" in
    *"$h"*) ;;
    "") note "PS1 is unset — no shell configuration in the image (open item, ADR-0017)" ;;
    *)  note "PS1 does not render the hostname, so nothing marks the side for a human" ;;
  esac
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

**This step is expected to be red on the first build**, because the shell
configuration does not exist yet. That is the intended behaviour: ADR-0017 records the
prompt as an open item, and a check that quietly omitted the unfinished third of the
mitigation would let it stay unfinished. `${DEVBOX-}` rather than `${DEVBOX}` because
the postcondition runs under `set -u`, where an unset variable would abort the script
before it could report that the variable is unset — the failure mode
`_schema.md` names, arriving through the shell rather than through a filter.

---

# Nothing in this round executes

**Confirmed and restated as a constraint, not a preference.** No probe from the
network round was run, no image was built, no container was started, and nothing in
the vault was modified — every artifact here lives in the plan file.

- **Probes 4, 4g, 5 and 8 stay unattended-forbidden.** They write nftables rules into
  a netns whose isolation is the thing under test, which makes "run it and see" a
  request to trust the boundary before measuring it. They run with a human watching or
  not at all.
- **`devbox-image-build` does not execute.** The base image is still open, and every
  step record here that names it — `toolchain-parity`, `devbox-ephemeral-writes`,
  `devbox-identity-markers` — is written against an image that does not exist yet.
  That is the normal state for a spec: `_schema.md` is explicit that the value is in
  a claim being writable as a postcondition, not in it having been run.
- **No changes to the current Debian image or wrapper**, per the scope note. Every
  decision above is a constraint on the replacement. Where a measurement of the
  current system appears, it is evidence, not a description of the target.

# Pending the network round

Not written, and deliberately not sketched — probes 4, 4g, 5 and 8 have not run:

- **The egress ADR.** Blocked on 4b/4c (can a `NET_ADMIN` sibling write nftables in a
  rootless pasta netns, and is a `--cap-drop=ALL` sibling genuinely denied), 4d/4f
  (does the drop bite, and does it survive an attempted flush), and 4g (does a folded
  `counter reject` still increment).
- **The pod + egress-guard runbook** — bring-up, teardown, and the
  read → re-run → read counter delta. Blocked on the same, plus probe 4's capture of
  the pod's resolver address, tap interface name and gateway, without which the
  ruleset cannot be written.
- **ADR-0001's headline.** It says rootless podman writes *no* nftables rules. The
  egress work writes them in a container netns. Whether that is an amendment or a
  distinction the ADR already implies is decided by 4-pre/4-post: if the host's `inet
  filter` hash moves, it is neither.

**Probe 8's exit reasoning gains an asymmetry worth recording now**, because it
strengthens the retire branch and it came from the wrapper reading rather than from a
probe: `--with-agent` is **opt-in and pre-checks that the agent actually holds keys**,
while the Dev Containers extension forwards `SSH_AUTH_SOCK` **unconditionally**, with
no working override (ADR-0003, vscode-remote-release#11413). So the two entry points
do not merely differ in whether egress is filtered — they differ in whether the SSH
agent is exposed by default. If `--pod` does not survive `runArgs`, retiring the
devcontainer path removes an unconditional exposure as well as an unfiltered one,
which is a materially stronger case than "the wrapper is easier to guard."

# Open items — later rounds

- **A clause for `_schema.md` convention 1.** The convention prefers `awk` over
  `grep -c` because awk prints `0` rather than exiting non-zero. That is only safe
  where `0` is the *unsafe* answer. The drop-counter diagnostic inverts it: `0`
  routes the operator toward "it's an outage." Roughly *prefer the construct whose
  empty answer fails toward suspicion, not toward all-clear.* Wants a worked case,
  which the egress round supplies.
- **Whether the two-reader cross-check generalises past 4g.** Asserting two
  independent readers agree **and** that both are non-zero caught a class that
  equality alone ratifies. The cost is two readers per diagnostic.
- **A POSIX-only lint for the collector.** ADR-0015 removes the accidental
  cross-implementation testing that caught the `\<` bug;
  `backlog/devbox-contents-audit` named this as the condition attached to installing
  gawk, and it is now due.
- **Re-read `backlog/devbox-contents-audit`'s man-page item.** Its objection was that
  a Debian base reports Debian's versions. On Arch that dissolves, and the
  `/usr/share/man` mount becomes the version-accurate channel 2 the root `CLAUDE.md`
  asks for. It still needs the `mandb`-against-read-only question answered.
- **`~/dev/devbox/devbox` and `Containerfile` are unread by this vault**, and are not
  reachable from inside the container. No artifact depends on them any more, but the
  wrapper is the only component of this system with no copy under version control
  here — the cwd guard's test, `git_args` and the full flag list are all unexamined.
  Mirroring the recipe repo is a task for the next round.
- **The container's shell configuration does not exist.** `archlinux:base` ships
  none, so there is no prompt, no `PS1`, and no human-facing side marker.
  `devbox-identity-markers` is red until it is written, deliberately.
- **`CLAUDE_CODE_PROJECT_DIR_NAME`'s composition is unlooked-up**, and the migration
  target for the orphaned `-work` tree depends on it — whether it replaces the whole
  directory name or one component. One launch with it set, watching what appears
  under `projects/`, answers it. Migrate after, not before.
- **`settings.json` is 0644 with content, and ADR-0006 said to tighten it.** Not
  secret, but the condition that record set has been met and nothing happened.
- **`~/.codex-default/config.toml` is still in the image** — ADR-0007's vestigial
  `COPY`, which was to be struck at the next rebuild. Measured present 2026-08-25, and
  it is now also the stale half of the seed drift ADR-0017 documents.
- **`WebFetch(domain:wiki.archlinux.org)` is still in the project allowlist**, which
  `state/notes.md` records as unreachable behind Anubis. Harmless, and a small worked
  example of an allowlist accumulating entries nobody revisits.
