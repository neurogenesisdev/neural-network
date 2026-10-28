---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [devenv]
tags: [devbox, containers, shell, identity, arch]
created: 2026-08-26T00:00:00Z
updated: 2026-08-26T00:00:00Z
commit: pending
---

# Write the devbox's shell configuration

> [!NOTE] Summary
> `archlinux:base` ships no shell configuration at all — no `PS1`, no prompt. That is the human-facing half of the identity mitigation [[../decisions/20260825-devbox-base-is-arch|ADR-0015]] needs and does not have, and it is why [[../spec/devenv#devbox-identity-markers|`devbox-identity-markers`]] is red by design.

## Why it is load-bearing rather than cosmetic

ADR-0015 accepts, as its headline negative consequence, that container and host become
hard to tell apart: same distro, same `pacman`, same `awk`. A host command typed in the
container now succeeds against the wrong database instead of failing with
`command not found`. Three things narrow that, and one of them does not exist:

| Mitigation | State |
|---|---|
| `--dbpath /host/pacman` makes the correct invocation look different | in [[../spec/_schema#Conventions\|the host-question rule]] |
| `DEVBOX=1` and `--hostname devbox…` | asserted by `devbox-identity-markers` |
| a prompt that renders the hostname | **does not exist** |

[[../decisions/20260825-harness-state-and-identity|ADR-0017]] then defers path, user and
home parity **partly on this**: taking parity would remove the last three visual tells
while the mitigation for their absence is still unwritten. So this item is one of the
two conditions that unblocks that decision.

## What the step already asserts, and why it asserts the artifact

`devbox-identity-markers` checks that some file in the image sets a `PS1` containing
`\h` or `\H` — the artifact, not `$PS1` in the running shell. A postcondition runs
**non-interactively**, where `PS1` is unset by definition and bash does not source
`.bashrc` at all; Arch's stock `.bashrc` opens with an early return for exactly that
reason. A check reading `$PS1` would have reported *"no shell configuration in the
image"* forever, including on an image whose prompt was perfect.

So the step is red now, goes green when this item is done, and its diagnostic already
names the true cause.

## Open questions for whoever writes it

- **Where it lands.** `/etc/bash.bashrc` is read by interactive non-login shells, which
  is what `podman exec` and `podman run bash` give; `/etc/profile.d/*.sh` is login-only.
  The step greps all three plus `~/.bashrc`, deliberately, because the location is part
  of this decision rather than settled by it.
- **Whether the hostname keeps its project suffix.** Measured 2026-08-25 the hostname is
  `devbox-infra-docs` — podman derives it from the container name, so it already says
  which project as well as which side. The step asserts the `devbox` **prefix** rather
  than equality so that a flat `--hostname devbox` does not become the fix.
- **Whether `DEVBOX=1` belongs in the image or the wrapper.** In the image it survives
  every entry point; in the wrapper it distinguishes the sealed path from the
  devcontainer one. They are different claims.

## Related

- [[../decisions/20260825-devbox-base-is-arch|ADR-0015]] — the consequence this mitigates
- [[../decisions/20260825-harness-state-and-identity|ADR-0017]] — defers parity partly on this
- [[../spec/devenv#devbox-identity-markers|`devbox-identity-markers`]] — the check
- [[devbox-recipe-repo-unmirrored]] — the recipe has to be reachable first
