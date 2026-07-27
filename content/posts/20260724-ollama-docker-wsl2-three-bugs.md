---
title: "Self-Hosting Troubleshooting - Three Loopbacks and a Blackhole"
date: 2026-07-24
draft: false
tags: ["docker", "wsl2", "ollama", "networking", "self-hosting"]
categories: ["Infrastructure"]
description: "Debugging a local LLM readiness check turned into a tour of every place the word localhost can lie to you — a container running nothing, a loopback bound to the wrong namespace, and Windows silently preferring an IPv6 address WSL2 quietly drops."
showToc: true
---

What started as a one-line question — "how do I check that my local LLM server is ready?" — turned into a tour of every place the word `localhost` can lie to you. The server looked healthy in `docker ps`, wasn't running at all, and once fixed, was unreachable from Windows for a completely unrelated reason. Three bugs, three different network layers, one recurring root cause.

## The setup

Ollama (a local large-language-model runtime) was running as a Docker container inside a WSL2 Ubuntu distribution, GPU-accelerated, with its API port published to the host. `docker ps` showed exactly what you'd want to see:

```
CONTAINER ID   IMAGE          COMMAND            STATUS       PORTS
d40d2ea61e9e   ubuntu:24.04   "sleep infinity"   Up 3 hours   0.0.0.0:11434->11434/tcp
```

Up, healthy, port published. The readiness check for Ollama is just an HTTP call — it exposes a REST API, so `curl` is the whole test:

```bash
curl -sf http://localhost:11434/          # -> "Ollama is running"
curl -s  http://localhost:11434/api/tags  # -> JSON list of installed models
```

That second endpoint is the one worth knowing: it tells you not merely that the process is alive but *which models you can actually prompt*. For true end-to-end confidence, a real generation request is the only honest test, since it's the first thing that touches the GPU.

The check returned curl exit code 56 — connection reset.

## Bug 1: a published port pointing at nothing

Look again at that `docker ps` output. The `COMMAND` column says `sleep infinity`. The container's job was to sit there; nothing ever launched the server. Inside it, the binary was installed but no process was running and nothing was listening on the port.

This is the trap in reading `docker ps` as a health signal. **A published port is a promise Docker makes about routing, not evidence that anything is listening on the other end.** Docker will happily forward traffic into a container where the port is dead, and the resulting reset looks a lot like a crashed service.

There was a second landmine waiting behind the first. Ollama binds `127.0.0.1` by default — the *container's own* loopback, a different network namespace from the host's. Published ports can't reach it. So even starting the server the obvious way would have reproduced the same symptom:

```bash
# Before — starts, but binds the container's private loopback; still unreachable
docker exec -d ollama ollama serve

# After — binds all interfaces inside the container, so the port publish can reach it
docker exec -d -e OLLAMA_HOST=0.0.0.0:11434 ollama ollama serve
```

That got it answering. But it was a manual fix that wouldn't survive a restart.

## Bug 2: making it survive a restart

The container's command was the problem, and a container's command can't be edited in place — it has to be recreated. Before touching anything, an inspection answered the question that decides whether recreating is safe or catastrophic: *where does the state live?*

Happily, the models (several gigabytes of downloads) were on a named volume, and the restart policy, port publish, GPU request, and environment variables were all already correct. The only wrong thing was the command. The container's writable layer turned out to hold nothing but the stock installer output, so a snapshot captured everything that mattered and doubled as a rollback:

```bash
docker commit ollama ollama-local:latest    # snapshot first — this is the undo button

docker run -d --name ollama --restart unless-stopped --gpus all \
  -p 11434:11434 -v ollama-models:/root/.ollama \
  -e OLLAMA_HOST=0.0.0.0:11434 \
  ollama-local:latest ollama serve          # was: sleep infinity
```

The verification that actually matters here isn't "does it respond now" — it's restarting the container and confirming it comes back with no manual intervention. It did, models intact, with the model reporting 100% GPU placement.

One honest caveat got recorded rather than papered over: an image built by `commit` can't be rebuilt from source. It's fine as a rollback, but the durable answer is either a three-line Dockerfile or the project's official image, which would have been a drop-in against the same volume.

## Bug 3: the blackhole

With the server solid, one thing remained: reaching it from Windows. `http://localhost:11434` timed out. The easy conclusion — "WSL port forwarding is broken" — was wrong, and testing four addresses instead of one is what exposed it:

| Address | Result |
|---|---|
| `http://127.0.0.1:11434` | works |
| `http://<host-lan-ip>:11434` | works |
| `http://localhost:11434` | hangs |
| `http://[::1]:11434` | hangs |

IPv4 had been working the entire time. Windows resolves `localhost` to the IPv6 address `::1` before `127.0.0.1` — standard behavior per RFC 6724, not a bug — and WSL's mirrored networking mode silently drops traffic to `::1`.

The silence is the real damage. Had that connection been *refused*, every HTTP client would have fallen back to IPv4 in milliseconds and nobody would have noticed anything. Because the packet vanishes instead, the client waits out its full timeout, and a routing quirk wears the costume of a dead server.

There is a one-line Windows command that reprioritizes IPv4 and makes `localhost` work — and it was deliberately not applied. It changes name resolution globally, for every application on the machine, to save typing four characters. Using `127.0.0.1` in client configuration is the better trade.

## Outcome & takeaways

The server now starts itself with the container, survives daemon and host reboots, keeps its models on a volume that outlives the container, runs fully on the GPU, and is reachable from Windows over IPv4 and from other machines on the network. Every claim there was verified by an actual request, including a full generation round-trip from PowerShell.

Three lessons generalize past this stack:

- **`localhost` means something different at every layer.** In this system there were four candidates: the container's loopback, the Docker bridge, the virtual machine's loopback, and Windows' loopback. A service "on localhost" is on exactly one of them. Bind `0.0.0.0` when you intend to publish; use explicit addresses when you connect. Explicit addresses can't be quietly reinterpreted by a layer you forgot existed.
- **A hang and a refusal point in opposite directions.** *Connection refused* means you found the right host and nothing is listening. A *hang* means your packets are going somewhere that isn't answering — wrong network path. This session produced one of each, and reading them correctly was the difference between a five-minute fix and an afternoon of suspecting the wrong component.
- **Snapshot before you recreate, and check where the state lives first.** Recreating a container is only reversible if you know what's on a volume, what's in the writable layer, and what you can rebuild. Two inspection commands turned a scary destructive step into a routine one.

The last one is broader than containers: the instinct to grab the visible symptom ("the server is down") and skip the layer-by-layer check is what makes this class of bug expensive. Testing four addresses took thirty seconds and overturned the diagnosis.
