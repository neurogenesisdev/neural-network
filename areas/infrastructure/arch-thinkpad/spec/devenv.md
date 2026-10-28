---
title: "Devenv — the agent image, its identity, and what it may lose"
type: spec
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-26
commit: pending
created: 2026-08-26
updated: 2026-09-02
tags: [spec, devenv, devbox, containers, agents, parity, identity]
---

# Devenv

The chapter two decision records have named since 2026-08-18 and nothing had written.
ADR-0007's frontmatter said `constrains: spec/devenv` outright; ADR-0006 has two rules
and [[containers|spec/containers]] implements only the first. Read
[[_schema|_schema.md]] first.

**It opened with three assertions and not the thing they assert about.** Until
2026-09-02 `devbox-image-build` — the step that actually builds the agent image from
the `Containerfile` — was **not defined here**, because the recipe repo was not in this
vault and had never been read from it. The recipe is now mirrored in
[[../devenv/_index|`devenv/`]] and the step is defined first below; the paragraphs that
follow describe why it was withheld and still apply to anything read from `_source/`.

That is an unusual shape for a chapter and it is deliberate. The alternative was to
write `devbox-image-build` from `_source/Block-C-Dev-Environment.md`'s 2026-08-18
transcript of the `Containerfile`, which is exactly the kind of claim the root
`CLAUDE.md` forbids: `_source/` is an input, wrong in known places, and a build step
transcribed from it would be `inherited-unverified` while looking like a specification.

Its sources are three decision records —
[[../decisions/20260825-devbox-base-is-arch|ADR-0015]] (Arch base, toolchain parity),
[[../decisions/20260825-devbox-pinning-four-layers|ADR-0016]] (four pins, OCI labels)
and [[../decisions/20260825-harness-state-and-identity|ADR-0017]] (harness state,
identity markers) — and one measurement session inside the running container.

## Where this chapter sits

```
first-boot ──→ collector-run ──→ collect-tool-versions      (spec/base)
 (spec/base)

devbox-image-build ─┬─→ toolchain-parity
                                ├─→ devbox-ephemeral-writes
                                └─→ devbox-identity-markers
```

**Two chains that share a subject and no edges.** `collect-tool-versions` grounds
`state/`; `toolchain-parity` asserts an invariant. An earlier draft joined them, and
the edge was wrong: `toolchain-parity` reads the host live and the image from its
labels and **never opens `state/`**, so the edge encoded an intention rather than a
dependency. `_schema.md` is explicit that `requires` is edges, not narrative, and an
intention in a `requires:` list is worse than no edge — it constrains a topological
sort for a reason that does not exist.

**`devbox-image-build` was this vault's only unresolved `requires`** until 2026-09-02,
and that was the first time [[areas/infrastructure/arch-thinkpad/spec/_index#Unresolved `requires`|the table]] has named `spec/devenv`. It
complained on 2026-08-24 that the measure "measures resolution, not coverage," with
`spec/devenv` as its worked case — named by two ADRs and by no `requires:` line, so the
table could not see it. It can now.

---

## devbox-image-build

Defined 2026-09-02, from the mirrored recipe in [[../devenv/_index|`devenv/`]]
rather than from Block-C's transcript. The build is `build.sh`: a first stage
builds the `Containerfile` with the three version pins as build args, the
image's own `awk` and `grep` are then *measured* by running them, and a second
stage re-labels the same image with those strings so `toolchain-parity` can read
them from labels. The three pins and the base digest live in exactly two files —
`build.sh` (versions, snapshot date) and `Containerfile` (base digest) — which is
ADR-0016's four layers minus the consumer, and the consumer is the wrapper's
digest reference.

**Not executed from this record.** The image on the machine was built by the
same script on 2026-08-26; nothing here re-ran it. What the postcondition checks
is that an image exists whose labels are populated and whose digest is the one
the wrapper pins.

```yaml
id: devbox-image-build
requires: [first-boot]
precondition: |
  set -uo pipefail
  test -x "$HOME/dev/devbox/build.sh" || { echo "recipe repo absent at ~/dev/devbox"; exit 1; }
  podman info >/dev/null 2>&1 || { echo "rootless podman not usable"; exit 1; }
command: |
  set -euo pipefail
  "$HOME/dev/devbox/build.sh"
postcondition: |
  set -uo pipefail
  NS=io.github.luis-gordon.devbox
  pinned=$(grep -oE 'localhost/devbox@sha256:[0-9a-f]{64}' "$HOME/dev/devbox/devbox" | head -1)
  test -n "$pinned" || { echo "RESULT=FAIL wrapper does not pin a digest"; exit 0; }
  have=$(podman image inspect --format '{{.Digest}}' "$pinned" 2>/dev/null) || have=
  test -n "$have" || { echo "RESULT=FAIL pinned image $pinned is not present"; exit 0; }
  for l in awk grep snapshot-date claude-version codex-version base-digest; do
    v=$(podman image inspect --format "{{ index .Labels \"$NS.$l\" }}" "$pinned" 2>/dev/null) || v=
    test -n "$v" || { echo "RESULT=FAIL label $NS.$l is empty"; exit 0; }
  done
  echo "RESULT=PASS image present at the wrapper's pinned digest, six labels populated"
```

*Measured 2026-09-02:* not run. The postcondition is written against the mirrored
files and `podman image inspect --help`; it is `inferred` until executed.

## toolchain-parity

**No `command`, and that is the point.** `_schema.md` records four shapes an absent
`command` covers, and this is the third: a standing invariant that no action
establishes. Nothing an executor runs makes the container's `awk` match the machine's —
the image build does — so this step can only check.

**It reads live on both sides.** `state/` is the record; a postcondition is the
assertion, and `_schema.md` is explicit that the spec records intent while `state/` and
executor output record observation. The host side is read with `readlink -f`; the image
side comes from ADR-0016's labels, so no container has to start and the check is cheap
enough to run often. A check that is expensive to run is a check that stops being run.

**This is the step that would have caught the original bug** — a `\<` escape that
matched nothing under mawk while the machine ran gawk, letting a verification pass
while exercising nothing.

### Two findings, two verdicts, two remedies

**Asserting the full version string would make this check red by default.** The host
line today is `GNU Awk 5.4.1, API 4.1, PMA Avon 8-g1, (GNU MPFR 4.2.2, GNU MP 6.3.0)` —
four independently versioned components, any of which moves on a host `-Syu`. And
ADR-0016 pins the container to a repo snapshot **precisely so that its userland lags
the rolling host**. Under a monthly rebuild ceiling the two strings would differ most of
the time, on a completely correct system. That is the permanently-red shape
[[disk#luks-format|`spec/disk`]] avoids, and a check that is red by default is a check
that stops being read — which would leave the substitution bug uncaught again, by a
different route.

| Finding | Verdict | Remedy |
|---|---|---|
| **Implementation substitution** — mawk for gawk, ugrep for GNU grep | `RESULT=FAIL` | the base image is wrong |
| **Version drift** — same implementation, different build | `DRIFT=<tools>` | a rebuild, or nothing |

Drift is expected under ADR-0016 and is reported, not judged. Substitution is the bug
this step exists for and is a hard failure.

**The implementation token is a policy literal, and that is allowed.** "Assert against
live state, never against constants" forbids values that vary per machine or per
rebuild; `GNU Awk` is neither. It is what the decision requires to be true, in the same
class as [[boot#tpm2-enroll|`tpm2-enroll`]] asserting the PCR set `[0, 2, 7]` as a
literal — the policy *is* the constant, and a check that resolved it at run time would
assert nothing.

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

---

## devbox-ephemeral-writes

`podman diff` reports changes to a container's writable layer. **Bind mounts do not
appear in it** — they are separate mounts, not part of the layer — so its output is
exactly the set of writes that die on the next rebuild, and nothing else.

**This half-closes ADR-0016's named gap.** That record concedes that labels "describe
the recipe's output; they cannot describe what an extension writes into a running
container afterwards," with the Copilot CLI agent as the worked case. A layer diff can.
It does not prevent the write, and it does not see anything written into a bind mount —
hence *half*.

**The parameter's hazard is a wrong target, so the parameter convention's third clause
applies in its original form.** Diffing the wrong container reports clean, and a clean
report is exactly what this step is looking for — the failure would be silent and
reassuring. So `$DEVBOX_CONTAINER` is validated against something the DAG established:
the container's image must be `$DEVBOX_IMAGE`. Same shape as
[[disk#luks-format|`luks-format`]] tying `$LUKSDEV` back to what `disk-partition`
produced, and the opposite of
[[containers#podman-first-pull|`$DEVBOX_IMAGE_DIGEST`]], whose hazard is a moving
reference and whose check is therefore a shape.

**No `command`.** Nothing an executor runs makes this true; a session does, and the step
can only look at the result afterwards.

**Why `awk` and not a `while read` loop**, since the loop is the obvious shape: it needs
its input from somewhere, and both routes are defects here. A heredoc breaks the
[[_schema#Conventions|indentation-survival rule]] outright — a step record is copied out
of a rendered page with its indentation, and a terminator that no longer sits at column
zero either hangs waiting for a delimiter or swallows the rest of the block. A pipe
avoids that and puts the loop in a subshell, where `fail=1` is set and then discarded
when the subshell exits, so the verdict line reports `PASS` over a list of violations it
just printed. `awk` carries the result out on stdout, which neither construct can lose.

```yaml
id: devbox-ephemeral-writes
requires: [devbox-image-build]
postcondition: |
  set -uo pipefail
  : "${DEVBOX_CONTAINER:?DEVBOX_CONTAINER must name a container that has run a normal session}"
  : "${DEVBOX_IMAGE:?DEVBOX_IMAGE must name the built agent image by digest, e.g. localhost/devbox@sha256:<64 hex>}"
  fail=0
  note() { echo "  $1"; fail=1; }
  # Validate against what the DAG established, not against the type. Diffing the
  # wrong container reports clean, and clean is the answer this step is hoping
  # for — the wrong target would be silent and reassuring.
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

*Unverified, with the lookup:* `podman diff`'s output columns and whether it offers
`--format json` — one `podman diff --help`, channel 1. The `awk` above assumes
`<op> <path>`; if a JSON form exists it is the better reader and the regexes go away.

*Also unverified:* that `podman inspect --format '{{.ImageName}}'` returns the digest
reference when the container was started from one, rather than a resolved tag. If it
resolves, the validation compares the wrong strings and is permanently red — the lookup
is one `podman inspect` against a container started by digest, and it should be run
before this step is relied on.

---

## devbox-identity-markers

ADR-0017's fourth decision promises three markers; this asserts them. It runs **inside**
the container, which is the only place the question makes sense.

**The hostname is asserted by prefix, not by equality, and that is a deliberate
weakening.** Measured 2026-08-25 the hostname is `devbox-infra-docs` — podman derives it
from the container name, so it currently carries *more* information than a flat `devbox`
would: it says which project as well as which side. Asserting equality with `devbox`
would fail on that and would push the fix toward discarding the project component.
Asserting the prefix keeps both.

**`/run/.containerenv` is checked alongside the markers, and it is the one that
matters.** `DEVBOX` and the hostname are values this system sets, so they can be set and
still be wrong. `/run/.containerenv` is written by podman and is a live reading of the
world (*measured present 2026-08-25*), in the same role `systemd-detect-virt -r` plays
for [[base#first-boot|`first-boot`]]'s chroot boundary. If the markers and the live
reading disagree, the markers are what is wrong.

**The prompt is asserted as an artifact in the image, not as `PS1` in this shell**, and
the first draft got that wrong in a way worth recording. A postcondition runs
non-interactively. `PS1` is unset in a non-interactive shell **by definition**, and bash
does not source `.bashrc` there — Arch's stock `.bashrc` opens with an early return for
exactly that reason. So a check reading `$PS1` would have reported *"PS1 is unset — no
shell configuration in the image"* permanently, **including on an image whose prompt was
written, correct, and rendering the hostname perfectly**. A permanently-red check is
bad; one whose message names a cause that is not the cause is worse than either half,
because it sends whoever reads it to fix something that is not broken.

The file list is a policy constant in the same class as the implementation token above —
it names where *this image* is built to put its shell configuration, not a value that
varies per machine.

*Deliberately not asserted here:* that the prompt actually renders.
`bash -ic 'echo "$PS1"'` comes closer and is unverified in a non-tty context — it can
emit job-control noise on stderr, which is the `wiki-search`/`stty` problem in
[[../backlog/devbox-wiki-mount|another form]]. Whether a prompt looks right belongs in a
runbook's verification section, confirmed once by eye, not in a postcondition trying to
prove something about a terminal it does not have.

```yaml
id: devbox-identity-markers
requires: [devbox-image-build]
postcondition: |
  set -uo pipefail
  fail=0
  note() { echo "  $1"; fail=1; }
  # Live reading first: podman writes this, nothing here sets it.
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
  # ${DEVBOX-} not ${DEVBOX}: under set -u an unset variable would abort the
  # script before it could report that the variable is unset. An unset marker
  # and a marker set to something else are different findings.
  case "${DEVBOX-}" in
    1)  ;;
    "") note "DEVBOX is unset — a shell that did not inherit the container environment, or an image built before ADR-0017" ;;
    *)  note "DEVBOX is '${DEVBOX}', want 1" ;;
  esac
  # The artifact, not $PS1. A postcondition runs non-interactively, where PS1 is
  # unset by definition and bash does not read .bashrc — reading it here would
  # report "no shell configuration" forever, including after one exists.
  ps1src=$(command grep -lE '^[[:space:]]*(export[[:space:]]+)?PS1=.*\\[hH]' \
             /etc/bash.bashrc /etc/profile.d/*.sh "$HOME/.bashrc" 2>/dev/null | head -1) || ps1src=
  [ -n "$ps1src" ] \
    || note "no shell configuration in the image sets a PS1 containing \\h or \\H — nothing marks the side for a human (open item, ADR-0017)"
  [ "$fail" -eq 0 ] && echo RESULT=PASS || echo RESULT=FAIL
```

**This step is expected red on the first build**, because the shell configuration
genuinely does not exist yet — `archlinux:base` ships none. That is intended: ADR-0017
records the prompt as an open item, and a check that quietly omitted the unfinished
third of the mitigation would let it stay unfinished. It goes green when someone writes
the file, rather than never.

---

## Open against this chapter

- ~~**`devbox-image-build` is undefined.**~~ Defined 2026-09-02 from the mirror in
  [[../devenv/_index|`devenv/`]]; its postcondition has not been executed. The wrapper
  has been read — cwd guard, `git_args`, flag list and agent pre-check are recorded in
  the mirror's index.
- **`devbox-snapshot-freshness` is specified in ADR-0016 and has no step record here.**
  The assertion is written out in that record's consequences; it wants
  `$DEVBOX_MAX_AGE_DAYS` and the same `$DEVBOX_IMAGE` parameter, and it belongs beside
  `toolchain-parity`. Left undefined rather than transcribed, on the same principle as
  `devbox-image-build`.
- **The sync-and-diff mechanism from ADR-0017 §3 has no step.** It is the repair for a
  measured drift — a 214-byte repo copy against a 609-byte live one — and it needs a
  manifest format decided before it can be written as a postcondition.
- **The container's shell configuration does not exist**, so `devbox-identity-markers`
  is red by design until it is written. Named in the backlog rather than left as a
  surprise.
- **Three `inferred` readings are load-bearing and each has a one-command lookup:**
  `podman diff`'s output columns, `podman inspect`'s `.ImageName` under a digest
  reference, and whether `bash -ic` is usable in a non-tty context.
- **Nothing here has been executed.** These three postconditions were written against
  measurements taken inside the running Debian container and against ADR text; the image
  they describe does not exist yet. That is the standing limitation of this spec and it
  is wider here than elsewhere, because the subject is a build that has not happened.
