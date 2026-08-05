---
title: "Node.js Runtime Management - The Filesystem Doesn't Care What Your Process Manager Thinks Is Running"
date: 2026-07-30
draft: false
tags: ["node-js", "abi-compatibility", "pm2", "windows-server", "process-manager"]
categories: ["DevOps"]
description: "Consolidating five Node.js installations down to three meant proving nothing still depended on the ones being removed — and the process manager's own state turned out to be the least reliable place to check."
showToc: true
---

A Windows box had five Node.js installations feeding a PM2 process manager: 14.16.0, 14.21.3, 16.20.2, 24.16.0, and a standalone 16.20.2 unpacked into its own directory. The trigger was mundane — two schedulers had been saved with a bare `node` interpreter, meaning the next reboot would resurrect them onto whatever runtime happened to be first on the machine's `PATH`, not the one they were tested against. Fixing that turned into a full consolidation, and the hard part wasn't upgrading anything. It was proving that nothing still depended on the runtimes I was about to delete.

## ABI, not version, is the thing that has to match

Eight scheduler processes were still on 14.16.0, which is end-of-life. The obvious worry with moving them off it is native modules — these projects load JNI bridges, canvas, and database drivers, all compiled against a specific Node ABI (Application Binary Interface), and getting that wrong kills the process at startup with `ERR_DLOPEN_FAILED`.

The ABI number is versioned independently of the release number, and that's the whole trick: every Node 14.x release shares ABI 83, Node 16 is ABI 93, Node 24 is ABI 137. So 14.16.0 to 14.21.3 is a *same-ABI* move — the prebuilt native binaries load unchanged, no rebuild required. Rather than trust that reasoning on paper, I verified it by loading each native module under the target runtime before touching anything:

```
project A     java OK   canvas OK   oracledb OK
project B     java OK               oracledb OK
project C     java OK   canvas OK   oracledb OK
```

One project reported failures, and my first read of that was "regression." It wasn't. Its binaries were ABI 93 — built for a sibling app that runs on Node 16 — and they failed identically on the *current*, unchanged runtime. The scheduler process never actually imported them, so nobody had noticed. The question that separates a real finding from a scary-looking distraction is simple: does it fail the same way before your change? Here it did, which meant it was a pre-existing condition, not something I'd broken.

## A hang that wasn't a hang

The first run of that verification table hung indefinitely. Nothing crashed, nothing errored — the process just never returned.

The cause was in what "loading" a JNI bridge actually does: it starts a JVM, and that JVM spins up non-daemon threads. A Node process doesn't exit while non-daemon-equivalent work is still alive under it, so the module had in fact loaded fine — my test harness just had no way to say so and kept waiting for a process exit that was never coming on its own. Adding an explicit `process.exit(0)` after the load-and-report step fixed it. Worth remembering if you ever write a similar smoke test: for any native module that spins up its own runtime underneath Node, "does it hang" and "did it fail" are different questions, and only one of them is about the thing you're testing.

## The process manager is the wrong place to look for "is this still used"

Once I'd repointed every PM2 entry away from the standalone runtime directories, the natural next move was to delete them. PM2 said nothing referenced them anymore. PM2 was wrong to trust on that specific question, because PM2 only knows about processes *it* started.

Removing the old runtimes required a sweep that went well past the process manager: its saved dump, live process executables (not just the dump — a process started by hand doesn't match either), machine and user `PATH`, Windows services, scheduled tasks, registry startup keys, the config of the service that supervises PM2 itself, and the global package directory. A plain filesystem search for the old install path, run after all of the above came back clean, still turned up two hits.

Both were simulator utilities that invoked the runtime *directly* — no PM2, no service, no scheduled task. They were idle at the time, so they showed up in no process listing and no manager state. Deleting the directory they pointed at would have broken both the next time someone ran them, with zero warning beforehand. Their native dependencies happened to use a version-independent interface, so repointing them was low-risk once found — the risk was entirely in *not* finding them.

Two more landmines turned up the same way, neither of them related to the runtime deletion directly:

- A stale launcher declared the *same* process name as a live, currently-managed service, but pointed at an older bundle on a different port. Running it by habit — muscle memory, or a script that assumes the name is unique — would have started a second instance that collided with the real one.
- A version-pin file still named a runtime that had already been removed. Nothing read it at launch time, but it would have sent the next person who opened the project chasing an installation that no longer exists.

None of these three would have surfaced from asking the process manager "what's using this." They surfaced from asking the filesystem "what mentions this path," which is a different question and, for a decommission, the one that actually matters.

## Outcome

All 21 processes now run on explicitly pinned runtimes: 13 on 14.21.3, 6 on 16.20.2, 1 on 24.18.0 — the last one itself a same-ABI move from 24.16.0, verified the same way before the swap. The end-of-life 14.16.0 install was removed cleanly through its own uninstaller, the unused standalone 16.20.2 copy was deleted only after the sweep above cleared it, and the two simulator utilities and the stale launcher were fixed at the same time so they wouldn't become the next surprise.

Two things are still open. The final directory deletion was blocked by a safety guard in my own tooling — every reference had already been found and repointed, so what's left is a one-line manual step, not more investigation. And the whole boot path — PM2 resurrecting all 21 processes cleanly on a cold restart — has been verified by inspecting every saved interpreter path, not by an actual reboot. Inspection is not the same claim as "I watched it come back up," and that gap is worth closing before calling this done.

## Takeaways

- **Same major version does not guarantee the same ABI, and different minor versions usually do.** Check the ABI number, then prove it by loading the module — don't infer compatibility from the version string alone.
- **A test harness that hangs on a native module might be reporting success, not failure.** If the module starts its own runtime underneath yours (a JVM via JNI, for example), that runtime's threads can be exactly why your process won't exit.
- **The tool that manages something is not the full set of things that reference it.** Before deleting shared infrastructure, search the filesystem directly — the most dangerous references are the ones held by whatever isn't currently running.
- **"Does it fail the same way before my change?" separates a real regression from a pre-existing condition that nobody had tripped over yet.**
