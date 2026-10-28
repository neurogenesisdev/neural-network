---
title: Container storage is a nested subvolume; project data is bind-mounted
type: decision
id: ADR-0006
aliases: [ADR-0006]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: high
created: 2026-08-16
updated: 2026-08-21
tags: [decision, containers, podman, btrfs, snapshots, backups]
constrains: spec/containers
---

# ADR-0006 — Container storage is a nested subvolume; project data is bind-mounted

- **Status:** accepted
- **Date:** 2026-08-16
- **Amended:** 2026-08-21 — added the Claude Code case, a concrete instance
  where upstream guidance conflicts with this rule
- **Provenance:** measured

## Context

Podman's rootless storage tree at `~/.local/share/containers` holds image
layers that are large, numerous, and reproducible. Left inside `@home` it would
be captured by every snapper snapshot and by restic, and it would reproduce the
failure recorded in the operational notes where 142 GB of deletions freed no
space because snapshots pinned the extents.

## Decision

Create `~/.local/share/containers` as a **nested btrfs subvolume**, before
pulling any images. Exclude it from restic.

Bind-mount project data from a normal path under `@home` (`~/dev/<project>`).
**Never use podman named volumes for anything you care about.**

## Rationale

Snapshots do not descend into nested subvolumes. Image layers therefore become
invisible to snapper in both directions — never pinned by `@home` retention,
never reverted by a rollback.

That is correct for images, which are build artifacts and reproducible from a
recipe. It is **wrong** for project data, and named volumes live in the same
unprotected tree. Bind-mounting from `@home` keeps project data inside the
snapshot and backup perimeter, and avoids a second failure: after a rollback,
code reverted while a named volume did not, leaving the two diverged.

Agent state at `~/.local/state/devbox/` is bind-mounted for the same reason.

## Worked example — Claude Code state, 2026-08-21

Anthropic's reference devcontainer configuration mounts a **named volume** at
`~/.claude` and sets `CLAUDE_CONFIG_DIR` to the same path. Following that
guidance on this machine would put Claude Code's session history, per-project
trust, MCP configuration and plugins into `~/.local/share/containers` —
covered by neither snapper nor restic, and liable to diverge from the code
after a rollback.

The devbox setup deviates deliberately: `CLAUDE_CONFIG_DIR=/home/dev/.claude`
in both the `Containerfile` and `devcontainer-template/devcontainer.json`, with
the host side bind-mounted from `~/.local/state/devbox/claude/`. Verified
present on the host: `.claude.json`, `.credentials.json`, `settings.json`,
`history.jsonl`, `sessions/`, `projects/`, `plugins/`, `backups/`.

The non-obvious part, and the reason a naive `~/.claude` mount fails: Claude
Code stores the OAuth account, personal MCP servers and per-project trust in
`.claude.json`, which sits *outside* the config directory by default. Mounting
the directory alone does not keep you signed in. `CLAUDE_CONFIG_DIR` is what
moves `.claude.json` inside the mounted path.

Note `settings.json` is 22 bytes (defaults) and mode 0644 while its neighbours
are 0600. Harmless while empty; tighten if it gains content.

## Consequences

- Container storage is protected by neither snapshots nor restic. Accepted.
- The recipe is what must be preserved: Containerfiles in git, base images
  pinned by digest (ADR-0007).
- `~/.config/containers/` stays on `@home` and **is** covered by restic — note
  `auth.json` holds registry credentials in cleartext.
- File operations on the storage tree require `podman unshare`, not `sudo`.
- Upstream devcontainer guidance for any tool should be read against this rule
  before adoption. Named volumes are the common default and are wrong here.
- For a deliberate export, `podman save --format oci-archive` to
  `~/containers-archive/`, which restic does cover. `skopeo` is the better tool
  once this is routine.
