---
title: "Self-Hosting - The Redis Outage That Recurred Every Ten Days, To the Second"
date: 2026-06-20
draft: false
tags: ["redis", "memurai", "onlyoffice", "windows-server", "nssm", "self-hosting"]
categories: ["Infrastructure"]
description: "An embedded document editor kept dying, and restarting it 'fixed' it every time. The real bug was the bundled Redis on Windows — Memurai Developer Edition, which self-terminates after exactly ten days. Here's reading the timestamp instead of the error, why the Docker fix was a trap on this host, and the Windows service-dependency gotcha that turned the permanent fix into a self-inflicted outage."
showToc: true
---

## A restart that keeps working is a snooze button

A self-hosted document server kept wedging. The embedded editor would open, spin forever, and show nothing; someone would restart the service, the spinner would clear, and everyone moved on. That worked. It also kept happening.

"Restart it and it's fine" is the most dangerous kind of fix, because it pays you back immediately and hides the actual cause. The question that mattered wasn't *how do I clear the spinner* — I already knew that — it was *why does this keep coming back?* The answer was one line in a log file nobody was reading, and the permanent fix then tripped over a Windows-specific trap worth the whole post on its own.

This is ONLYOFFICE Docs running on Windows Server, but nothing here is really about ONLYOFFICE. It's about recurring outages, reading the right log, and what "disable this service" actually does on Windows.

## Read the timestamp, not the error

The failure-day logs were a wall of one repeated line — tens of thousands of `redisClient error ECONNREFUSED`, roughly every half-second, from one instant until a manual restart hours later. The reflex is to stare at the error text. Redis connection refused: is it the connection pool, a config, a network blip?

The error text was a red herring. The *timestamp* was the clue. The flood didn't ramp up the way load does; it began on one exact second — `13:52:04` — and never recovered on its own. A health check happened to run hours later and simply landed inside the outage, which is why it looked like *the health check* was flaky.

> An outage that starts on a precise second and never self-heals doesn't look like load, and it doesn't look like a crash. It looks like something was switched off.

Something was. The consumer screaming `ECONNREFUSED` wasn't the culprit — the thing it was connecting *to* had gone away. So I stopped reading the application logs and went to read the dying component's own log.

## The bundled "Redis" was a trial that self-destructs

The "Redis" on this box isn't Redis. ONLYOFFICE on Windows bundles **Memurai**, a Redis-compatible server, and the edition that ships by default is **Memurai Developer Edition**. Its own log says what it is, in plain words:

```
08 Jun 13:52:04  Usage of Memurai Developer Edition in a production environment is
                 prohibited; it also automatically shuts down after 10 days.
18 Jun 13:52:04  Memurai Developer Edition automatic shutdown... will now exit, bye bye
```

Started June 8 at `13:52:04`. Self-terminated June 18 at `13:52:04`. Exactly ten days, to the second. Not a crash, not memory pressure, not a network partition — a licensing time-bomb with a countdown.

Two more details turned each detonation from a blip into a multi-hour outage:

- **Memurai exits cleanly** (exit code 0). It thinks it's doing the right thing, so Windows sees a graceful stop.
- **Nothing was watching it.** No supervisor, no alert. It just vanished and stayed gone until a human noticed the editor was broken.

That combination — a clean exit plus no watchdog — is why "restart it" had become a recurring ritual on a ten-day clock.

## The obvious fix was a trap on this host

The textbook move is "run Redis in a Docker container." On this host, that's a dead end, and chasing it would have meant rebooting a production server to find out.

The official `redis` image is Linux-only. Windows Server natively runs *Windows* containers; running a *Linux* container on Windows means a Linux backend (WSL2 or Hyper-V), which requires **nested virtualization** from the host. This box is a standard public-cloud VM, and standard cloud instances generally don't expose nested virt to the guest. So the Docker path likely fails — but only *after* you enable Windows features and reboot to try it. On a production server, "let's reboot and see" is not a plan.

The lesson I keep relearning: when the popular fix needs a reboot to even test, stop and find the path that doesn't.

## Native Windows Redis, supervised properly

The replacement was **real Redis built for Windows**, installed with Chocolatey:

```
choco install redis    # redis-windows: a genuine Redis build for Windows, no expiry, no license
choco install nssm
```

This `redis-windows` package is an actual upstream Redis compiled for Windows — no ten-day fuse, no "developer edition" clause. The package ships just the binaries and a `start.bat`, so it isn't a service yet. I wrapped it with **NSSM** (the Non-Sucking Service Manager) so it behaves like infrastructure should: auto-start at boot, **auto-restart on crash**, and a graceful shutdown signal instead of a hard kill.

Two details that matter more than they look:

- **Keep state out of the package directory.** Config, data, and logs went in `C:\ProgramData\Redis`, deliberately *outside* Chocolatey's lib folder, so a future `choco upgrade redis` can't wipe the data with it.
- **Mind the blast radius of a public IP.** This box is internet-reachable, so before flipping anything I confirmed the config kept `bind 127.0.0.1 -::1` and `protected-mode yes`, and verified the listener answered on localhost only. An unauthenticated Redis exposed to the internet would have been a far worse incident than the one I was fixing — swapping the backend is exactly the moment that mistake sneaks in.

## The trap: a disabled dependency blocks startup beneath the app

Here's where the permanent fix bit back. With the new Redis running and Memurai stopped and **disabled**, I restarted the ONLYOFFICE services — and got a blunt `Cannot start service`. DocService, Converter, *and* the Proxy ended up stopped. The application logs said nothing useful, because the failure happened below them.

The ONLYOFFICE services carry a hard Windows **Service Control Manager dependency** on a service named literally `Memurai` (alongside `postgresql-x64-18` and `RabbitMQ`). The SCM refuses to start a service whose dependency is *disabled* — and it rejects it *before* the service wrapper, and therefore the Node process, ever runs. That's why nothing showed up in any ONLYOFFICE log. The evidence wasn't in the application's world at all; it was here:

```powershell
Get-Service DsDocServiceSvc | Select-Object -ExpandProperty ServicesDependedOn
```

The fix is to repoint the dependency from `Memurai` to the new `Redis` service. The catch: `sc config depend=` **replaces the entire list** rather than appending, so every dependency you still need has to be re-listed or you'll silently drop it:

```
sc.exe config DsDocServiceSvc depend= postgresql-x64-18/Redis/RabbitMQ
sc.exe config DsConverterSvc  depend= postgresql-x64-18/Redis/RabbitMQ
```

Then start in dependency order — Converter, DocService, Proxy. The health endpoint returned `true`, and a fresh DocService log showed zero `redisClient` errors.

There was a self-inflicted few-minute outage in the middle of this, caused by disabling Memurai *before* repointing the dependency. That ordering mistake is the most transferable lesson of the day: on Windows, disabling a service can break its dependents at a layer your application logs can't see.

## Takeaways

- **A recurring outage on a fixed interval, with an identical on-the-second timestamp, is a scheduled or expiry event — not load and not a crash.** Read the dying component's own log before theorizing about the consumers screaming about it.
- **A restart that "fixes" a recurring problem is a snooze button.** Ask why it keeps happening, not how to clear it this time.
- **Bundled "developer" or "free" editions can be production time-bombs.** Memurai Developer Edition is prohibited for production use and self-terminates every ten days, and it ships with ONLYOFFICE-on-Windows by default. Check what your "Redis" actually is.
- **Clean-exit processes defeat Windows Service Recovery.** Recovery actions fire on crashes, not graceful exits, so a service that shuts *itself* down is never auto-restarted. Wrap console apps that must stay up with a supervisor like NSSM.
- **Before disabling a service on Windows, check who depends on it.** A disabled dependency blocks its dependents at the SCM layer, invisibly to application logs — and `sc config depend=` rewrites the whole list instead of appending.
- **Swapping a network-facing backend is the moment to re-verify it binds to localhost.** Especially on a box with a public IP.
