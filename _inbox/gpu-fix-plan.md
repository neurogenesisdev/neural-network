# Diagnosis & Fix: Dell XPS 9570 Shutdown BSOD + NVIDIA GPU Failure

## Context

After installing Womic (virtual audio driver), the system crashed into a boot loop. The user recovered by uninstalling Womic and its drivers. The system now **BSODs on every shutdown/restart attempt**. The NVIDIA GPU shows Error Code 43, and reinstalling the NVIDIA driver has failed through multiple methods (NVIDIA app, Driver Easy, manual download, DDU in Safe Mode).

## Diagnostic Findings

### What we confirmed
- **BSOD code is `0x0000009F` (DRIVER_POWER_STATE_FAILURE)**, parameter 0x3 = "a device object blocked a power IRP too long". Occurs during every shutdown/restart.
- **NVIDIA GTX 1050 Ti** is visible on the PCIe bus (`PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1`)
- **Error Code 43** (CM_PROB_FAILED_POST_START) on the NVIDIA GPU — the only device reporting a problem
- **A DCH driver package appears present**: `oem137.inf` (original: `nvdmi.inf`), version 582.28, `PnpLockdown=1`. The kernel binary `nvlddmkm.sys` (110MB) exists at the DriverStore path referenced by the service ImagePath. SetupAPI.dev.log shows installation exiting with status 0x00000000 — but the device immediately enters Error 43 on restart, so the install may not be fully healthy.
- **`nvlddmkm` service is STOPPED**, exit code 1077 (`ERROR_SERVICE_NEVER_STARTED`) — the kernel driver has never successfully loaded in the current session
- **Intel UHD 630** is working fine (Status: Started)
- **BIOS version**: 1.42.0 (Dec 2024), system: Dell XPS 15 9570
- **No Womic remnants** found in the DriverStore
- **5 minidump files** from recent BSODs: `C:\Windows\Minidump\042226-22343-01.dmp` (latest)
- **Two NVIDIA display packages** in the driver store: `oem137.inf` (nvdmi.inf, v582.28, Jan 2026) and `oem1.inf` (nvdmsi.inf, v581.57, Oct 2025)

### What we have NOT confirmed
- Whether the blocked device stack in the BSOD is specifically the NVIDIA PDO/FDO (the minidumps have not been analyzed)
- Whether the GPU hardware is functional at the PCIe/firmware level
- Whether the driver package is truly intact vs. partially corrupted despite SetupAPI success

### Hypothesis (not proven)
The GPU hardware or firmware is likely failing to initialize, despite the driver package appearing present. Possible causes:
1. **GPU hardware failure** — common on XPS 9570 (thermal BGA solder joints). The Womic crash may have been coincidental or a final trigger.
2. **GPU VBIOS/firmware corruption** — the system crash may have corrupted the GPU's firmware during a power transition.
3. **PCIe link or power delivery issue** — physical connection to the discrete GPU is damaged.
4. **ACPI/power management problem** — system ACPI tables governing GPU power states may be damaged (BIOS update could help).
5. **Driver package corruption** — the package appears present but may have subtle corruption not caught by SetupAPI.

---

## Phase 1: Stop the Shutdown BSODs

**Goal:** Restore normal shutdown by disabling the GPU device so Windows stops sending power IRPs to it.

**Why Safe Mode:** The BSOD occurs during power state transitions. Disabling the device in normal mode may not succeed cleanly if the broken driver stack is partially loaded.

### Step 1.1 — Boot into Safe Mode
```
shutdown /r /o /t 0
→ Troubleshoot → Advanced Options → Startup Settings → Restart → Press 4 (Safe Mode)
```

### Step 1.2 — Disable the NVIDIA GPU device
Run as Administrator:
```powershell
pnputil /disable-device "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"
```
This is the main lever — disabling the devnode should prevent Windows from starting the device or managing its power state in subsequent boots, which in turn should stop the 0x9F BSOD on shutdown. This is not an absolute guarantee (a disabled devnode is still present), but it is the standard first step and works in most cases.

### Step 1.3 — (Optional) Disable NVIDIA services
These are user/service-layer components, not the kernel display miniport, so they are unlikely to be on the 0x9F shutdown path. Disabling them is a secondary cleanup, not the primary fix:
```powershell
sc.exe config NVDisplay.ContainerLocalSystem start=disabled
sc.exe config nvagent start=disabled
```

### Step 1.4 — Reboot normally and test shutdown
```powershell
shutdown /s /t 30
```
(Cancel with `shutdown /a` if needed.)

- **If shutdown succeeds**: the GPU device was on the blocked path. Proceed to Phase 2.
- **If shutdown still BSODs**: the blocked device may not be the NVIDIA GPU. Proceed to **Phase 1.5** to analyze the minidump before any further driver changes.

### Step 1.5 — Minidump analysis (only if Phase 1.4 still BSODs)
Install WinDbg (from Microsoft Store or Windows SDK), then:
```
windbg -z C:\Windows\Minidump\042226-22343-01.dmp
```
In WinDbg:
```
!analyze -v
```
This will identify the exact device object and driver that blocked the power IRP. If it's not `nvlddmkm`, the plan needs to change direction entirely.

---

## Phase 2: Attempt to Restore the GPU

**Goal:** Determine whether the GPU can be recovered, starting from least-invasive steps.

### Step 2.1 — Full power drain (reset GPU hardware state)

**Preferred method** (true power removal — fully drains standby power from dGPU path):
1. Shut down the laptop completely (should work after Phase 1)
2. Unplug the AC adapter
3. Open the bottom panel (T5 Torx) and disconnect the internal battery cable
4. Hold the power button for 30 seconds
5. Wait another 30 seconds
6. Reconnect battery cable, close panel, reconnect AC, boot up

**Best-effort alternative** (if you cannot open the chassis — this only triggers an EC reset and may not fully remove standby power from the dGPU):
1. Shut down, unplug AC adapter
2. Hold the power button for 45 seconds
3. Reconnect AC, boot up

### Step 2.2 — Check for BIOS update
Current BIOS: **1.42.0** (Dec 2024). Check Dell support for XPS 15 9570 — a newer BIOS may fix ACPI tables or GPU initialization issues. **Note**: whether a "disable discrete GPU" BIOS option exists is firmware-version-dependent. Not all XPS 9570 BIOS versions expose this.

### Step 2.3 — Re-enable GPU and test
Boot into Safe Mode. Run as Administrator:
```powershell
pnputil /enable-device "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"

# Re-enable services
sc.exe config NVDisplay.ContainerLocalSystem start=auto
sc.exe config nvagent start=demand
```
Reboot normally. Check device status:
```powershell
pnputil /enum-devices /instanceid "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"
```
- **If Status = Started** (no Problem Code): GPU is recovered. Test shutdown.
- **If Status = Problem (Code 43)**: proceed to Step 2.4.

### Step 2.4 — (Optional experiment) Try the older driver version
The DriverStore contains a second NVIDIA package (`oem1.inf` / `nvdmsi.inf`, v581.57). It has not been verified to match this GPU's hardware ID (`PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028`), so this step may not produce a working bind. If it fails or you want to skip it, proceed directly to Step 2.5 (the verified fresh-install path).

Boot into Safe Mode. **Before starting**, reduce the chance of automatic driver rebinding during cleanup:
```powershell
# Discourage auto-reinstall during hardware re-enumeration.
# NOTE: This setting mainly affects Windows Update search behavior and is NOT
# a hard guarantee against PnP reusing packages already staged in the DriverStore.
# Deletion may still fail or rebind may still occur. If it does, re-enter Safe Mode and retry.
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\DriverSearching" /v SearchOrderConfig /t REG_DWORD /d 0 /f
```

Then uninstall the device and swap to the older package:
```powershell
# Uninstall device (removes the devnode)
pnputil /remove-device "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"

# Remove the newer driver package. This may fail if PnP has already re-bound it.
# If it fails, retry the remove-device + delete-driver sequence.
pnputil /delete-driver oem137.inf /force

# Verify it's gone, and oem1.inf (older, nvdmsi.inf v581.57) remains
pnputil /enum-drivers

# Re-enable auto-driver search
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\DriverSearching" /v SearchOrderConfig /t REG_DWORD /d 1 /f

# Trigger re-enumeration — Windows should find the GPU and bind the remaining older package
pnputil /scan-devices
```
Check device status. If still Error 43, the driver version is not the issue.

**If this makes things worse** (e.g., display resolution drops or Intel adapter is affected): there is no clean rollback to the previous driver state. The forward path is Step 2.5 — download a fresh NVIDIA package and install it cleanly.

### Step 2.5 — Clean install with fresh download
Boot into Safe Mode. Block auto-install as in Step 2.4, then:
```powershell
# Remove the device
pnputil /remove-device "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"

# Remove ALL NVIDIA display driver packages
pnputil /delete-driver oem137.inf /force
pnputil /delete-driver oem1.inf /force

# Verify clean state
pnputil /enum-drivers
```

Download the NVIDIA driver for GTX 1050 Ti from nvidia.com. Extract with 7-Zip (don't run the installer). Then **find and verify the correct INF** before installing:
```powershell
# Search ALL .inf files in the extracted folder for this GPU's hardware ID.
# Do NOT assume the filename — NVIDIA package layouts vary across driver versions.
Get-ChildItem -Path "path\to\extracted" -Filter "*.inf" -Recurse |
    Select-String -Pattern "DEV_1C8C" |
    Select-Object Filename, LineNumber, Line

# From the results, identify the display-class INF (look for Class=Display and
# ClassGUID={4d36e968-e325-11ce-bfc1-08002be10318} near the top of the matching file).
# NVIDIA packages may contain multiple INFs or sections matching DEV_1C8C.
# Choose the one whose Class is "Display" and whose [Manufacturer] section
# targets this notebook's subsystem (SUBSYS_087C1028). Note its full path.

# Only after confirming the hardware ID match in a display-class INF:
pnputil /add-driver "path\to\the\verified\display.inf" /install

# Re-enable auto-driver search
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\DriverSearching" /v SearchOrderConfig /t REG_DWORD /d 1 /f

pnputil /scan-devices
```
Check device status. If still Error 43 after a verified clean install, the GPU hardware or firmware has very likely failed — though subtle driver-level issues cannot be fully ruled out without hardware-level diagnostics (e.g., testing the PCIe link with vendor tools).

---

## Phase 3: If GPU Cannot Be Recovered

If the GPU still shows Error 43 after power drain, BIOS update, and verified clean driver install, the GPU has very likely failed at the hardware or firmware level. The goal of this phase is to reach a stable terminal state where the laptop works reliably on the Intel iGPU alone.

**Intended terminal state:** The NVIDIA devnode and all NVIDIA display driver packages are fully removed from the system (not just disabled), so that Windows does not attempt to manage, start, or power-transition the GPU. This is a stronger action than the Phase 1 disable.

### Step 3.1 — Remove the GPU device and driver packages
If you completed Phase 2 Step 2.5, the device and packages are already gone — verify with the commands below and skip to Step 3.2 if nothing NVIDIA-related appears. Otherwise, boot into Safe Mode (Administrator):
```powershell
# Inspect current state
pnputil /enum-devices /class Display
pnputil /enum-drivers

# If the NVIDIA device still appears, remove the devnode:
pnputil /remove-device "PCI\VEN_10DE&DEV_1C8C&SUBSYS_087C1028&REV_A1\4&1cfd4d04&0&0008"

# If NVIDIA display packages still appear in enum-drivers, delete them:
pnputil /delete-driver oem137.inf /force
pnputil /delete-driver oem1.inf /force

# Confirm only Intel UHD 630 remains
pnputil /enum-devices /class Display
```

### Step 3.2 — (If available) Disable discrete GPU in BIOS
Restart → F2 → look for a toggle under **Video** or **Switchable Graphics**. If present, this removes the device from the PCIe bus entirely, preventing Windows from ever re-enumerating it. **This option is firmware-version-dependent and not present on all XPS 9570 BIOS revisions.** If unavailable, the Step 3.1 removal alone may not be permanent — a removed devnode (unlike a disabled one) can be re-enumerated on reboot or by Windows Update re-staging a driver package. If that happens, re-enter Safe Mode and repeat Step 3.1, or use `pnputil /disable-device` instead of `/remove-device` to keep the devnode present but stopped.

### Step 3.3 — Ongoing use
The Intel UHD 630 handles general productivity, video playback, and light photo/video editing. You lose gaming and GPU-compute capability, but the laptop is otherwise fully functional. If you later replace the motherboard or want to re-test the GPU after a BIOS update, re-run Phase 2 starting from Step 2.3.

---

## Other Issues Noted (independent of GPU)

These appeared in the System event log and may need separate attention:
- **OpenSSH SSH Server** crash-looping (5 restarts per boot)
- **DNS Name Resolution Policy Table corrupted** (Event 1023) — try `netsh winsock reset` + `ipconfig /flushdns`
- **WMI/CIM returning empty results** for standard queries — try `winmgmt /salvagerepository`
- **Windows Biometric Service** and **Realtek DHCP Service** crashing
- **Intel SUR QC Software Asset Manager** service file missing

---

## Verification Checklist

| After Phase | Test | Expected Result |
|-------------|------|-----------------|
| Phase 1 | `shutdown /s /t 0` | Clean shutdown, no BSOD |
| Phase 2 | `pnputil /enum-devices /instanceid "PCI\VEN_10DE..."` | Status: Started, no Problem Code |
| Phase 2 | Open Device Manager → Display adapters | NVIDIA GTX 1050 Ti with no yellow triangle |
| Phase 3 | `pnputil /enum-devices /class Display` | Only Intel UHD 630 listed |
| Phase 3 | `shutdown /s /t 0` then `shutdown /r /t 0` | Both work cleanly |
