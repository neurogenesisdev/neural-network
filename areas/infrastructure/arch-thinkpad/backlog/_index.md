---
type: index
device: arch-thinkpad
updated: 2026-09-02
tags: [index, backlog]
---

# Backlog

Everything known to be undone, decomposed from the old state reference §1.3 and
Appendix C, plus findings from the 2026-08-21 collector build. Ordered by
consequence, not by section number.

**Nothing here is required for the machine to work.** This is the list of things
that will bite later or that would make daily use materially better.

## Active Items

| Item | Priority | Area | Summary |
|------|----------|------|---------|
| [[shutdown-hang-black-screen]] | medium | boot | Twice a Plasma shutdown ended at a black screen needing the power button. The journal ends normally, so the hang is after journald stops and hidden by `quiet`. Debug hook installed; next shutdown produces `/shutdown-log.txt` |
| [[apparmor-enforce-stage-2]] | high | security | Was ~2083 profiles all in complain. **2026-09-02: Discord and Spotify enforced** with local overrides, and the post-upgrade re-enforce hook exists. Firefox withdrawn — apparmor.d ships it unconfined on purpose, so confining it is a decision. The other ~1950 are still complain and unreviewed |
| [[restic-restore-untested]] | high | backups | Two repositories hold a current snapshot and **nothing has ever come back out of B2**. Restore untested; the backup timer is enabled but has never taken a backup, and the metered guard's *proceed* branch has never been reached — a guard that has only ever said no is indistinguishable from one that always says no. **2026-09-02: restores from both B2 and T5 verified byte-identical, a manual 5% check passed, and a `$HOME`-less cache defect that would have failed the B2 timer's first run is fixed.** What remains is item 2 — the timer has never got past the metered guard — and the off-host restore |
| [[devbox-shell-configuration]] | medium | devenv | `archlinux:base` ships no shell configuration, so the human-facing half of ADR-0015's identity mitigation does not exist. `devbox-identity-markers` is red by design until it does, and ADR-0017 defers path parity partly on it |
| [[collector-posix-awk-lint]] | medium | vault | ADR-0015 closes the mawk class and removes the accidental cross-implementation testing that caught it. Covers **portability**, not vacuous correctness — both defects this round were valid gawk, and shellcheck is not run against the collector at all |
| [[devcontainer-ipc-bridge-hardening]] | medium | security | `.bashrc` shim unsetting the host IPC bridges. Mitigation, not a boundary — does not stop anything enumerating `/tmp` |
| [[devbox-contents-audit]] | medium | devenv | Container built for one purpose, now used for three. 13 manual packages measured against current use — `awk` is mawk where the target is gawk, and that has already caused a check that passed while testing nothing |
| [[devbox-wiki-mount]] | medium | devenv | Local ArchWiki invisible inside the devbox, so the lookup rule's fallback channel is unreachable where the work happens. Needs the `wiki-search` script as well as the data — its paths are hardcoded |
| [[devbox-cleanup-next-rebuild]] | low | devenv | Four items to fold into the next rebuild — Copilot Chat, vestigial COPY, extension host check, ArchWiki text dump |
| [[codex-manual-fetch-broken]] | low | devenv | The OpenAI Docs manual helper rejects the usual `--help` probe, sending recovery toward source inspection and web search even though its supported invocation works locally; the fallback must be decided with the pending egress round |

## Resolved 2026-09-03

| Item | Outcome |
|------|---------|
| ~~[[seagate-ntfs-migration]]~~ | Stays on NTFS by decision: `ntfs3` handles it, no Windows remains, only `fsck` is lost. usbguard block and dirty flag were the real barriers, both cleared 2026-09-02 |
| ~~[[ssh-key-diversification]]~~ | GitHub split to its own key, three dead machines' keys deleted from the account. Tailnet hosts keep sharing one key, accepted |
| ~~[[luks-second-keyslot]]~~ | Keyslot 1 added, header re-backed-up with all three slots. A first attempt keyed the slot with the clipboard's expired contents — the command line — and was killed and redone; the lesson is in the item. Slot verified with `--test-passphrase` |

## Resolved 2026-09-02

| Item | Outcome |
|------|---------|
| ~~[[undeliberate-listeners-review]]~~ | [[../decisions/20260902-what-listens-and-why\|ADR-0031]] — passim and avahi masked, ollama disabled; sshd and syncthing recorded as the deliberate listeners |
| ~~[[login-path-unreviewed]]~~ | [[../decisions/20260902-plasma-login-path-accepted\|ADR-0032]] — same PAM stack as console login plus the wallet hook; wallet and login are one secret, accepted behind FDE; no autologin |
| ~~[[devbox-recipe-repo-unmirrored]]~~ | Mirrored into `devenv/` with a provenance index; the four unread wrapper questions answered; `devbox-image-build` defined, so the spec has **no unresolved `requires`**. The premise had aged — the recipe has its own GitHub remote |
| ~~[[explain-two-ethernet-controllers]]~~ | Two different chips (RTL8168ep with DASH, RTL8168h) on two root ports with consecutive MACs — onboard, not a dock. Mapping to the RJ45 and the Ethernet extension connector is `inferred`; PSREF not consulted |
| ~~[[collector-stale-file-detection]]~~ | Staging directory, published together only after the last block; one generated `collector-run.md` carries the run marker. Closed by a different shape than proposed, and the failure it describes happened the same day |
| ~~[[collector-desktop-blind-spots]]~~ | Charge thresholds collected; bounded package-presence list with `<absent>`; `ksecretd`/`klipper` were not package names |
| ~~[[restic-exclusion-list-maintenance]]~~ | Part 2 by the collector option: agent state directories listed beside the live exclusion list in `harness.md` |
| ~~[[git-hooks-activation-and-parity]]~~ | Hook refuses to run if its pattern set differs from `redact.sh`; activation documented in `CLAUDE.md` and warned by the collector; still per-clone, accepted |
| ~~[[dotfiles-versioning]]~~ | Resolved by [[../decisions/20260902-dotfiles-bare-git-repo\|ADR-0030]] — bare git repo with an explicit allowlist, `~/.ssh/config` first. Ten files in. Private GitHub remote `neurogenesisdev/dotfiles` |
| ~~[[backup-staleness-alarm]]~~ | Built as a **local last-success marker** written by `restic-b2.service` only on full success, read by a daily user-session timer that raises a critical desktop notification and a failed unit when the marker is missing, older than **7 days** (assumed, not argued), or the monthly integrity check has failed. First run on install fired truthfully: no successful backup has ever been recorded. Spec step `restic-staleness-alarm` |

## Resolved 2026-08-30

| Item | Outcome |
|------|---------|
| ~~[[remote-restic-repository]]~~ | Resolved by [[../decisions/20260830-remote-repository-is-b2\|ADR-0028]] — **Backblaze B2** over the S3 backend, 12.809 GiB stored for about $0.03/month, and [[../decisions/20260818-backup-cadence-open\|ADR-0012]] decided with it. **Two of the item's three options had stopped existing and it did not know**: the Storage Box it recorded as "already owned" was gone, `inherited-unverified` for nine days while being the load-bearing input to the decision it blocked. The two netcup products that turned out to be the real comparison were never in it. Successor items above cover what the round left unverified |

## Resolved 2026-08-28

| Item | Outcome |
|------|---------|
| ~~[[lid-and-hibernation-handling]]~~ | Resolved by the desktop switch — `plasma-powerdevil.service` handles lid, lock and suspend natively and they work with no configuration written. The item was blocked on `hypridle` being gone and Noctalia having no hibernate verb; **both blockers stopped existing** rather than being cleared. Its `logind.conf`/`sleep.conf` proposal is moot — powerdevil owns the policy. **The original symptom, a bag overnight draining flat, has not been re-tested**: suspend on lid close is not hibernation before the battery is gone |

## Resolved 2026-08-25

| Item | Outcome |
|------|---------|
| ~~[[claude-code-history-backup-scope]]~~ | Resolved by [[../decisions/20260825-harness-state-and-identity\|ADR-0017]] — the exclusion narrows to credentials and cache, session history is kept, and [[../decisions/20260818-restic-exclusions\|ADR-0008]] carries the amendment. The item predicted its own residual correctly: an exclude-based list **fails open** as tooling changes, and that is accepted rather than solved |

## Resolved 2026-08-21

| Item | Outcome |
|------|---------|
| ~~Tailscale Mullvad exit nodes unavailable~~ | Resolved — exit nodes now list. `capMap` is still `null`, so the §1.4 diagnostic was wrong; see [[state-notes]] |
| ~~Copy `/tmp/collect.sh` somewhere persistent~~ | Done — the collector lives in `collect/` and is committed |
| ~~Regenerate §7 with the config-map command~~ | Superseded — the config map is generated into `state/` on every run |
| ~~Dangling `.wants` symlinks after waybar/mako removal~~ | Verified clean |
| ~~fcitx5 not autostarted~~ | Was wrong — started by `app-org.fcitx.Fcitx5@autostart.service` |
| ~~Duplicate media-key bind in hyprland.lua~~ | Corrected during the Noctalia migration |

## Deliberately not tracked here

Decisions with a rationale go in [[../decisions/_index|decisions/]]. Diagnostics
whose error message points away from the cause go in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]].
This file is only for work that is undone.
