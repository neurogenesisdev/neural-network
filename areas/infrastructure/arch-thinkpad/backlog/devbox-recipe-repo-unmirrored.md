---
type: backlog-item
status: resolved
priority: high
device: arch-thinkpad
projects: [devenv, vault]
tags: [devbox, containers, vault, provenance]
created: 2026-08-26T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Mirror the devbox recipe repo into this vault

> [!NOTE] Summary
> `~/dev/devbox` — the `Containerfile`, the `devbox` wrapper, `codex-config.toml`, `devcontainer-template/` — has never been read from this vault. It is the only component of this system with no copy under version control here, and it is what blocks [[../spec/devenv|spec/devenv]] from defining `devbox-image-build`.

> [!SUCCESS] Resolved 2026-09-02 — `measured`
> Mirrored into [[../devenv/_index|`arch-thinkpad/devenv/`]] from the working tree
> of `~/dev/devbox` at commit `d40c092` plus one uncommitted change (the wrapper's
> `--replace` and digest pin). **The premise had aged**: the recipe *is* under
> version control — its own repo with a GitHub remote — just not readable from
> here. The four unexamined points are answered in the mirror's index, and
> [[../spec/devenv#devbox-image-build|`devbox-image-build`]] now exists, so the
> spec has no unresolved `requires`. `reference/ssh-config.example` was left out
> on purpose: retired hosts by tailnet address.

## Why this blocks a step rather than merely being untidy

[[../spec/devenv|spec/devenv]] defines three assertions about the agent image and
**not** the build that produces it. That is deliberate. The only description of the
build available to this vault is `_source/Block-C-Dev-Environment.md` §5, a 2026-08-18
transcript — and `_source/` is an input, wrong in known places, and explicitly not a
ground for factual claims per the root `CLAUDE.md`. A `devbox-image-build` step
transcribed from it would be `inherited-unverified` while looking like a specification,
which is the exact failure mode the vault's provenance vocabulary exists to prevent.

So `devbox-image-build` is [[../spec/_index#Unresolved `requires`|the one unresolved
`requires`]] in the spec, and this item is what closes it.

## What is unexamined

Measured 2026-08-25/26 from inside the running container: three bind mounts, `id -u`
1000, `$HOME` `/home/dev`, project at `/work`. That is what the wrapper *produces*.
What it *says* has never been read:

- **the cwd guard** — Block-C calls it "the single most valuable line in the script",
  and its actual test is unknown here;
- **`git_args`** — Block-C §3.7 records it being built and never expanded, caught only
  by a runtime check, so its current state is worth confirming;
- **the full flag list** — five flags are recorded in Block-C's table; whether that is
  the whole set is not established;
- **`--with-agent`'s pre-check** — reported to verify the agent actually holds keys
  before forwarding, which is the asymmetry that makes the wrapper path stronger than
  the devcontainer path. Reported, not read.

## Two findings already rest on a host reading rather than on this vault

Both are recorded as `inherited-unverified` and neither is load-bearing, which was
arranged deliberately so that no artifact depends on an unread file:

- `useradd --uid 1000` as a literal — [[../decisions/20260825-devbox-base-is-arch|ADR-0015]]
  regrounds this as a *requirement* (container UID equals the UID owning host-side
  state) measured from inside, so the literal is corroboration rather than evidence.
- the wrapper's `devbox:latest` short name —
  [[../decisions/20260825-devbox-pinning-four-layers|ADR-0016]] stands on the moving-tag
  argument instead, so this is noted rather than depended on.

## What to do

Mirror the repo into the vault, or vendor the two files that matter into
`arch-thinkpad/devenv/`. The choice is whether this vault holds the recipe or points at
it; either resolves the provenance problem, and pointing at it does not.

## Related

- [[../spec/devenv|spec/devenv]] — the chapter this unblocks
- [[devbox-cleanup-next-rebuild]] — the changes waiting for the next rebuild
- [[devbox-contents-audit]] — what the container holds, measured from inside
