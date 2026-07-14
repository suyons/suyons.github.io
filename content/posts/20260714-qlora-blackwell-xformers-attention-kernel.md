---
title: "QLoRA Fine-Tuning Troubleshooting - The Attention Kernel My GPU Was Too New For"
date: 2026-07-14
draft: false
tags: ["qlora", "unsloth", "pytorch", "windows", "gpu", "fine-tuning"]
categories: ["Infrastructure"]
description: "Moving a QLoRA fine-tuning notebook from Colab's T4 to a local Windows box with a Blackwell-generation GPU broke in two unrelated ways before a single training step ran: a pip install that poisoned itself, and an attention kernel that had never heard of my GPU."
showToc: true
---

## The setup

I wanted to run a QLoRA fine-tuning notebook (Unsloth + `trl`, a Qwen3-4B base model) against my own GPU instead of queuing for a shared Colab T4 runtime. The GPU was an RTX 5060 Laptop GPU — Blackwell generation, compute capability `sm_120`. Nothing about the notebook itself needed to change; the model, the LoRA config, the training loop were all identical to the Colab version. Getting to the point where any of that code could run took two unrelated fights, neither of which had anything to do with the model.

## Gotcha 1: one `pip install` line quietly breaks itself

The natural instinct is to throw every package at `pip`/`uv` in one command and let the resolver sort it out:

```bash
uv pip install torch "unsloth==2025.5.1" "unsloth-zoo==2025.5.1" \
  "transformers==4.51.3" "accelerate==1.1.1" triton-windows rich \
  "datasets>=3" "dill>=0.4" "huggingface-hub<1" \
  --extra-index-url https://download.pytorch.org/whl/cu128
```

That resolves inconsistently. PyTorch's `cu128` extra index doesn't host only CUDA wheels of `torch` — it also mirrors older copies of ordinary packages like `requests` at its own priority, alongside whatever PyPI has. Handing the resolver both indexes and the full package list in one shot means it can pick a stale version of something that has nothing to do with CUDA, just because of how the two indexes' priorities interact for that one package. The fix isn't a smarter set of pins — it's not asking the resolver to reconcile two indexes at once:

```bash
# Pass 1: CUDA-specific packages, from the cu128 extra index
uv pip install torch "unsloth==2025.5.1" "unsloth-zoo==2025.5.1" \
  "transformers==4.51.3" "accelerate==1.1.1" triton-windows rich ipykernel \
  --extra-index-url https://download.pytorch.org/whl/cu128

# Pass 2: ordinary packages, from default PyPI only
uv pip install -U "datasets>=3" "dill>=0.4" "huggingface-hub<1"
```

Every pin in pass 1 exists for a reason I only found by hitting the failure first:

- **`torch` from the `cu128` index specifically** — PyPI's default `torch` wheel is CPU-only on Windows. An RTX 50-series card needs the `cu128` build or CUDA never initializes.
- **`unsloth==2025.5.1`** — the last version with native Windows support. `2026.x` added a hard dependency on `torchao`, which doesn't ship a Windows wheel at all.
- **`transformers==4.51.3`** — the version actually validated against `unsloth==2025.5.1`. Newer `5.x` releases require `huggingface-hub>=1`, which conflicts with the pin below.
- **`accelerate==1.1.1`** — `unsloth==2025.5.1` doesn't cap `accelerate`'s upper bound, so an unconstrained install grabs whatever's latest. With `accelerate>=1.14`, training crashes immediately with `NameError: FP8BackendType`. Pinning to a version known to work with this Unsloth release avoids that entirely.
- **`triton-windows`**, not `triton` — the official `triton` package has never shipped Windows wheels.
- **`rich`** — an undeclared dependency of the `trl` version this stack pulls in; without it, imports fail with no useful error pointing at why.

And pass 2's pins exist for a separate reason: `dill>=0.4` and `datasets>=3` are there because Python 3.12.11+ changed pickle internals in a way that breaks `datasets.load_dataset()` on older versions of both, with a `TypeError` that gives no hint it's a pickle version mismatch. `huggingface-hub<1` is there because `transformers==4.51.3` rejects `huggingface-hub` 1.x outright.

None of this is exotic — it's just version pins nobody should have to rediscover by trial and error, which is exactly why writing them down with the reason attached is worth the extra two comment lines in the install cell.

## Gotcha 2: an attention kernel that assumed it could always find itself a GPU

With the environment installed, the model loaded and patched fine. Running inference through it did not:

```
No operator found for `memory_efficient_attention_forward`
```

That's `xformers` failing to find a prebuilt attention kernel for the GPU. `xformers`' prebuilt kernels don't cover Blackwell (`sm_120`) yet — the wheel simply has no compiled kernel for that compute capability, so the "efficient" attention path has nothing to dispatch to.

The frustrating part: Unsloth's Qwen3 attention implementation calls its `xformers_attention` function unconditionally. It doesn't check a `HAS_XFORMERS` capability flag before calling it — the flag exists in the codebase, but the Qwen3 code path doesn't consult it before reaching for `xformers`. So there's no config flag or `use_xformers=False` argument that avoids this; the only way around it is replacing the function itself before the model runs:

```python
import torch.nn.functional as F
import unsloth.models.qwen3 as _unsloth_qwen3

def _sdpa_xformers_attention(Q, K, V, attn_bias=None):
    # Inference calls this with 5D tensors; training under gradient
    # checkpointing calls it with 4D tensors. Both need handling.
    if Q.dim() == 5:
        b, m, h, g, d = Q.shape
        Q2 = Q.reshape(b, m, h * g, d).transpose(1, 2)
        K2 = K.reshape(b, m, h * g, d).transpose(1, 2)
        V2 = V.reshape(b, m, h * g, d).transpose(1, 2)
        out = F.scaled_dot_product_attention(Q2, K2, V2, is_causal=True)
        return out.transpose(1, 2).reshape(b, m, h, g, d)
    else:
        Q2, K2, V2 = Q.transpose(1, 2), K.transpose(1, 2), V.transpose(1, 2)
        out = F.scaled_dot_product_attention(Q2, K2, V2, is_causal=True)
        return out.transpose(1, 2)

_unsloth_qwen3.xformers_attention = _sdpa_xformers_attention
```

`torch`'s built-in `scaled_dot_product_attention` (SDPA) has a Blackwell-compatible backend, so swapping the module-level function reference to a same-shape-in-same-shape-out SDPA implementation sidesteps `xformers` entirely, for both the shapes I actually saw: 5D tensors on the plain inference path, 4D tensors once gradient checkpointing was active during training. I reassigned the patch again right before calling `trainer.train()`, specifically so re-running the training cell alone — without re-running the cell that originally defined the patch — still works. A monkeypatch that only survives one specific notebook run order is worse than no patch, because it fails silently instead of loudly.

## Takeaways

- A dependency resolver's output can change based on how many indexes and packages you hand it at once — not just which pins you specify. If a `pip`/`uv` install pulls a version that makes no sense for a package unrelated to what you're actually installing, suspect index interaction before suspecting your pins, and try splitting the install into separate calls with separate index priorities.
- When a library's internal capability check exists (`HAS_XFORMERS` in this case) but a specific code path doesn't consult it, a hard error naming a missing kernel op is a stronger signal than the check ever was. Verify what actually gets called, not just what flag claims to gate it.
- A monkeypatch on a private module attribute is a stopgap tied to one library version, not a fix — and it should be idempotent and reapplied at the point it's actually needed, not just once wherever it was first defined, or a change in cell execution order silently undoes it.
