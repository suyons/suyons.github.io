---
title: "nginx Troubleshooting - Three Small Fixes Between a Blank Page and a Working Login"
date: 2026-10-02
draft: false
tags: ["nginx", "reverse-proxy", "angular", "webpack-dev-server", "windows"]
categories: ["Infrastructure"]
description: "Putting nginx in front of an Angular dev server and its API looked like a five-minute reverse proxy. It took three separate, differently-silent failures before a real login worked."
showToc: true
---

The goal looked trivial: put nginx on Windows in front of an Angular dev server and its backend API, so a browser only ever talks to one origin. nginx listens on one public port, forwards everything to the dev server, and routes `/api/` to the backend. Three small, unrelated failures stood between that plan and an actual working login.

## The base configuration

A second `server` block went into `nginx.conf`, leaving the default one on port 80 untouched:

```nginx
server {
    listen       0.0.0.0:33040;
    server_name  _;

    location / {
        proxy_pass http://127.0.0.1:4200;
        # websocket upgrade so the dev server's live reload keeps working
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /api/ {
        proxy_pass http://127.0.0.1:31040;
    }
}
```

Because `proxy_pass` in the `/api/` block has no URI path of its own, the `/api` prefix is forwarded to the backend unchanged — no `rewrite` needed. The Windows firewall also needed an inbound rule for the public port, or nothing outside the machine could reach it at all.

`nginx -t` passed immediately. That checks syntax only — it proved nothing about whether nginx was actually running. It wasn't; `nginx -s reload` failed with an "invalid PID" error, which is nginx's way of saying there's no master process to signal. Starting it fresh was the real fix, and the first lesson of the session: a clean `-t` is not a clean bill of health.

## Fix 1: "Invalid Host header"

The first request to the public hostname came back with a 19-byte page: `Invalid Host header`. The response also carried `X-Powered-By: Express`, which is easy to misread as "wrong app entirely." It wasn't — it was webpack-dev-server, which Angular's CLI uses under the hood and which happens to be built on Express.

webpack-dev-server refuses any `Host` header that isn't localhost, as a defense against DNS-rebinding attacks. nginx was forwarding the original public hostname straight through, so the dev server rejected every request on principle:

```nginx
# Before
proxy_set_header Host $host;

# After: send the upstream's own host instead of the public one
proxy_set_header Host $proxy_host;
```

The alternative is whitelisting the public hostname in the Angular dev-server's `allowedHosts` setting. Rewriting one nginx header was the smaller change and left the app's own config untouched — worth doing when the proxy is the thing you control and the app config isn't, or isn't worth touching for a dev-only setup.

## Fix 2: 502 on login

The page now loaded fine. The login `POST` came back as a 502, even though the backend was clearly listening and handling the request — curling it directly worked. The nginx error log had the actual reason:

```
upstream sent too big header while reading response header from upstream
```

The login response's headers — a large `Set-Cookie`, most likely, or an oversized auth token — exceeded nginx's default proxy buffer, which defaults to one memory page (4k or 8k, platform-dependent). Once the headers alone overflow that buffer, nginx can't read the rest of the response and just gives up:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:31040;
    proxy_buffer_size        32k;
    proxy_buffers            8 32k;
    proxy_busy_buffers_size  64k;
}
```

This is the kind of failure that only shows up once a response actually carries a large header — a login, specifically, is exactly where that tends to happen, since it's the response most likely to set a session cookie or hand back a token.

## Outcome and takeaways

After both fixes and a reload, login worked end to end: one public port, front end and API behind a single origin, firewall open for it.

- **A 502 from nginx tells you almost nothing by itself.** The error log names the actual reason. Read it before guessing at proxy config.
- **`X-Powered-By: Express` on a dev-server response doesn't mean you hit the wrong app.** webpack-dev-server is built on Express; that header will show up for any Angular (or Vue, or Create React App) dev server behind a proxy.
- **A dev server sitting behind a reverse proxy usually needs its `Host` header rewritten, or its allowed-hosts list extended.** Both work; changing the proxy is less invasive when the app config isn't yours to touch.
- **`nginx -t` only validates syntax.** It says nothing about whether the process is running or whether the upstreams actually respond. Send a real request after every change, not just a syntax check.
- One open item: nginx was started as a plain foreground-ish process and won't survive a reboot. Running it as a proper Windows service is the obvious next step if this setup needs to stick around.
