---
title: "Home Server on Proxmox VE: VM Layout, nginx Reverse Proxy, and Cloudflare"
date: 2026-05-25
draft: false
tags:
  [
    "proxmox",
    "nginx",
    "cloudflare",
    "ubuntu",
    "rocky-linux",
    "windows",
    "oracle",
    "self-hosted",
    "networking",
  ]
categories: ["Infrastructure"]
description: "War story of setting up a Proxmox VE home server — choosing the right guest OS for each workload, wiring up an nginx reverse proxy, and layering Cloudflare proxy with Zero Trust access control."
showToc: true
---

## Why Proxmox VE

Running multiple distinct workloads on a single machine creates a tension: each service wants its own OS environment, yet a bare-metal Linux install means every service shares one kernel and one package manager. That turns into dependency hell quickly.

The three workloads I needed to host made this especially stark:

- A general application stack that runs comfortably on Ubuntu
- Oracle Database 19c, which Oracle only certifies on RHEL-family distributions
- MetaTrader 5 with its Python SDK, which is distributed exclusively for Windows

All three on one machine, but each needing its own OS. Proxmox VE is a type-1 hypervisor built on KVM/QEMU and LXC that runs on bare Debian — it handles this cleanly without requiring a host OS GUI, a cloud subscription, or a VMware license.

---

## Hardware

| Component   | Detail                                    |
| ----------- | ----------------------------------------- |
| CPU         | AMD Ryzen 7 8845HS (8 cores / 16 threads) |
| RAM         | 16 GB                                     |
| Boot mode   | EFI, Secure Boot disabled                 |
| PVE version | `pve-manager/9.0.11`                      |
| Kernel      | `Linux 6.14.11-4-pve`                     |

The Ryzen 7 8845HS is a mobile chip (laptop/mini-PC class), which keeps the idle power draw low — important for a machine that runs 24/7.

---

## Initial PVE Setup

### Network Bridge

PVE uses a Linux bridge (`vmbr0`) to give VMs and containers access to the LAN. The host's physical NIC is enslaved to the bridge, and the bridge interface holds the static IP.

`/etc/network/interfaces` on the PVE host:

```
iface vmbr0 inet static
    address  192.168.0.10/24
    gateway  192.168.0.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

Every VM and container gets a virtual NIC attached to `vmbr0`. From the guest's perspective it's just a regular LAN interface — DHCP or static, same subnet, same gateway.

### Storage

| Pool        | Type      | Total   | Used    | Used for                      |
| ----------- | --------- | ------- | ------- | ----------------------------- |
| `local`     | Directory | ~100 GB | ~25 GB  | ISOs, backups, CT templates   |
| `local-lvm` | LVM-Thin  | ~853 GB | ~147 GB | VM disks, CT root filesystems |

LVM-Thin allows thin provisioning: a 100 GB virtual disk doesn't consume 100 GB up front. Space is allocated on write, which matters when several VMs are all over-provisioned relative to actual usage.

### Authentication

Single user: `root@pam` — Proxmox's PAM realm maps directly to the Linux `root` account. No additional realms or LDAP needed for a single-owner setup.

---

## Guest Layout: The Reasoning Behind Each Choice

### 1. Ubuntu Server — Main Application Host

Ubuntu LTS is the path of least resistance for most self-hosted applications. Package support is wide, container images default to it, and documentation assumes it. Nothing exotic here — it runs the application services that don't have hard OS requirements.

### 2. Rocky Linux — Oracle Database 19c

This one required deliberate thought. Oracle Database 19c is only certified on RHEL and its binary-compatible derivatives (Oracle Linux, Rocky Linux, AlmaLinux). Ubuntu is explicitly unsupported — Oracle will refuse a support case if you're running on it, and the RPM-based installer simply won't proceed cleanly on a Debian system.

The obvious alternative was to run Oracle inside a Docker container on the Ubuntu VM. Oracle publishes official container images, so this does work. But it introduces a problem: you're now running a container inside a VM — two abstraction layers for a database that already demands significant resources and careful I/O tuning. Container networking adds latency, volume mounts add indirection, and troubleshooting I/O issues becomes harder when you can't tell where in the stack the bottleneck lives.

The cleaner solution: add a Rocky Linux LXC container. Rocky Linux is a downstream rebuild of RHEL with no license cost. The Oracle installer treats it as RHEL and proceeds normally. The LXC container is lighter than a full VM — it shares the host kernel — so the overhead is lower than running a nested container inside a VM.

LXC containers on PVE do require `privileged: true` for Oracle's installation scripts to work (they need kernel parameter tweaks via `sysctl`). That's an acceptable trade-off in a home environment.

### 3. Windows 11 — MetaTrader 5 Python SDK

MetaTrader 5's Python integration (`MetaTrader5` pip package) is a Windows-only library. It communicates with the MT5 terminal process via a named pipe — a Windows IPC primitive. There is no Linux build, and the API is not documented for Wine.

Running it under Wine on the Ubuntu container is technically possible: install Wine, install Python for Windows, install MT5 terminal, install the pip package inside the Windows Python environment, then try to get the named pipe IPC to work through Wine's IPC layer. People have gotten partial versions of this running, but it is fragile and unsupported.

For the same reason I rejected the nested-container approach for Oracle — adding virtualization layers to work around an OS restriction when the right OS can be a first-class guest — I added a Windows 11 VM instead. The MT5 SDK works as documented. No Wine-specific debugging.

The trade-off is resource cost: a Windows 11 VM with the MT5 terminal idle consumes roughly 2–3 GB RAM. On 16 GB with three other guests, this is manageable.

---

## Network Architecture

```
Internet
  └─ Cloudflare Edge (proxy + Zero Trust)
        └─ Home Router (port 80/443 forwarded to PVE host)
              └─ PVE Host at 192.168.0.10 (nginx)
                    ├─ Ubuntu Server at 192.168.0.11
                    │     ├─ :4000  (notes viewer)
                    │     └─ :8080  (application)
                    └─ PVE Web UI at 192.168.0.10:8006
```

The PVE host also acts as the nginx reverse proxy host. The router forwards ports 80 and 443 to `192.168.0.10`. nginx routes by `server_name` (the `Host` header) to the appropriate upstream.

---

## Cloudflare Configuration

### DNS Proxy (Orange Cloud)

All subdomains used for this setup have the Cloudflare proxy enabled. This means:

- Public DNS resolves to Cloudflare's anycast edge IPs, not the home IP
- Cloudflare absorbs DDoS, bot traffic, and port scans before they reach the home router
- The home router's public IP is not exposed in DNS

The side effect: nginx sees Cloudflare's edge IPs as the client IP, not the real visitor's IP. This is corrected by the `cloudflare_address_list.conf` snippet covered in the nginx section below.

### Zero Trust Access (Cloudflare One)

Services that should be reachable over the internet but restricted to a single authenticated user are protected via Cloudflare Zero Trust Access:

1. In the Cloudflare Zero Trust dashboard, create an **Access Application** for the subdomain.
2. Set the policy to identity-based: allow only the owner's account (email/SSO).
3. When a browser hits the subdomain, Cloudflare intercepts before the request reaches nginx and prompts for login.
4. After authentication, Cloudflare issues a signed JWT in a cookie. nginx only sees already-authenticated requests — it never handles the login flow itself.

This replaces nginx-level authentication (Basic Auth, PAM) for externally exposed admin services. The PVE web UI (`pve.mydomain.com`) is a concrete example: it's protected by Zero Trust rather than adding an nginx `auth_basic` block in front of it.

---

## nginx Configuration

nginx runs on the PVE host and handles all inbound HTTP/HTTPS traffic.

### Real IP Restoration

Since traffic passes through Cloudflare's proxy, the `$remote_addr` nginx sees is a Cloudflare edge IP. The actual client IP is in the `CF-Connecting-IP` header. The `set_real_ip_from` directives trust Cloudflare's published IP ranges, and `real_ip_header` tells nginx to replace `$remote_addr` with the header value.

`/etc/nginx/conf.d/cloudflare_address_list.conf`:

```nginx
# Cloudflare IPv4
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;

# Cloudflare IPv6
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2a06:98c0::/29;
set_real_ip_from 2c0f:f248::/32;

real_ip_header CF-Connecting-IP;
```

Cloudflare publishes its current IP ranges at `https://www.cloudflare.com/ips/`. The list above should be refreshed if Cloudflare adds new ranges — the canonical source is always their published list, not this config file.

### Default Catch-All (Silent Drop)

Any request that doesn't match a configured `server_name` is silently dropped with status `444`. This prevents nginx from serving a default page to scanners probing random hostnames.

`/etc/nginx/sites-available/00-default.conf`:

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate /etc/nginx/certificate/mydomain.com.pem;
    ssl_certificate_key /etc/nginx/certificate/mydomain.com.key;

    return 444;
}
```

### Shared Proxy Directives

Every virtual host uses the same set of proxy headers and settings. These could be extracted to an `include` file but are shown inline here for clarity.

```nginx
# WebSocket support
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";

# Disable response buffering (required for real-time UIs, SSE, etc.)
proxy_buffering off;

# Standard forwarding headers
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# SEO prevention — instruct crawlers not to index or archive
add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
```

The `X-Robots-Tag` header is a safeguard: even if a search engine somehow crawls an internal service, it won't index it.

### Routing Table

| Domain             | Upstream            | Internal Protocol |
| ------------------ | ------------------- | ----------------- |
| `pve.mydomain.com` | `192.168.0.10:8006` | HTTPS             |
| `not.mydomain.com` | `192.168.0.11:4000` | HTTP              |

### Proxmox VE Reverse Proxy Quirks

The PVE virtual host has three requirements that differ from a typical HTTP service:

**1. Upstream must be HTTPS.** PVE's built-in web server does not listen on plain HTTP — it only serves on port 8006 over TLS. nginx needs `proxy_pass https://` and `proxy_ssl_verify off` (PVE uses a self-signed cert by default).

**2. `proxy_redirect` must rewrite PVE's redirect headers.** PVE issues HTTP redirects to paths like `/` during login flows. Without the rewrite, the `Location` header points to `https://192.168.0.10:8006/` — the internal address — instead of the public domain. nginx's `proxy_redirect` fixes this.

**3. WebSocket is required for the console.** PVE's noVNC and SPICE console use WebSocket connections. Without proper `Upgrade` handling, the VM console opens a blank pane and immediately disconnects.

`/etc/nginx/sites-available/pve.mydomain.com.conf`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name pve.mydomain.com;

    ssl_certificate /etc/nginx/certificate/mydomain.com.pem;
    ssl_certificate_key /etc/nginx/certificate/mydomain.com.key;

    # Allow large uploads — ISO files through the PVE web UI
    client_max_body_size 1G;

    location / {
        proxy_pass https://192.168.0.10:8006;
        proxy_ssl_verify off;

        # Rewrite PVE's internal redirect headers to the public domain
        proxy_redirect https://192.168.0.10:8006/ /;

        # WebSocket support (required for noVNC/SPICE console)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name pve.mydomain.com;

    return 301 https://$host$request_uri;
}
```

The `client_max_body_size 1G` directive is necessary if you want to upload ISO images through the PVE web interface. The default nginx limit is 1 MB, which will cause the upload to fail with a `413 Request Entity Too Large` without a clear error in the UI.

### A Standard HTTP Upstream (Notes Viewer)

For contrast, a plain HTTP upstream with no special requirements:

`/etc/nginx/sites-available/not.mydomain.com.conf`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name not.mydomain.com;

    ssl_certificate /etc/nginx/certificate/mydomain.com.pem;
    ssl_certificate_key /etc/nginx/certificate/mydomain.com.key;

    location / {
        proxy_pass http://192.168.0.11:4000;

        add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name not.mydomain.com;

    return 301 https://$host$request_uri;
}
```

The structure is identical to the PVE config minus the HTTPS upstream handling and the large body limit.

---

## SSL Certificate

A wildcard certificate for `*.mydomain.com` covers all subdomains with a single cert/key pair. Both the PEM certificate and private key are stored at `/etc/nginx/certificate/`. The wildcard means adding a new subdomain doesn't require provisioning or deploying a new certificate — just a new nginx `server` block pointing at the same files.

---

## What This Stack Solves (and Doesn't)

**What it solves well:**

- OS-level isolation without a cloud subscription — each workload gets the OS it actually needs
- Single public-facing entry point: one router port forward, one nginx process, one certificate
- Real IP visibility preserved through Cloudflare proxy via `CF-Connecting-IP`
- Zero Trust access control without adding auth logic to individual services
- PVE console works through the proxy (WebSocket + HTTPS upstream properly handled)

**What to be aware of:**

- The PVE host doubles as the nginx host — a misconfigured nginx restart could briefly cut off the PVE web UI itself. On a single-node cluster this is acceptable; at scale you'd separate the proxy from the hypervisor
- Cloudflare Zero Trust is the only authentication layer for admin services. If your Cloudflare account is compromised, that layer is gone. MFA on the Cloudflare account is non-optional
- The Cloudflare IP list in `cloudflare_address_list.conf` is static. Cloudflare occasionally expands it — worth checking against the canonical list when setting this up fresh
