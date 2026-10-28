---
type: index
device: arch-thinkpad
provenance: measured
stale: false
last_verified: 2026-09-02
updated: 2026-09-02
tags: [index, devenv, devbox]
---

# devenv — mirror of the devbox recipe

A **copy**, not the source. The recipe lives in `~/dev/devbox`, its own git
repository with a remote at `github.com/neurogenesisdev/devbox`. These files were
copied from that working tree on 2026-09-02, when it stood at commit `d40c092`
**plus one uncommitted change to `devbox`** — `--replace` on the `podman run`
line and the image reference pinned to a digest instead of `devbox:latest`
(ADR-0016's fourth layer). The copy carries that working-tree state, which is what
runs. Closes [[../backlog/devbox-recipe-repo-unmirrored|devbox-recipe-repo-unmirrored]]
and lets [[../spec/devenv#devbox-image-build|`devbox-image-build`]] be written.

| File | What it is |
|------|------------|
| `Containerfile` | The image: Arch base by digest, ALA snapshot by date, agent installers by version, OCI labels — ADR-0015/0016 |
| `build.sh` | Two-stage build that measures `awk`/`grep` in the image and writes them as labels; holds the three version pins |
| `devbox` | The wrapper: cwd guard, git identity pass-through, `--with-agent` pre-check, bind mounts, digest-pinned `podman run` |
| `codex-config.toml` | Seeded into the host state dir on first use; Codex's inner sandbox settings |
| `devcontainer-template/devcontainer.json` | The VS Code Dev Containers counterpart to the wrapper — same mounts and flags, `devbox:latest` by tag |

**Not mirrored:** `reference/ssh-config.example`. It is an old copy of
`~/.ssh/config` naming two retired hosts by tailnet address; it would trip the
redaction gate and records nothing current. `~/.ssh/config` itself is versioned
by ADR-0030.

## What reading the wrapper settled (backlog questions, 2026-09-02)

- **The cwd guard** is a `case "$PROJECT" in "$HOME"/dev/*)` — refuses to run
  unless the working directory is under `~/dev`. Nothing subtler.
- **`git_args`** is built as an array and expanded as `"${git_args[@]}"` — the
  Block-C §3.7 defect is fixed in the current file.
- **The full flag list** is one flag: `--with-agent`. Everything else is fixed.
- **`--with-agent`'s pre-check** is real: `SSH_AUTH_SOCK` must be set and a
  socket, and `ssh-add -l` must succeed, or the wrapper exits before `podman run`.

## Keeping it current

Manual. When `~/dev/devbox` changes, copy again and note the commit here. A
drift check belongs in the collector; not built.
