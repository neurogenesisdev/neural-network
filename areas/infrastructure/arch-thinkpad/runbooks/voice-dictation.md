---
title: Voice dictation with whisrs — setup and the KDE Wayland fixes
type: runbook
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-11-09
commit: pending
created: 2026-11-09
tags: [runbook, dictation, whisrs, groq, kde, wayland]
---

# Voice dictation (whisrs)

**Fires when:** dictation stops working after a rebuild, a KDE or whisrs update, or a new microphone; or when setting the laptop up again from scratch. Set up 2026-11-09; every step below was run on this machine.

## What runs

- `whisrs-git` from the AUR (built 0.2.0). The daemon is the user service `whisrs.service`; the hotkey runs `whisrs toggle`, which talks to the daemon over `/run/user/1000/whisrs.sock`. Press once to record, once more to transcribe and type at the cursor.
- Backend: Groq, model `whisper-large-v3-turbo`. The API key sits in `~/.config/whisrs/config.toml` (0600) and in KeePassXC. Cloud trade-off accepted: every dictation goes to Groq, same as Wispr Flow on the phone.
- Typing goes through uinput (`/dev/uinput`, user in group `input`), because KDE's compositor does not offer the Wayland virtual-keyboard protocol.
- KDE shortcut: System Settings → Keyboard → Shortcuts → Add New → Command `whisrs toggle`, bound to Meta+B. whisrs does not need to know the binding.

## Install

```bash
paru -Syu && paru -S --nocheck whisrs-git      # --nocheck: one self-test fails in a user cgroup, unrelated to dictation
sudo usermod -aG input "$USER"                 # log out/in afterwards
whisrs setup                                   # backend, key, language, mic, service
systemctl --user enable --now whisrs.service
```

## The two things that were not obvious

1. **Microphone.** The ALSA `default` device gave silence (log: `speech none dBFS`, `peak 0.0`). The internal mic is `alsa_input.pci-0000_06_00.6.HiFi__Mic1__source` (`pactl list short sources | grep -v monitor`). Set it with `whisrs config` → audio device.
2. **Keyboard layout.** whisrs builds its virtual keyboard from the system X11 layout, which was `(unset)` on this machine (log: `failed to create XKB keymap for layout '(unset)'`). Fix: `sudo localectl set-x11-keymap us`, then `systemctl --user restart whisrs.service`. An `XKB_DEFAULT_LAYOUT` drop-in on the service did **not** help on its own. The warning may keep appearing in the log after the fix while typing works.

## Verify

```bash
localectl status | grep -i x11            # X11 Layout: us
systemctl --user is-active whisrs.service  # active
journalctl --user -u whisrs -n 20 --no-pager | grep -iE 'speech|transcription'
```

Then Meta+B in any text field, speak, Meta+B again: text appears within about a second.

## Open

- Language is `en`; switch to `auto` in `whisrs config` for German, or bind a second shortcut to `whisrs toggle -l de`.
- KDE exposes no focused-window query, so whisrs cannot restore focus after recording: the text goes wherever focus is when transcription finishes.
