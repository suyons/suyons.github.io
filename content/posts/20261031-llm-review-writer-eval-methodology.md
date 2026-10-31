---
title: "LLM Evaluation - When the Small Model Copies the Example Instead of the Facts"
date: 2026-10-31
draft: false
tags: ["llm-evaluation", "vllm", "prompt-engineering", "qwen", "model-evaluation"]
categories: ["Backend"]
description: "Three days spent raising the score of a 4B model that writes pharmaceutical quality-report prose, and the lesson that the biggest gain came from taking a decision away from the model, not from better prompting."
showToc: true
---

A 4-billion-parameter model (Qwen3-4B, served with vLLM) writes the review prose for an annual pharmaceutical quality report: four short sections that read tables of deviations, out-of-specification results, and process-capability indices, then render a verdict on the year. The scores were low. The instinct is to reach for prompting tricks. What actually moved the numbers was mostly about learning to trust the measurement, and about taking decisions away from the model wherever code could make them instead.

## The metric that looked good was lying

Two graders score each draft. A rule-based grader checks what code can verify: every number traces back to the input, required control IDs are mentioned, the format is right, and a "causal-link" proxy counts sentences that contain two control IDs plus a relation word like "due to" or "caused by." An LLM judge — Claude Sonnet, called through the CLI — scores each draft 0 to 3 against a human-written gold answer, on four axes: facts, causality, verdict, and tone.

The causal-link proxy sat at 50%, and the obvious move was to push it toward 100%. Reading the actual sentences it had counted showed why that would have been wrong. In the source language, the relation word for "caused by" happens to sit as a substring inside an unrelated, much more common verb meaning "to confirm." Any sentence that said "confirm X and Y" tripped the same substring match. Half of the gold answers' follow-up rows were passing the causal-link check for a reason that had nothing to do with causality.

```python
# Before: a substring match anywhere in the sentence
if any(word in sentence for word in CAUSAL) and len(anchors.ids_in(sentence)) >= 2:
    out.append(sentence.strip())
```

```python
# After: only count a relation word at the start of a word, not buried inside one
# (the same substring also appears inside the verb meaning "to confirm")
HANGUL_RANGE = chr(0xAC00) + '-' + chr(0xD7A3)  # the Hangul syllable block
CAUSAL_WORD = re.compile('(?<![%s])(?:%s)' % (HANGUL_RANGE, '|'.join(map(re.escape, CAUSAL))))

if CAUSAL_WORD.search(sentence) and len(anchors.ids_in(sentence)) >= 2:
    out.append(sentence.strip())
```

The fix is one negative lookbehind on the Hangul Unicode block, so the relation word only counts when it isn't preceded by another syllable character, plus a regression test. The bigger lesson arrived right after: running a sentence-by-sentence comparison for one product, the rule-based score and the judge disagreed constantly, and the judge was the one that was right. A sentence can name two deviation IDs and the word "because" and still invent a connection the underlying data doesn't support — a pattern-matcher can't tell a real causal claim from a well-formed fake one. From that point on, the rule-based checks were demoted to guards (no invented numbers, no missing IDs) and the LLM judge became the actual score to optimize.

## Make the measurement able to tell signal from noise

The next experiment fed computed facts into the prompt: root causes grouped across deviations, counts of open deviations, counts of open and overdue corrective actions. The judge scores moved a little. Was that real, or noise?

Generation ran at temperature 0, so each configuration produced exactly one draft per section — and a changed input rewrote the whole text, not just the part that changed. One section's score swung by 2 points on a single sample. That's not a measurement, it's a coin flip. Two changes fixed it:

```js
// Sampling settings, overridable for measurement only.
// At temperature 0 every seed gives the same text, so comparisons
// raise the temperature and vary the seed to draw several drafts per input.
export const SEED = Number(process.env.PQR_SLM_SEED || 7);
export const TEMPERATURE = Number(process.env.PQR_SLM_TEMPERATURE || 0);
```

Each arm now generates at temperature 0.3 with seeds 1, 2, and 3, and the judge runs once per seed — 48 judge calls per arm across four training products. The three-seed means for a given arm differed by at most 0.2 on any axis, and the judge alone moved about 0.15 when scoring identical drafts twice. So the working rule became: a gap smaller than 0.3 between two arms' three-seed means is noise, not a result.

The other hygiene rule was less obvious: whenever an experiment changes what a section's input looks like, re-score the baseline drafts against the new input too. The judge sees the input alongside the draft, so comparing a draft judged against the old input to one judged against the new input mixes two variables into one number.

This baseline run also surfaced two failures that no amount of sampling noise explained: the verdict axis never exceeded 1 out of 3 across all 48 section-seeds, and the overall-summary section scored 0 on causality in three of four products, on every seed. Those weren't noise. They were where the real work was.

## Let code choose what code can choose

The follow-up-actions section is a table, one row per issue that needs attention next year. Its rows aren't a matter of judgment — they follow deterministic rules: every Critical deviation (grouped when they share a root cause, with any out-of-specification result or recall on the same lot attached), every deviation with an unexplained root cause, every open corrective action, the remaining out-of-specification results and recalls, and every capability index that falls short of target.

So code was given the job of picking the rows, handing the model a numbered list and asking it only to fill in two cells per row — what to do, who owns it — which code then merges back by row number. If there's nothing to follow up, the model isn't called at all.

```js
// Before: the model chose the rows and wrote the whole table
outputs[name] = await slm.generate(prompt, { model });
```

```js
// After: code chooses the rows; with nothing to choose, the model is skipped entirely
if (name === '12.4' && !parser.followUps(secs).length) { outputs[name] = parser.NO_FOLLOW_UP; continue; }
outputs[name] = await slm.generate(prompt, { model });
if (name === '12.4') outputs[name] = parser.followUpTable(secs, outputs[name]);
```

This breaks one of the project's own standing rules, which forbids putting the must-mention control IDs directly into the prompt — otherwise the "did it omit anything" metric passes by transcription, not by understanding. The exception is documented inline: for this one section, the omission metric is now measuring the code's row-selection logic, not the model's writing, and that's fine precisely because code is what decides the rows.

It was the single change that moved the judge decisively, on identical judge input for both arms: facts 1.4 → 2.2, causality 1.2 → 2.2, verdict 0.6 → 1.3, tone 1.9 → 2.6. The causal-link proxy actually went *down* at the same time it got better — the sentences it used to count were now table cells instead of prose, which is one more reason not to steer by that proxy directly.

One wording detail fell out of this too. A capability row that read "target 1.33 not met (close to target, monitor)" was read by the judge as hedging. Dropping the parenthetical and keeping just "not met" was both the more honest phrasing and the one the judge preferred.

## Four ideas that didn't work, and why writing that down mattered

Each of the following was measured with the same three-seed method and reverted when it failed the 0.3-gap bar:

- **Scoping the computed facts to only the tables each section reads.** Looked cleaner. The judge moved by 0.1 or less, the omission metric got worse, and one gold answer cites a figure that the narrower scope had removed.
- **Per-issue-type hints for the follow-up table** (e.g. "for an out-of-specification row, confirm the investigation and close it"). Small gains, all under the noise floor. A row with too few samples picked up an "unstable process" hint that didn't apply, and one product's rows kept a phrase copied verbatim from the worked example.
- **Removing the worked example from the follow-up-table prompt.** Worse on every axis. Actions went vague, cells shifted by one row, one row came back empty. The example lends phrasing the judge otherwise dislikes, but without it the 4B model can't produce the table shape at all.
- **Explicit cross-references for the overall-summary section.** Code joined each Critical deviation to the out-of-specification results, stability findings, and recalls from the same lot — exactly the links the gold answers make. The section stayed flat (causality 0.38 → 0.33). The model ignored the computed list and retold the worked example's story instead, substituting the current numbers: three products echoed another product's line about "everything that got worse points to one place," and one product described a recall that never happened, lifted wholesale from a different example.

That last result is the most informative failure of the four. The summary section wasn't short of facts. It was dominated by its worked example — the model was taking the example's plot and rewriting it with the current product's numbers, facts supplied or not.

## Where it landed

What shipped: a word-boundary fix in the causal-link grader, a three-seed measurement protocol with environment-variable overrides for temperature and seed, and code-chosen rows for the follow-up table — the one change that moved a section by roughly a full point on causality and facts. Across all four sections, judge means moved from roughly 1.4 / 0.9 / 0.8 / 1.7 to 1.5 / 1.3 / 1.0 / 1.9 on facts / causality / verdict / tone. Every rejected experiment is recorded with its numbers, not just the ones that worked.

Still open: the overall-summary section. The next candidate is choosing the worked example by the shape of the year — was there a Critical deviation, an out-of-specification result, a recall, a capability shortfall — instead of by text similarity, so a quiet year stops inheriting an eventful year's storyline. The catch is a pool of only three other products to draw examples from, so some years will have no close match at all.

## Takeaways

- Read what your metric actually counted before you optimize it. A proxy sitting at 50% that's half false positives is not a target to push toward 100% — it's a bug.
- One generation per experimental arm isn't an experiment if a changed input rewrites the whole output. Sample several seeds and decide in advance what gap counts as signal.
- Give the judge identical input across both arms you're comparing, or you're measuring two changes at once and can't tell which one moved the number.
- When an output follows a deterministic rule, take that decision away from the model entirely. The largest gain in this whole effort came from code choosing table rows, not from a better-written prompt.
- A small model follows its worked example's narrative more than it follows your instructions or your facts. Supplying the right data doesn't help if the example tells a different story — the example is the lever, not the prompt text around it.
