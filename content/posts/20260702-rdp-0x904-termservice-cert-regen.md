---
title: "Windows RDP Troubleshooting - Fixing Error 0x904 by Forcing Certificate Regeneration"
date: 2026-07-02
draft: false
tags: ["windows-server", "rdp", "remote-desktop", "certificate", "cmd", "troubleshooting"]
categories: ["Infrastructure"]
description: "RDP error 0x904 / extended error 0x7 often means the server's self-signed TLS certificate has expired or corrupted. Restarting TermService discards the broken cert and generates a fresh one — no manual certificate management needed."
showToc: true
---

## The error

You try to RDP into a Windows Server and get:

```
Error code: 0x904
Extended error code: 0x7
```

The remote desktop client can't negotiate a TLS session. The server is reachable, the firewall port is open, the service appears to be running — but the connection dies before the login screen.

## What's actually broken

Windows Remote Desktop uses a **self-signed certificate** to authenticate the server side of the TLS handshake. That certificate is generated and owned by the **Remote Desktop Services** daemon (TermService). If it expires or gets corrupted — after a long uptime, a botched system update, or a disk-level issue — RDP stops working with exactly this symptom.

The certificate isn't sitting somewhere you can cleanly replace. The right way to regenerate it is to stop TermService and let it create a new one on startup. No manual certificate management, no MMC snap-ins, no `certlm.msc` digging.

## The fix

Run this CMD script **as administrator directly on the server** — not through an existing RDP session (you'd sever the connection mid-script), and if RDP is already down you need physical console access or an out-of-band management interface. More on that below.

```cmd
@echo off
:: Remedy for RDP "Error code: 0x904, Extended error code: 0x7"
:: Run AS ADMINISTRATOR on the Windows Server (target machine)

echo.
echo ==== Step 1: Checking Remote Desktop firewall rules ====
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
echo.

echo ==== Step 2: Restarting Remote Desktop Services (TermService) ====
:: This forces Windows to discard any corrupted/expired
:: self-signed RDP certificate and generate a fresh one.
net stop termservice /y
net start termservice
echo.

echo ==== Step 3: Verifying Remote Desktop is enabled ====
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
echo.

echo ==== Step 4: Restarting related services (RD listener) ====
net stop UmRdpService /y
net start UmRdpService
echo.

echo ==== Done ====
echo If the client still cannot connect after this, a full server
echo reboot is the next step, as some certificate/session state
echo only clears fully on restart.
echo.
pause
```

Each step addresses a different layer where the connection can be stuck:

**Step 1 — Firewall rules.** Ensures the "Remote Desktop" rule group is enabled. Usually fine, but a policy change or a failed update can flip it.

**Step 2 — TermService restart.** The actual fix. Stopping the service forces Windows to discard and regenerate the self-signed RDP certificate on restart. The `/y` flag auto-confirms stopping the dependent `UmRdpService`, which is why step 4 re-starts it explicitly.

**Step 3 — Registry flag.** `fDenyTSConnections = 0` means "allow connections". If this got set to `1` by a policy or accident, RDP is disabled at a layer below the certificate and the restart wouldn't have helped.

**Step 4 — UmRdpService restart.** The User Mode Port Redirector handles clipboard and device redirection. Restarting it cleans up any stale port listener state left behind from the TermService stop.

## What a successful run looks like

Step 2 is the one to watch. A clean certificate regeneration looks like this:

```
==== Step 2: Restarting Remote Desktop Services (TermService) ====
The following services are dependent on the Remote Desktop Services service.
Stopping the Remote Desktop Services service will also stop these services.

   Remote Desktop Services UserMode Port Redirector

The Remote Desktop Services UserMode Port Redirector service is stopping..
The Remote Desktop Services UserMode Port Redirector service was stopped successfully.

The Remote Desktop Services service is stopping.
The Remote Desktop Services service was stopped successfully.

The Remote Desktop Services service is starting.
The Remote Desktop Services service was started successfully.
```

No error, no hung service. After this, try connecting from the client. The client may prompt "The identity of the remote computer cannot be verified" — that's the new self-signed cert, which the client hasn't seen before. Accept it.

If the client still can't connect after a clean run, reboot the server. Some certificate and session state in Windows only fully clears on restart.

## The bootstrap problem

When RDP is down, you can't RDP in to run the fix. In order of least friction:

**Physical access.** Plug in a monitor and keyboard. If the server is at a client site, this means dispatching someone on-site.

**Out-of-band management.** iDRAC, iLO, IPMI — these provide a remote console independent of the OS. If the server has one and you have credentials, use it before scheduling a dispatch.

**Another remote channel.** If SSH or WinRM is running, you can execute `net stop termservice /y && net start termservice` directly from a remote shell, without the batch file.

## Delivering the script to an on-site technician

If you're sending this to someone else to run rather than running it yourself, note that **email filters commonly block `.cmd` attachments**. The workaround: rename the file to something like `fix-rdp.txt`, attach that, and instruct the recipient to rename it back to `.cmd` before right-clicking → "Run as administrator."

This script has no PowerShell dependency and runs on any Windows Server version from 2008 onward.
