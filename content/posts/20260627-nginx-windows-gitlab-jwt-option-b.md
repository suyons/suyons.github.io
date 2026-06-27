---
title: "Windows Server Troubleshooting - The nginx Proxy That Fell Apart at the JWT Boundary"
date: 2026-06-27
draft: false
tags: ["nginx", "windows-server", "gitlab", "docker", "self-hosting", "tls", "chocolatey", "jwt", "nssm"]
categories: ["Infrastructure"]
description: "Chocolatey nginx isn't apt nginx. NSSM-hosted services can't receive signals from your shell. And a proxy_redirect-plus-sub_filter edge works right up until the registry needs a signed JWT. Here's the full arc from a manual D:\\nginx to a working GitLab container registry."
showToc: true
---

## The goal was a boring afternoon

Install nginx on Windows Server 2019, give it TLS, put a self-hosted GitLab behind it. Nothing novel. The interesting parts were three places where the Linux mental model was quietly wrong, and one place where a clever workaround hit a hard ceiling.

## Read before write: the box wasn't empty

The plan was `choco install nginx`. Before running it, I checked what was already there — and nginx was already there. A manual install at `D:\nginx` (version 1.24) was acting as a live reverse proxy for an existing service, with NSSM already present from a previous Chocolatey install.

Running `choco install` without knowing this would have planted a *second* nginx under `C:\tools\nginx-1.31.2`. Its Chocolatey shim would have shadowed the manual one on PATH; every subsequent `nginx -t` would have tested the wrong config. So the read happened first, the collision was spotted, and the operator decided to start clean by deleting `D:\nginx`. That deletion was an informed choice — not a surprise.

## Chocolatey isn't apt, and nginx config isn't exempt

On Linux, `apt upgrade nginx` is boring: the binary lands at `/usr/sbin/nginx`, config stays at `/etc/nginx`, the systemd unit doesn't change. Chocolatey is structurally different. Each version installs into its own folder (`C:\tools\nginx-1.31.2`), and the default config lives *inside* that folder. If your Windows service is pinned to the old binary path, `choco upgrade nginx` adds a new folder and the service keeps launching the old binary. Your config changes are in a directory that nothing reads anymore.

The fix is to reconstruct, by hand, the three things apt keeps stable:

**A persistent config prefix.** Move config, logs, and temp into `C:\nginx`, and launch nginx with `nginx -p C:\nginx`. Config now survives version changes.

**A stable binary pointer.** The Chocolatey nginx package creates no shim, so there's no auto-updating wrapper to exploit. Instead: a directory junction `C:\nginx-bin` → `C:\tools\nginx-<current>`. The NSSM service path points to the junction; the junction points to whichever version just installed.

**An upgrade hook.** A Chocolatey post-install/upgrade hook (`hooks\post-{install,upgrade}-nginx.ps1`) repoints the junction to the new version, runs `nginx -t`, and restarts the service. After this, `choco upgrade nginx` behaves the way `apt upgrade nginx` does: swap the binary, keep the config, restart.

## nginx -s reload fails from your shell

The service runs under NSSM as LocalSystem. That's fine — until you try to reload config. Running `nginx -s reload` from an interactive shell returns:

```
OpenEvent("Global\ngx_reload_<pid>") failed (5: Access is denied)
```

The mechanism: `nginx -s reload` opens a named event the master process is watching. The master process lives in **session 0** as SYSTEM. Your interactive shell is in a different session. Windows doesn't let your session open an event owned by session 0. The signal never arrives; nginx never reloads.

The fix: a Windows scheduled task named `nginx-reload`, configured to run as SYSTEM, that fires the signal from the right session. An `nginx.cmd` wrapper placed on PATH intercepts the `-s reload` subcommand and routes it through this task; all other flags (`-t`, `-v`, `-s stop`) pass straight through to the binary with the correct `-p C:\nginx` prefix.

From the operator's point of view, `nginx -s reload` works. The session-0 routing is invisible.

## TLS: HTTP-01 over per-host, and a Windows path quirk

A wildcard certificate (`*.example.com`) needs the ACME **DNS-01** challenge, which requires DNS provider API access or a manual TXT record on every renewal. Per-host **HTTP-01** is fully automatable with no DNS access needed — at the cost of being per-subdomain rather than wildcard. That was the pragmatic choice here.

Two details worth keeping:

**Bootstrap with a self-signed placeholder.** `nginx -t` fails if a config block references a certificate file that doesn't exist yet. Put a self-signed cert in place first, validate the entire HTTPS config against it, then swap in the real certificate. No further config edits needed. As a side effect, the catch-all server block — which returns `444` to any SNI that doesn't match a configured vhost — gets a self-signed cert permanently and never needs a real one.

**Absolute paths for certificates on Windows.** nginx resolves relative `include` paths against the *config directory*, but relative `ssl_certificate` paths against the *prefix* (`C:\nginx`). On a standard Linux install these coincide; on Windows they're different directories. The safe rule: use absolute paths for `ssl_certificate` and `ssl_certificate_key`.

`certbot` isn't available on Chocolatey. `win-acme` handles the webroot challenge: it writes the ACME token to `C:\nginx\challenge`, served from a `^~ /.well-known/acme-challenge/` location that's explicitly exempt from the HTTP→HTTPS redirect. It exports PEM files and registers a SYSTEM daily renewal task that fires the same `nginx-reload` task on success.

## GitLab's one-identity rule: why Option B seemed fine

GitLab was running in a Docker container on a NAS on the same LAN, configured via environment variables (the entrypoint regenerates all config from env on boot — edits inside the container don't persist). It had been told its host was an internal IP address and its scheme was `http`, so it emitted `Location: http://gitlab.example.com/...` redirect headers — a downgrade that would cause an https→http→https loop from external clients and serve broken clone URLs.

The core constraint here is hard: **GitLab supports exactly one canonical URL, and the scheme is global.** Tell it `GITLAB_HTTPS=true` and it forces `https://` everywhere, including when accessed by its internal IP — which speaks no TLS. You cannot have it be HTTP internally and HTTPS externally at the same time.

Two options:

**Option A** — fix it at the app. Update four env vars (`GITLAB_HOST=gitlab.example.com`, `GITLAB_PORT=443`, `GITLAB_HTTPS=true`, `SSL_SELF_SIGNED=false`) and recreate the container. Clean. Requires stopping the container.

**Option B** — fix it at the edge. Leave the container untouched and do all the http→https translation in nginx.

The operator's constraint: the container must not stop. Option B.

The nginx proxy location needed several pieces working together:

```nginx
proxy_redirect http://gitlab.example.com/      https://gitlab.example.com/;
proxy_redirect http://<nas-ip>:<port>/         https://gitlab.example.com/;

proxy_cookie_flags ~ secure;          # session cookie was missing the Secure flag

proxy_set_header Accept-Encoding "";  # allow sub_filter to decompress and read the body
sub_filter_once off;
sub_filter_types text/css text/javascript application/javascript application/json;
sub_filter "http://<nas-ip>:<port>"      "https://gitlab.example.com";
sub_filter "http://gitlab.example.com"   "https://gitlab.example.com";
```

Verified end-to-end: `Location` headers were `https://`, the sign-in response body had no `http://` internal references, and the session cookie carried `HttpOnly; Secure`. Option B was working.

One latent bug surfaced during this config work: a `location` block with its own `add_header` directive **does not inherit `add_header` directives from the server block**. HSTS was declared at the server level; `location /` had its own `add_header` for something else, so HSTS was silently not being sent. Fixed by re-declaring HSTS inside the location. This is a well-known nginx behavior, but it bites in exactly the cases where you're adding a security header and then touching the location for unrelated reasons.

## The registry exposed Option B's ceiling

The GitLab Container Registry is not a separate subdomain. The Docker client always talks to `/v2/` — the same host, cert, and port 443 as the web UI. What was needed was a single additional `location /v2/` block in the existing vhost:

```nginx
location /v2/ {
    proxy_pass http://<nas-ip>:<registry-port>;  # no trailing slash — preserve the /v2/ prefix
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;
    proxy_read_timeout 900s;
    proxy_send_timeout 900s;
}
```

Large image layers are multi-GB. They must stream, not buffer; the long timeouts are not optional.

The registry is a separate `registry:2` container, and the two are wired together by a **JWT token triangle**:

1. `docker push` hits `/v2/`.
2. Gets `401` with `WWW-Authenticate: Bearer realm="https://gitlab.example.com/jwt/auth", service="container_registry"`.
3. Docker fetches a token from GitLab's `/jwt/auth` endpoint.
4. Presents it to the registry, which validates it using a **shared signing certificate** mounted into both containers.

For step 3 to produce a *valid* token, GitLab must know its own canonical URL is `https://...` and that the registry exists. That is exactly the self-knowledge Option B deliberately withheld — Option B's entire premise was that GitLab still thought it was `http://<nas-ip>:<port>`. You cannot `sub_filter` your way to a cryptographically signed token.

**The registry forced Option A.** There was no workaround.

A debugging note from wiring this up: it was a legacy `--link` (bridge) deployment, so container-name DNS doesn't resolve the way it would on a user-defined network. The registry URL GitLab calls had to be the host's IP, not the container name. And a `/v2/` health check that returned an SSH banner — instead of the expected `401` with a Bearer realm — caught a port mix-up early. One port was GitLab's SSH; the registry was on the next one. Verify which port is which before assuming the service is misconfigured.

## Recreating a stateful container without losing the instance

Recreating a GitLab container that holds real data and live 2FA secrets is where careful beats fast.

The safety approach:

1. Run `runlike` on the existing container to reconstruct the exact `docker run` command. That output is the rollback artifact — not the starting point.
2. Preserve `GITLAB_SECRETS_*` (three values) verbatim. These seed every encrypted token, session secret, and 2FA TOTP secret in the database. Regenerate them and everything encrypted is permanently unreadable.
3. Layer *only* the new variables on top: `GITLAB_HOST`, `GITLAB_PORT=443`, `GITLAB_HTTPS=true`, `SSL_SELF_SIGNED=false`, the five `GITLAB_REGISTRY_*` variables, and a read-only mount of the shared signing key.

One operational footnote: delivering a multi-line shell script over a chat window is a reliability problem. Line-continuation characters get mangled by copy-paste; a base64 blob was flatly rejected. The thing that worked was the simplest: write a plain `.sh` file to disk and let the operator run it. When someone is frustrated, reduce the complexity of the transport, not the content of the script.

Verification from the nginx host, using `--resolve gitlab.example.com:443:127.0.0.1` to bypass DNS:

- `https://gitlab.example.com/` → `302` to the login page. Native HTTPS. No proxy tricks in the response path.
- `https://gitlab.example.com/v2/` → `401` with `Bearer realm="...jwt/auth", service="container_registry"`. The triangle is wired.
- `https://gitlab.example.com/jwt/auth?...` → `403`. GitLab is alive and enforcing — refusing an anonymous push rather than returning `404` on an unknown route.

With Option A live, the Option B block became dead weight. The `proxy_redirect` directives, the `sub_filter` blocks, and — critically — the `Accept-Encoding ""` override that was decompressing every upstream response just so `sub_filter` could read the body: all gone. The vhost now reads as what it is.

## Takeaways

- **Chocolatey installs to versioned directories; config lives inside them.** To get `choco upgrade` to swap the binary without orphaning your config, reconstruct what apt provides for free: a stable config prefix (`nginx -p`), a stable binary pointer (a directory junction, since Chocolatey creates no shim for nginx), and an upgrade hook that repoints the junction and restarts the service.
- **nginx running as SYSTEM in session 0 can't receive signals from your interactive session.** The `Access is denied` from `nginx -s reload` is Windows session isolation. Route the reload through a SYSTEM scheduled task and hide it behind the native command.
- **Bootstrap TLS with a self-signed placeholder first.** `nginx -t` rejects a config that references a certificate that doesn't exist. Validate the whole HTTPS configuration against a throwaway cert; swap in the real one afterward.
- **Use absolute paths for ssl_certificate on Windows.** nginx's prefix-relative resolution for cert directives diverges from its config-relative resolution for includes. The paths coincide on Linux; they don't on Windows.
- **`add_header` in a location block does not inherit server-level `add_header` directives.** A security header set at the server level is silently dropped for any location that sets its own. Always re-declare inside the location.
- **A reverse-proxy translation layer has a ceiling: anything cryptographic.** `proxy_redirect` and `sub_filter` can rewrite URLs in headers and bodies. They cannot mint a signed JWT. The moment a feature needs the app to know its real canonical identity — registry auth, OIDC, webhook signatures — the edge-faking escape hatch closes. Fix it at the app.
- **The GitLab Container Registry shares the web UI's host and port.** It's one `location /v2/` block with streaming enabled, not a separate vhost or subdomain.
- **When recreating a stateful container, treat `runlike` output as your rollback, not your template.** Preserve `GITLAB_SECRETS_*` verbatim; layer only the new environment on top.
