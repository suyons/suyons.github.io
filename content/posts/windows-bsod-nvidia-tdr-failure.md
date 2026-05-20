---
title: "Diagnosing a Windows BSOD Caused by NVIDIA TDR Failure"
date: 2026-05-07
draft: false
tags: ["windows", "bsod", "nvidia", "debugging", "event-viewer", "minidump"]
categories: ["Troubleshooting"]
description: "A practical walkthrough of diagnosing repeated BSODs on a Windows machine using Event Viewer and minidump files, tracing the root cause to an NVIDIA driver TDR failure, and resolving it step by step."
showToc: true
---

## Background

A Windows laptop at the office started crashing with blue screens several times a day. The machine had recently received a BIOS update and had its GPU mode switched from Hybrid to Discrete. Sent over the Event Viewer logs for analysis — this post is a write-up of the full diagnostic and resolution process.

## Reading the Event Viewer Logs

When Windows crashes and recovers, it writes a series of events to the System log. The relevant events appear in roughly this order after each crash:

### Event ID 41 — Kernel-Power (Critical)

```
Source: Microsoft-Windows-Kernel-Power
Level:  Critical
Task:   (63)

BugcheckCode: 278
BugcheckParameter1: 0xffffa781475b6010
BugcheckParameter2: 0xfffff8024a007960
BugcheckParameter3: 0xffffffffc000009a
BugcheckParameter4: 0x4
```

Event 41 is the kernel's record of an unclean reboot. The `BugcheckCode` field is the raw stop code (decimal). `278` decimal = `0x116` hex — that is `VIDEO_TDR_FAILURE`.

A second crash shortly after produced `BugcheckCode: 307` = `0x133` hex — `DPC_WATCHDOG_VIOLATION`.

### Event ID 1001 — WER SystemErrorReporting

```
Source: Microsoft-Windows-WER-SystemErrorReporting
Level:  Error

param1: 0x00000116 (0xffffa781475b6010, 0xfffff8024a007960, 0xffffffffc000009a, 0x4)
param2: C:\WINDOWS\Minidump\<timestamp>-<id>.dmp
```

This event confirms the stop code in human-readable hex and records the path to the minidump file on disk.

### Event ID 162 — volmgr

```
Source: volmgr
Level:  Error

\Device\HarddiskVolume3
```

Windows successfully wrote the minidump. If this event is missing, the dump either didn't complete or is stored elsewhere (check `SystemPropertiesAdvanced` → Startup and Recovery).

### Event ID 6008 — EventLog

```
Source: EventLog
Level:  Error

The previous system shutdown at <time> on <date> was unexpected.
```

This appears after the reboot, confirming the machine went down uncleanly.

---

## The Actual Culprit: `nvlddmkm`

Before the `0x116` crash, the System log contained a cluster of `nvlddmkm` events that fired in rapid succession:

| Event ID | Message |
|----------|---------|
| 13 | `Graphics FECS Exception: UCODE Fatal Error` |
| 153 | `GpuRcReset TDR occurred on GPUID:100` |
| 14 | `GPU recovery action changed from 0x0 (None) to 0x1 (GPU Reset Required)` |
| 153 | `Resetting TDR occurred on GPUID:100` |

`nvlddmkm` is the NVIDIA kernel-mode display driver. The sequence here is:

1. GPU firmware (FECS microcode) encountered a fatal exception.
2. Windows attempted a Timeout Detection and Recovery (TDR) — the mechanism that tries to reset a hung GPU without a full crash.
3. TDR reset was initiated but apparently couldn't recover the device in time.
4. Windows stopped the system and wrote a minidump rather than continue with a potentially corrupted graphics subsystem.

The second BSOD (`0x133 DPC_WATCHDOG_VIOLATION`) followed shortly after. DPC_WATCHDOG fires when a Deferred Procedure Call runs longer than the kernel's timeout — consistent with a driver that is stuck trying to recover from the GPU reset.

### Why after a BIOS update and GPU mode change?

Switching from Hybrid Graphics to Discrete (dGPU-only) mode disables the iGPU at the BIOS level. On many laptops, the graphics driver installed for Hybrid mode carries assumptions about the Advanced Optimus mux switch that don't hold in pure Discrete mode. A BIOS revision on top of this can change the power delivery contract the driver expects. The result is often exactly this: GPU intermittently fails to respond, TDR triggers, fails, BSOD.

---

## Resolution Steps

### Step 1: Clean-uninstall the display driver with DDU

Simply reinstalling the driver leaves registry artifacts and residual driver files that perpetuate the problem.

1. Download **Display Driver Uninstaller (DDU)**.
2. Boot into **Safe Mode** (press Shift at the login screen → Restart → Troubleshoot → Advanced options → Startup Settings → Safe Mode with Networking).
3. Run DDU, select GPU type NVIDIA, and choose **"Clean and restart"**.

### Step 2: Install the previous stable driver version

After the clean reboot, avoid installing the very latest Game Ready Driver (GRD). Instead:

- Download the most recent **Studio Driver** from the NVIDIA website — Studio drivers go through additional stability testing.
- Alternatively, download the GRD release that predates the current one by one version.
- Do **not** use Windows Update for driver delivery; use the manual installer with the **Custom** install path and tick **"Perform a clean installation"** for a second layer of safety.

### Step 3: Set power management to maximum performance

Low-power states cause the GPU to transition between P-states. The driver/BIOS combination may not handle these transitions cleanly.

1. Open **NVIDIA Control Panel**.
2. Navigate to **Manage 3D settings** → **Power management mode**.
3. Set to **Prefer maximum performance**.

### Step 4: Check the GPU mode in BIOS

If crashes occurred specifically after switching to Discrete Graphics mode, go back into BIOS and test with **Hybrid Mode** re-enabled. If the crashes stop, the Discrete-mode path in the new BIOS has a bug — check Lenovo's support site for a newer BIOS patch for the specific laptop model.

### Step 5: Disable hardware acceleration in browsers and other apps

If crashes happen during general browsing:

- Chrome/Edge: Settings → System → turn off **"Use hardware acceleration when available"**.
- Discord: Settings → Advanced → disable **Hardware Acceleration**.

This shifts GPU command submission from the browser/app path to pure CPU rendering, which sidesteps the driver's problematic fast-path.

### Step 6: Repair system files

The `ntoskrnl.exe` involvement in the `0x133` crash means it is worth checking for OS-level corruption as well:

```cmd
sfc /scannow
dism /online /cleanup-image /restorehealth
```

Run both from an elevated Command Prompt. `sfc` checks the Windows Component Store; `dism` fetches clean replacements from Windows Update if any are needed.

---

## Summary

The crashes traced to a driver-kernel TDR conflict in the NVIDIA display driver (`nvlddmkm`), surfacing after a BIOS update changed the power delivery contract that the installed driver version expected. The fix is a DDU clean uninstall followed by a known-stable driver, not the latest release. If the same stop code returns after a clean driver install, the next step is a BIOS rollback.

The diagnostic chain — Kernel-Power 41 for the stop code, WER 1001 for the minidump path, nvlddmkm events for the immediate cause — is reusable for any GPU-related BSOD regardless of vendor.
