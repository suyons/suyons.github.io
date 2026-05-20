---
title: "Enabling SSH on Windows Server for Lightweight Remote Management"
date: 2026-05-19
draft: false
tags: ["windows", "ssh", "sftp", "openssh", "powershell", "rdp"]
categories: ["Infrastructure"]
description: "How to enable the OpenSSH Server feature on Windows Server, connect from any SSH client, and use SFTP via FileZilla — useful when RDP's single-session limit makes collaborative server work painful."
showToc: true
---

## The Problem with RDP for Team Environments

Windows Remote Desktop Protocol (RDP) supports only one interactive session per user account on Windows Server (without Remote Desktop Services licensing). If two team members need to interact with the server simultaneously — one deploying a build, another checking a log — one of them gets kicked out.

For most routine operations — uploading build artifacts, restarting services, tailing logs — a terminal session is sufficient. OpenSSH Server ships with modern Windows and solves this neatly: multiple SSH sessions can run concurrently without displacing each other, and the shell is PowerShell 7 if it is installed.

---

## 1. Enable OpenSSH Server

OpenSSH Server is a Windows optional feature available on Windows Server 2019 and later (and Windows 10 1809+).

### Option A: Settings UI

**Settings** → **System** → **Optional features** → **Add a feature** → search **OpenSSH Server** → **Install**.

### Option B: PowerShell (preferred for server environments)

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Verify installation:

```powershell
Get-WindowsCapability -Online | Where-Object Name -Like 'OpenSSH*'
```

Expected output shows both client and server with `State: Installed`.

---

## 2. Start and Configure the `sshd` Service

```powershell
# Start the service
Start-Service sshd

# Set it to start automatically after reboot
Set-Service -Name sshd -StartupType 'Automatic'
```

The OpenSSH installer creates a firewall rule named `OpenSSH-Server-In-TCP` that allows inbound TCP on port 22. Verify it is active:

```powershell
Get-NetFirewallRule -Name 'OpenSSH-Server-In-TCP'
```

If the rule is missing or disabled:

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True -Direction Inbound -Protocol TCP `
  -Action Allow -LocalPort 22
```

---

## 3. Connecting via SSH

From any client machine — Windows CMD, PowerShell, macOS/Linux terminal:

```
ssh Administrator@192.168.x.x
```

On the first connection, the client displays the server's host key fingerprint and asks for confirmation:

```
The authenticity of host '192.168.x.x (192.168.x.x)' can't be established.
ED25519 key fingerprint is: SHA256:<fingerprint>
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes`. The key is saved to `~/.ssh/known_hosts` and this prompt will not appear again for this host. Enter the account password when prompted.

A successful connection drops into the server's default shell:

```
PowerShell 7.6.1
PS C:\Users\Administrator>
```

If PowerShell 7 is installed, it becomes the default shell for SSH sessions automatically (OpenSSH reads it from the `DefaultShell` registry value). Otherwise you get the legacy `cmd.exe`.

To set PowerShell 7 explicitly:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Program Files\PowerShell\7\pwsh.exe" `
  -PropertyType String -Force
```

---

## 4. SFTP for File Transfers

Because SFTP runs over the same SSH channel, no additional server configuration is needed. Any SFTP client that can connect to the SSH server will work.

### Command-line `scp`

```
scp .\build\app.zip Administrator@192.168.x.x:C:\apps\app.zip
```

### FileZilla (recommended for interactive use)

For browsing directories, dragging and dropping multiple files, or managing a mix of uploads and downloads, a GUI client is faster than `scp`.

1. Download **FileZilla Client** from the official site.
2. In the **Quickconnect** bar at the top:
   - **Host**: `sftp://192.168.x.x`
   - **Username**: `Administrator`
   - **Password**: (your password)
   - **Port**: `22`
3. Click **Quickconnect**.

FileZilla detects the SFTP protocol from the `sftp://` prefix and presents a split-pane view: local file system on the left, remote server on the right. Drag files between the panes to upload or download.

---

## 5. Key-Based Authentication (Optional but Recommended)

Password authentication works immediately but is less secure. To switch to key-based authentication:

### Generate a key pair on the client

```
ssh-keygen -t ed25519 -C "your-identifier"
```

This creates `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public).

### Authorize the public key on the server

For standard user accounts, append the public key to `C:\Users\<username>\.ssh\authorized_keys`.

For members of the local **Administrators** group, Windows OpenSSH uses a different path:

```
C:\ProgramData\ssh\administrators_authorized_keys
```

This is a deliberate security design: administrator-level authorized keys are stored in a system-protected path rather than the user's profile directory.

Paste the contents of `id_ed25519.pub` into that file, then fix permissions:

```powershell
icacls C:\ProgramData\ssh\administrators_authorized_keys /inheritance:r
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant "SYSTEM:(F)"
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant "BUILTIN\Administrators:(F)"
```

After this, connecting with the matching private key requires no password.

---

## Summary

OpenSSH Server on Windows Server is a first-class feature — no third-party software, no Cygwin layer. Enabling it takes three commands. The resulting SSH and SFTP access can run concurrently alongside existing RDP sessions, making it a straightforward complement for teams where RDP's single-session limitation causes friction.

The main gotcha: administrators use `C:\ProgramData\ssh\administrators_authorized_keys` for authorized keys, not the per-user `.ssh\authorized_keys` path you may be used to from Linux.
