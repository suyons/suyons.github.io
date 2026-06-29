---
title: "Docker Deployment - Ship a Vendor SDK Image as a Kilobyte Patch"
date: 2026-06-27
draft: false
tags: ["docker", "containers", "registry", "dockerfile", "ci-cd", "self-hosting", "maven"]
categories: ["Infrastructure"]
description: "When your image bakes in a vendor SDK, every patch ships gigabytes — unless you order the Dockerfile so stable layers sit below volatile ones, and split into a base image and a thin app image. Also: when docker login fails with 'no route to host' on an address you never typed, look at the WWW-Authenticate realm in the /v2/ 401."
showToc: true
---

## The patch that ships gigabytes

Our service bundles a third-party document processing SDK — native `.so` libraries, system packages, the servlet container — all baked into the image. The first build is unavoidably large, and that's fine. The problem was every subsequent patch. Change one line in application code, rebuild, push to the registry: gigabytes go over the wire. Pull on the server: gigabytes come back. For a service under active development, this gets painful fast.

The image size isn't the problem. The layer order is.

## Layer ordering: stable down, volatile up

Docker builds images as a stack of layers. Each `RUN`, `COPY`, and `ADD` instruction is one layer. The registry's deduplication skips unchanged layers on push and pull — but only if the layer cache is intact. **Any change to a layer invalidates every layer above it.** If you copy the application artifact early, then install heavy dependencies below it, every rebuild forces the heavy layers to retransfer even though they didn't change.

Before the fix:

```dockerfile
# Artifact is early — invalidates everything below it on every patch
FROM tomcat:10.1-jdk21-temurin
COPY ROOT.war /usr/local/tomcat/webapps/   # changes on every patch...
RUN apt-get update && apt-get install -y fontconfig libgomp1 freetype2-demos
COPY native-libs/ /opt/leadtools/lib/      # ...so these heavy layers rebuild too
```

After reordering:

```dockerfile
# Stable layers first, volatile artifact last
FROM tomcat:10.1-jdk21-temurin
RUN apt-get update && apt-get install -y fontconfig libgomp1 freetype2-demos
COPY native-libs/ /opt/leadtools/lib/      # cached across patches
COPY ROOT.war /usr/local/tomcat/webapps/  # only this small layer changes per patch
```

The principle is just: stable things go low in the stack, volatile things go high. The registry's deduplication does the rest.

## The base/app split: take it further

Layer ordering helps within a single image, but there's a ceiling. A fresh machine — a new CI runner, a production host that hasn't seen the image yet — still has to pull everything. And there's a build-host problem specific to vendor SDKs: the native libraries often aren't redistributable. You can't bake them into an image in CI without the SDK present, which rules out the "build everywhere" model.

The solution is two images:

**Base image** — the OS, runtime, system packages, and native SDK libraries. Rebuilt rarely, only when the vendor releases a new SDK version. Built once on a machine that has the SDK, pushed to the registry. After that, it sits there.

**App image** — `FROM base`, plus the deployable artifact. Rebuilt on every patch. Can be built anywhere — CI included — because the non-redistributable SDK is already in the base and doesn't need to be present on the build host.

The push output for a patch build shows exactly what happens:

```
2e3ed237b280: Mounted from <registry>/base    # base layers already in the registry — skipped
<app-layer-digest>: Pushed                     # only the application artifact transfers
```

Production pulls look the same: base layers are already cached, only the new app layer downloads. A patch that used to move gigabytes now moves a WAR file.

Rollback is "pull the previous app tag, restart." Nothing about the SDK layer needs to change.

## Registry login fails on an address you didn't type

After moving the registry behind HTTPS, `docker login` should work. It didn't. The error:

```
Get "http://<registry-host>:30000/jwt/auth?service=container_registry":
    dial tcp <registry-host>:30000: connect: no route to host
```

That internal plain-HTTP address is not something we typed on the client. The registry's two-hop auth handshake is where the problem lives:

1. `docker login` (or `docker push`) hits `/v2/`.
2. The registry returns `401` with a `WWW-Authenticate` header naming a *token realm* URL.
3. Docker fetches a bearer token from that realm.
4. The token is presented back to the registry.

We had wrapped step 1 in HTTPS, but the registry's `401` response in step 2 still advertised an internal HTTP realm — an address unreachable from outside the LAN. Docker faithfully went there and hit a dead end.

"No route to host" is not a firewall problem. It's that the realm URL in the `WWW-Authenticate` header points somewhere the client can't reach. Inspect it directly:

```bash
curl -sv https://<registry>/v2/ 2>&1 | grep -i www-authenticate
```

If the `Bearer realm=` value is an internal or plain-HTTP address, that's the thing to fix — on the **registry server**, in its advertised token realm configuration, not on the client. Once the realm points to the external HTTPS URL, login works.

## Delete the workaround once the root cause is gone

Moving the registry to HTTPS also exposed a stale mitigation in our Maven settings. An earlier build-tool error — "plain-HTTP repository blocked" — had been neutralized with a mirror override:

```xml
<!-- Neutralized the build tool's HTTP-repository blocker -->
<mirror>
    <id>maven-default-http-blocker</id>
    <mirrorOf>dummy</mirrorOf>
    <url>http://0.0.0.0/</url>
    <blocked>false</blocked>
</mirror>
```

The registry is now HTTPS. The blocker never fires. This override is dead weight, and worse: it signals to the next reader that something is still wrong and requires a workaround. It doesn't. Delete it.

```xml
<!-- Nothing. The registry is HTTPS; the blocker doesn't apply. -->
```

This is a general pattern worth internalizing: when you fix the root cause, delete the mitigation. A workaround kept past its expiry date doesn't just add noise — it implies the problem it was masking is still present.

## Takeaways

- **Layer order is a performance decision.** Stable layers — runtime, system packages, native libraries — go low. Volatile layers — your application artifact — go high. The registry's deduplication handles the rest; you just need to stop invalidating the stable cache on every build.
- **A base/app split separates SDK lifecycle from application lifecycle.** The base is rebuilt on vendor releases; the app is rebuilt on every patch. The split also solves the build-host problem for non-redistributable native libraries: bake them into the base once, then app builds need only the artifact.
- **Registry auth failures showing "no route to host" on an address you didn't type are a realm problem, not a firewall problem.** Read the `WWW-Authenticate` header from the `/v2/` `401` response. Fix the realm URL on the registry server.
- **Workarounds have expiry dates.** When you fix the root cause, remove the mitigation. A mitigation in place after the problem is gone is misleading to everyone who reads it next.
