---
type: index
device: arch-thinkpad
updated: 2026-09-02
tags: [index]
---

# arch-thinkpad

ThinkPad T14 Gen 1 AMD. Arch, KDE Plasma (Wayland), LUKS2 + btrfs + snapper,
systemd-boot UKIs, Secure Boot with own keys, TPM2+PIN, nftables, usbguard,
AppArmor in complain mode, rootless podman, restic, Tailscale.

The desktop was Hyprland + Noctalia until 2026-08-26/27; see
[[decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]]. Records written
before that date may still describe the old userland — the ones known to have been
swept are listed in [[sessions/20260828-desktop-kde-migration|the session log]].

## Contents

| Folder | Contents |
|--------|----------|
| [[decisions/_index\|decisions/]] | 24 ADRs |
| [[backlog/_index\|backlog/]] | 8 open items |
| [[state/_index\|state/]] | Generated inventory and [[areas/infrastructure/arch-thinkpad/state-notes\|operational notes]] |
| collect/ | Collector script, redaction filter, map generator |
| [[devenv/_index\|devenv/]] | Mirror of the devbox recipe — `Containerfile`, `build.sh`, wrapper, Codex config, devcontainer template |
| [[spec/_index\|spec/]] | Build specification — schema settled, five chapters written; `spec/desktop` is named by five ADRs and does not exist |
| [[runbooks/_index\|runbooks/]] | 2 procedures |
| [[sessions/_index\|sessions/]] | Working narrative — dead ends, discovery order, revised conclusions |

## Regenerating state

Run `./collect/collect.sh` from this directory, then `git diff` — the diff is
the changelog. Requires sudo. The filter refuses to run if `redact.map` is
missing, rather than emitting secrets.
