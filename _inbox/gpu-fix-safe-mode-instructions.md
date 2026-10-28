---
title: GPU Fix — Safe Mode Procedure (Phone-Friendly)
created: 2026-04-23
status: active
tags: [gpu-fix, windows, bsod, phaseB]
---

# GPU Fix — Safe Mode Procedure

> You are here because the 0x9F shutdown BSOD diagnosis completed, attribution locked onto the **Intel PEG root port** (the PCIe bridge hosting your NVIDIA GPU), and the next step requires disabling that bridge from Safe Mode. This is **Step B.2** of the approved plan at `C:\Users\luisw\.claude\plans\lexical-painting-dahl.md`.

## What we know
- The **bridge** is at `PCI\VEN_8086&DEV_1901&...`. Its only child is the broken NVIDIA GPU.
- Disabling the bridge stops Windows from sending power IRPs through it at shutdown — which should stop the BSOD.
- Fully reversible. The command to undo is at the bottom of this note.

---

## Before you start

- [ ] **Save any open work.** The shutdown may BSOD one last time on the way down (which is fine — Advanced Startup still activates).
- [ ] Have this note open on your phone. Can't be in Obsidian on Windows if Windows is rebooted.
- [ ] Know that you can always roll back — the Undo command is at the end.

---

## Step 1 — Boot to Safe Mode

1. Open **PowerShell as Administrator** on the laptop.
2. Run:
   ```
   shutdown /r /o /t 0
   ```
3. Windows shuts down (possibly BSODs — OK), reboots into the recovery menu.
4. Navigate: **Troubleshoot → Advanced options → Startup Settings → Restart**.
5. On the Startup Settings screen, press **`4`** for **Safe Mode** (not "with Networking", not "with Command Prompt").
6. Sign in as usual.

---

## Step 2 — Disable the bridge

Once logged in to Safe Mode:

1. Open **PowerShell as Administrator** (right-click Start → Windows Terminal (Admin) or PowerShell (Admin)).
2. **Copy-paste this exact command and hit Enter:**

```
pnputil /disable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"
```

3. Read the output carefully. It will be one of four outcomes:

### ✅ Outcome A — "Device disabled successfully" (no reboot required)
Proceed to Step 3.

### ✅ Outcome B — "Reboot required to complete operation"
The disable is staged but not yet active. From Safe Mode, run:
```
shutdown /r /t 0
```
Boot back to normal Windows (not Safe Mode this time — just let it boot normally). Then proceed to Step 3.

### ⚠️ Outcome C — "Device provides critical system functionality" (or similar refusal)
**Stop. Do not retry with `/force` yet.** Take a photo or copy the exact error text and send it to me. We agreed in the plan that `/force` requires explicit sign-off because the "critical functionality" label is there for a reason. I will assess and confirm before you escalate.

### ❌ Outcome D — "Device not found" or any other error
Copy the exact error text and send it to me. Do not improvise further commands.

---

## Step 3 — Verify the disable took effect

Still in PowerShell (Admin), run:

```
pnputil /enum-devices /instanceid "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"
```

Expected output includes the line:
```
Status:                     Disabled
```

If you see `Status: Disabled` → we're green. Proceed to Step 4.
If you see `Status: Started` (still enabled) → something didn't apply. Send me the output.

---

## Step 4 — Return to normal mode

From Safe Mode:
```
shutdown /r /t 0
```
Let it boot normally — no key presses, just wait for the normal Windows login.

---

## Step 5 — The moment of truth: shutdown test

Once logged in normally:

1. Open PowerShell (Admin).
2. Run:
   ```
   shutdown /s /t 30
   ```
   (You have 30 seconds to cancel with `shutdown /a` if you need to.)
3. Watch what happens. Expected: a clean shutdown, no BSOD, no long hang.
4. Power the laptop back on.
5. Once logged in again, come back to our session and tell me: **did shutdown work cleanly? Any BSOD? How long did it take?**

---

## 🔙 Undo command (use anytime to roll back)

If anything looks wrong at any point, or you just want to get back to the pre-Phase-B state:

```
pnputil /enable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"
```

This returns the system to exactly the state it was in before Step 2. The original BSOD returns too — that's the pre-fix baseline.

---

## Quick-reference commands (copy-paste friendly)

- **Disable bridge:** `pnputil /disable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"`
- **Re-enable bridge:** `pnputil /enable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"`
- **Check status:** `pnputil /enum-devices /instanceid "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"`
- **Reboot:** `shutdown /r /t 0`
- **Test shutdown:** `shutdown /s /t 30`
- **Cancel pending shutdown:** `shutdown /a`

---

## If you run into anything unexpected

Your safety net: **do nothing destructive, run nothing undocumented, and reach out.** Take photos of any unexpected dialogs or error text. The plan file at `C:\Users\luisw\.claude\plans\lexical-painting-dahl.md` has full detail on every branch, and the handoff note `gpu-fix-claude-handoff.md` in this same folder lets a new Claude session pick up wherever you are.
