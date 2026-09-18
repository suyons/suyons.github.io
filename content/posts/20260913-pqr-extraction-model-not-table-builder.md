---
title: "Report Generation Design - The Model's Only Job Was Naming Which Number Was pH"
date: 2026-09-13
draft: false
tags: ["llm", "small-language-models", "document-generation", "fine-tuning", "pharma"]
categories: ["Backend"]
description: "A design review reset a two-stage report generator's architecture with one line — the model doesn't build tables, it labels values — and that reframing dissolved three infrastructure problems before any of them had to be solved."
showToc: true
---

I sat down to design the piece of the [Product Quality Review generator](/posts/20260822-pqr-optional-feature-adapter/) that still didn't exist: the test-results tables, the ones with a process-capability score in the last column. The plan I brought to review had a small language model reading a document's raw text extract and writing the finished markdown table straight out — n, min, max, mean, Cpk, pass/fail, the lot. One line from the reviewer undid that plan: the model doesn't build tables. Everything in that table except two fields is either arithmetic or a copy. The model's entire job is finding the one thing that isn't: "this value is pH."

## Sorting the columns by what actually needs a model

Every column in the target table falls into one of two buckets. Document number and approval date are copied verbatim from the source record. n, min, max, mean, Cpk, and pass/fail are all computed from a set of measured values — not read, computed. Neither bucket needs a language model at all; both are deterministic once you have the raw numbers. The only thing in the whole table a model is actually useful for is the step before either bucket: given a page of unstructured text, find the reading and say what it's a reading *of*. Everything downstream of that is arithmetic and formatting, and arithmetic and formatting are exactly the things you don't want a generative model doing when the output has to be exactly right — the same lesson I ran into later [debugging why a fine-tune couldn't hit 100% on numeric accuracy](/posts/20260909-qlora-evaluation-rubric-ceiling/), except here it heads the problem off at the design stage instead of surfacing three weeks in.

So the model's output stops being a table and becomes a small JSON object: field name, value, maybe a unit. A second, entirely separate step — plain code — takes those objects, runs the statistics, and renders the markdown. I'll call the two halves the extraction step and the report step, since that's what they actually do.

## Skip the fine-tune, force the shape instead

The instinct when you need JSON out of a model is to ask nicely for JSON in the prompt. That's the failure mode that eats a training budget: the model drifts on field names, wraps the object in a sentence, or drops a field it decided wasn't worth mentioning. The fix doesn't need training data at all — it needs the serving layer to enforce the shape:

```python
import ollama

schema = {
    "type": "object",
    "properties": {
        "field": {"type": "string"},
        "value": {"type": "number"},
    },
    "required": ["field", "value"],
}

response = ollama.generate(
    model="llama3.1",
    prompt=extraction_prompt,
    format=schema,  # constrains decoding to schema-valid JSON
)
```

Ollama's `format` parameter takes a JSON schema directly on recent versions (older releases only accept the literal string `"json"`, which guarantees valid JSON but not the fields you asked for); vLLM has the same idea under `guided_json`, and llama.cpp enforces it with a GBNF grammar. None of that is fine-tuning — it's the serving layer refusing to emit a token that doesn't fit. If plain extraction under a forced schema gets the right values out, there's nothing to fine-tune for this half of the pipeline at all. A fine-tuned adapter for extraction only gets added later, and only if forcing the shape doesn't fix the *content*.

## One model on the wire at a time, on purpose

The report step still needs a model — someone has to turn "capable, one deviation logged this quarter" into a paragraph a reviewer can read. My first sketch had both models loaded together, one small adapter swapped in per stage on top of a shared base, "like changing the bit on a drill." That metaphor turned out to describe multi-LoRA serving, which vLLM supports — pick the adapter per request against one running server. Ollama doesn't: an `ADAPTER` line in a Modelfile bakes a LoRA into a new named model, so a base and two adapters become two separately registered models, not one model with a swap parameter. The layer underneath Ollama can reportedly hot-swap adapters at request time; Ollama hasn't exposed that yet, and there's apparently an open feature request for it — I haven't independently confirmed the exact issue number, so treat that specific detail as unverified.

None of that ends up mattering, because the two stages never run at the same time anyway. Extraction has to finish, a human has to fill in whatever it couldn't find, and only then does the report step run on the completed data — a real dependency, not a scheduling choice. So the pipeline is three separate command invocations: run extraction, wait for manual entry, run the report step. One model is ever loaded. The few seconds a model load costs is nothing against a five-minute-per-report target, and the adapter hot-swap problem stops being a problem because the thing it was solving never existed.

## Keeping training and grading from lying to each other

Splitting the two stages into separate commands also settles a fine-tuning question I hadn't fully closed: whether to eventually train one model on both extraction and report-writing, since they run back to back anyway. Don't. Extraction has one correct answer per field, gradeable by exact string match in well under a second. Report-writing needs judgment — an LLM-as-judge setup, slower and fuzzier by nature. Training one model on both tasks with a small dataset risks exactly the pathology [I later diagnosed from the outside](/posts/20260909-qlora-evaluation-rubric-ceiling/): the model learns the narrative's shape fine and starts inventing the numbers that shape wants, because a handful of examples pin down structure long before they pin down "copy this digit exactly." Keeping the tasks in separate models with separate eval harnesses means a bad report score can't hide behind a good extraction score, or the reverse — each stage gets checked against what it actually had to do.

The eval set for extraction also needs a shape the reviewer specifically pushed back on: not just documents from the format I'd been developing against. The plan now is three buckets — the development format, a second real format with different column order and different field labels that I haven't looked at while building, and a handful of "trap" documents where the field genuinely isn't present and the only correct answer is saying so. The metric that matters isn't either format's score alone, it's the *gap* between them — reusing the same "under 10 percentage points between formats" bar the report generator already holds itself to elsewhere, just retargeted from PDF layouts to this JSON extract. A model that only works on the format it saw during development hasn't learned to extract pH, it's learned to parse one file layout, and a 95% score against a pile of that one format would prove exactly that and nothing else.

## The number that isn't in the source at all

One question from review cut straight to a wrong assumption I'd been carrying: is Cpk in the source record, or does something have to compute it? It's not in the source. Cpk comes from the mean and standard deviation of measured values pulled across a batch of records, and that arithmetic doesn't belong anywhere near the model — [I've already had to fix that function once](/posts/20260901-cpk-zero-variance-guard-floating-point/) for floating-point edge cases a hand-picked test array never exercised, and I'd rather fix `cpk()` again someday than debug a model's arithmetic, which you can't fix, only retrain and hope. The pipeline settled into: extract raw values under a forced schema, hand them to the same rule-based `cpk()`, and let a rule-based renderer build the table from the result. The model touches exactly the part of this that has no other way to be automated — reading unlabeled prose — and nothing else.

## Outcome and takeaways

None of this is built yet; it's the architecture that came out of one review pass, before a line of the extraction prompt is written. What's settled: model output is a labeled value, not a table or a computed statistic; the two stages stay separate models with separate eval harnesses; adapter hot-swapping was a solution to a scheduling problem that a strict pipeline order removes on its own; and the eval set has to include a format the model never saw during development, or a high score proves nothing.

- **Before asking what a model should output, sort the target's fields into "copied," "computed," and "found."** Only the last bucket needs a model at all — the first two are usually the majority of the columns.
- **A serving-layer schema constraint beats a training run for getting structure right.** Fine-tune for content the model gets wrong under a forced shape, not for the shape itself.
- **A dependency between two stages can retire an infrastructure problem instead of requiring you to solve it.** Adapter hot-swapping only mattered under an assumption — concurrent models — that the actual workflow never needed.
