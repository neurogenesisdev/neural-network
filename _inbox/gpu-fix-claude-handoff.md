---
title: GPU Fix — Claude Session Handoff (Post-B.2)
created: 2026-04-23
status: active
tags: [gpu-fix, windows, bsod, phaseB, handoff]
---

# GPU Fix — Claude Session Handoff

> **If you are a fresh Claude session, read this first.** This is a multi-session debugging effort. The canonical plan is at `C:\Users\luisw\.claude\plans\lexical-painting-dahl.md`. Memory index at `C:\Users\luisw\.claude\projects\C--\memory\MEMORY.md`. User's Safe Mode procedure at `C:\Users\luisw\Documents\neural-network\_inbox\gpu-fix-safe-mode-instructions.md`.

## TL;DR of the situation

Dell XPS 15 9570 laptop, Windows 11 Pro, BSODs `0x0000009F` on every shutdown/restart. The user's NVIDIA GTX 1050 Ti shows Error Code 43 and the driver can't reinstall. Two prior Claude sessions investigated:

1. **Session 1** (plan: `async-stirring-goblet.md`) hypothesized broken NVIDIA driver/GPU. Disabled the GPU devnode via `pnputil /disable-device`. Did NOT fix the BSOD.

2. **Session 2** (plan: `lexical-painting-dahl.md`) — the current plan — used WinDbg on all four minidumps. **Attribution locked: the blocked power IRP is at the Intel PEG root port** (`PCI\VEN_8086&DEV_1901&...`), the bridge hosting the GPU. Device stack: `\Driver\pci` → `\Driver\ACPI` → `\Driver\pci` (PDO). Current frame of the blocked IRP sits on `\Driver\ACPI` with `pci!PciBusPowerUpCompletion` as the pending completion routine. Same pattern across all four crash dumps.

## Current state when this file was written

**B.1 verification complete and PASSED:**
- Bridge exists at instance id `PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08`, Status: Started.
- GPU's `DEVPKEY_Device_Parent` matches the bridge exactly.
- Bridge has **exactly one child** — the NVIDIA GPU — via `DEVPKEY_Device_Children`.
- Safety gate cleared. B.2 authorized.

**B.2 staged but not executed** — requires Safe Mode, user is executing manually from `gpu-fix-safe-mode-instructions.md`. The command to run in Safe Mode (admin PowerShell):
```
pnputil /disable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08"
```

## What to do when the user returns

**Step 1: Ask the user what happened.** Specifically:
- Did Safe Mode boot complete?
- What did `pnputil /disable-device` output?
- Did it succeed, refuse (critical-device), or error?
- After reboot to normal mode, did `pnputil /enum-devices /instanceid ...` show `Status: Disabled`?
- Did the `shutdown /s /t 30` test produce a clean shutdown or another 0x9F?

**Step 2: Based on the user's answer, branch per plan Step B.2 / B.3:**

### If disable succeeded + shutdown was clean (Outcome: Phase B success)
1. Verify no new dumps: `Get-ChildItem C:\Windows\Minidump | Sort LastWriteTime -Descending | Select -First 3`.
2. Verify no new Event 41: `Get-WinEvent -FilterHashtable @{LogName='System'; Id=41; StartTime=(Get-Date).AddHours(-1)}`.
3. Update task #7 → completed, #8 → in_progress → completed.
4. Move on to Step B.4 (BIOS discrete-GPU toggle check) if the user wants a more permanent fix, OR Phase C (discuss GPU-recovery options).

### If disable succeeded but shutdown still BSODs
1. The dump should be fresh in `C:\Windows\Minidump\`. Find it with the Get-ChildItem command above.
2. **Re-run Phase A on the new dump.** Symbols are already cached at `C:\ProgramData\Dbg\sym\` so this is fast. Use the WinDbg script pattern from this session:
   - Parse the new dump's Arg2, Arg3, Arg4 from the WER event (Event ID 1001 in System log) or from the dump file itself via `!analyze -v`.
   - Write a new `phaseA_commands_dumpN.txt` in `C:\Users\luisw\dump_analysis\` with those addresses.
   - Launch `DbgX.Shell.exe -z <newdump> -c "$$><<script>" -Q`.
3. Interpret the new PDO:
   - **If same DEV_1901 bridge:** the pnputil disable isn't fully suppressing ACPI interactions. Escalate to Plan Step B.4 (BIOS toggle) or consider `/force`.
   - **If a different PDO:** we have a different offender now. Apply Phase A attribution logic fresh.

### If disable refused with "critical system functionality"
1. Per plan, explicit user sign-off is required before `/force`.
2. Re-run B.1.c to re-verify no unexpected children have appeared between sessions.
3. Ask the user to confirm they understand and accept the risk that `/force` is designed for devices Windows considers critical — and that on this laptop, the only critical thing behind DEV_1901 is an already-broken GPU.
4. Only after explicit sign-off, run: `pnputil /disable-device "PCI\VEN_8086&DEV_1901&SUBSYS_087C1028&REV_07\3&11583659&0&08" /force` (in Safe Mode).

### If any other error
Capture the error text and pause for instructions. Do not improvise.

## Key locations

| Thing | Path |
|---|---|
| Current plan | `C:\Users\luisw\.claude\plans\lexical-painting-dahl.md` |
| Prior plan (superseded) | `C:\Users\luisw\.claude\plans\async-stirring-goblet.md` |
| Memory index | `C:\Users\luisw\.claude\projects\C--\memory\MEMORY.md` |
| Codex-review feedback memory | `C:\Users\luisw\.claude\projects\C--\memory\feedback_codex_plan_reviews.md` |
| User's Safe Mode instructions | `C:\Users\luisw\Documents\neural-network\_inbox\gpu-fix-safe-mode-instructions.md` |
| WinDbg dump analysis logs | `C:\Users\luisw\dump_analysis\dump*.log` |
| WinDbg command scripts used | `C:\Users\luisw\dump_analysis\phaseA_commands*.txt` |
| Minidumps | `C:\Windows\Minidump\042226-*.dmp` |
| Symbol cache | `C:\ProgramData\Dbg\sym\` |
| WinDbg (Store) | `C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2603.20001.0_x64__8wekyb3d8bbwe\DbgX.Shell.exe` |

## Key evidence (from Phase A, all four dumps)

| Dump | Timestamp | Arg2 (PDO) | DeviceInst confirmed |
|---|---|---|---|
| 042226-14656-01.dmp | 20:31 PM 2026-04-22 | 0xffffcb8e4e155360 | `PCI\VEN_8086&DEV_1901&...&08` |
| 042226-22343-01.dmp | 18:47 PM 2026-04-22 | 0xffffe1841517f360 | (same structural stack) |
| 042226-21125-01.dmp | 16:22 PM 2026-04-22 | 0xffff9905db182360 | `PCI\VEN_8086&DEV_1901&...&08` |
| 042226-28656-01.dmp | 11:29 AM 2026-04-22 | 0xffff9d8fc71a1360 | `PCI\VEN_8086&DEV_1901&...&08` |

All four dumps: `Arg1=0x3` (device blocked IRP too long), `FAILURE_BUCKET_ID: 0x9F_3_ACPI_IMAGE_pci.sys`, current frame on `\Driver\ACPI` / `pci!PciBusPowerUpCompletion`.

## Important cautions for the next session

1. **Do not assume the GPU is the root cause.** The dumps only prove the PEG bridge path timed out. Candidate causes include: unresponsive GPU hardware, Dell BIOS 1.42.0 ACPI bug, Windows PEG policy interaction. Phase B mitigates under any of them.
2. **Codex will review plan changes.** User routes plans through Codex. When "codex review:" appears, treat findings as technical claims to verify, not preferences. This session fixed: 0x9F Arg2/3/4 mapping (Arg2=PDO, Arg3=TRIAGE_9F_POWER, Arg4=IRP), DPC watchdog vs power IRP timeout distinction, `!thread <addr>` not bare `!thread`, cross-dump interpretation not an assumption, parent-vs-children direction for safety gate, `/force` requires explicit sign-off.
3. **Plan file is authoritative.** Read it before acting. The executive summary in this handoff is lossy.
4. **Third-party audio drivers (VBCable, Krisp, Bome) are NOT the cause.** Phase A ruled them out. Do not re-investigate unless new evidence emerges.
5. **Do NOT re-run `.reload /f` in WinDbg.** Lazy symbol loading is much faster. Symbols are already cached from Phase A.

## What good looks like

The user returns saying the shutdown test was clean. Tasks 7 and 8 close. Optional discussion of Phase B.4 (BIOS toggle for more durable fix) and Phase C.2 (GPU-recovery options as a menu for future experiments). Case closed barring later regressions.
