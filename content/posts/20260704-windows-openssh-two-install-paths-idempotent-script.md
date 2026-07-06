---
title: "Windows Server Troubleshooting - An Idempotent Install Script That Only Knew One Way sshd Gets Installed"
date: 2026-07-04
draft: false
tags: ["windows-server", "openssh", "sshd", "powershell", "idempotency", "troubleshooting"]
categories: ["Infrastructure"]
description: "A PowerShell script to install OpenSSH on a custom port assumed one installation path and one config-generation timing. Both assumptions broke on a box set up by hand."
showToc: true
---

## The setup

A Windows Server needed `sshd` listening on a non-default port (32222, to keep it off the noise that port 22 attracts), reachable only from one external IP plus the internal subnet. The plan was a single idempotent PowerShell script: install OpenSSH if missing, set the port, lock the firewall rule down, done. Re-run it any time and it should no-op cleanly on a box that's already configured.

It worked the first three times. It broke on the fourth box, because that one had OpenSSH installed a different way than the script assumed.

## Gotcha 1: "is it installed" has two different answers

The original check used the Windows Optional Feature API:

```powershell
$capability = Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
if ($capability.State -ne 'Installed') {
    Add-WindowsCapability -Online -Name $capability.Name
    Write-Host "OpenSSH.Server installed successfully." -ForegroundColor Green
} else {
    Write-Host "OpenSSH.Server is already installed." -ForegroundColor Yellow
}
```

That's correct if OpenSSH always arrives through `Add-WindowsCapability`. It doesn't, if someone earlier just downloaded a Win32-OpenSSH release zip and ran its bundled `install-sshd.ps1` directly — which registers the `sshd` and `ssh-agent` services without ever touching the Capability store. To `Get-WindowsCapability`, that server looks exactly like one with nothing installed. The script would try to install the capability on top of an already-running manual install, which is at best redundant and at worst a source of duplicate/conflicting `sshd` registrations.

The fix checks both ways OpenSSH could be present:

```powershell
$capability = Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
$sshdAlreadyInstalled = ($capability.State -eq 'Installed') -or (Get-Service -Name sshd -ErrorAction SilentlyContinue)
if (-not $sshdAlreadyInstalled) {
    Add-WindowsCapability -Online -Name $capability.Name
    Write-Host "OpenSSH.Server installed successfully." -ForegroundColor Green
} else {
    Write-Host "OpenSSH.Server is already installed (capability or manual Win32-OpenSSH)." -ForegroundColor Yellow
}
```

`Get-Service -Name sshd` doesn't care how the service got registered. If it exists, something already put `sshd` on this box, and the script should treat that as "installed" regardless of which path got it there. The lesson generalizes: when a script's idempotency check queries one specific installation mechanism, it's implicitly assuming that's the *only* mechanism anyone will ever use. On a fleet where different people set up boxes at different times, that assumption has a shelf life.

## Gotcha 2: the config file that doesn't exist until the service has run once

Past the install check, the script edits `sshd_config` to change the listening port:

```powershell
$sshdConfigPath = "$env:ProgramData\ssh\sshd_config"
```

On the Capability-installed boxes, this file was always there. On the manually-installed box, it wasn't — `Test-Path $sshdConfigPath` came back false, and the script hit its own guard clause and exited with "sshd_config file not found." The install had clearly succeeded: `sshd.exe` was on disk, the service was registered. But nothing had ever generated the default config.

Here's why: `install-sshd.ps1` registers the Windows services. It does not write `sshd_config` or generate host keys. Those get created by `sshd` itself, the first time it actually starts, from a template it carries internally. A Capability install happens to trigger that first start as part of Windows' own setup sequence; a bare manual install just leaves the services registered and stopped, with no config in sight, until someone starts `sshd` for the first time.

The fix is to force that first start before touching the file:

```powershell
if (-not (Test-Path $sshdConfigPath)) {
    # Manual Win32-OpenSSH installs don't write the default config until sshd runs once.
    Write-Host "sshd_config not found; starting sshd once to generate the default config..." -ForegroundColor Yellow
    Start-Service sshd
    Start-Sleep -Seconds 2
    Stop-Service sshd
}

if (-not (Test-Path $sshdConfigPath)) {
    Write-Error "sshd_config file not found at: $sshdConfigPath (check OpenSSH installation)."
    exit 1
}
```

Start it, give it two seconds to write its defaults and generate host keys, stop it again, then re-check. The original hard failure stays in place as a second guard — if the file still isn't there after a forced start, something is actually wrong with the install, and the script should still refuse to proceed rather than blindly write config for a path that doesn't exist.

## A smaller win from the same pass

The allowlist for the firewall rule went from a single string to an array in the same change:

```powershell
# Before
$AllowedRemoteIP = "192.0.2.10/32"

# After
$AllowedRemoteIP = @("192.0.2.10/32", "192.168.0.0/24")
```

`New-NetFirewallRule -RemoteAddress` accepts multiple values, so adding the internal subnet to the allowlist didn't need any change to the rule-creation logic further down the script — just the input list. Worth knowing before writing a rule builder that assumes single-address input: check whether the underlying cmdlet already accepts an array before adding your own loop around it.

## The uninstall script nobody had written yet

None of this would have been easy to debug without a clean way to tear a manual install back down and start over. That didn't exist, so it got written alongside the fix — a mirror image of the install script:

```powershell
<#
.SYNOPSIS
    Cleanly removes the manually-installed Win32-OpenSSH server and reverts the
    firewall/config changes made by Install-OpenSSH-Port32222.ps1.

.DESCRIPTION
    1) Stops sshd / ssh-agent
    2) Runs the upstream uninstall-sshd.ps1 (removes services + registry keys)
    3) Removes the restricted firewall rule and re-enables the default OpenSSH rule
    4) Deletes C:\Program Files\OpenSSH and $env:ProgramData\ssh (config + host keys)
    5) Strips the OpenSSH entry from the Machine PATH

.NOTES
    - Must be run as Administrator.
    - If script execution is restricted, run with:
      powershell -ExecutionPolicy Bypass -File .\Uninstall-OpenSSH-Port32222.ps1
#>

#Requires -RunAsAdministrator

$ErrorActionPreference = "Stop"
$OpenSshDir = "C:\Program Files\OpenSSH\OpenSSH-Win64"
$RuleName = "OpenSSH-Server-In-TCP-32222-Restricted"

Stop-Service sshd -ErrorAction SilentlyContinue
Stop-Service ssh-agent -ErrorAction SilentlyContinue

if (Test-Path "$OpenSshDir\uninstall-sshd.ps1") {
    & "$OpenSshDir\uninstall-sshd.ps1"
} else {
    Write-Warning "uninstall-sshd.ps1 not found; removing services manually."
    sc.exe delete sshd | Out-Null
    sc.exe delete ssh-agent | Out-Null
}

Get-NetFirewallRule -DisplayName $RuleName -ErrorAction SilentlyContinue | Remove-NetFirewallRule
if (Get-NetFirewallRule -DisplayName "OpenSSH SSH Server (sshd)" -ErrorAction SilentlyContinue) {
    Enable-NetFirewallRule -DisplayName "OpenSSH SSH Server (sshd)"
}

Remove-Item -Path "C:\Program Files\OpenSSH" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:ProgramData\ssh" -Recurse -Force -ErrorAction SilentlyContinue

$machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
$cleanedPath = ($machinePath -split ';' | Where-Object { $_ -and $_ -notlike "*OpenSSH*" }) -join ';'
[Environment]::SetEnvironmentVariable("Path", $cleanedPath, "Machine")

Remove-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sshd.exe" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\ssh-agent.exe" -Recurse -Force -ErrorAction SilentlyContinue
```

Two details worth calling out. First, it prefers the upstream `uninstall-sshd.ps1` shipped inside the Win32-OpenSSH release folder over hand-rolled `sc.exe delete` calls, and only falls back to the manual path if that script is missing — the vendor's own uninstaller knows about service dependencies and cleanup steps a two-line `sc.exe` loop doesn't. Second, it deletes leftover Image File Execution Options registry entries for `sshd.exe` and `ssh-agent.exe`. Win32-OpenSSH's installer creates these; deleting the program files and service registrations doesn't remove them, and they'll still be sitting under `HKLM:\...\Image File Execution Options` the next time someone runs a fresh install on the same box. Exactly what carries over in those keys is installer-internal detail worth confirming against the current Win32-OpenSSH release before relying on it — but leaving them behind is the kind of residue that turns a "clean reinstall" into a debugging session three months later.

## Takeaway

An idempotency check is only as good as its model of every way the target state could have been reached. This script had one install path in mind and one component (the Capability store) it trusted to report the truth. A single box set up by hand, months earlier, by someone following a README instead of the company script, was enough to break both assumptions on the same day.
