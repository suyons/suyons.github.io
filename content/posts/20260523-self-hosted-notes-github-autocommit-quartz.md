---
title: "Self-Hosted Notes: Auto Git Commit + Quartz Web Viewer on Ubuntu Server"
date: 2026-05-23
draft: false
tags: ["obsidian", "quartz", "git", "ubuntu", "self-hosted", "bash", "inotify", "systemd"]
categories: ["Infrastructure"]
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

## Optional: Automatic `git pull` Triggered by Push from Another Machine

If you edit notes on a second machine (not the Ubuntu server) and push to GitHub, the server won't automatically pull those changes — inotifywait only reacts to local filesystem events. You can close this gap with a GitHub webhook that triggers `git pull` on the server whenever a push lands on the `note` repository.

### How It Works

```
Another machine
  └─ git push → GitHub
                  └─ POST https://<your-webhook-domain>/hooks/note-pull
                              └─ git pull on Ubuntu Server
```

### Install webhook

```bash
apt-get install -y webhook
```

The `webhook` binary (github.com/adnanh/webhook) listens for HTTP POST requests and executes a configured script when the trigger rules match.

Disable the default system unit — we'll create a dedicated one:

```bash
systemctl disable --now webhook
```

### The Pull Script

Create `/usr/local/bin/note-pull.sh`:

```bash
#!/bin/bash
git -C /root/note pull origin HEAD
```

```bash
chmod +x /usr/local/bin/note-pull.sh
```

### Hook Configuration

Create `/etc/webhook/hooks.json`:

```json
[
  {
    "id": "note-pull",
    "execute-command": "/usr/local/bin/note-pull.sh",
    "command-working-directory": "/root/note",
    "response-message": "pull triggered",
    "trigger-rule": {
      "and": [
        {
          "match": {
            "type": "payload-hmac-sha256",
            "secret": "<your-webhook-secret>",
            "parameter": {
              "source": "header",
              "name": "X-Hub-Signature-256"
            }
          }
        },
        {
          "match": {
            "type": "value",
            "value": "push",
            "parameter": {
              "source": "header",
              "name": "X-GitHub-Event"
            }
          }
        }
      ]
    }
  }
]
```

The `payload-hmac-sha256` rule verifies the `X-Hub-Signature-256` header that GitHub signs with the shared secret, so only genuine GitHub requests can trigger the pull. Generate a secret with:

```bash
openssl rand -hex 32
```

Use the same value in this file and in the GitHub webhook settings.

### Systemd Service

Create `/etc/systemd/system/note-webhook.service`:

```ini
[Unit]
Description=GitHub webhook listener for /root/note pull
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/webhook -hooks /etc/webhook/hooks.json -port 5300 -verbose
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now note-webhook
```

The listener binds on `0.0.0.0:5300`. Point a reverse proxy at it so GitHub can reach it over HTTPS.

### GitHub Webhook Settings

In your `note` repository: Settings → Webhooks → Add webhook.

| Field | Value |
|---|---|
| Payload URL | `https://<your-webhook-domain>/hooks/note-pull` |
| Content type | `application/json` |
| Secret | the value from `openssl rand -hex 32` above |
| Events | Just the push event |

### Caveat: Cloudflare Zero Trust Access

If the webhook domain is protected by Cloudflare Zero Trust Access, GitHub's requests will be redirected to the Cloudflare login page and never reach the server. You need to bypass Access for the `/hooks/note-pull` path.

The cleanest fix is to create a second Access application scoped to `<your-webhook-domain>/hooks/note-pull` with a **Bypass** policy. Cloudflare matches the most specific hostname and path first, so this exempts the webhook endpoint while leaving everything else protected.

If your Access policy is applied to a wildcard domain (e.g. `*.example.com`), create an explicit application for the exact webhook hostname with the Bypass policy — exact hostnames take precedence over wildcards.

## Wrapping Up

- The git-based approach gives full version history and zero sync risk — all writes flow one direction (server → GitHub).
- Quartz renders Obsidian-style markdown including backlinks, tags, and graph view, and works well on iPhone Safari.
- The debounced inotify watcher keeps commits clean without spamming one commit per keystroke.
- The whole stack (watcher + Quartz) survives reboots via systemd and has no runtime dependencies beyond Node.js.
