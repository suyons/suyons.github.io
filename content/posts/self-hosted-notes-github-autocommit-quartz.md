---
title: "Self-Hosted Notes: Auto Git Commit + Quartz Web Viewer on Ubuntu Server"
date: 2026-05-23
draft: false
tags: ["obsidian", "quartz", "git", "ubuntu", "self-hosted", "bash", "inotify", "systemd"]
categories: ["DevOps"]
description: "How I set up an Obsidian vault on Ubuntu Server with automatic git commit/push on every save, and Quartz as a web viewer — after abandoning SilverBullet and Syncthing."
showToc: true
---

## Why I Abandoned SilverBullet + Syncthing

### SilverBullet

I initially installed SilverBullet to serve my Obsidian vault through a browser. It looked promising — self-hosted, markdown-based, and easy to set up. But it didn't hold up in practice:

1. The UI was minimal to the point of feeling unfinished.
2. Pages frequently failed to load on iPhone Safari, which was a dealbreaker since mobile access was the whole point.
3. Custom fonts and plugin installation required digging through undocumented internals.

### Syncthing

I also tried Syncthing to keep the vault in sync between the server and my MacBook. Two problems:

1. **Fatal data loss risk.** Syncthing is bidirectional. If it starts syncing *after* files are deleted on the server, the deletion propagates to the Mac — irreversibly. I actually lost my notes this way during setup. Because the sync happened before I noticed, there was no recovery path on the server side.
2. **Port control.** Syncthing operates at the TCP transport layer (port 22000). My home network runs a Proxmox VE host as the gateway with nginx handling reverse proxy at the application level. Nginx can route HTTP/HTTPS by domain name, but it can't intercept raw TCP — so Syncthing required a separate port forward that I had no visibility into from the nginx config. That felt out of control.

### The Pivot

I settled on two tools that solve these problems cleanly:

- **GitHub private repo** — version-controlled, no sync conflicts, recoverable from any machine, stays behind HTTPS.
- **Quartz** — a static site generator built for Obsidian vaults. Renders beautifully in mobile Safari, no special client needed.

I edit notes in a VS Code Remote Tunnel session connected from my MacBook to the Ubuntu server, so the vault lives only on the server. No sync needed at all.

## Architecture Overview

```
MacBook (VS Code Remote Tunnel)
  └─ edits /root/note on Ubuntu Server
        │
        ├─ inotifywait detects file save
        │     └─ git add -A && git commit "save: <ISO8601>" && git push
        │           └─ Quartz rebuilds /opt/quartz/public/
        │
        └─ Quartz --serve watches content dir
              └─ serves static site at :4000
```

## Setting Up the Vault

The vault is a plain directory of markdown files. No special initialization needed beyond what git requires.

```bash
mkdir -p /root/note
cd /root/note
git init
git remote add origin https://<username>:<token>@github.com/<username>/note
```

## Auto Commit and Push on Save

### Install inotify-tools

```bash
apt-get install -y inotify-tools
```

### The Watcher Script

Create `/usr/local/bin/note-autopush.sh`:

```bash
#!/bin/bash
WATCH_DIR="/root/note"
cd "$WATCH_DIR"

while true; do
    inotifywait -r -e close_write,moved_to,moved_from,create,delete \
        --exclude '(\.git|\.swp|\.tmp|\.swx)' "$WATCH_DIR" 2>/dev/null

    # Debounce: drain additional events within 2 seconds of silence
    while inotifywait -r -e close_write,moved_to,moved_from,create,delete \
        --exclude '(\.git|\.swp|\.tmp|\.swx)' -t 2 "$WATCH_DIR" 2>/dev/null; do
        :
    done

    TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")
    git -C "$WATCH_DIR" add -A
    if git -C "$WATCH_DIR" diff --cached --quiet; then
        continue
    fi
    git -C "$WATCH_DIR" commit -m "save: $TIMESTAMP"
    git -C "$WATCH_DIR" push origin HEAD
    PATH=/root/.nvm/versions/node/v25.1.0/bin:$PATH \
        /opt/quartz/quartz/bootstrap-cli.mjs build 2>/dev/null &
done
```

Key points:

- The debounce loop drains all events until 2 seconds of silence, so rapid multi-file saves are batched into a single commit.
- `git diff --cached --quiet` skips the commit if there are no actual changes.
- Commit message is ISO 8601 UTC with milliseconds: `save: 2026-05-23T13:00:00.123Z`.
- Quartz rebuild runs in the background (`&`) so it doesn't block the watcher.

```bash
chmod +x /usr/local/bin/note-autopush.sh
```

### Systemd Service

Create `/etc/systemd/system/note-autopush.service`:

```ini
[Unit]
Description=Auto commit and push /root/note on file changes
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/note-autopush.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now note-autopush
```

## Installing Quartz

### Prerequisites

Node.js and pnpm are required.

```bash
npm install -g pnpm
```

### Clone and Install

```bash
git clone https://github.com/jackyzha0/quartz.git /opt/quartz
cd /opt/quartz
pnpm approve-builds --all
pnpm install
```

### Link the Vault as Content

Quartz reads markdown from its `content/` directory. Symlink the vault there:

```bash
rm -rf /opt/quartz/content
ln -s /root/note /opt/quartz/content
```

### Initial Build

```bash
cd /opt/quartz
pnpm quartz build
```

Output goes to `/opt/quartz/public/`. Nothing is written back into `/root/note`, so the autopush watcher is not triggered.

### Systemd Service

Create `/etc/systemd/system/quartz.service`:

```ini
[Unit]
Description=Quartz static site server for /root/note
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/quartz
Environment=PATH=/root/.nvm/versions/node/v25.1.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/opt/quartz/quartz/bootstrap-cli.mjs build --serve --port 4000
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now quartz
```

The `--serve` flag builds once, then watches the content directory and rebuilds automatically on any change. This means even files written directly on the server (not through the watcher) will appear in the browser without a manual rebuild.

### Open the Port

```bash
ufw allow 4000/tcp
```

Access at `http://<server-ip>:4000`. Point a reverse proxy at it for HTTPS and a clean domain.

## Wrapping Up

- The git-based approach gives full version history and zero sync risk — all writes flow one direction (server → GitHub).
- Quartz renders Obsidian-style markdown including backlinks, tags, and graph view, and works well on iPhone Safari.
- The debounced inotify watcher keeps commits clean without spamming one commit per keystroke.
- The whole stack (watcher + Quartz) survives reboots via systemd and has no runtime dependencies beyond Node.js.
