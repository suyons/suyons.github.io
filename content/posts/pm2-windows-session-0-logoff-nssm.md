---
title: "Closing RDP Kept My pm2 Daemon Alive. Signing Out Killed It."
date: 2026-06-13
draft: false
tags: ["pm2", "nodejs", "windows-server", "nssm", "rdp", "devops"]
categories: ["DevOps"]
description: "After a week of making 22 Node apps survive a reboot under pm2 on Windows, they vanished overnight on a box that never rebooted. The daemon ran inside an interactive RDP session, and signing out — not disconnecting — destroyed it silently. The fix was moving the daemon into session 0 with NSSM."
showToc: true
---

## "It was fine when I went to sleep"

A Windows Server box runs 22 production Node apps under pm2. By the time I got to this point I'd spent days making the dump honest and the daemon healthy — pinning the one app whose node came from fnm, downgrading pm2 to a version its node could actually run. The last check before bed: `pm2 ls`, 22 online. Good night.

Morning: `pm2 ls` showed nothing. The operator had already resurrected by hand. The apps were just *gone* — and the box had never rebooted.

Every reflex I'd built over the previous days said "reboot ate them, fix the boot story." Every reflex was wrong. The daemon didn't die from a reboot or a crash. It died because someone clicked **Sign out** instead of just closing the window, and on Windows those are not the same thing. If you run any long-lived process inside an RDP session, that one sentence is the post.

## The corpse with no wound

The instinct from a week of scar tissue was "reboot." The evidence said otherwise, and it's worth walking the evidence because the conclusion is counterintuitive.

Three facts, none of which fit a reboot or a crash:

- `LastBootUpTime` was the night before, and uptime was continuous straight through to morning. **The box never rebooted overnight.** Whatever killed the apps wasn't a restart.
- `pm2.log` between the daemon's start the previous evening and the manual resurrect in the morning was **completely empty**. A daemon that crashes leaves a death rattle — an uncaught exception, an exit code, *something*. This one left nothing.
- The morning's resurrect logged `New PM2 Daemon started` — meaning there was **no daemon alive to connect to**. The process was simply gone, with no record of how it left.

A process that vanishes with no log line, on a machine that never rebooted, was not killed by anything it could see coming. It was torn down by the OS. So the question narrows to: which OS event destroys every process in one shot, silently, without a reboot?

## Disconnect is not logoff

The Windows Security log had the answer with a timestamp: event **4634, logon type 8**, carrying the same `LogonId` as the session that had launched the pm2 daemon 97 seconds earlier. The operator had checked `pm2 ls`, then signed out, then gone to sleep. The apps didn't fade through the night. They died a minute and a half after the bedtime check, and the box sat idle until morning.

Here's the assumption every prior day had quietly leaned on, the one nobody says out loud: that *disconnect equals logoff*. It does not, and on Windows the difference is everything.

- **Closing the RDP window** is a *disconnect*. The session keeps running. Every process in it — including a pm2 daemon you started at a shell — stays alive. You can reconnect later and find it exactly as you left it.
- **Signing out** is a *logoff*. Windows destroys the session and **every process bound to it**, with no graceful shutdown and no log entry. A console app gets no `SIGINT`-equivalent it can catch and report; it's just unmade.

The pm2 6 daemon ran inside the operator's interactive RDP session, because that's where `pm2 resurrect` was typed. So it inherited the session's lifetime. Disconnecting would have been harmless. Signing out was fatal. A week of reboot-proofing, undone by a menu click — not because the work was wrong, but because it answered a different question than the one that actually killed the apps.

"Survives a reboot" and "survives a logoff" are **different guarantees**, and `pm2 resurrect` on its own gives you neither for free until the daemon lives somewhere a session teardown can't reach.

## The only real fix: get out of the session

You can patch around a lot of this — disable logoff, train the operator to only ever disconnect — but every workaround depends on a human never clicking the wrong thing on a box that 22 production apps depend on. The durable fix is to make the daemon's lifetime independent of any interactive session.

On Windows, that means **session 0**. Interactive logins get session 1, 2, 3… and each dies with its user. Session 0 is the isolated, non-interactive session where Windows *services* run. Nothing a user does at the desktop — disconnect, logoff, even fast-user-switch — touches it. A process in session 0 outlives every interactive session by construction.

The catch: `pm2 startup`, which on Linux generates the systemd unit that does exactly this, **does not exist on Windows**. So you wire it up yourself. The tool for turning an arbitrary process into a real Windows service is [NSSM](https://nssm.cc/), the Non-Sucking Service Manager.

The design, with every path pinned to a *stable* install location (never fnm's per-shell temp dir — the trap from the earlier rounds of this saga):

- A service named `PM2` runs `node` (the stable v16 install path, not a shell shim) against a small foreground supervisor script, `pm2-service.js`. NSSM needs something that stays in the foreground to supervise; the pm2 daemon itself daemonizes and would escape that.
- The supervisor runs `pm2 resurrect` on start. On stop, it does `taskkill /F /T` against the daemon PID read from `pm2.pid` — deliberately **not** `pm2 kill`, whose Windows named-pipe handshake is the exact thing that hangs for minutes when the daemon is unhealthy. You don't want your service's stop action to be the one operation that can wedge.
- `PM2_HOME` and `PM2_BIN` are baked into the service environment, so the interactive CLI you type at a shell talks to the *same* daemon and the *same* dump. Skip this and you get a split-brain: your shell spawns its own second daemon on a different home, and now two pm2s fight over one socket.
- The service account is `.\Administrator`, **not** `LocalSystem`. The apps read from a NAS over SMB, and `LocalSystem` / `LocalService` can't authenticate to a network share — so the service needs a real user with credentials. (This is a constraint I'd hit days earlier and it never stopped being true.)
- Start type is **Automatic (Delayed)**, with a dependency on `LanmanWorkstation` and `Tcpip` so the SMB redirector and network stack are up before `pm2 resurrect` fires and the apps reach for the NAS.

That last pair of choices is a bonus: a session-0 service that starts automatically at boot also gives you reboot survival, for free, with no scheduled task and no boot-time wrapper. The thing I'd been chasing all week fell out of solving the logoff problem correctly.

## The secure prompt that wouldn't prompt

Setting the service account's password ran into a wall worth a warning, because it fails *silently* and you can lose an hour to it.

A tidy PowerShell script — `Read-Host -AsSecureString` to grab the password without echoing it — returned **instantly without ever asking**. Run it again, same thing. No prompt, no error, no password set. The service stayed on `LocalSystem`, which would have crash-looped every NAS-reading app the moment it tried to start.

The culprit is **SSH**. `Read-Host -AsSecureString` needs a real interactive console to read from. Over Windows OpenSSH there isn't one, so the secure-string read pulls from a non-interactive stdin, gets nothing, and gives up quietly — setting nothing.

The fix is to stop being clever and set the account directly in the operator's own SSH session:

```
nssm set PM2 ObjectName .\Administrator <password>
```

The password is visible only in that terminal and never goes into a script or a transcript. NSSM grants the account the "Log on as a service" right automatically. `nssm get PM2 ObjectName` confirms `.\Administrator`. If you script a secure prompt for a remote Windows box, test it over the exact transport you'll use — a console-bound prompt that works in RDP can no-op over SSH and leave you with a misconfigured service that looks fine until it starts.

## Cutting over costs exactly one restart

There's no in-place handoff. The 22 running apps were children of the daemon in the old interactive session, and Windows can't reparent a live process tree into session 0. The daemon *has* to be replaced — there is no version of this that doesn't restart everything exactly once.

The one constraint was that clients were connected, so a hard kill was off the table. The sequence:

1. Back up the dump first (`dump.pm2.bak-pre-nssm-*`). Always.
2. **Graceful** `pm2 kill` — this sends `SIGINT`, so every app closes its connections cleanly. The log read `All Applications Stopped.`
3. `nssm start PM2` → the supervisor → `pm2 resurrect`, which exited `code=0` with no hang this time, because pm2 6 on node 16 is a healthy RPC (the unhealthy combination was a different chapter).

Then the only check that actually mattered — proof the new daemon really lives in session 0:

```
New daemon PID: 2408
Daemon SessionId (must be 0): 0
Apps online: 22 / 22
Apps with restart_time>0: 0
```

Every child reparented to the new daemon, all in session 0, zero restarts. A sign-out can no longer find them. That single integer — `SessionId == 0` — is the whole victory, and it's worth verifying explicitly rather than assuming the service wrapper put you there.

## What I'd tell myself before blaming the reboot

The transferable lesson isn't about pm2. It's that **on Windows, an interactive session is a process container with a lifetime, and "disconnect" and "logoff" sit on opposite sides of that lifetime.** Anything long-lived you start at an RDP shell — a daemon, a dev server, a background worker — inherits the session's mortality. It'll happily survive a closed window and a reconnect, which lulls you, and then disappear without a trace the first time someone signs out instead.

So if you're keeping something alive on a Windows box:

- Don't reason from the CLI when a process vanishes. Read the OS: `LastBootUpTime` to rule out a reboot, an empty process log to rule out a crash, and the Security log (event 4634) to catch the logoff that leaves no other fingerprint.
- A process that must outlive the person who started it belongs in **session 0** — a real service. On Windows without `pm2 startup`, that's NSSM, with stable paths, the right service account for whatever network resources it needs, and a stop action that doesn't depend on the daemon being healthy.
- Verify session 0 directly. `SessionId == 0` is the difference between "survives a reboot" and "survives everything," and nothing else in the output tells you which one you got.

It survives the night now — not because the apps got more robust, but because the daemon finally lives somewhere a menu click can't reach.
