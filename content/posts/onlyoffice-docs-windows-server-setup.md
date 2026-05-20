---
title: "Self-Hosting OnlyOffice Docs on Windows Server: Setup, Redis Pitfalls, and Font Management"
date: 2026-05-13
draft: false
tags: ["onlyoffice", "windows-server", "redis", "jwt", "document-server", "edms"]
categories: ["Infrastructure"]
description: "A practical guide to installing OnlyOffice Docs on a Windows Server VM, wiring up JWT authentication, debugging the Redis ECONNRESET failure that causes the editor to spin forever, and reloading custom fonts."
showToc: true
---

## Context

An EDMS (Electronic Document Management System) was running a legacy office suite integration for in-browser document editing. The goal was to replace it with a self-hosted OnlyOffice Docs server. The target environment was a Windows Server VM hosted on a cloud provider — the team was more familiar with Windows Server administration than Linux, which drove the OS choice.

This post covers the practical steps of getting OnlyOffice Docs running on Windows, the configuration that tripped us up, and what to do when the editor loads but spins forever.

---

## 1. OnlyOffice Docs Edition Selection

OnlyOffice offers several products that sound similar:

| Product | What it includes |
|---------|-----------------|
| **Docs Community** | Editor only — open-source, free, connect via Integration API |
| **Docs Developer** | Editor only — paid, same integration approach, adds enterprise support |
| **DocSpace** | Editor + file storage — different product, not what you want for EDMS integration |

For embedding into an existing application via the [Integration API](https://api.onlyoffice.com/), pick **Docs Community** (free) or **Docs Developer** (paid). DocSpace is a standalone product and adds complexity you don't need.

For Windows, download the installer from the official OnlyOffice website. It bundles NGINX (reverse proxy), the DocService Node.js app, a Converter service, PostgreSQL, Redis (via a Cygwin port), and RabbitMQ.

---

## 2. Core Configuration: `local.json`

The installer creates a default `default.json` at:

```
C:\Program Files\ONLYOFFICE\DocumentServer\config\default.json
```

Never edit `default.json` directly. All overrides go into `local.json` in the same directory. Values in `local.json` are deep-merged on top of `default.json`.

A working baseline `local.json` for a single-node Windows deployment:

```json
{
  "services": {
    "CoAuthoring": {
      "sql": {
        "dbHost": "localhost",
        "dbPort": "5432",
        "dbUser": "onlyoffice",
        "dbPass": "YOUR_DB_PASSWORD",
        "dbName": "onlyoffice"
      },
      "redis": {
        "host": "localhost"
      },
      "server": {
        "port": "8000"
      },
      "utils": {
        "utils_common_fontdir": "C:/Windows/Fonts"
      },
      "token": {
        "enable": {
          "request": {
            "inbox": true,
            "outbox": true
          },
          "browser": true
        },
        "inbox":  { "header": "Authorization" },
        "outbox": { "header": "Authorization" }
      },
      "secret": {
        "browser":  { "string": "YOUR_JWT_SECRET" },
        "inbox":    { "string": "YOUR_JWT_SECRET" },
        "outbox":   { "string": "YOUR_JWT_SECRET" },
        "session":  { "string": "YOUR_JWT_SECRET" }
      }
    }
  },
  "rabbitmq": {
    "url": "amqp://guest:guest@localhost"
  },
  "license": {
    "license_file": "C:/ProgramData/ONLYOFFICE/Data/license.lic"
  }
}
```

Use the same string for all four `secret.*` fields to keep things simple on a single node. On the integration side, every editor `config` object and every callback from OnlyOffice to your application must be signed with a JWT using this same secret.

---

## 3. The Infinite Spinner: Redis ECONNRESET

The most common issue on a fresh Windows install: the editor opens in the browser, the loading spinner appears, and nothing ever loads. No network errors in the browser's Dev Tools either.

### Where to look first

OnlyOffice writes structured logs to:

```
C:\Program Files\ONLYOFFICE\DocumentServer\Log\docservice\
```

Open the most recent `DocService_<date>.out.txt`. If you see lines like these, Redis is the culprit:

```
[ERROR] nodeJS - redisClient error Error: read ECONNRESET
[ERROR] nodeJS - redisClient error AggregateError [ECONNREFUSED]:
    at internalConnectMultiple (node:net:1122:18)
```

OnlyOffice's DocService uses Redis as its session and pub/sub store. Without Redis, it cannot complete the document handshake with the editor client — hence the infinite spinner.

### Verify Redis is actually responding

```powershell
cd "C:\Program Files\Redis-Windows"
.\redis-cli.exe ping
```

If you get `Connection refused` rather than `PONG`, the Redis service is down.

### Fix: restart the Redis Windows service

```powershell
Stop-Service -Name Redis
Start-Service -Name Redis
.\redis-cli.exe ping
# Expected output: PONG
```

### Then restart all OnlyOffice services

```powershell
Get-Service |
  Where-Object { $_.DisplayName -like "*ONLYOFFICE*" } |
  Restart-Service
```

After this, reload the editor page — it should open within a few seconds.

### Why does Redis go down?

The Redis binary bundled with the Windows installer runs inside Cygwin (a Linux environment emulation layer for Windows). Cygwin-based services are less robust than native Windows services; they can silently crash after a server restart or when the system's TCP stack resets. Setting the Redis service to restart automatically on failure mitigates this:

```powershell
sc.exe failure Redis reset= 0 actions= restart/5000/restart/5000/restart/5000
```

For a more stable setup, install a native Windows Redis port (e.g., Memurai) or move to a Linux-hosted OnlyOffice instance if the team is comfortable with it.

---

## 4. Adding Custom Fonts

OnlyOffice loads all fonts installed on the server into its editor UI. Korean fonts and other non-Latin typefaces are not bundled — they need to be installed on the OS and then re-indexed by OnlyOffice.

### Install the font on Windows

Right-click the TTF/OTF file → **Install for all users**. Installing for the current user only does not make the font available to the OnlyOffice service account.

### Regenerate the font cache

```powershell
cd "C:\Program Files\ONLYOFFICE\DocumentServer\bin"
.\documentserver-generate-allfonts.bat
```

The script rebuilds the font list, regenerates presentation themes, clears JS caches, and then restarts all three OnlyOffice services (DocService, Converter, Proxy). Sample output:

```
Generating AllFonts.js, please wait...Done
Generating presentation themes, please wait...Done
Generating js caches, please wait...Done
The ONLYOFFICE Document Server EE DocService service is stopping.
...
The ONLYOFFICE Document Server EE Proxy service was started successfully.
```

After this completes, the newly installed fonts appear in the editor's font dropdown.

---

## 5. POC Testing Checklist

Before signing off on the OnlyOffice integration, the following areas are worth testing systematically. These were the points verified during our evaluation:

### Rendering compatibility

- High-resolution images (PNG, JPG) in documents — load without loss
- Grouped shapes and complex SmartArt — rendered correctly
- Advanced effects (transparency, shadow, neon) — reproduced accurately
- Text wrap and anchor settings — maintained after round-trip
- Korean fonts (e.g., Malgun Gothic) inside text boxes — no garbling

### Co-editing

- **Fast mode**: simultaneous keystrokes from two users reflect in real time
- **Strict mode**: changes sync only on explicit save, preventing write collisions during careful editing
- Three or more concurrent editors on the same paragraph — no conflicts observed

### Track Changes

Track Changes in OnlyOffice is a *recorder*, not a *scanner*. This means:

- Enabling Track Changes **before** editing is required to capture attribution.
- Changes made without the mode active leave no record.
- The Accept / Reject workflow is clean and works as expected.

### Permission modes

| Mode | Toolbar | Typing | Print/Download |
|------|---------|--------|----------------|
| Read-only | Disabled | Blocked | Configurable via JWT payload |
| Review | Enabled (comments only) | Tracked | Configurable |
| Edit | Full | Allowed | Configurable |

Granular permissions (disable print, disable download, disable copy) are set in the `permissions` block of the editor config object on the integration side, not in `local.json`.

### Observations

- macOS Safari: editor loads noticeably slower than Chromium-based browsers. This appears to be a known issue with Safari's WebAssembly execution path.
- Files with issues in the legacy office suite (missing shapes, partial page rendering, shape edit hangs) all reproduced correctly in OnlyOffice without modification.

---

## Summary

The Windows OnlyOffice installation is straightforward. The two places where people lose time are:

1. **Redis**: the Cygwin-based Redis service goes down silently; the symptom is an infinite spinner with no network errors. Check the DocService log, restart Redis, restart OnlyOffice services.
2. **JWT**: every request in both directions — editor to server and server to callback — must carry a signed JWT. If you see `403` responses in the integration, the secret in `local.json` doesn't match what your application is signing with.
