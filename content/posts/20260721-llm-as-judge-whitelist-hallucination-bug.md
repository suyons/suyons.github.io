---
title: "LLM Evaluation - The Judge Agreed With the Answer and Still Failed It"
date: 2026-07-21
draft: false
tags: ["llm-as-judge", "evaluation", "prompt-engineering", "local-llm", "machine-learning"]
categories: ["Backend"]
description: "An LLM-as-judge evaluation harness ran clean and produced a confident 3.6/5 average with a 30% hallucination rate. Both numbers were wrong: the judge had silently turned a minimum checklist of key points into an exhaustive whitelist, penalizing answers for quoting the source text verbatim."
showToc: true
---

I was porting a small-language-model training and evaluation notebook, originally written for a hosted cloud GPU, to run against a local model server instead. Most of the port was mechanical. The instructive part was the evaluation step: a harness that used one LLM to grade another's answers — the common LLM-as-judge pattern — ran to completion without a single error and handed back numbers that were confidently wrong.

## The setup

With no external API credentials available, both roles were filled locally: a 4B-parameter model as the system under test, and an 8B-parameter model, a same-family model roughly twice the size, as the grader. Each eval item carried a `key_points` list — a checklist of what a correct answer should contain, meant as a *minimum* bar.

The run completed cleanly. Average score: 3.6 out of 5. Hallucination rate: 30%. For a 4B instruction-tuned model on a task no harder than "summarize this review and don't make things up," both numbers were suspiciously bad — bad enough that the more likely explanation was the measurement, not the model.

## What the judge actually did

The judge had interpreted `key_points` as an *exhaustive whitelist* rather than a minimum checklist. Any content in an answer that wasn't in that list got flagged as fabrication — even when it was quoted verbatim from the source text being summarized. One item was penalized for citing a word that is literally the first word of the source review. The clearest tell: on a deliberately unanswerable trap question, the model gave the correct "not stated" response, the judge's own written justification agreed with it, and it still awarded the lowest possible score.

The original grading instruction was short:

```text
Criteria: coverage of key points (accuracy), adding facts not present (hallucination),
concision. For trap questions, admitting "unknown / not present" is the correct answer.
```

Terse enough that "hallucination" had no explicit anchor — the model was free to infer one, and it inferred the wrong one: content, not source.

The fix defines hallucination against the source material instead of the answer key, and states the trap-question rule explicitly instead of leaving it implied:

```text
1. Accuracy: does it cover the key points? The key points are a MINIMUM checklist;
   different wording with the same meaning counts as correct.
2. Hallucination: set true ONLY when the answer invents content found neither in the
   original source text nor in the key points. Quoting or summarizing wording that
   actually appears in the source is NOT hallucination, even if absent from the key points.
3. For trap questions, answering "unknown / not stated" IS the correct answer -> score 5,
   hallucination false. Inventing a specific number, name, or brand -> score 1-2, true.
4. Concision is a tiebreaker, not a deduction.
```

It's worth being fair to the original judge here: it wasn't uniformly wrong. It correctly caught two genuine model failures — a misread of a phrase that was actually a compliment, and a trap question where the model invented an average rating and a review count out of a single data point. That second catch is exactly what the trap item was designed to detect, which is evidence the eval set itself discriminates properly. The bug was specifically in how "hallucination" got defined, not in the judge's competence generally.

## Why this is easy to miss

An evaluation harness that runs without errors *looks* trustworthy precisely because it's quantitative — a 3.6 with a clean JSON schema reads as more authoritative than it is. The only way to catch the whitelist interpretation was to read individual judgments, not the aggregate score. Reading a handful of the actual `reason` fields the judge produced took a few minutes and surfaced the bug immediately; staring at the average alone would never have.

There's a second, quieter lesson underneath the first: once the rubric was fixed, the headline metric moved more from rewording the *grading prompt* than it would have from swapping the model under test. That's uncomfortable if you're using this kind of score to compare models, and worth knowing before quoting any such number in a report.

## Outcome and limitations

With the corrected rubric, the same harness stopped penalizing verbatim quotation and started distinguishing "unknown" from "wrong" on trap questions correctly. One limitation carries forward regardless: an 8B same-family local grader is materially weaker than a frontier model used as judge, which is why a manual spot-check of a sample of judgments stays in the loop rather than trusting the automated score outright.

## Takeaways

- **Verify the measurement before trusting the measured.** A harness that runs without errors can still be producing noise. Reading a handful of individual judgments caught what the aggregate score concealed.
- **Ambiguous rubric fields get interpreted, not obeyed.** "Key points" silently became a whitelist. If a field is a minimum bar, say so in the prompt — graders don't infer intent, they pick one.
- **A judge that's right most of the time can still have one systematic, high-impact error.** Being mostly correct is exactly what makes the wrong part hard to notice.
- **Rephrasing the grading prompt can move the score more than changing the model does.** That's a reason to distrust small differences between models graded with different prompt versions.
