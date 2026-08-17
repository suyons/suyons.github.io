---
title: "On-Prem LLM Design - Never Let the Model Do the Math"
date: 2026-08-15
draft: false
tags: ["llm", "qlora", "rag", "fine-tuning", "evaluation", "synthetic-data"]
categories: ["Backend"]
description: "Designing a fine-tuned local LLM to draft regulated compliance reports means the eval harness has to exist before the model does — and the sharpest design decision is which parts of the document the model is never allowed to touch."
showToc: true
---

I've been scoping a pipeline that fine-tunes a small, locally hosted LLM to draft a specific kind of document: a pharmaceutical Product Quality Review, the report a QA team writes once a year per product, pulling deviations, CAPAs, out-of-spec results, and batch records out of half a dozen systems and turning them into a document with a fixed regulatory structure. Nothing here is trained yet. This post is about the eval harness, which I built before writing a line of training code, and about the one design rule that shaped everything else: the model is never allowed to compute a number.

## Why on-prem, and why that constrains the model choice

The source data — batch records, deviation counts, lab results — is exactly the kind of thing a manufacturer will not hand to a commercial LLM API. So the whole pipeline has to run inside the customer's network: weights, retrieval index, everything. That one constraint eliminates most of the easy choices, and it means the base model needs a license that actually permits shipping it inside a paid product rather than research use only. That's why the base model is Qwen3-4B-Instruct (Apache-2.0), fine-tuned with QLoRA via `unsloth`, exported to GGUF, and served locally through Ollama on an 8GB card.

## Three jobs, three different owners

The instinct with a document-generation task is to hand the whole thing to the model: give it the source records, ask for the report. I split it three ways instead, and the split is the actual design decision this project rests on.

- **Numbers are computed by code, never by the model.** Deviation counts, CAPA overdue counts, batch pass rates — a small aggregation function reads the source records and hands the model a value to cite. The model's only job with a number is to put it in a sentence in the right place. Two reasons: a 4B model counting forty records is going to get it wrong sometimes, and in a document like this a wrong number isn't a typo, it's grounds to throw the whole report out.
- **Regulatory facts and citations come from RAG, not from weights.** "Does a Product Quality Review need to cover CAPA effectiveness checks" is a question with a sourced answer that can change when a regulation is revised. Baking that into fine-tuned weights means re-training every time a guideline updates. Retrieval keeps it swappable and keeps the citation traceable back to a real document.
- **Structure, section order, and register are what fine-tuning is for.** The skeleton, subheading wording, and the flat, declarative tone these documents are written in are fixed and don't change per product. That's a style problem, and QLoRA is the right tool for a style problem in a way it is not the right tool for "did this number come out correctly."

The retrieval half — `bge-m3` embeddings, a Chroma index, BM25 hybrid search — is the same setup from [an earlier retrieval eval I wrote up](/posts/20260804-rag-retrieval-eval-hybrid-search/), reused as-is because there was no reason to redesign a retriever that already had a working eval attached to it.

## No real data yet, so the ground truth has to be manufactured on purpose

The obvious problem: there's no training data, because the customer's actual records don't exist in this project yet. The fix is a two-stage synthetic pipeline, and the reason it's two stages instead of one is what makes automated grading possible at all.

Stage one generates structured source records as JSON — deviations, CAPAs, batch history, stability results — with specific numeric values planted in them on purpose. Stage two generates the target report from *only* that JSON, with the prompt forbidding any claim that isn't traceable to a field in it. Collapse those two stages into one call and you get a plausible-looking report with no way to check where any of its numbers came from. Keep them separate and the JSON becomes an answer key: pull the same values back out of the generated report and diff them against the source.

```python
import re

def anchor_citation_accuracy(draft_text, anchors):
    """For each planted ground-truth value, check whether the generated
    section cites it verbatim as a number."""
    hits = sum(
        1 for value in anchors.values()
        if re.search(rf'\b{re.escape(str(value))}\b', draft_text)
    )
    return hits / len(anchors)

anchors = {"deviation_count": 12, "capa_overdue_count": 3, "batch_pass_rate": 97.2}
draft = "Deviations totaled 12 for the period, with 3 CAPAs overdue against plan."
print(anchor_citation_accuracy(draft, anchors))  # 0.666... — batch_pass_rate never cited
```

That splits into two separately-tracked scores that a single "is this report correct" grade would hide: whether an upstream step correctly reconstructs the JSON values in the first place, and whether the generation step then actually quotes them. A drop in the second without a drop in the first means the retrieval or prompt is losing the value on the way to the sentence, not that the source data is wrong — a distinction that matters for knowing what to fix.

## Splitting by product, not by row

The eval set holds out entire products, not a random sample of report sections. In production the model has to write a PQR for a product it has never seen a single record from — every deviation, CAPA, and batch number will be new. A random split lets the model see records from the same product in both training and eval, which measures something closer to memorization than generalization. Splitting on the product boundary is the only split that matches how the tool actually gets used.

## The VRAM ceiling turned into an architecture decision

An 8GB card with a 4-bit base model, a LoRA adapter, and optimizer state leaves a sequence length ceiling of about 2048 tokens. A finished report runs tens of pages, so generating the whole document in one pass doesn't fit. The fix is to make the unit of generation a single section instead of the whole document: each section gets only the aggregated numbers and retrieved regulatory context it needs, and code stitches the sections together afterward.

The constraint turned out to match the real usage pattern anyway — a QA reviewer is far more likely to ask "regenerate the CAPA section with this quarter's numbers" than to want the entire report rewritten because one section changed. A hardware limit forcing chunked generation is not always a lucky accident; here it happened to be one.

## Deliberately breaking the data

Alongside the clean synthetic scenarios, some are broken on purpose — a month of batch records missing, a deviation total that doesn't match the sum of the itemized list. The question these test isn't formatting, it's whether the model states that data is missing or inconsistent instead of inventing a plausible number to keep the sentence coherent. That failure mode matters more here than anywhere else in the eval: a fluent paragraph with a fabricated pass rate is worse than an empty section, because the empty section gets caught in review and the fabricated one might not.

## Comparing against a baseline that can't lose on the metric everyone asks about first

A fair amount of what a PQR needs is aggregation dropped into a fixed template — which means the first objection anyone hears is "why not just fill a template with the aggregated numbers and skip the model entirely." So that's in the comparison as a real baseline, not a strawman: a zero-LLM rule-based renderer, alongside the untuned base model with RAG and a handful of examples, alongside the fine-tuned model.

The honest tradeoff is that the rule-based baseline wins the numeric-accuracy column outright, by construction — it has no path to inventing a number, because it never generates prose at all. If numeric accuracy were the only column in the table, fine-tuning would look like it wasn't worth doing. It isn't the only column. The rule-based baseline has no answer at all for "why does this CAPA delay matter" or "does this report satisfy a specific regulatory citation," and it has no way to *say* "data missing" versus silently rendering a blank field — it can't attempt those questions, fine-tuned or not. The case for fine-tuning is the columns a template structurally can't compete in, not the one it wins by default.

## What's still just a plan

Every number in the design above — the split sizes, the citation-accuracy target, the trap-detection target — is a target, not a result, because none of this is trained yet. I built the harness in this order on purpose: if the fine-tuned model can't clear the untuned baseline on the trap category once it exists, I'd rather the eval catch that than find out from a customer. Whether Qwen3-4B has enough capacity left after QLoRA to hold both the section-format habits and the "say I don't know" behavior at the same time is the open question the next phase actually has to answer.
