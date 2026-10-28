---
type: backlog-item
status: active
priority: high
device: arch-thinkpad
projects: [security]
tags: [apparmor, confinement, security]
created: 2026-08-21T00:00:00Z
updated: 2026-09-10T00:00:00Z
commit: pending
---

# Enforce apparmor.d profiles (Stage 2+)

> [!NOTE] Summary
> **2026-09-12:** the `git` profile turned out to be enforced (not only Discord/Spotify) and blocked the bare dotfiles repo (ADR-0030); local override added. Lesson: a `DENIED` from an enforced profile surfaces as the tool's own error text ("not a git repository"), and `dmesg` without root shows nothing — use `sudo journalctl -k | grep DENIED` first.
>
> `apparmor.d` is installed and loaded with ~2083 profiles, but **every one is in complain mode** — nothing is confined. This is the largest remaining security gap on the machine and the reason C.1 item 1 has been open since 2026-08-16.

> [!INFO] Progress 2026-09-02 — `measured`
> **Discord and Spotify are enforced** (`discord`, `discord-chrome-sandbox`,
> `spotify`), verified by launching each in the user session under enforcement
> four times, adding the denials to `/etc/apparmor.d/local/{discord,spotify}`
> between rounds, until a 45-second run logged nothing. Both attach: `ps -o label`
> shows `discord (enforce)` / `spotify (enforce)`. Local overrides survive package
> upgrades; the flags edit does not, so **step 3 is done**:
> `/usr/local/bin/apparmor-reenforce` re-applies the list in
> `/etc/apparmor/enforced-profiles` and exits 1 if any profile is not enforced
> afterwards, run by `/etc/pacman.d/hooks/zz-apparmor-reenforce.hook` after every
> `apparmor.d` install or upgrade, `AbortOnFail`.
>
> **Firefox cannot be enforced from this package.** apparmor.d ships
> `/etc/apparmor.d/firefox` as `flags=(unconfined)` with the comment *"this
> profile allows everything and only exists to give the application a name"*.
> The loaded mode reads `complain` only because the observation window's
> `aa-complain` sweep rewrote the flag. Confining Firefox means writing a profile
> against upstream's judgement, which is a decision, not a step — it is
> **removed from step 2** and left here as an open question.
>
> **Discord's denial list is worth reading once.** The binary that runs is
> `~/.config/discord/app-<ver>/Discord` — the self-updated copy, not
> `/usr/share/discord` — and its voice, utils and erlpack modules are downloaded
> native code under `~/.config` that the profile must allow to `mmap` and, for the
> GPU encoder helper, to execute. The local override grants that with `ix`, so the
> downloaded code inherits the same confinement; it is still downloaded code being
> executed from a config directory, which enforcement makes visible rather than
> creates.
>
> **Core dumps during testing — resolved 2026-09-02, later the same day.** Every
> forced `systemctl --user stop` of a test instance left a core dump (Spotify
> `SIGTRAP`). The operator then opened and closed both apps normally and
> `coredumpctl list --since today` showed nothing new: the dumps were the abrupt
> stops, not enforcement. `measured`.
>
> The measured baseline before this round: 1954 complain, 77 enforce, 61
> unconfined, 107 processes attached. ADR-0002 is unchanged and still `proposed`;
> none of the three enforced profiles references `hypr`. Step 1 (full
> `aa-logprof` review of the other ~1950) has not been done.

> [!INFO] Progress 2026-09-09 — `measured`
> **keepassxc, kde-powerdevil, dolphin and tlp are enforced**, appended to
> `/etc/apparmor/enforced-profiles` and applied with `apparmor-reenforce` (exit 0).
> Rules came from 30 days of complain-mode log, written to
> `/etc/apparmor.d/local/{keepassxc,kde-powerdevil,tlp,tlp_systemctl}`; dolphin's
> only denial (dolphinstaterc link) is already covered upstream, no override.
> keepassxc: the database is in `~/Syncthing`, not `@{user_passwordstore_dirs}`,
> so its rules are mirrored for that directory; `wl-copy` runs `rPx` under its own
> (complain) profile; `ptrace read peer=ksecretd`. Zero `DENIED` in the first
> 20 minutes with all three user processes attached (`ps -o label`). **KeePassXC
> save under enforcement not yet exercised by the operator** — a denial there shows
> as a save error, not data loss. `tlp` could not be verified: `tlp start` hung in
> an ACPI/EC kernel wedge that began 8 minutes *before* enforcement (see notes,
> "AC power_supply reads hang"); re-verify after reboot with
> `tlp start && journalctl -k | grep DENIED`.
>
> **Triaged out:** `git` (profile confines projects to `~/Projects`; this machine
> uses `~/dev`, and root runs git under `/root` and `/var/tmp` — would need the
> `@{XDG_PROJECTS_DIR}` tunable extended in `tunables/xdg-user-dirs.d/site.local`
> plus a decision about root), `kwin_wayland` and `xdg-desktop-portal-kde`
> (session and file-dialog blast radius), `firefox` (unconfined upstream).
> Snapshot 543 precedes the change.

> [!INFO] Progress 2026-09-10 — `measured`
> **kded and git are enforced** (list now 9 profiles + discord's sandbox helper).
> kded's only denial class in 30 days was spawning KIO workers into a null
> profile; `local/kded` hands them to the `kioworker` profile with `rPx`. git
> needed the projects directory: `tunables/xdg-user-dirs.d/site.local` now adds
> `dev` to `@{XDG_PROJECTS_DIR}`, which every profile using
> `@{user_projects_dirs}` inherits on its next reload. `local/git` also covers
> root-run git in the vault and `/root/.claude`; `local/git_ssh` adds the
> ControlMaster socket directory and the user `ssh-agent.socket`. Verified: fetch,
> status and log as luiswg, status as root, kded attached `(enforce)` with 29
> modules loaded, zero denials except one `dac_override` from root git — see
> notes, "root git in the vault": the rule is now to run vault git as luiswg.
>
> **Second-round rules 2026-09-10** for keepassxc, tlp, kde-powerdevil and
> spotify after a day under enforcement (uevent, gdbus, rfkill, cpufreq boost,
> nvme autosuspend, wakeup sources under devices, crash reporter).
>
> **Observation window was being destroyed:** `SystemMaxUse=500M` rotated the
> kernel audit log to under a day. Raised to 2G in
> `/etc/systemd/journald.conf.d/50-size.conf`. Profiles with evidence lost this
> way and deferred to the next batch: `xdg-desktop-portal-kde` (file-chooser
> reads under `~/Downloads`), `kscreen_osd_service`, `systemsettings`,
> `plasma-browser-integration-host`. Re-read the log after a week.

## Context

Block B installed `apparmor.d` (AUR) on 2026-08-16 and put it in complain mode for an observation window. 58 processes attached. The session ended mid-verification of the Firefox profile disable mechanism, before Stage 2.

Container tooling denials from `podman`, `crun` and `conmon` appear in the log from 2026-08-17 onward. These are new noise, not a regression, and must be triaged separately from the Firefox/Discord/Spotify set — enforcement is per-profile and the container denials must not contaminate that decision.

## Steps

1. Complete the observation window; review denials with `aa-logprof`.
2. ~~Enforce Firefox, Discord, Spotify individually.~~ Discord and Spotify done 2026-09-02; Firefox withdrawn — upstream ships it unconfined, see above.
3. ~~Write the post-upgrade re-enforce script~~ Done 2026-09-02 (`apparmor-reenforce` + pacman hook) — **`apparmor.d` package updates silently revert enforced profiles back to complain.** Without this, enforcement decays invisibly.

## Constraints

- **Never grant `hypr/**`** in any enforced profile — see [[20260816-hyprland-ipc-never-granted|ADR-0002]]. That record is `proposed`, not accepted; decide it as part of this work.
- Obsidian and the VS Code UI process stay unconfined — no upstream profile exists.
- **The compositor-IPC constraint has moved and is unresolved (2026-08-28).** This bullet
  previously read "Noctalia is unconfined by construction (requires Hyprland IPC)", which is no
  longer true — the package is uninstalled and the desktop is KDE Plasma
  ([[20260828-plasma-replaces-hyprland-userland|ADR-0018]]). ADR-0002's rule is **not** amended
  here and nothing is added to it. What is recorded is that its premise moves from a socket path
  to the session bus, which AppArmor mediates differently, and that the set it leaves unconfined
  is larger under Plasma than under Hyprland. See
  [[../sessions/20260828-desktop-kde-migration|the 2026-08-28 session log]]; resolving it is part
  of deciding ADR-0002, which is part of this item.

## Related

- [[20260816-hyprland-ipc-never-granted|ADR-0002]]
