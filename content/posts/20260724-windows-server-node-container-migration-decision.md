---
title: "Windows Server Infrastructure - The Container Migration I Decided Not to Do"
date: 2026-07-24
draft: false
tags: ["windows-server", "docker", "nodejs", "pm2", "process-management"]
categories: ["DevOps"]
description: "Weighing Docker against a process manager's per-app interpreter for running five Node.js versions on one Windows Server host, and why the numbers killed the container migration before it started."
showToc: true
---

Should the production Node.js apps on our Windows Server move from a process manager to Docker? They run a spread of runtime versions — anywhere from 14 to 24 — and juggling that on one host is exactly the kind of mess containers are supposed to clean up. I ran the numbers and killed the idea. The return wasn't marginal; it wasn't close.

## The problem I thought I had

The framing was "multiple apps, incompatible Node versions, one Windows Server, this is getting unmanageable." Containers are the reflex answer: each app carries its own runtime, no host-level version conflicts, images as deployable artifacts, clean rollback. All genuinely good things.

But writing down the actual costs on *this* platform changed the picture, because Docker on Windows Server gives you exactly two paths and both are expensive.

**Native Windows containers.** The official Node images are Linux-only. Windows Node images are community-built and effectively unmaintained, which means building and maintaining my own base images on Server Core — for every version from 14 through 24, indefinitely. On top of that, the supported container runtime for Windows Server is a commercial product. So: a licensing line item, plus becoming the maintainer of a private base-image fleet, in exchange for isolation.

**Linux containers on Windows.** This needs a Linux virtual machine underneath. That works, and the hypervisor is first-party. But look at what it builds: a Linux server that happens to be hosted on a Windows box. At that point the Windows Server has been demoted to a hypervisor and contributes nothing else. If that's the destination, the honest move is a Linux host — not a Linux VM wearing a Windows costume.

## The cheaper answer was already installed

The decisive realization: PM2, the process manager already running in production, supports a **per-application interpreter**. Each app can point at its own Node binary, under a single daemon. That is precisely the feature I was about to build a container platform to obtain.

**Before** — one host runtime, apps fighting over it:

```js
// implicit: every app runs on whatever `node` resolves to
module.exports = { apps: [
  { name: 'legacy-service', script: 'server.js' },
  { name: 'new-api',        script: 'dist/main.js' },
]}
```

**After** — each app pinned to its own runtime, no new infrastructure:

```js
module.exports = { apps: [
  { name: 'legacy-service', script: 'server.js',
    interpreter: 'C:\\Program Files\\nodejs\\v14.21.3\\node.exe' },
  { name: 'new-api',        script: 'dist/main.js',
    interpreter: 'C:\\Program Files\\nodejs\\v22.14.0\\node.exe' },
]}
```

Install the runtimes side by side with a version manager and pin the **full version path**. Don't point at a "current" symlink — a stray version switch on the host should never be able to silently change what production is executing.

One caveat that applies either way: native modules compiled against a specific runtime must be rebuilt per version. Containers don't exempt you from that, so it isn't a point in their favor.

## The problem I actually had

Reframing the version spread as a support question rather than an operations question was clarifying. Most of that range no longer receives security patches at all — the older releases have been end-of-life for years, and even the mid-range one aged out earlier this year. Only the two newest lines in the spread are still maintained.

That reframes containerization as a non-solution. Wrapping an unpatched runtime in an image doesn't patch it; it freezes it and adds ceremony. The base images for those old versions are just as unpatched as the host installs. The real work isn't isolation — it's **consolidation**: move everything that can move onto a supported release, and use the per-app interpreter only for the stragglers that genuinely can't.

## Outcome & takeaways

Decision: no Docker on the Windows Server. Instead — consolidate apps onto currently-supported runtimes; use per-app interpreters for whatever can't move yet; and if containers become worthwhile later, adopt them on a Linux host where they're boring, rather than paying a platform tax to simulate that environment on Windows.

Worth being clear about what was *not* rejected. Containers are good technology and the motives were sound — reproducible builds and image-based rollback are real wins. What failed the ROI test was Docker *on Windows Server specifically*. Same tool, different platform, completely different economics.

Three things I want to remember:

- **Cost out the platform, not the technology.** "Should we use containers" has a famous answer. "Should we use containers *on Windows Server, for Node, across five major versions*" has a very different one, and only the second question was mine to answer. Best practices are context-dependent by definition; a recommendation that ignores the host platform isn't a recommendation.
- **Check what your existing tools already do before adopting new ones.** The winning feature had been sitting in the process manager's configuration schema the entire time. The gap wasn't capability — it was that I never read that far into the docs. The cheapest migration is the one you don't perform.
- **When a migration is proposed as a fix, verify it fixes the thing.** Containers would have organized the version sprawl beautifully while leaving every unpatched runtime exactly as unpatched. That's motion, not progress — and it's the most expensive kind, because it looks like progress on the way out the door.
