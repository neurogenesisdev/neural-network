# Block B — Real Confinement via apparmor.d

**Target:** `arch-thinkpad` · **Closes:** State Reference §4.3, Appendix C.1 item 1
**Prerequisite for:** Block C (contained dev environment, C.1 item 2)

Objective: move from *0 processes in enforce mode* to a measured, non-zero count, with Firefox and the Electron apps demonstrably unable to read `~/.ssh` and the KeePassXC database.

Success is a denial you can point at, not a profile count. §4.3 exists because "78 profiles in enforce mode" was mistaken for confinement.

---

## Decisions already made

| Decision | Rationale |
|---|---|
| apparmor.d over hand-written profiles | AppArmor profiles are default-deny whitelists; there is no "block only these paths" mode. Minimum viable AppArmor is a complete profile per app |
| **Non-FSP build** | Full System Policy adds a `default-user` fallback confining every unprofiled binary. On Hyprland that means the compositor stack under GNOME-shaped rules. Non-FSP leaves unprofiled programs unconfined — which is their current state |
| Flatpak deferred | apparmor.d confines native Firefox in place, so KeePassXC native messaging keeps working. Flatpak revisited only for future apps with no profile |
| Default parser cache location | `/var/cache/apparmor`, which sits on the `@cache` subvolume — outside root snapshots. Upstream's `earlypolicy` cache location is an FSP requirement and does not apply |

---

## Stage 0 — Recovery path and escape hatches

Do not skip. AppArmor policy loads at boot; a bad state that survives reboot needs a way out that does not depend on the running system.

### 0.1 Confirm recovery paths exist

```bash
# Arch (arch-thinkpad)
sudo bootctl list | grep -E 'title|id'
```

Expect four UKI entries: `arch-linux`, `arch-linux-fallback`, `arch-linux-lts`, `arch-linux-lts-fallback`.

Note: the root account is locked (State Ref §7.3), so `rescue.target` and `emergency.target` are unavailable — `sulogin` refuses a locked account. Recovery is the fallback UKI, the LTS entry, or the signed Arch rescue medium (§8.3).

### 0.2 The escape hatch — memorise this

Unloads all policy immediately. No reboot, no UKI rebuild, no BIOS interaction.

```bash
sudo aa-teardown
```

Works from any TTY as long as you can get a shell. If the graphical session is the thing that broke, `Ctrl+Alt+F2` for a TTY, log in, run it, then `sudo systemctl restart apparmor` when you want policy back.

Escalation ladder if `aa-teardown` is not reachable:

| Situation | Action |
|---|---|
| Session broken, TTY works | `sudo aa-teardown` |
| System boots, no login possible | Boot `arch-linux-lts` — same policy, but a different kernel rules out kernel-side interaction |
| Policy load hangs or panics at boot | Boot `arch-linux-fallback` |
| Still unbootable | Arch rescue medium → `arch-chroot` → `pacman -R apparmor.d` |

### 0.3 Snapshot and back up existing policy

`snap-pac` will create pre/post snapshots around the pacman transaction automatically, but the `parser.conf` edit in Stage 1 is not a pacman operation.

```bash
# Arch (arch-thinkpad)
sudo snapper -c root create -d "pre apparmor.d — Block B stage 0"
sudo cp -a /etc/apparmor.d /root/apparmor.d.stock.$(date +%Y%m%d)
sudo aa-status > ~/aa-status-before.txt
```

Keep `aa-status-before.txt` — Stage 4 diffs against it.

---

## Stage 0b — Container storage subvolume

Do this now, before Block C pulls images. `~/.local/share/containers` currently holds ~10 MB (the alpine test pull); it will reach several GB. Under hourly `@home` snapshots every discarded image layer stays pinned until retention expires — the same mechanism that cost 142 GB in §9.

```bash
# Arch (arch-thinkpad)
mv ~/.local/share/containers ~/.local/share/containers.old
btrfs subvolume create ~/.local/share/containers
cp -a ~/.local/share/containers.old/. ~/.local/share/containers/
rm -rf ~/.local/share/containers.old
```

Verify — the new subvolume must appear, and podman must still see the alpine image:

```bash
sudo btrfs subvolume list / | grep containers
podman images
```

Nested subvolumes are not captured by the parent's snapshot, so this is now outside snapper entirely. That also means it is **not** protected by rollback, which is correct: container images are reproducible.

**Restic ripple (§8):** add `~/.local/share/containers` to the exclude list. When Flatpak eventually lands, exclude `~/.local/share/flatpak` but **not** `~/.var/app` — the latter is per-app data (browser profiles, vaults), the former is redownloadable runtimes. The intuitive glob catches both; write them as separate lines.

---

## Stage 1 — Parser cache and install

### 1.1 Parser cache

apparmor.d is roughly 100k rules. Without caching, every boot recompiles the lot.

```bash
# Arch (arch-thinkpad)
printf 'write-cache\nOptimize=compress-fast\n' | sudo tee -a /etc/apparmor/parser.conf
cat /etc/apparmor/parser.conf
```

Deliberately **not** setting `cache-loc` — the default `/var/cache/apparmor` lives on `@cache`, keeping compiled policy out of root snapshots. Upstream's `earlypolicy` location exists for FSP's initramfs policy load, which we are not doing.

### 1.2 Install

```bash
# Arch (arch-thinkpad)
paru -Ss apparmor.d
```

Pick the stable package over any `-git` variant. **Before confirming the build, read the PKGBUILD** — `paru` shows it by default. Two things to check:

1. The build target is the standard one, **not** `--full` / FSP. If the PKGBUILD offers a choice, take the default.
2. Whether it declares `conflicts`/`replaces` against files owned by `apparmor`. File conflicts in `/etc/apparmor.d/` are expected and normally handled; if pacman refuses with a conflict list, stop and report it rather than forcing with `--overwrite`.

```bash
paru -S apparmor.d
```

Build is a Go project and takes a few minutes. Expect ~1500 profiles installed.

### 1.3 Confirm the Hyprland stack is untouched

```bash
# Arch (arch-thinkpad)
ls /etc/apparmor.d/ | grep -iE 'hypr|waybar|mako|swaybg|wlsunset'
```

**Empty output is the expected and desired result** — the compositor stack has no profiles, therefore stays unconfined, therefore keeps working exactly as today.

Non-empty means apparmor.d has grown Hyprland support. Do not treat those profiles as trusted on sight; they go into complain mode with everything else in Stage 2 and get the same five days of observation.

### 1.4 Verify the Firefox profile is real

The stock Arch profile was `flags=(unconfined)` — a name badge, not a policy. The replacement should be neither.

```bash
grep -c 'flags=(unconfined)' /etc/apparmor.d/firefox
grep -rl 'private-files-strict' /etc/apparmor.d/firefox*
```

First command should return `0`. Second should name the profile — `abstractions/private-files-strict` is the upstream abstraction denying `~/.ssh`, `~/.gnupg`, wallets and keyrings, and it is the mechanism that closes §4.3.

If `private-files-strict` is absent, the profile may rely on a plain whitelist instead. Not a failure, but Stage 4's test becomes the only proof — do not skip it.

---

## Stage 2 — Complain mode, five days

Complain mode logs what *would* have been denied and allows it. This is the observation window; nothing is enforced yet.

### 2.1 Set everything to complain

```bash
# Arch (arch-thinkpad)
sudo find /etc/apparmor.d -maxdepth 1 -type f -exec aa-complain {} +
sudo systemctl restart apparmor
sudo aa-status | head -5
```

Expect a large *profiles in complain mode* count and **0 in enforce mode**.

### 2.2 Reboot and use the machine normally

Reboot so policy loads the way it will in production. Then use the system as you actually do for ~5 days. Deliberately exercise the awkward paths at least once each, because they are where the GNOME-tuned profiles will miss on Hyprland:

- Copy and paste between Firefox and a terminal (`wl-clipboard`)
- Screen share or screenshot from a browser (portal + `grim`/`slurp`)
- File upload and download dialogs (portal file chooser)
- KeePassXC browser autofill on a site
- KeePassXC SSH agent serving a key to `git push`
- Syncthing reconciling a change from the Pixel
- A `podman run` and a `virsh` VM start
- Mullvad connect/disconnect, Tailscale up/down
- Suspend and resume, lid close, `loginctl lock-session`

### 2.3 Review denials

```bash
# Arch (arch-thinkpad)
# Everything since boot, grouped by profile
sudo journalctl -b | grep -oP 'apparmor="[A-Z]+".*?profile="\K[^"]+' | sort | uniq -c | sort -rn

# Full detail for one profile
sudo journalctl -b | grep 'profile="firefox"' | grep DENIED

# Interactive review — proposes rules, writes to /etc/apparmor.d/local/
sudo aa-logprof
```

`aa-logprof` is the tool, but **do not let it write rules unreviewed.** Read §2.4 before running it.

### 2.4 The one rule that must never be granted

You will see denials on `@{run}/user/@{uid}/hypr/**` or `hyprctl`.

Any process that can write the Hyprland IPC socket can issue `hyprctl dispatch exec <anything>`, and the spawned process is a child of Hyprland — inheriting Hyprland's **unconfined** state, not the caller's profile. Granting that access to a confined app converts every enforced profile on this system into decoration.

**Standing rule:**

- `hypr/` denials are handled case by case, never with a blanket allow
- Never copy a `hypr/` rule between local override files
- Firefox, Discord, Obsidian, VS Code and Spotify never legitimately need it
- Legitimate needs are limited to waybar modules calling `hyprctl` and your own scripts

If `aa-logprof` proposes it, answer **Deny**, not Allow.

### 2.5 Additive hardening the shipped profiles will not have

The profiles were tuned on GNOME and KDE. Explicit `deny` rules override allows granted by included abstractions, so these tighten things the upstream authors never considered. Create per app:

```
# /etc/apparmor.d/local/firefox
# also: discord, obsidian, code, spotify

deny owner @{HOME}/.config/hypr/** w,
deny owner @{HOME}/.config/waybar/** w,
deny owner @{HOME}/.config/systemd/user/** w,
deny owner @{HOME}/.local/bin/** w,
deny owner @{HOME}/.bashrc w,
deny owner @{HOME}/.bash_profile w,
deny owner @{run}/user/@{uid}/hypr/** rw,
```

`hyprland.lua` is executable configuration with `hl.dsp.exec_cmd` available — write access to it is arbitrary code execution at next login, no exploit required. On GNOME the equivalent is a dconf database the shipped profiles already handle; here it is a Lua file nobody upstream has considered.

Local overrides survive package updates. Reload after editing:

```bash
sudo systemctl reload apparmor
```

---

## Stage 3 — Selective enforce

Only after the complain log is quiet for the apps concerned.

```bash
# Arch (arch-thinkpad)
for p in firefox discord obsidian code spotify syncthing qbittorrent ollama; do
  [ -f "/etc/apparmor.d/$p" ] && sudo aa-enforce "/etc/apparmor.d/$p"
done
sudo systemctl reload apparmor
sudo aa-status | head -5
```

Profile filenames may differ (`code` vs `visual-studio-code`, `discord` vs `Discord`). Check with `ls /etc/apparmor.d/ | grep -i <app>` and adjust.

Then launch each app and exercise it. Expect breakage here — that is the point of doing eight apps rather than all 1500.

**Per-app rollback**, if something breaks and you need it working now:

```bash
sudo aa-complain /etc/apparmor.d/firefox && sudo systemctl reload apparmor
```

Back to logging-only for that app alone, no effect on the others.

### Known interaction risks

| Component | Symptom | First response |
|---|---|---|
| Rootless podman | Container fails to start | `podman run --security-opt apparmor=unconfined` to confirm cause, then fix the profile |
| tailscaled / mullvad-daemon | Looks like a network outage, not a policy failure | Check `journalctl -b \| grep DENIED` **before** troubleshooting the VPN. §5 notes already make VPN state confusing |
| usbguard | USB device stops appearing | §9 already says usbguard is the first suspect; AppArmor is now the second |
| Clipboard / screen share | "Paste stopped working in Discord" — silent, no error | Wayland-under-AppArmor classic. Grep DENIED first |

Standing rule for Stage 3 onward: **any weird behaviour → `sudo journalctl -b | grep -i DENIED` before anything else.**

---

## Stage 4 — Verify the objective

This is the stage that justifies the block.

### 4.1 Confinement is actually happening

```bash
sudo aa-status | head -6
diff <(head -6 ~/aa-status-before.txt) <(sudo aa-status | head -6)
```

*Processes in enforce mode* must be greater than 0. Before this block it was 0 with 78 profiles loaded — that gap is the whole reason §4.3 exists.

### 4.2 Firefox cannot reach the credentials

`aa-exec` runs a command under a named profile without launching the app:

```bash
# Arch (arch-thinkpad)
aa-exec -p firefox -- cat ~/.ssh/id_ed25519
aa-exec -p firefox -- ls ~/.ssh/
aa-exec -p firefox -- head -c 32 /path/to/your.kdbx
```

Interpreting the result:

| Outcome | Meaning |
|---|---|
| `Permission denied` + a DENIED line naming `profile="firefox"` | **Pass.** The objective is met |
| Denial on executing `cat` itself | **Pass, weaker evidence.** The profile is enforcing and does not permit arbitrary exec. Confirm the specific path with `sudo apparmor_parser -p /etc/apparmor.d/firefox \| grep -i ssh` |
| File contents printed | **Fail.** The profile permits it. Do not proceed to Stage 5 — investigate whether the profile actually attached |

Confirm the denial appeared:

```bash
sudo journalctl -b | grep 'profile="firefox"' | grep -i 'ssh\|denied' | tail
```

### 4.3 Record it

Capture the passing output. The `aa-status` figure and one denial line are what the State Reference §4.3 rewrite is built on.

---

## Stage 5 — Expand

Only after Stage 4 passes and the eight apps have been stable for a week.

Broaden enforce to daemons with genuine input surface first: `NetworkManager`, `bluetoothd`, `pipewire`, `dbus-daemon`, `polkitd`, `libvirtd`. Take them in small batches, not all at once — a failure in a batch of three is diagnosable, a failure in a batch of forty is a bisect.

Leave unconfined, deliberately:

- The entire Hyprland stack (unprofiled anyway)
- Anything you installed to `~/.local/bin`, `/opt`, or via `pipx` / `npm -g` — no attachment path matches, so no profile applies. Containment there comes from Block C's podman work, not from policy

---

## Residual exposure — stated honestly

Not closed by this block, in any configuration:

- **Portal-mediated screen capture.** `xdg-desktop-portal-hyprland` and pipewire stay unconfined; an app that obtains portal consent can capture the screen. This is a consent-model gap, not an AppArmor-solvable one. The control is not clicking through share prompts
- **Unprofiled userland.** Anything without a matching attachment path runs exactly as it does today
- **KeePassXC socket.** A compromised confined browser can still speak the browser-integration protocol. Unchanged from the current state; the control is per-site association plus the confirmation prompt
- **apparmor.d maturity.** Upstream describes the project as early development. You are a tester. `aa-teardown` is the reason that is acceptable

---

## Documentation updates on completion

| Document | Change |
|---|---|
| State Ref §4.3 | Rewrite. Title stops being "AppArmor is not confining anything". Record the new `aa-status` figures and the Stage 4 denial |
| State Ref §6.2 | Add `apparmor.d` to AUR/foreign packages with version |
| State Ref §7.1 | Add `/etc/apparmor/parser.conf` and `/etc/apparmor.d/local/*` to the config map |
| State Ref §8 | Restic excludes: `~/.local/share/containers` added |
| State Ref §3.2 | New nested subvolume at `~/.local/share/containers` |
| State Ref §9 | New operational note: the `hypr/` IPC socket rule from §2.4, plus "DENIED before troubleshooting" |
| State Ref Appendix B | Add `sudo aa-teardown`, `aa-exec -p <profile> -- <cmd>`, the DENIED grep |
| State Ref Appendix C.1 | Close item 1. Item 2 becomes Block C |

---

## Order of operations

```
0.1 → 0.2 → 0.3 → 0b → 1.1 → 1.2 → 1.3 → 1.4 → 2.1 → reboot
    → 5 days normal use → 2.3 → 2.4/2.5 → 3 → 4 → 1 week → 5
```

Stop and report at three points: after 1.2 if the build or file conflicts look wrong, after 1.3 if the Hyprland grep is non-empty, and after 4.2 whatever the result.
