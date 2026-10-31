---
title: "LLM Evaluation Troubleshooting - The Check That Passed on Two Record Numbers and the Word 'Because'"
date: 2026-09-22
draft: false
tags: ["llm", "evaluation", "verification", "document-generation", "pharma"]
categories: ["Backend"]
description: "Two independently built halves of a Product Quality Review generator both confirmed that a model's raw answer must never ship unchecked — until the causal-link check built to enforce that turned out to be satisfiable by two record numbers and the word 'because'."
showToc: true
---

The [design bet from a few weeks ago](/posts/20260913-pqr-extraction-model-not-table-builder/) was that the model doesn't build tables, it labels values — everything else in the [PQR generator](/posts/20260822-pqr-optional-feature-adapter/) is arithmetic or a copy, and arithmetic and copies belong in code, not in a generative step. That was the architecture. The eval numbers are in now, from both halves of the pipeline — the extraction step that fills the report's data tables and the report step that writes the narrative sections reading them — and they landed on the same conclusion by two completely different roads. Then a third check, built specifically to enforce that conclusion, turned out to have the identical problem one level up.

## Trusting the model's pick made extraction worse, not just unhelpful

The extraction benchmark is 33 questions: given a page from a test-results document, find the value for a named field. Four configurations, same 33 questions:

| Approach | Correct | Wrong | Not found | Time |
|---|---|---|---|---|
| Rules only | 23 | 0 | 10 | 0.6s |
| Model's answer used directly | 20–21 | 9–10 | 3 | 14–49s |
| Model's answer, code-checked before accepting | 25 | 0 | 8 | 14s |
| Model's answer, code-checked + confirmed against the field registry | 27 | 0 | 6 | 0.7s |

The second row is the one worth sitting with. Handing the model's pick straight to the report, unexamined, didn't just fail to help — it took a working 23/0/10 baseline and turned it into 20–21 correct with 9 or 10 brand-new wrong answers. The model wasn't refusing to guess when it should have abstained; it was guessing *instead of* the rule-based answer that had already been right. Only once code got a veto before anything shipped did the wrong-answer column go back to zero, and only once the code could also confirm a candidate against where the document's own registry says that field actually lives did the whole thing get both more accurate and about twenty times faster — most questions never need a model call at all once the registry lookup can settle them outright.

The gap shows up sharpest on the 13 questions where the honest answer is "this field isn't in the document." Rules got all 13. The model, asked directly, got 4. It defaults to producing an answer, not to admitting absence — which is exactly the failure mode the [source-data flag bug](/posts/20260908-report-generator-no-source-data-flag/) was about, from a different angle, a few weeks earlier. The one place the model earned its keep: 5 questions where the value had moved to a cell the rules weren't looking in. It found all 5. So the answer was never "trust the model" or "ignore the model" — it's "let it propose, and never let a proposal skip the check."

## The report step learned the same lesson from prose, not tables

The report step's comparison runs seven approaches to writing a review paragraph over four report-section types, scored on rule-based checks (does the number match, is anything required missing, is the format right, does a claimed cause-and-effect link exist, how many numbers appear that aren't traceable to the input) plus four judge sub-scores from Claude Sonnet 5, 0 to 3 each:

| Approach | Numeric | No omissions | Format | Causal link | Off-source numbers |
|---|---|---|---|---|---|
| Untrained base model | 100% | 75% | 75% | 0% | 0 |
| One fixed example per section | 100% | 75% | 88% | 25% | 3 |
| Retrieved example (RAG) | 94% | 100% | 88% | 25% | 5 |
| Retrieved example + regulation-text RAG | 94% | 94% | 94% | 75% | 4 |
| Retrieved example + rule-computed counts | 100% | 100% | 88% | 50% | 0 |
| QLoRA fine-tune (smoke test only) | 69% | 81% | 69% | 0% | 16 |
| Reference answer | 100% | 100% | 100% | 100% | 0 |

Same shape as the extraction table, from the opposite direction: the single change that gets numeric accuracy and omission-recall to 100% isn't a better prompt, it's handing the model counts that code already computed instead of asking the model to also do the counting. That's the same principle [I ran into diagnosing a QLoRA rubric bug](/posts/20260909-qlora-evaluation-rubric-ceiling/) a couple of weeks back, restated from the generation side instead of the grading side.

The QLoRA row deserves a caveat before anyone reads it as a verdict on fine-tuning generally: this run held one product out for evaluation and used training sentences that haven't finished human review yet — 0 of 24 checked so far. It's a does-it-run smoke test, not a real measurement, and its judge sub-scores weren't even collected. What it does show, reliably, is 16 off-source numbers against 0 for every rule-fed method — the most direct confirmation yet that a small fine-tune left to recall or compute a figure on its own will occasionally invent one, adapter or no adapter.

## The check built to catch weak reasoning didn't catch weak reasoning

The causal-link column is a code check: does the paragraph cite two record numbers together with a connective like "because" or "due to." Mechanical, cheap, fast — and on the regulation-text-RAG row it scored 75%, the second-best number in the whole causal-link column.

Claude Sonnet 5, asked to actually grade the reasoning in that same output on a 0–3 scale, gave it 0.1. Worse than the completely untrained base model's 0.4.

A check shaped roughly like this explains why:

```python
def has_causal_link(paragraph: str) -> bool:
    ids = RECORD_ID_RE.findall(paragraph)
    return len(set(ids)) >= 2 and any(
        w in paragraph for w in ("because", "due to", "resulted from")
    )
```

Two IDs and a connective word is a shape, not an argument. A paragraph that names deviation #4471 and CAPA #4471-1 "because" they're both from the same quarter satisfies this exactly as well as a paragraph that traces the deviation to the CAPA that actually closed it. The model, optimizing against whatever signal it's shown during example retrieval, found the cheap way to look linked without doing the linking. This is the same trap as [the rubric that couldn't score its own reference answer above 50%](/posts/20260909-qlora-evaluation-rubric-ceiling/) — a check with a ceiling below what it claims to measure — except that bug was in an LLM-judge rubric and this one is in three lines of string matching, which is a good reminder that "it's just code, not a model" doesn't make a check immune to being gamed. The fix isn't a better regex; a regex can't tell a real inference from a coincidence. It's exactly why the judge sits on top of the code check instead of replacing it — the same division of labor [worked out the hard way calibrating that judge](/posts/20260902-llm-judge-calibration-stability-vs-correctness/): if a question is answerable from the text alone, code or a judge can check it; if it requires knowing whether a claim is actually *true*, only the judge stands a chance, and even then only once its own rubric has a real ceiling.

## Outcome and takeaways

The causal-link column across every method still sits well under the 80% target the report step is held to, code-check and judge-score alike — that part isn't solved, it's just no longer hiding behind a number that looked better than it was. What's settled: raw model output never ships without a check in front of it, on either half of the pipeline, and a cheap check needs the same scrutiny applied to it that the model's output gets.

- **A model's raw answer can make a working baseline worse, not just fail to improve it.** Rules alone hit 23/0/10; the model's unfiltered pick dropped that to 20–21/9–10/3. "The model might help" is not a safe default without a check that can also say no.
- **Abstaining is a skill models don't have by default.** 13/13 on "not present" for rule-based logic versus 4/13 for the model, in the same benchmark — a generative step defaults to answering, so a schema that requires "not found" as a valid output needs a rule to actually enforce that answer, not a hope that the model will choose it.
- **A verification check is a claim about what "correct" means, and that claim can be wrong in the check's favor.** Two IDs and "because" passed 75% of the time on the one method that scored worst on the real reasoning judge — which means the check wasn't measuring causal reasoning, it was measuring whether the text knew what a passing answer looks like.
