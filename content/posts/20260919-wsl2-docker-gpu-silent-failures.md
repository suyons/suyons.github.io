---
title: "WSL2 Troubleshooting - Four Silent Failures That All Looked Like Success"
date: 2026-09-19
draft: false
tags: ["wsl2", "docker", "cuda", "windows", "networking"]
categories: ["Infrastructure"]
description: "Turning a docker-compose sketch for a GPU-backed Jupyter container into a working setup surfaced a wrong version pairing, a mount silently pointed at the wrong filesystem, a restart policy that wasn't enough, and a firewall rule scoped to the wrong network profile — none of which threw an error."
showToc: true
---

I had a docker-compose sketch for a second, GPU-enabled container on a training box — Jupyter, isolated from the inference server already running there. Turning that sketch into a real, working container took one session and surfaced four bugs. The thing they have in common: none of them threw an error. Each one looked, at first glance, like it had already worked.

## The version pairing was wrong, not just unconfirmed

The sketch pinned the image to `pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime`. Rather than trust that pairing, I ran a small CUDA smoke test against it first — allocate a tensor, move it to the GPU, run a matrix multiply. It failed:

```
NVIDIA GeForce RTX 5060 Laptop GPU with CUDA capability sm_120 is not compatible with the current PyTorch installation.
The current PyTorch install supports CUDA capabilities sm_50 sm_60 sm_70 sm_75 sm_80 sm_86 sm_90.
RuntimeError: CUDA error: no kernel image is available for execution on the device
```

The RTX 5060 is a Blackwell-generation GPU — the same hardware and compute capability (`sm_120`) that also broke `xformers`' prebuilt attention kernels for me [in an earlier fine-tuning session](/posts/20260714-qlora-blackwell-xformers-attention-kernel/). This time the trap was a container image tag instead of a Python package: it needs CUDA 12.8 or newer to ship kernels compiled for `sm_120` at all. Swapping to `pytorch/pytorch:2.8.0-cuda12.8-cudnn9-runtime` and re-running the same smoke test passed cleanly.

A version pairing that looks reasonable on paper is still a guess until something actually runs a GPU op against it. The failure mode here — silently wrong kernels — wouldn't have surfaced until deep into a real training run, several hours in, with no obvious reason to suspect the base image.

## From a compose file to a single `docker run`

With only one service, a docker-compose file was pure indirection. The GPU device-reservation block Compose uses collapses into a single `--gpus all` flag; ports, volumes, and command carry over directly:

```yaml
# Before — docker-compose.yml
services:
  jupyter:
    image: pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    ports:
      - "8888:8888"
    volumes:
      - ~/training/data:/workspace/data
    command: jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root
```

```bash
# After — single docker run
docker run -d --name jupyter --restart unless-stopped --gpus all \
  -p 8888:8888 \
  -v /mnt/c/Users/<user>/training/data:/workspace/data \
  pytorch/pytorch:2.8.0-cuda12.8-cudnn9-runtime \
  bash -c "pip install -q jupyterlab && jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root"
```

The base image doesn't ship JupyterLab, so it gets installed at container start instead of baked into a custom image — a deliberate shortcut that trades a few seconds of reinstall time on every restart for not maintaining a Dockerfile, since nothing else about the image needed customizing.

## A shell-expansion bug silently mounted the wrong filesystem

The first attempt at the bind mount used `~/training/data`, run through a Windows-side Bash tool that shells out to `wsl.exe`. That combination is deceptive: `~` gets expanded by the *Windows-side* shell, not the WSL shell, before `wsl.exe` ever sees the argument. Because the two shells' home directories are spelled similarly, the resulting path was syntactically valid *inside WSL's own virtual disk* — not a path into the actual Windows `C:` drive. The container started, the mount "worked," and nothing looked wrong until it was checked directly:

```bash
# Looked like it was writing to the Windows C: drive:
-v ~/training/data:/workspace/data

# Actually resolved, inside WSL, to a directory on WSL's own disk —
# coincidentally valid, but not the Windows drive:
/c/Users/<user>/training/data

# Correct: WSL's explicit mount point for the real Windows C: drive
-v /mnt/c/Users/<user>/training/data:/workspace/data
```

The fix is to always use WSL's explicit `/mnt/<drive>/...` form when a command crosses the Windows/WSL boundary, and to check with a direct listing rather than trust that a successful mount resolved where it looked like it should. The mistaken directories were empty and safe to discard here, but the same bug with real training data in flight would have looked like a working backup right up until a Windows reboot or WSL disk reset made it disappear.

## A restart policy that looked complete and silently wasn't

`--restart unless-stopped` reads like the whole answer to "survive a reboot." It silently wasn't, because WSL2 itself doesn't start at Windows boot unless something asks it to — no Docker daemon, no restart policy to even apply. Two independent pieces were needed:

1. `--restart unless-stopped` on the container, plus WSL's own `systemd=true` setting (already in place), so `dockerd` comes up as soon as the WSL VM does.
2. A Windows Scheduled Task, triggered at user logon, running `wsl.exe -d Ubuntu -e true` — enough to boot the WSL VM, at which point systemd and Docker take over and bring the container back.

Neither piece alone was sufficient. Without the scheduled task, the VM — and therefore Docker and the container — simply never started after a reboot until someone happened to open a WSL terminal. No crash, no log entry, just a service that was down until manually noticed.

## A firewall rule that looked correct and only matched half the network

Opening TCP 8888 on the Windows firewall for LAN access initially "worked": connections from the Windows host itself succeeded. A genuinely remote client on the same Wi-Fi network still couldn't connect. The cause was a firewall-profile mismatch — the rule was scoped to the "Private" network profile, but Windows had categorized the active Wi-Fi network as "Public," a categorization with nothing to do with whether the network is actually a home LAN, just how Windows happened to classify it that session. A rule scoped to the wrong profile is silently dropped for non-matching traffic rather than producing any error:

```powershell
# Rule existed, was enabled, looked right — but only applied to one profile
New-NetFirewallRule -DisplayName 'WSL Jupyter 8888' -Direction Inbound -Protocol TCP -LocalPort 8888 -Action Allow -Profile Private

# Fix: check which profile the active network actually uses, and match it
Get-NetConnectionProfile   # showed NetworkCategory: Public
Set-NetFirewallRule -DisplayName 'WSL Jupyter 8888' -Profile Private,Public
```

An existing SSH firewall rule had the identical Private-only scoping and got widened the same way, since it would fail for the same reason under the same conditions.

## Outcome and takeaways

The container now runs with GPU acceleration verified against the actual GPU rather than assumed from the image tag, survives a host reboot through a restart policy plus a logon-triggered scheduled task, and is reachable from other devices on the LAN.

- **A version pairing, a mount, a restart policy, and a firewall rule can all "work" and still be wrong.** None of the four bugs here threw an error; each one produced a state that looked identical to success until checked directly — an actual GPU op, an actual directory listing, an actual reboot, an actual connection from a second device.
- **Cross a shell boundary (Windows Bash tool → `wsl.exe` → WSL shell) and `~` expands in the wrong shell.** Use explicit `/mnt/<drive>/...` paths whenever a command has to cross that boundary, never a tilde.
- **"Survives a restart" has more than one layer under WSL2.** A container restart policy assumes the daemon comes back; the daemon assumes the WSL VM comes back; the VM doesn't, by default, until something boots it.
