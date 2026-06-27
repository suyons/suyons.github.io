---
title: "Infrastructure - Three Container Traps When You Don't Own the Image"
date: 2026-06-27
draft: false
tags: ["docker", "containers", "security", "alpine", "musl", "glibc", "onlyoffice", "nginx", "self-hosting"]
categories: ["Infrastructure"]
description: "Alpine won't load your vendor's glibc-compiled native libraries. A secret passed inline shows up in ps. And a config edit inside a vendor container evaporates on the next image pull. Three lessons from the first day of running two commercial services in containers."
showToc: true
---

## When you deploy a service you didn't build

Deploying containers you built yourself is one problem. Deploying containers built by vendors — where you can't change the source, the image comes from someone else's registry, and the default configuration is optimized for their demo, not your production setup — is a different problem. The traps are different too.

This post is about three mistakes made on the first day of running two commercial document services in containers on the same Ubuntu host. None of them were obscure edge cases; all of them are things you'd hit the first time.

## Alpine won't load a glibc-compiled native library

The instinct when picking a base image for a container you build yourself is to reach for Alpine: small image, fast pulls, sensible defaults. For most things this is fine.

For services that ship **native libraries** — `.so` files compiled for a specific C standard library — it can be a silent but fatal choice. Alpine uses **musl libc**. Most Linux distributions (Debian, Ubuntu, RHEL, CentOS) use **glibc**. A native library compiled against glibc will not load on a musl system:

```
error while loading shared libraries: libSomeVendorLib.so: cannot open shared object file
```

The LEADTOOLS document processing SDK ships native `.so` libraries compiled against glibc. The container for it must use a glibc base. The right starting point is something like `tomcat:10.1-jdk21-temurin` — Debian-based, which pins the servlet container version and the JDK alongside a glibc that can actually load the vendor's libraries.

The Alpine "optimization" would have saved some disk space. It wouldn't have saved any runtime cost — containers aren't virtual machines, there's no hypervisor overhead to reduce. And it would have produced a container that fails at startup, not at build time, which is the worst way to discover a base image was wrong.

**The rule:** if a vendor ships native binaries, check which libc they were compiled against before picking a base image. `file <library.so>` and `ldd <library.so>` both tell you. Alpine is off the table if the answer is glibc.

## A secret passed inline shows up in ps

Standing up the document editor for the first time, a JWT secret for the service's API was configured like this in the systemd unit that runs the container:

```
--env JWT_SECRET=${JWT_SECRET}
```

This looks fine — it's pulling the value from the environment, not hardcoding it. But the `${JWT_SECRET}` substitution happens at shell evaluation time, before the container starts. The expanded value gets passed as a literal command-line argument. That means:

```
$ ps aux | grep container
... docker run --env JWT_SECRET=<the-actual-value> ...
```

Anyone with access to process listings on the host — or anything that logs command lines — can read the secret. No exploit required.

The fix is a one-character change: **pass the variable name, not the value.**

```
# Before: value is interpolated onto the command line
--env JWT_SECRET=${JWT_SECRET}

# After: pass-through — the daemon reads the value from the unit's environment, off the command line
--env JWT_SECRET
```

When you omit the `=value`, Docker reads the variable from its own environment and injects it into the container without ever putting the value on the command line. The secret stays out of `ps`, out of audit logs, out of anything that records argv.

After switching, rotate the leaked value. Then verify the container actually received it — you can confirm the variable is set without printing its value:

```bash
docker exec <container> sh -c '[ -n "$JWT_SECRET" ] && echo "set (len=${#JWT_SECRET})"'
```

This pattern — pass secrets by name, not by value — applies to any `--env` flag in `docker run`, any `environment:` entry in Compose, and any equivalent in Podman or Kubernetes.

## The right layer for disabling a vendor welcome page

Both services shipped a browser welcome page at `GET /`. Both needed to be API-only. The obvious move — find the file inside the running container and delete it — is exactly wrong, and for a reason that's easy to miss the first time.

**Editing a running container is impermanent.** The container's writable layer is discarded when it's recreated. On the next `docker pull` and restart, the welcome page comes back. You are not changing the image; you're leaving a note on a whiteboard that gets erased every time someone enters the room.

The right layer depends on who owns the image.

**For a service you build yourself**, the right layer is build time. Exclude the welcome page from the web archive before packaging:

```xml
<!-- In the Maven WAR plugin config: exclude the SDK's demo landing page at build time -->
<packagingExcludes>index.html,index.js</packagingExcludes>
```

`GET /` returns `404`. The API and health endpoints are untouched. This change travels with every image you build; it doesn't need to be re-applied after an update.

**For a vendor image you don't build**, the right layer is the edge proxy — which you *do* own, and which survives any image update. Block the welcome path at nginx instead:

```nginx
location = /          { return 404; }    # exact match: bare root
location ^~ /welcome/ { return 404; }    # ^~ beats the vendor image's internal regex location
```

The `^~` modifier matters here. The vendor image's internal server likely uses a regex location to match the welcome path. In nginx, a regex location (`~` or `~*`) beats a plain prefix location — but `^~` suppresses that regex matching and wins regardless of what the upstream registers internally. Without it, the nginx location would be overridden by the upstream's regex.

There's a third option when even direct access (bypassing the proxy) must be API-only: build a thin image `FROM` the vendor image that removes the welcome file at build time, with a build-time assertion that fails loudly if the vendor restructures their config in a future release. This keeps the customization in a layer you own, makes it visible in your image history, and doesn't depend on the proxy being in the path.

## When "open the port" is actually a bind address

A related question that came up while getting both containers reachable from a separate proxy host: "open ports 10000 and 10001." The host firewall was already fully permissive — there was nothing to open.

The real issue was the container's published bind address:

```
# Before: loopback only — reachable only from the same host
PUBLISH=127.0.0.1:10001:8080

# After: all interfaces — the separate proxy host can reach it
PUBLISH=10001:8080
```

With Docker, published ports don't need a firewall rule — the daemon programs the packet filter rules itself. "Open the port" almost always means "change the bind address so the client can see it," not "add a firewall rule." The security implication follows directly: a plain-HTTP backend bound to all interfaces is reachable by anyone who can reach the host. The control belongs at the proxy's source — a host firewall rule that allows traffic only from the proxy's IP, or a cloud security group rule that does the same.

## Takeaways

- **Check the libc before choosing a base image** for any service that ships native libraries. A vendor SDK compiled against glibc will not load on Alpine's musl. `file` and `ldd` tell you up front; a failing container tells you later.
- **Pass secrets by name, not value.** `--env VAR` injects the value without putting it on the command line; `--env VAR=${VAR}` expands it onto argv where `ps` and log collectors can see it. Rotate anything that was ever passed the wrong way.
- **Editing a running container is not a durable change.** The right layer depends on who owns the image: build time if you own the build, the edge proxy if you don't, or a thin image layered on top of the vendor image if you need it off the proxy path too.
- **Docker manages packet-filter rules for published ports.** "Open the port" is usually a bind-address decision, not a firewall rule. Bind to `127.0.0.1:<port>` to keep a service local; omit the address to make it reachable from outside the host. Lock plain-HTTP backends to the proxy's source address — the daemon won't do that for you.
