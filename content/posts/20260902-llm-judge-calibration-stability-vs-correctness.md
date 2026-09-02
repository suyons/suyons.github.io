---
title: "LLM Evaluation Troubleshooting - Stable Was Not the Same Question as Correct"
date: 2026-09-02
draft: false
tags: ["llm", "evaluation", "llm-as-judge", "prompt-engineering", "pharma"]
categories: ["Backend"]
description: "Calibrating an LLM judge for a regulated-document eval harness took four rounds to stop flickering — and the round where the score finally held still is the round it became confidently, unanimously wrong."
showToc: true
---

I've written before about [the eval harness I built](/posts/20260815-onprem-llm-regulatory-report-eval/) for a fine-tuned model that drafts sections of a pharmaceutical Product Quality Review. Numbers are checked in code — an anchor value either matches the source record or it doesn't, no judgment call involved. But two of the checks aren't numeric: did the draft skip something it was legally required to mention, and does the prose actually hold together. For those I planned to use an LLM as the judge, the way a second reviewer would read a colleague's draft. Before wiring that into anything, the plan was to spend thirty minutes confirming the judge doesn't flip its own verdict on the same text. It took four rounds, and the round where it finally stopped flickering is the round it started being wrong on purpose.

## The setup

Five sample outputs, three of them clean sections pulled from the report I'd already used as the format reference, one deliberately defective — a version of the annual-summary section with the year's single most severe incident (a sterility failure that triggered a batch disposal) quietly removed, while two lesser incidents stayed in. Each sample gets graded three times by the same judge model (Claude Sonnet 5, invoked non-interactively through the CLI with tool access switched off, so it can only grade text and can't go do anything else with the session). If the three scores for one sample disagree by more than a point, that sample is unstable.

## Round 1: holistic 1–5, and one real signal buried in the noise

The first pass asked for a single 1–5 quality score with a rubric paragraph. One of the five samples came back `5, 3, 3` — a two-point spread on the exact same input, well past the stability threshold. Fail on the stated goal.

But it wasn't a wash. Averaged across the three defect-injected runs (2.67) against the three clean runs of the equivalent section (3.67), the judge was consistently harsher on the version with the missing incident. Wrong stability, right direction. That asymmetry is the thread the next three rounds kept losing.

## Round 2: replace the vibe with a checklist, and lose the one good signal

Holistic scores are exactly the kind of thing that's hard to pin down, so the obvious fix is to make the question mechanical: three yes/no checks (does it mention the required incident, does it cite sources with a confidence label, does it use the fixed judgment vocabulary), and let code — not the model — turn the count of "yes" answers into a score.

```python
def bucket_score(yes_votes, total=3):
    """Collapse 3 repeated yes/no calls on one question into a 5/3/1 bucket."""
    if yes_votes == total:
        return 5
    if yes_votes == 0:
        return 1
    return 3
```

Result: worse on both axes. Unstable samples went from 1 to 3 out of 5. And the one useful signal from round 1 was gone — the defective sample and its clean counterpart landed on the identical score sequence, `[3, 5, 5]`. Whatever this checklist was measuring, it wasn't the thing I actually cared about.

Digging into which question was flipping revealed a rubric bug, not a model failure: one of the three checks only makes sense for the section that reports a process-capability index — it asks whether the judgment vocabulary for that specific metric was used correctly. Applied to a section that never mentions that metric at all, the question is unanswerable, and the judge was answering it anyway, consistently, for the wrong reason. A "stable" score on a question that structurally can't apply isn't a good sign, it's a rubric that needs to branch by section.

## Round 3: branch the rubric by section, still no better

Fixed that — each section type now gets the checklist questions that actually apply to it. Stability got worse again: three unstable samples, and one of them swung the full range, `1, 5, 1`. The direction signal was also completely gone now — the defective sample and the clean sample averaged exactly the same score, 3.0 apiece.

Three rounds in, a pattern was clear across all of them: every fix so far had been a prompt change, and none of it touched the actual scoring mechanism. `bucket_score()` maps a count of yes/no answers to `{5, 3, 1}` — flip a single answer on a single run and the score jumps two points. A model that doesn't give byte-identical answers to the same prompt every time (which no model does) will look wildly unstable through a bucket that amplifies every disagreement by 2x.

## Round 4: stop amplifying, and the real problem stops hiding behind noise

Same section-branched rubric as round 3, but the aggregation changed: no bucket. Report the raw agreement — unanimous or split, three-way vote — and a continuous 0–3 score (the plain average of "yes" counted as 1). That's it.

```python
from collections import Counter

def raw_score(votes):
    """votes: list of bool, one per repeated judge call on the same question."""
    tally = Counter(votes)
    unanimous = len(tally) == 1
    return sum(votes) / len(votes), unanimous
```

The noise vanished. Four of five samples hit unanimous agreement across all three questions on all three runs; the fifth split only once, out of nine total votes. By the stated goal — does the score hold still — round four passed cleanly.

And that's exactly where it went wrong. The defective sample — the one missing the year's single most severe incident — scored a unanimous, three-for-three "yes, the required incident is mentioned." Its total (3.0) came out *higher* than the clean sample's (2.0). I went back and reread the defective text by hand to make sure the sample generation hadn't accidentally left the incident in. It hadn't. The incident was genuinely absent. The judge was wrong, and it was wrong the same confident way three times in a row.

What happened: the defective section still mentions two *other* severe incidents, because I'd only removed the one. The judge, reading that one paragraph in isolation with no list of what was supposed to be in it, correctly noticed "a severe incident is discussed here" and had no way to notice that the specific one that mattered wasn't among them. Every rewrite of the question up to this point had been trying to get a stable answer out of a judge that structurally cannot answer it — telling it "check whether anything required is missing" when the only thing it can see is the text that's present, never the list of what should be there.

## Where the line actually is

Stability and correctness turned out to be two different axes, and three rounds of prompt engineering moved the first one without ever touching the second. The fix wasn't a fifth rewrite of the question. It was noticing that "was anything omitted" isn't a judgment call at all — it's a lookup against a reference list, and the harness already had exactly that check running in code, comparing the draft against the set of incidents that legally must be named. Asking the judge to re-derive that same answer from a single paragraph, with none of the source data it would need to actually check, was never going to work regardless of how the question was phrased.

What's left for the judge is narrower than the original plan, and it's also the part that turned out to be genuinely stable in round four once the noise was gone: does the reasoning in the paragraph actually follow from what's stated, and does it stay inside the fixed judgment vocabulary — questions answerable by staring at the one thing you were shown, nothing else required. The general shape: if a grading question is answerable from the fragment alone, an LLM judge can do it and hold still doing it. If answering requires knowing what *wasn't* shown to you, it's a retrieval problem wearing a grading costume, and no rubric rewrite fixes that — only moving the check out of the judge does.

## What I haven't verified yet

Five samples, three runs each, is a calibration pilot, not a statistically solid stability measurement — I'd want at least a few dozen samples before trusting round four's "unanimous" results as a general property of the narrower rubric rather than a lucky small set. I also haven't re-run the must-mention check as a five-question rubric with the omission questions actually removed; round four still asked them; the next run needs a rubric that no longer includes a question the judge can't answer, not just a result that shows why it shouldn't.
