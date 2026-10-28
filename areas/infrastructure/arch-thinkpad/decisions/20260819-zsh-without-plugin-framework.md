---
title: zsh runs without a plugin framework
type: decision
id: ADR-0010
aliases: [ADR-0010]
status: accepted
provenance: inferred
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: medium
created: 2026-08-19
tags: [decision, shell, zsh, supply-chain]
constrains: spec/shell
---

# ADR-0010 — zsh runs without a plugin framework

- **Status:** accepted
- **Date:** 2026-08-19
- **Provenance:** inferred — the exposure is reasoned from what frameworks
  auto-source, not from an observed compromise

## Context

The login shell moved from bash to zsh (`chsh -s /usr/bin/zsh`, `luiswg` only;
root and all system scripts untouched). The conventional next step is
oh-my-zsh or an equivalent framework.

## Decision

No plugin framework. The two plugins in use are Arch packages, sourced by
absolute path from `/usr/share/zsh/plugins/`.

## Rationale

oh-my-zsh and its peers are a large auto-sourced plugin ecosystem running in an
unconfined host shell with full access to `~/.ssh`, the KeePassXC database and
`~/Sync` — the same exposure class as the unconfined-Electron problem in
ADR-0002, but with a much larger and less scrutinised supply chain.

Arch packages are signed and reviewed. That is a meaningfully different trust
position from `curl | sh` plus a plugin list.

## Consequences

Ordering constraints in `.zshrc`, all of which fail **silently** if violated:

1. `zsh-syntax-highlighting` must be sourced **last** — it wraps the ZLE
   widgets, and anything sourced after it loses highlighting.
2. `atuin` must init **after** `fzf` — both bind `Ctrl+R` and the last one
   wins. Atuin's is the intended one. `--disable-up-arrow` keeps plain `Up` as
   conventional history.
3. History file is `$XDG_STATE_HOME/zsh/history`; the directory is created in
   `.zshrc` because zsh will not create it.

Layout:

- `~/.zshenv` — read by *every* zsh invocation including non-interactive
  scripts. Environment only: `EDITOR`, `PAGER`, `XDG_*`, and `PATH` (with
  `typeset -U path` for de-duplication, which is why `PATH` belongs here rather
  than in `.zshrc`).
- `~/.zshrc` — interactive only.

Tooling initialised from `.zshrc`: `starship`, `zoxide`, `fzf`, `direnv`,
`atuin`. Atuin is local-only — no account, no sync. The sync server would hold
shell history off-machine and is deliberately not enabled.

Rollback is `chsh -s /bin/bash`; `.bashrc` and `.bash_profile` are retained.

## Related

Aliases that change a tool's *interface* rather than its output are a separate
hazard — see the operational notes. `alias diff='difft'` is retained knowingly;
`alias grep='rg'` was intended to be removed and was not.
