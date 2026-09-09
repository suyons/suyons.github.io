---
title: "QLoRA Evaluation Troubleshooting - The Reference Answer Couldn't Score Full Marks on Its Own Rubric"
date: 2026-09-09
draft: false
tags: ["llm", "evaluation", "fine-tuning", "qlora", "small-language-models"]
categories: ["Backend"]
description: "A QLoRA fine-tune looked worse than a rule-based baseline on four of five metrics, and the one place it seemed to differ turned out to be a rubric that couldn't score even a correct answer above 50%."
showToc: true
---

I was pulled in to review the evaluation table for a small-data QLoRA fine-tuning experiment: a model trained on 16 examples to draft the narrative sections of a structured report — the parts that state what happened and connect it to a cause, not just fill in numbers. Four rows, five metrics, and a first read that says the fine-tune failed. The actual finding was more useful than that, but getting there meant distrusting the table before distrusting the model.

## The table that looked like a verdict

| Approach | Numeric accuracy | No omissions | Evidence cited | Format | Causal link |
|---|---|---|---|---|---|
| Rule-based | 100% | 100% | 100% | 100% | 0% |
| Base model | 100% | 67% | 100% | 62% | 8% |
| QLoRA (16 examples) | 71% | 67% | 100% | 79% | 8% |
| Reference answers | 100% | 100% | 100% | 100% | 50% |

The obvious read: QLoRA loses to a plain rule-based generator on four of five columns, and the one column where it might matter — linking an outcome back to its cause, the thing a rule-based template genuinely can't do — shows 8% against 0%. An eight-point gap on a low base is not a result you'd want to present as the payoff for fine-tuning anything.

The project's own notes already had half of the right diagnosis: a rule-based approach scoring 100% isn't a compliment. The checker just happens to measure exactly the things a fixed template is good at — correct arithmetic, the right heading, no omitted field. None of that requires understanding the report. So the instinct was to read QLoRA's weak numbers as "needs more training data." That instinct skipped a check that should come first.

## Checking the ceiling before blaming the floor

Row four is the reference answers — the hand-written correct output — scored against the same rubric used to grade the model. That row exists to answer one question: if you feed the rubric a genuinely correct answer, does it say 100%? For four of the five columns it does. For causal link, the correct answer scores 50%.

That's not a property of any model. It's a property of the rubric. If the right answer can't clear its own bar, the bar is measuring the wrong thing, and no amount of retraining will move a number that's capped below 100% by design. This is the same failure mode as an LLM-as-judge question that's structurally unanswerable from the text it's given — [I ran into a version of that same trap calibrating a judge for a different project](/posts/20260902-llm-judge-calibration-stability-vs-correctness/) — except here the checker is plain code, not a model, and the fix is correspondingly more mechanical once you see it.

## The denominator was counting things that can't be counted

The project's scenarios split into two kinds: some have a genuine cause to report, some don't — a deviation with no discernible root cause is a valid, correct outcome to write up as "no link found." Two of the scenario types in this dataset are exactly that: nothing to connect. The causal-link metric's denominator, as built, counted every scenario, including the ones where the honest answer produces no link. Scoring "correctly reported no link" as a miss is why the reference answer — which gets those two scenarios exactly right — still only reaches 50%.

The fix is to shrink the denominator to the scenario types where a link is actually expected. Once you do that, the reference answer's causal-link score goes from 50% to 100%, because it was never wrong on the two "nothing to connect" scenarios — the metric was just charging it for scenarios that were never scoreable in the first place. The same correction applies to QLoRA's 8%: that number was computed against the same six-scenario denominator, so it's undercounting too, though by how much isn't something I have in hand yet — recomputing it against the corrected four-scenario base is the first thing to check before drawing any conclusion about whether QLoRA learned causal linking at all.

## Once the metric's honest, the real failure is still there

Fixing the denominator changes the ceiling, not the model's numeric accuracy column, and that one doesn't need a second look: 71% against a required 100%. With 16 training examples, the model learned the shape of the narrative — the sentence structure, where a number goes, how a cause clause is phrased — and it fills the number slot with something plausible rather than the value that was actually in the source data. That's not a fine-tuning bug to train away with more of the same 16 examples. It's what small-data fine-tuning does when a task requires both structure and specific facts: it generalizes the structure fine and hallucinates the facts, because 16 examples aren't enough to pin down "copy this exact digit" as a hard constraint rather than a soft pattern.

## Take the numbers out of the model's hands

If the requirement is that numbers must be exactly right, don't ask a 16-example fine-tune to reproduce them from context. Replace the numeric literals in the training targets with named placeholders — `{deviation_count}`, `{capa_exceeded}`, and so on — train the model to produce the placeholder in the right grammatical slot, then substitute the real value back in with a plain string replace after generation. Numeric accuracy stops being something the model is graded on and becomes something the substitution code guarantees by construction. What's left for the model to actually learn is the part small-data fine-tuning is plausibly good at: sentence structure and which cause goes with which outcome.

This is the same idea as keeping a numeric check in code instead of asking an LLM judge to eyeball a number — push the part that has one correct answer out of the generative step entirely, and only ask the model to do the part that requires judgment.

## Order of operations under a deadline

With a demo and a live Q&A still ahead and not much runway left, the tempting move is to jump straight to augmentation — more training examples, hope the numeric accuracy climbs. That's the most expensive fix and it doesn't touch the rubric bug at all. The corrected order:

1. **Fix the denominator.** Pure scoring code, no retraining, done in about an hour, and it's the only change that makes the causal-link number mean anything.
2. **Move numbers out of the generation target.** Also no new data collection — reshape the existing 16 examples to use placeholders, retrain once, and numeric accuracy becomes structurally 100%.
3. **Augment only if the first two aren't enough**, and even then, prefer holding back a subset of scenarios from an existing larger pool over hand-writing new anchored examples — cheaper, and it doesn't require re-deriving ground truth from scratch for every new case.

Steps one and two require no new labeled data, which matters when data collection is the slow part.

## Outcome and takeaways

The table didn't get worse once the rubric was fixed — it got legible. QLoRA's headline numbers move once the denominator is corrected, but that correction hadn't been run yet as of this review, so the honest state is: the rubric bug is identified and fixable in an hour, and the actual causal-link number for QLoRA is currently unknown rather than 8%.

- **Before concluding a model failed a metric, run the metric on a known-correct answer.** If the correct answer can't score 100%, you've found a bug in the test, not in the model.
- **A denominator that counts unscoreable cases silently caps every score below it.** "No link exists" and "failed to find the link" are different outcomes and need different buckets, or the metric can't tell success from a structural ceiling.
- **When a fact must be exactly right, don't train a small model to reproduce it — template it out.** Fine-tuning on a handful of examples is good at teaching structure and bad at teaching "copy this digit exactly." Give the model the structure and let code own the fact.
