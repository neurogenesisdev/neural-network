---
title: Hyprland IPC is never granted in an enforced AppArmor profile
type: decision
id: ADR-0002
aliases: [ADR-0002]
status: proposed
provenance: inferred
device: arch-thinkpad
stale: false
last_verified: 2026-08-21
commit: pending
impact: high
created: 2026-08-16
tags: [decision, apparmor, hyprland, confinement, security]
constrains: spec/apparmor
---

# ADR-0002 — Hyprland IPC is never granted in an enforced AppArmor profile

- **Status:** proposed
- **Date:** 2026-08-16
- **Provenance:** inferred — reasoned from the compositor's capabilities, not
  demonstrated by an exploit

## Context

`apparmor.d` ships ~2083 profiles. Applications running under Hyprland will
generate denials for the compositor's IPC socket (`hypr/**`) during the
complain-mode observation window, and `aa-logprof` will offer to allow them.

## Decision

Never grant `hypr/**` in any profile placed into enforce mode. This holds
regardless of what the denial log proposes.

## Rationale

The Hyprland IPC socket accepts dispatch commands, including `exec`. A confined
process that can reach it can ask the compositor to run an arbitrary command on
its behalf, and that command runs unconfined. The confinement boundary is then
decorative — anything inside the profile can escape through a socket the
profile itself permitted.

This is a confinement escape by design of the protocol, not a bug to be waited
out.

## Consequences

- Any application that legitimately requires Hyprland IPC cannot be confined by
  an enforced profile. It stays unconfined by construction.
- Noctalia falls into this category as of 2026-08-19: it holds layer-shell
  surfaces, `ext-session-lock-v1`, the system tray, clipboard history, gamma
  control, and Hyprland IPC. It is materially more privileged than the
  waybar/wofi/mako set it replaced.
- Obsidian and the VS Code UI process remain unconfined for a different reason
  — no upstream profile exists.
- The residual exposure list in the state notes must be kept current, because
  it is the honest accounting of what this rule costs.

## Revisit when

Never, unless Hyprland's IPC protocol gains a capability-scoped mode that can
expose read-only queries without `dispatch exec`.
