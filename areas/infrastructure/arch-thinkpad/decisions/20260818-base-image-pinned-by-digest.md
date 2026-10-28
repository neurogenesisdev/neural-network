---
title: The devbox base image is pinned by digest
type: decision
id: ADR-0007
aliases: [ADR-0007]
status: superseded
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-18
updated: 2026-08-26
tags: [decision, containers, devbox, reproducibility]
constrains: spec/devenv
superseded_by: ADR-0016
---

# ADR-0007 — The devbox base image is pinned by digest, and the image is the version pin

- **Status:** superseded 2026-08-25 by
  [[20260825-devbox-pinning-four-layers|ADR-0016]]
- **Date:** 2026-08-18
- **Provenance:** measured

> **Superseded, not reversed.** ADR-0016 keeps the digest pin unchanged and adds three
> layers around it. This record's judgement that the agent installers "cannot be pinned
> this way without vendoring their installation" was correct on the evidence available
> in August 2026; both installers were measured on 2026-08-25 to accept a version, so
> the evidence changed rather than the reasoning being wrong. The *Revisit when*
> section below named the trigger and it fired.

## Context

The devbox `Containerfile` installs Claude Code and Codex CLI. Both installers
fetch the latest version at build time. Pinning the *Containerfile* therefore
pins nothing about the agent versions — two builds from an identical
Containerfile on different days produce different environments.

## Decision

Pin the base image by **digest**, not tag. Treat the built image, not the
Containerfile, as the version pin for Claude Code and Codex. Record both agent
versions in the commit message on every rebuild.

## Rationale

A tag is a moving reference. A digest is content-addressed and immutable, so
the base layer is reproducible.

The agent installers cannot be pinned this way without vendoring their
installation, which was judged not worth the maintenance. Recording the
resulting versions in the commit message gives an audit trail without that
cost: the image is the artifact, git holds the provenance.

## Consequences

- Rebuilding is a deliberate act with a recorded outcome, not a no-op.
- A rebuild will silently change agent versions unless the commit message is
  written. If the message is skipped, the record is lost — there is no
  automated capture of this.
- The recipe lives in `~/dev/devbox`, pushed to `Luis-Gordon/devbox` (private):
  `Containerfile`, `codex-config.toml`, the `devbox` wrapper, a
  `devcontainer-template/`, and `reference/ssh-config.example`.
- `CODEX_HOME` must be relocated to `~/.codex-state` to avoid a bind-mount
  conflict — Codex's runtime otherwise lives in the directory being mounted.
- A vestigial `COPY` of `~/.codex-default` became unnecessary once the
  `CODEX_HOME` relocation was adopted. Strike it at the next rebuild rather
  than rebuilding for cosmetics.

## Revisit when

Copilot Chat installed a Copilot CLI agent into the container unasked — an
agent with shell access that is not in the Containerfile, will not survive a
rebuild, and is not in git. Disabling it is on the backlog. If more uninvited
tooling appears, the "image is the pin" position needs re-examination, because
it assumes the image contents are a function of the recipe.
