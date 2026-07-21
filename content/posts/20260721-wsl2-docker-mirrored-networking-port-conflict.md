---
title: "WSL2 Troubleshooting - A Stale Port Forward Rule Survived the Network It Was Built For"
date: 2026-07-21
draft: false
tags: ["wsl2", "docker", "windows", "networking", "containers"]
categories: ["Infrastructure"]
description: "A Docker connection error on Windows led through a WSL2 networking migration to mirrored mode, where a container's lost bridge endpoint and a leftover netsh portproxy rule from the old NAT setup looked like one failure and turned out to be two."
showToc: true
---

`docker ps` on Windows failed with `failed to connect to the docker API at npipe:////./pipe/docker_engine`. The obvious read — "the daemon isn't running" — was wrong, and chasing the real cause went through a WSL2 networking migration and two failures that looked like one.

## The error that wasn't what it looked like

The named pipe `//./pipe/docker_engine` is created by Docker Desktop. No Docker Desktop, no pipe. This machine had no Docker Desktop at all, only a standalone Docker CLI that defaults to that pipe. Meanwhile the actual Docker Engine was alive and healthy inside a WSL2 Ubuntu distribution, listening on its Unix socket. CLI and daemon were both fine; they just had no channel between them.

Worth internalizing: a connection error names the transport it tried, not the thing that's broken. Reading `npipe://…` as "daemon down" sends you off restarting a service that was never the problem.

The daemon's config already declared a TCP listener:

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://127.0.0.1:2375"]
}
```

WSL2 forwards localhost, so pointing the Windows CLI at it was one line:

```powershell
[Environment]::SetEnvironmentVariable('DOCKER_HOST','tcp://localhost:2375','User')
```

A detour worth recording: before reading that config file, I'd added `-H` flags to the daemon's systemd override. Docker refuses to start when the same directive appears as both a flag and a config-file entry, so it crash-looped — and then systemd's restart rate limiter kicked in with `Start request repeated too quickly`, a second, misleading error that hides the first. When a service won't start after several retries, run `systemctl reset-failed` before trusting the most recent log line.

## Changing direction: WSL-only, but reachable from the LAN

The goal shifted: run Docker only from inside WSL, but have published container ports reachable from other machines on the network. So the TCP exposure got reverted — daemon back to Unix-socket-only, the Windows environment variable removed.

There were two routes to external reachability. The legacy approach forwards specific Windows ports to the WSL VM's IP with `netsh portproxy`, but it needs per-port setup and the VM's IP changes on every reboot. The modern approach is WSL2 **mirrored networking**, where the VM shares the Windows host's network interfaces directly — every published port lands on the host's LAN address, for all present and future containers, nothing per-port:

```ini
[wsl2]
networkingMode=mirrored

[experimental]
hostAddressLoopback=true
```

After `wsl --shutdown` and a restart, the Linux side showed an interface carrying the host's own LAN address instead of a private NAT address. Mirrored mode was live.

## Two failures that looked like one

A container that had been publishing a database port came back with **no ports at all**. `NetworkSettings.Networks` was `{}`, the bridge interface was down, and its network namespace held only loopback — no `eth0`.

The tempting conclusion was "mirrored networking broke Docker." A throwaway container disproved that in seconds: it attached to the bridge and published its port fine. When a shared subsystem looks broken, test it with a fresh, minimal case before blaming it — that one command split the problem cleanly into "the platform" versus "this one container," and it was the latter.

The container had lost its bridge endpoint across the VM restart. Neither `restart` nor `stop`/`start` rebuilt it, because Docker decides what to attach at start time from the recorded `Networks` map, and that map was empty — so start faithfully attached nothing. The repair is to record the network while the container is stopped, then start it:

```bash
docker stop <container>
docker network connect bridge <container>   # must be while stopped
docker start <container>
```

Attempting the network-connect step on a still-*running* container fails, and that failure exposed the second problem: `failed to bind host port 0.0.0.0:3306/tcp: address already in use` — even though nothing inside Linux was listening on it. The port was held on the **Windows** side, by a stale `netsh portproxy` rule left over from the old NAT setup, pointing at a WSL IP that no longer existed.

That rule had been harmless for as long as the two networks were separate. Mirrored mode is precisely the change that makes them share one port space, so a dormant Windows-side listener suddenly became a hard conflict. Deleting the obsolete rule freed the port, and the stop/connect/start sequence restored the container with its published port intact.

One more piece was needed: under mirrored networking, inbound traffic to the VM is filtered by the Hyper-V firewall, which defaults to Block. It has to be set to Allow for outside hosts to reach anything.

## Outcome

Docker now runs exclusively inside WSL, and published container ports bind directly to the host's LAN address with no per-port forwarding. The database container came back up with its port published, and the throwaway test container was removed.

One honest caveat: reachability was only confirmed from the same machine, which doesn't traverse the Windows Firewall. A genuine test from another device on the network is still outstanding. Testing a network path from the host itself proves the listener exists — it doesn't prove the path from outside is open.

## Takeaways

- **A connection error names the transport, not the culprit.** `npipe://…` meant "this channel is absent," not "the service is dead."
- **Read the existing config before adding to it.** The TCP listener was already configured; adding it again as a flag caused the conflict that crash-looped the daemon.
- **A restart limiter's message will mask the original error.** Reset the failure counter and read the *first* failure, not the most recent one.
- **Isolate with a minimal fresh case** before concluding a platform change broke everything.
- **Merging two network namespaces surfaces dormant conflicts.** Anything previously listening on one side can collide with the other; audit leftover forwarding rules before switching to a shared-port model.
