---
title: "ROCm Troubleshooting - Why My AMD iGPU Was a Dead End Before I Wrote a Line of Code"
date: 2026-09-19
draft: false
tags: ["llm", "gpu", "rocm", "vllm", "wsl2"]
categories: ["Infrastructure"]
description: "Trying to run vLLM against an AMD laptop iGPU under WSL2 turned into an afternoon of checking ROCm's hardware support list instead of writing code, and the fix ended up being a two-machine split between inference and training."
showToc: true
---

I wanted to run vLLM inside a Docker container on WSL2, accelerated by the AMD integrated GPU on my laptop. Before touching Docker, I checked whether the GPU was even on ROCm's supported list. It wasn't, and the reason it wasn't mattered more than the "no" itself.

## ROCm's supported-hardware list is short, and iGPUs are barely on it

ROCm is AMD's GPU compute stack — the rough equivalent of NVIDIA's CUDA, and the thing vLLM's AMD backend depends on entirely to talk to the hardware. Officially supported hardware is mostly data-center cards and a handful of high-end discrete Radeon GPUs. Integrated GPUs are almost never on that list, and mine wasn't an exception: a Ryzen 7 4700U paired with Vega-generation graphics, 7 compute units.

The usual unofficial escape hatch — telling the ROCm runtime to treat the GPU as a newer, officially supported architecture via an environment variable override — doesn't help here. That trick works by pretending to be a chip that's close enough in instruction set to the real one; Vega on a 4700U is old enough that there's no "close enough" newer target left to pretend to be.

And even in the world where it ran: 7 compute units isn't much parallel throughput, and an iGPU shares system RAM instead of having dedicated VRAM, so the memory-bandwidth advantage a GPU normally has over a CPU mostly disappears too. CPU inference would likely have been competitive with a working ROCm setup on this specific chip, which makes forcing the issue not worth it.

## The conclusion: check the support list before you build anything

For this class of hardware, GPU-accelerated vLLM via ROCm isn't realistic, full stop. The fallback is CPU-only inference through something built for it — llama.cpp with a quantized model — or moving the workload to hardware that actually has driver support. I took the second option, and it turned into a bigger restructuring than "swap the GPU."

## The pivot: split inference and training across two machines

A colleague had a PC with a discrete NVIDIA GPU (RTX 5060), so model serving moved there. That solved inference, but opened a second problem: the training code and datasets lived on my original machine, and the GPU capable of training LoRA adapters now lived somewhere else entirely.

The setup that came out of that constraint:

- Code stays in a git repository, synced between machines by pushing and pulling rather than copying files around by hand.
- The dataset gets copied to the GPU machine once — datasets don't need live-syncing mid-training, so there's no reason to treat them like code that changes every few minutes.
- Editing and running code happens directly on the GPU machine through SSH-based remote development tooling, from an editor on the original machine. No edit-transfer-run loop, no keeping two copies of a codebase in sync manually.

## One WSL instance, two isolated containers

The GPU machine was already running vLLM inside a Docker container under WSL2, with GPU passthrough working through the NVIDIA Container Toolkit. Rather than add training libraries into that same running container, the plan was a second container, dedicated to training and notebook access, sitting alongside the inference container under the same WSL instance — each one requesting GPU access independently through Docker's own GPU device reservation, so neither container needs to know the other exists.

The reason to keep them separate: an inference server and a training stack pin conflicting versions of the same libraries often enough that sharing one environment invites a dependency collision — PyTorch, CUDA bindings, `transformers`, `peft`, `bitsandbytes` all move at different paces depending on which workload you're optimizing for. Splitting them also means an out-of-memory event during training can't take the inference server down with it. Both containers reuse the same GPU passthrough path that was already proven to work, so there's no new GPU configuration to debug — just a new container.

## Outcome and takeaways

Inference got solved by moving it to hardware ROCm was never going to support on this laptop. Training got solved by treating "where the code lives" and "where the data and GPU live" as separate problems: code stays portable through git, data gets copied once, and a second isolated container keeps training from ever touching the running inference service.

Two things stayed open at this point: confirming the exact CUDA/PyTorch pairing the RTX 5060 needed, since it's a newer GPU generation I hadn't run anything against yet, and setting up SSH tunneling to the training container instead of exposing its port on the LAN directly. Both turned into their own debugging sessions.

- **Check the vendor's official hardware support list before investing time in GPU acceleration.** The driver/runtime boundary, not the framework running on top of it, is usually the real constraint — and it's a five-minute check that would have saved an afternoon of dead ends.
- **An iGPU's shared-RAM architecture caps how much a working setup would even have bought me.** No dedicated VRAM means the usual GPU-over-CPU bandwidth advantage mostly doesn't apply.
- **When splitting compute across machines, separate "where code lives" from "where data and GPU live."** Code is cheap to sync repeatedly, so treat it that way. Data isn't — copy it once and leave it.
