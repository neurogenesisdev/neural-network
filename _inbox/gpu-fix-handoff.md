# GPU Fix Session Handoff — 2026-04-22

## What happened before this session

My Dell XPS 9570 (Intel UHD 630 + NVIDIA GTX 1050 Ti) started BSODing on every shutdown/restart after installing Womic (virtual audio driver). Womic was uninstalled but the damage persisted. The NVIDIA GPU shows Error Code 43 and the driver cannot be reinstalled.

## Diagnosis completed

- **BSOD**: `0x0000009F` (DRIVER_POWER_STATE_FAILURE) — a device blocks a power IRP during shutdown
- **GPU**: NVIDIA GTX 1050 Ti, instance ID `PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008`
- **Error Code 43** (CM_PROB_FAILED_POST_START) — GPU fails to initialize despite driver appearing present
- **Driver**: DCH package `oem137.inf` (nvdmi.inf v582.28) is installed, `nvlddmkm.sys` exists in DriverStore, but the kernel driver never loads (service STOPPED, exit 1077)
- **SetupAPI.dev.log** confirms driver install succeeds (exit 0) but device immediately enters Error 43 on restart
- **Hypothesis**: GPU hardware/firmware failure (not proven), possibly triggered by Womic crash
- **Intel UHD 630** works fine throughout
- **BIOS**: 1.42.0 (Dec 2024)
- **Two NVIDIA packages in DriverStore**: `oem137.inf` (v582.28) and `oem1.inf` (nvdmsi.inf, v581.57)
- **No Womic remnants** found

## What was just done (Phase 1)

1. **Disabled the NVIDIA GPU devnode** via `pnputil /disable-device` — Status changed from "Problem" to "Disabled"
2. **Disabled NVIDIA services**: `NVDisplay.ContainerLocalSystem` and `nvagent` set to `start=disabled` and stopped
3. **About to test shutdown** — if you're reading this after a reboot, it either worked or didn't:

## What to do next

### If shutdown worked (no BSOD):
- Phase 1 is complete. Proceed to **Phase 2** (attempt GPU recovery).
- Full plan is at `C:\Users\luisw\Desktop\gpu-fix-plan.md` and `C:\Users\luisw\.claude\plans\async-stirring-goblet.md`
- Phase 2 steps in order:
  1. **Step 2.1**: Full power drain (disconnect battery, hold power 30s)
  2. **Step 2.2**: Check Dell support for BIOS update newer than 1.42.0
  3. **Step 2.3**: Re-enable GPU in Safe Mode, reboot, check if Error 43 is gone
  4. **Step 2.4**: (Optional) Try older driver — skip this, go to 2.5
  5. **Step 2.5**: Clean install with fresh NVIDIA download, verify INF matches `DEV_1C8C`

### If shutdown still BSODed:
- The blocked device may not be the NVIDIA GPU
- Need to analyze minidumps with WinDbg: `windbg -z C:\Windows\Minidump\042226-22343-01.dmp` → `!analyze -v`
- This changes the entire direction of the investigation

## Key file locations
- Full approved plan: `C:\Users\luisw\Desktop\gpu-fix-plan.md`
- Plan (Claude internal): `C:\Users\luisw\.claude\plans\async-stirring-goblet.md`
- Minidumps: `C:\Windows\Minidump\`
- SetupAPI log: `C:\Windows\INF\SetupAPI.dev.log`
- Diagnostic scripts: `C:\Users\luisw\AppData\Local\Temp\diag*.ps1`

## Session state
- GPU devnode: DISABLED
- NVIDIA services: DISABLED + STOPPED
- Intel GPU: working normally
- Reviewer (Codex) approved the plan with manual-approval mode
