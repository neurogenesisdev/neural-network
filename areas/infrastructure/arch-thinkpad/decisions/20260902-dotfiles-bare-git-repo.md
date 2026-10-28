---
type: decision
id: ADR-0030
aliases: [ADR-0030]
status: accepted
impact: medium
provenance: measured
device: arch-thinkpad
constrains: backlog/dotfiles-versioning
stale: false
created: 2026-09-02T16:20:00Z
updated: 2026-09-02T17:10:00Z
last_verified: 2026-09-02
commit: pending
supersedes:
superseded_by:
tags: [decision, dotfiles, git, versioning, ssh, kde]
---

# ADR-0030 — Dotfiles are versioned in a bare git repo with an explicit allowlist

## Summary

`~/.dotfiles.git` is a bare repository whose work tree is `$HOME`; only files
added by hand are tracked, and `~/.ssh/config` was the first. No chezmoi, no stow.

## Context

[[../backlog/dotfiles-versioning|dotfiles-versioning]] had been open since
2026-08-21 with three options and no chooser. It stopped being hypothetical on
2026-08-31 when `~/.ssh/config` was overwritten and could not be recovered — no
version existed and the backup timer had never run. The file set had also changed
shape: Plasma writes a long tail of rc files by dialog, so "which of these files
hold decisions?" is now the real question, and any mechanism that tracks a
directory wholesale answers it wrongly.

The operator was undecided and accepted a recommendation; this record exists so the
choice is not re-argued from the same three options next time.

## Decision

Version dotfiles in a bare git repository at `~/.dotfiles.git`, work tree `$HOME`,
`status.showUntrackedFiles=no`, driven by a two-line wrapper `~/.local/bin/dots`.
Files enter the repo only by an explicit `dots add`. Machine-written KDE state is
excluded unless a file carries a decision; the three named by ADR-0018 and ADR-0019
(`kdeglobals`, `kwinrc`, `kwalletrc`) are in from the start.

## Alternatives Considered

1. **chezmoi** — templating and secret integration this machine does not need yet:
   there is one Linux desktop, and the per-device blocks in `~/.ssh/config` are a
   difference between machines that do not share a home. Rejected on weight, not on
   capability; the moment a second workstation needs the same shell config, this
   record should be revisited rather than worked around.
2. **GNU stow** — a symlink farm answers deployment, not history, and moves every
   tracked file out of its path. Solves the smaller half of the problem.

## Consequences

### Positive

- Zero dependencies; the repo is ordinary git and survives any tooling change.
- The allowlist *is* the answer to "which files hold decisions" — the question is
  asked once per file, at `dots add`, instead of being inferred from a diff.
- `~/.ssh/config` now has a history. The 2026-08-31 loss class is closed for every
  tracked file.

### Negative

- Nothing is tracked automatically. A new hand-written file is unversioned until
  someone remembers `dots add`; `dots status` shows nothing for it by design.
- ~~No remote yet.~~ **Amended 2026-09-02, same day:** pushed to a **private**
  GitHub repository, `Luis-Gordon/dotfiles`, on the operator's decision. The file
  set holds full SSH topology and no secrets; private is the condition, and it
  must stay so. The dead-disk loss class is now covered by two independent copies
  (this remote and restic).
- The wrapper lives in the repo it drives. A fresh machine bootstraps with the
  long form once: `git clone --bare <url> ~/.dotfiles.git` then
  `git --git-dir=$HOME/.dotfiles.git --work-tree=$HOME checkout`.

## Verification

`measured` 2026-09-02:

```bash
# arch-thinkpad, as luiswg
dots log --oneline        # c7f4e59 Add dots wrapper / 112c93f Initial import
dots ls-files             # 10 files: .zshrc .zshenv .gitconfig .ssh/config .local/bin/dots
                          #   2 user units, kdeglobals kwinrc kwalletrc
dots status --short       # clean
```

## Amended 2026-09-12 — interaction with AppArmor enforcement

`git` is confined by apparmor.d's `git` profile, and on 2026-09-12 it was found
**enforced** (measured: `apparmor="DENIED" … profile="git"
name="/home/luiswg/.dotfiles.git/HEAD"`). The profile allows ordinary `.git/`
directories inside repositories, not a bare repository at `~/.dotfiles.git`, so
every `dots` command failed with `fatal: not a git repository` — a misleading
message for a MAC denial. Fix: `/etc/apparmor.d/local/git` gains
`owner @{HOME}/.dotfiles.git/{,**} rwkl,` and the profile is reloaded. The
strace tell was `openat(... HEAD) = EACCES` on a 0600 file the user owns.

The work tree needs the same treatment: the profile does not let `git` list
`~/` or read arbitrary dotfiles, so the override also carries `owner @{HOME}/ r,`
and **one line per tracked file**, generated from `dots ls-files`. That keeps
`~/.ssh` keys and everything untracked out of git's reach, at the cost of a new
rule whenever the allowlist grows — the allowlist and the AppArmor override are
now the same list in two places, and adding a dotfile means editing both.

## Revisit when

A second machine needs the same shell configuration (chezmoi's case). Whether a
GitHub-hosted copy of the SSH topology should move to self-hosting belongs with
[[../../netcup-vps/decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]].

## Related

- [[../backlog/dotfiles-versioning|dotfiles-versioning]] — closed by this
- [[20260828-secret-service-provider-is-ksecretd|ADR-0019]], [[20260828-plasma-replaces-hyprland-userland|ADR-0018]] — name the three KDE files tracked
