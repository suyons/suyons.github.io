---
title: "vLLM Troubleshooting - The Silent Truncation Bug That Faked My Baseline"
date: 2026-09-19
draft: false
tags: ["llm", "vllm", "lora", "quantization", "evaluation"]
categories: ["Backend"]
description: "Moving a fine-tuned report generator from Ollama to vLLM for multi-adapter LoRA serving looked like a server swap, until an evaluation error exposed that most of the published baseline had been silently generated from truncated prompts."
showToc: true
---

I've been building a small-language-model pipeline that fine-tunes Qwen3-4B with QLoRA to draft sections of a [pharmaceutical Product Quality Review](/posts/20260815-onprem-llm-regulatory-report-eval/) — [one adapter per report section](/posts/20260913-pqr-extraction-model-not-table-builder/), selected per request. It had been serving through Ollama with a quantized GGUF file. The goal was to move serving to vLLM so one base model could carry both adapters at once. What looked like swapping the server turned into four days of discoveries, most of them failures that never raised an error — including one that had quietly been faking an evaluation baseline for weeks.

## First attempt: keep the GGUF, just change the server

The original plan kept the training pipeline exactly as it was — train, merge the adapter, convert to a 4-bit `q4_k_m` GGUF — and only replaced the server. vLLM speaks the OpenAI chat API, so the client changed from Ollama's `/api/chat` to `/v1/chat/completions`:

```js
// Before (Ollama)
const payload = { model, messages, stream: false,
  options: { num_ctx: 8192, temperature: 0, seed: SEED } };
answer = await request('/api/chat', { body: payload });
if (answer.done_reason === 'length') throw new Error('cut off at the context limit');
const content = answer.message.content;

// After (vLLM, OpenAI-compatible)
const payload = { model, messages, stream: false, temperature: 0, seed: SEED };
answer = await request('/v1/chat/completions', { body: payload });
const choice = answer.choices[0];
if (choice.finish_reason === 'length') throw new Error('cut off; raise --max-model-len on the server');
const content = choice.message.content;
```

Getting vLLM itself running on an 8GB RTX 5060 laptop GPU under WSL2 hit two walls first. vLLM 0.29 had moved GGUF support out of its core into a separate plugin, so pointing it at a local `.gguf` file crashed while it tried to parse the file as a JSON config — installing `vllm-gguf-plugin` into a derived image fixed that. Then the engine died with `UVA is not available`: vLLM disables pinned host memory under WSL, and its newer model runner requires it. Setting `VLLM_USE_V2_MODEL_RUNNER=0` fell back to the older runner, and the server came up.

## The error that exposed a bad baseline

Re-running the evaluation on vLLM failed at the first block: "generation cut off at the context limit." The prompt was around 2,500 tokens; the model then generated 5,700 more, all of it reasoning, before running out of room. Digging into why led to the actual base model pulled from Ollama as `qwen3:4b`: its GGUF metadata identified it as Qwen3-4B-**Thinking**-2507, a thinking-only variant that ignores the usual "skip the reasoning" prompt switch. The model actually being fine-tuned was the original hybrid Qwen3-4B, so the "base versus tuned" comparison in the published results had been comparing two different models the whole time.

The worse finding was why Ollama had never complained. Its llama.cpp backend logs, once I went looking, showed exactly what happens at the context limit:

```
slot context shift, n_keep = 4, n_left = 8187, n_discard = 4093
stop processing: n_tokens = 4855, truncated = 1
```

Instead of stopping, it kept the first 4 tokens, discarded the next 4,093 — most of the instructions and input — and kept generating anyway. In the evaluation run that had produced the published baseline, 24 of 28 requests were truncated this way. The reported numbers looked reasonable. They were scoring answers written after most of the prompt had already been thrown away. Switching the base to the official hybrid Qwen3-4B made every request finish inside the context window in 4 to 22 seconds, and produced an honest — and lower — baseline.

## One base, two adapters: GGUF can't do it

With the baseline fixed, the original goal came back: one base model plus two independently selectable adapters. vLLM refused to even start a GGUF base with LoRA enabled:

```
AttributeError: 'VocabParallelEmbedding' object has no attribute 'weight'. Did you mean: 'qweight'?
```

vLLM's LoRA layers expect plain weight matrices, and a GGUF file quantizes even the embedding layer, so there's nothing for LoRA to attach to. That constraint rules out any workflow of "one base model, swap adapters per task" while staying in GGUF — the only option inside GGUF is merging each adapter into the base and exporting a whole new file per task, which throws away the point of having separate adapters at all.

The alternative is AWQ (Activation-aware Weight Quantization): a 4-bit format stored as plain safetensors, and one vLLM handles explicitly in its LoRA code path (`--enable-lora`). A pre-quantized `Qwen/Qwen3-4B-AWQ` checkpoint (2.67GB) replaced the merge-and-convert step entirely — the training pipeline is unaffected up through producing the LoRA adapter, and everything after that (merge, GGUF export) just gets skipped. At serving time, vLLM keeps one base resident in memory and hot-swaps LoRA weights per request by name, with no restart to switch tasks. That's a meaningfully different shape than the FP8 alternative (let vLLM quantize the full-precision 8GB checkpoint at startup — safer LoRA compatibility in some vLLM versions, but a ~4.5GB memory footprint that eats into the KV-cache budget on an 8GB card) or than staying on GGUF with fully merged files per task (simplest mentally, but ~4 minutes of downtime per task switch, and it doesn't scale past two adapters). AWQ was the only one of the three that didn't trade away either memory headroom or switching speed.

A throwaway container with the existing adapter confirmed the base and the adapter gave different answers to the same prompt, so the LoRA weights were actually taking effect. The pipeline got simpler, not more complex:

```bash
# Before: train, merge, convert, quantize, serve one merged file
finetune.py --merge && export_gguf.py && serve_wsl.sh   # one model per container

# After: train an adapter, serve the base plus every adapter found
finetune.py --out models/adapters/part-b
docker run ... vllm/vllm-openai:v0.29.0 --model /models/Qwen3-4B-AWQ \
  --enable-lora --max-lora-rank 16 --lora-modules part-b=/models/adapters/part-b part-a=/models/adapters/part-a
```

Requests now pick the adapter by name. The merge step, the GGUF converter, the plugin image, and roughly 20GB of old model files were deleted. One detail mattered enough to catch before it caused a silent mismatch: the chat template shipped with the AWQ checkpoint differs from the one used in training, so the server has to load the tokenizer saved alongside the adapter, not the one bundled with the base. Training and serving have to see the identical prompt, character for character, or the adapter is being evaluated on inputs it never actually saw during fine-tuning.

## Smaller traps along the way

- Windows file names are case-insensitive. Downloading a differently-capitalized copy of an existing GGUF file silently overwrote it.
- Git's `core.autocrlf` checked scripts out with CRLF line endings, which WSL bash rejects outright. A single `.gitattributes` line, `* text=auto eol=lf`, fixed it. One JavaScript file stayed broken because it contained a raw NUL byte used as a map-key separator, which makes Git classify the whole file as binary; replacing the byte with the `\0` escape sequence produced the identical string at runtime and let Git treat the file as text again.
- Inside WSL's login shell, `NAME` is already set to the machine's hostname, so a script's `${NAME:-model}` default silently served the model under the machine's name instead of the intended one.
- The container had no restart policy and stayed down after any reboot; `--restart unless-stopped` fixed that.

## Outcome and takeaways

The pipeline now trains one LoRA adapter per report section and serves a single AWQ base with both adapters from one vLLM container. Still open: re-measuring the baseline on the AWQ base rather than the GGUF one, training the second adapter, and pointing the other section's model calls — still configured for the now-stopped Ollama server — at vLLM.

- **A server that silently truncates instead of erroring can corrupt a baseline for weeks without anyone noticing.** Ollama's context-shift behavior kept generating past a blown context window instead of stopping; the "cut off at the context limit" error I got from vLLM was the single most useful thing that happened all week, because it was the only failure that forced a look underneath a number that had already been published.
- **Before trusting a base-versus-tuned comparison, confirm both sides are actually the same base model.** A model tag (`qwen3:4b`) is not a guarantee of which checkpoint it points to — GGUF metadata was the only way to catch that the "base" had silently become a different variant.
- **A quantization format choice is a LoRA-compatibility choice, not just a size/speed tradeoff.** GGUF quantizes the embedding layer, which vLLM's LoRA code can't attach to at all — a hard blocker discovered only by trying, since the failure mode is an `AttributeError` deep in model loading, not a documented incompatibility flagged up front.
