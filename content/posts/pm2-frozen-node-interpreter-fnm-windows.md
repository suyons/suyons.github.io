---
title: "Your pm2 Dump Pins a node Path. fnm Deletes It on Reboot."
date: 2026-06-12
draft: false
tags: ["pm2", "nodejs", "fnm", "windows-server", "process-manager", "devops"]
categories: ["DevOps"]
description: "Running 22 Node apps under pm2 on Windows, one app refused to survive a reboot. The cause wasn't the app — it was that pm2 resolves each process's node interpreter from a PATH frozen into the dump, and fnm's per-shell node lives at a path that vanishes when the machine restarts."
showToc: true
---

## "pm2 resurrect, but it brought nothing back"

A Windows Server box runs 22 production Node apps under pm2. Reboot it, run `pm2 resurrect`, and 21 come back fine. One doesn't — a document-management backend with a native addon. And chasing that one app dragged me through three separate misunderstandings of what pm2 actually stores and what runtime it runs your code under.

The short version, because it's the part worth keeping: **pm2 does not run your app on the daemon's node.** It runs each app on whatever `node` resolved to *in the shell where you last ran `pm2 start` or `pm2 save`*, and it bakes that resolved path into the dump. On a machine where node is managed by [fnm](https://github.com/Schniz/fnm), that path is frequently a per-shell temporary directory that does not exist after a reboot. So the dump points at a `node.exe` that is gone, and the app fails to resurrect while every app pinned to a stable path comes up clean.

If you run fnm (or nvm) on a server and lean on pm2 to survive reboots, that single sentence is the post. The rest is how I got there and the two adjacent traps that wear the same disguise.

## The dump freezes a path, not "node"

Here's the model most people carry: the pm2 daemon runs on some node, and when you `pm2 resurrect`, all your apps run on that same node. Reasonable. Wrong.

`exec_interpreter: "node"` in the dump is resolved **per app**, against the `PATH` that was in effect when that app's process definition was created. pm2 stores the result. So a dump for 22 apps can — and on this box did — point at three different node binaries at once:

| Apps | Interpreter the dump froze | Survives reboot? |
| --- | --- | --- |
| 16 legacy | `C:\Program Files\nodejs` → v14 | yes |
| 5 schedulers | absolute paths to a node 16 install | yes |
| the doc backend | `...\fnm_multishells\13644_<timestamp>\node.exe` → v24 | **no** |

The 16 legacy apps were started from a shell where the system-wide node 14 was on PATH, so they froze a stable absolute path. The 5 schedulers were pinned deliberately. All 21 of those are reboot-proof by accident or design.

The doc backend was different. It needs node 24 specifically — its native `java` addon (a JNI bridge that spins up a JVM at startup for document generation) is compiled against node 24's ABI and will not load on 14 or 16. And the only node 24 on the box came from fnm. When you `fnm use 24` in a shell, fnm activates that version by putting a **per-shell temporary directory** on PATH:

```
...\fnm\fnm_multishells\13644_1718000000000\node.exe
```

That `13644_<timestamp>` directory is created for that shell session. It is garbage-collected, and it certainly is not there after the machine reboots. So `pm2 save` from that shell froze a path to a `node.exe` inside a directory that the next boot would not have. Resurrect, and that one app points into the void.

## The fix is an absolute path to fnm's *stable* node, not its shell shim

fnm keeps two kinds of paths, and the distinction is the whole game:

- The ephemeral shell shim: `...\fnm_multishells\<pid>_<ts>\node.exe` — what's on your PATH after `fnm use`. Dies with the shell.
- The durable install: `...\fnm\node-versions\v24.16.0\installation\node.exe` — where the version actually lives. Stable across reboots.

The shim points at the install. Your interactive shell uses the shim and everything works, which is exactly why this is so easy to miss — it's correct in every session you're actually typing in. It only breaks for a process that has to start with no shell, at boot, from a frozen definition.

So the durable fix is to pin that one app's interpreter to the install path. I did it directly in the dump, on that app and only that app:

```
exec_interpreter:
  C:/Users/Administrator/AppData/Roaming/fnm/node-versions/v24.16.0/installation/node.exe
```

The 21 apps on stable paths keep riding on relative `"node"`; the one app whose runtime came from fnm gets an absolute pin matching its `.node-version`. After that, `pm2 resurrect` brings up all 22 on the right node, with no boot-time scheduled task and no wrapper script — just a human typing one command after a reboot.

One caveat that cost me an extra cycle, and that you must internalize before you trust this: **a dump-only edit is real, but a later `pm2 save` overwrites it.** If you hand-edit the dump to pin the interpreter, then someone runs `pm2 start ecosystem.config.js` and `pm2 save`, the save re-resolves `"node"` from *that* shell's PATH and silently reverts your pin. The dump is the source of truth only until the next write wins. Either pin it in the place the next `save` will read from, or know that `save` is a loaded gun pointed at your fix.

## The adjacent trap: pm2's own node has to satisfy its `engines`

While untangling the interpreter, an everyday command started hanging. `pm2 ls`, `pm2 jlist`, `pm2 save` — all sat there for *five minutes* and never returned. Meanwhile the apps were up: healthy child processes, low CPU, serving traffic. Only the CLI status calls hung.

The daemon (the "God daemon" process) sat at 0.6 seconds of CPU — not spinning, **deadlocked**. CLI clients piled up unanswered on the Windows named pipe pm2 uses for RPC.

The cause was the same family of bug one level up: **the wrong node, but for pm2 itself.** The daemon was pm2 7.0.1, and pm2 7 declares `engines: { node: ">=18.0.0" }`. The node it happened to be running on was 16.20.2. Running pm2 7 below its required runtime, on Windows, doesn't fail with a clean error — it wedges the named-pipe RPC while leaving the worker children alive and serving. You get a daemon that runs your apps but won't answer `pm2 ls`.

The lesson I'd weaponize from this: on Windows especially, **don't trust a hanging pm2 CLI to tell you the truth about your apps.** Check the OS process list and the dump file. `pm2 resurrect` kept working on that wedged daemon even while `pm2 ls` hung, because resurrect doesn't need the same RPC round-trip that status does.

The registry settled the version question cleanly:

| pm2 | engines node | runs on node 16.20.2? |
| --- | --- | --- |
| 7.x | `>=18.0.0` | no — the deadlock |
| **6.0.14** | `>=16.0.0` | yes — newest compatible |
| 5.4.3 | `>=12.0.0` | yes, but old |

Because the daemon was stuck on node 16 (fnm's default on this box, and not something I could change out from under the other apps), the answer was pm2 6.0.14 — the newest pm2 whose `engines` still admits node 16 — installed via plain npm under node 16. After that, `pm2 ls` returned in about 1.2 seconds. The five-minute hang was a version mismatch the whole time.

## Two more things the dump format will do to you

Downgrading the daemon surfaced one last surprise, and it's worth a sentence so it doesn't eat your afternoon: **pm2 6 silently drops the per-app `env` block when it resurrects a dump written by pm2 7.** Apps with a hard-coded default port shrugged it off. The ones that read their port from the environment came up with no port and crash-looped. The repair was to recreate each affected app with `pm2 delete` + `pm2 start` *with the env set in the shell* — because `pm2 start` bakes the shell environment into the process definition, whereas `pm2 restart --update-env` does not persist across the next resurrect. Then `pm2 save` to rewrite the dump in the new format with env baked in.

And distrust pm2's error text on Windows. A crash-looping app reported `Port undefined is already in use` — which was two bugs in a trench coat. The "undefined" half was the missing env above. The "already in use" half was real but unrelated: orphaned node processes, zombies left by thousands of crash-restart cycles, still squatting on the ports and invisible to pm2's process table. Reading the message literally sent me hunting for a config bug; the actual fix was `taskkill` on the orphans and then a clean `pm2 start`.

## What I'd tell myself before touching the box

The transferable lesson isn't about pm2 quirks. It's that **on a machine with a node version manager, "node" is ambiguous, and pm2 freezes the answer at the moment you weren't paying attention.** Every disaster here was a version-resolution surprise wearing a different costume: the app's interpreter frozen to an ephemeral fnm shim, the daemon itself running below its own `engines` floor, a dump migrated between pm2 majors that quietly mutated.

So if you run pm2 on top of fnm or nvm, especially on Windows:

- Pin any app whose runtime comes from the version manager to the **install** path (`node-versions/<v>/installation/node.exe`), not the shell shim. The shim is correct in every session except the one that matters — boot.
- Make sure the pm2 daemon's node satisfies pm2's `engines`, or expect a wedged RPC and a five-minute `pm2 ls`. Match the major of pm2 to the node you're stuck on, not the other way around.
- After any pm2 major version change, re-verify the env blocks survived the dump migration, and confirm via the OS process list — not a hanging CLI — that apps are actually on the node you think.

It works now: one `pm2 resurrect` after a reboot brings all 22 back, each on the runtime it needs, no scheduled tasks and no wrapper scripts. The fix was four characters of absolute path — once I understood that pm2 had never been running what I assumed it was.
