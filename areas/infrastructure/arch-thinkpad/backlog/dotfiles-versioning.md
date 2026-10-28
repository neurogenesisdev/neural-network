---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [vault]
tags: [dotfiles, chezmoi, versioning, recovery, kde, ssh]
created: 2026-08-21T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Version the dotfiles

> [!NOTE] Summary
> `.zshrc`, `.zshenv`, systemd user units, a growing nvim config and — since 2026-08-26 — the whole of Plasma's configuration are hand-built or hand-adjusted and unversioned. Appendix C.3 called this "the single largest thing a dead disk would cost." Still true, and the shape of the problem changed.
> **It stopped being hypothetical on 2026-08-31**: `~/.ssh/config` was destroyed
> by an overwrite and could not be recovered, because no version of it existed
> anywhere. See below.

> [!SUCCESS] Resolved 2026-09-02 by [[../decisions/20260902-dotfiles-bare-git-repo|ADR-0030]] — `measured`
> Option 2, bare git repo. `~/.dotfiles.git`, work tree `$HOME`, untracked files
> hidden, wrapper `dots`. Ten files in the first two commits: `.zshrc`, `.zshenv`,
> `.gitconfig`, **`.ssh/config`**, `ssh-agent.service`, `udiskie.service`,
> `kdeglobals`, `kwinrc`, `kwalletrc`, and the wrapper itself. Left out on
> purpose: `noctalia.service` (dead unit from the old desktop — delete it, do not
> version it), `plasma-org.kde.plasma.desktop-appletsrc` (machine-written, no
> decision in it), `klipperrc` (does not exist yet). Pushed the same day to a
> private GitHub remote; the ADR records why private is the condition.

## Context

chezmoi is **not installed** — confirmed 2026-08-19 and again by the package collector 2026-08-21. An earlier session recorded an action item assuming it was installed and in scope; that was wrong.

restic covers `/home`, so the files are backed up. What is missing is *version history* and *reproducibility* — knowing what changed and being able to lay the config down on a fresh install.

**Amended 2026-08-28 — the file set changed, and so did the difficulty.** This item originally named `hyprland.lua`, `hyprlock.conf` and the Noctalia state file. None of those exists: the desktop is KDE Plasma ([[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]]). `.zshrc`, `.zshenv`, the nvim config and the user units carry over unchanged.

What replaced them is **larger and machine-written**: `~/.config/plasma*`, `kwinrc`, `kdeglobals`, `klipperrc`, `kwalletrc` and a long tail of KDE rc files, all written by settings dialogs rather than by hand.

That is not merely a longer list — it changes which option is right. A hand-written `hyprland.lua` produces a readable diff where every line was typed deliberately. A GUI-written rc file churns on unrelated interactions, records window geometry and per-session state next to real preferences, and produces diffs nobody can review. **Whatever is chosen has to answer "which of these files hold decisions?"**, and that question did not exist a week ago.

Two of those files also now hold security-relevant configuration — `kwalletrc` and `klipperrc` are named as open questions in [[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] and [[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]] respectively. Versioning them would make those answerable from the vault instead of only from the machine.

## The first real loss, 2026-08-31

`~/.ssh/config` was replaced wholesale with a three-block fragment, destroying
every other host block in it. The immediate cause was a miscommunication — a
fragment presented as if it were the whole file — but **the reason it was
unrecoverable is this item.**

What made it cheap rather than expensive was luck: only `Host github.com` was
lost, and it failed loudly on the next `git push`. Had a rarely-used host been in
there, it would have failed silently weeks later, with nothing to reconstruct the
block from.

Two properties of that file are worth noting for whatever option is chosen:

- **restic holds a copy, not a history — and at the time, not even a copy.** The
  backup timer is enabled but has never taken a backup
  ([[restic-restore-untested]]), so there was no snapshot to restore from. Two
  open items compounding is what turned a mistake into a loss.
- **It holds no secrets but full topology** — every host reached, its user, port,
  and which key. Exactly the kind of file where *version history* matters more
  than *backup*, which is this item's whole argument in one example.

Interim mitigation, adopted the same day and not a substitute: a dated copy at
`~/.ssh/config.bak-<date>`.

**This is the concrete cost the item previously lacked.** The earlier framing was
a hypothetical about a dead disk. The actual first loss came from a two-second
editing mistake on a working machine — a far more frequent event, arguing for the
same fix more strongly.

## Options

1. **chezmoi** — templating for per-machine differences, KeePassXC integration for secrets, `chezmoi status`/`diff` for drift detection. Heaviest, most capable.
2. **Bare git repo** — zero dependencies, no templating, no secret handling.
3. **GNU stow** — symlink farm; simple, no templating.

## Note

`~/.ssh/config` should be in scope for whichever option is chosen. It is
hand-written, it holds decisions rather than machine state, and it is the one file
in this set with a demonstrated loss — which makes it the natural first file to
bring under versioning rather than an afterthought. Its per-device blocks also
differ between machines, so it is a good early test of whether templating
(option 1) is actually needed.

If chezmoi is chosen, scope must be decided against the devbox repo — they do different jobs and should not nest. The devbox repo versions the *recipe* (Containerfile, wrapper, devcontainer template); chezmoi would version the dotfiles that reference it.

Do not carry forward the note about `chezmoi cd` opening a subshell as verified — it is marked `inherited-unverified` in [[state-notes]].
