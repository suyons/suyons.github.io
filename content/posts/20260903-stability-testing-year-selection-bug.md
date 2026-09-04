---
title: "Database Troubleshooting - The Twelve-Month Test That Never Landed In a Review Year"
date: 2026-09-03
draft: false
tags: ["sql", "database-design", "data-modeling", "time-series", "pharma"]
categories: ["Databases"]
description: "A stability-testing query selected records by the calendar year a test plan started. That made the one checkpoint most of the report's numbers depend on structurally impossible to select — every plan's twelve-month result belongs to a different year than the plan itself."
showToc: true
---

I've been generating realistic synthetic data for [the pharma report generator I've written about before](/posts/20260822-pqr-optional-feature-adapter/), and one section needs a real stability-testing decline: a product's measured content at time zero, checked again at three, six, nine, and twelve months, so the report can state how much it dropped over a year. Five of the six scenarios this section covers anchor their numbers to a twelve-month decline figure for the evaluation year. The query meant to source that figure couldn't return it. Not "returned it wrong" — structurally could not return it, for any plan, ever.

## What the query actually selected

The query that picks which stability plans belong to a given year's review filtered on the plan's own start date:

```sql
-- Before: select plans that started in the evaluation year
SELECT * FROM stability_plans
WHERE EXTRACT(YEAR FROM start_date) = :evaluation_year;
```

That reads as reasonable — "which stability programs began this year" sounds like the right question for an annual review. It isn't the question a decline figure needs answered. A twelve-month checkpoint, by definition, happens twelve months after the plan starts. A plan that started in 2025 has its twelve-month test in 2026. A plan that started in 2024 has its twelve-month test in 2025 — but this query, run for the 2025 review, never selects it, because its start date says 2024.

Push that through: there is no calendar year in which a plan's own twelve-month timepoint falls inside the year selected by its own start date. Not "rare," not "usually" — the twelve-month checkpoint of a plan selected this way lands in next year, always, by construction. Nine-month, six-month, three-month timepoints can land in the start year depending on when in the year the plan began. Twelve-month structurally cannot, short of a plan that starts on January 1st and is measured on December 31st of the same year to the day.

## Why it took a while to notice

The query wasn't throwing errors and wasn't returning zero rows — it was returning real plans, with real zero-month and real three/six/nine-month measurements, just never the twelve-month one for the year that mattered. A row count check would have looked fine. What actually surfaced it was working backward from the anchor: five of six scenarios needed "twelve-month decline in the evaluation year" as a concrete number, and no query against the selected data could produce one, because the row that number depends on was never in the selected set to begin with.

That's the same trap as any bug where the query runs clean and the absence looks like sparse data rather than a wrong selection — the fix isn't in the data, it's in which records "belong to this year" in the first place.

## The fix: select by the event, not by the parent's start date

The right question isn't "which plans started this year," it's "which plans have a checkpoint that was actually tested this year." A stability plan already joins to its own timepoint records — the join needed for the fix already existed, it just wasn't the join the selection was using:

```sql
-- After: select plans with any checkpoint tested in the evaluation year
SELECT DISTINCT p.* FROM stability_plans p
JOIN stability_timepoints t ON t.plan_id = p.id
WHERE EXTRACT(YEAR FROM t.test_date) BETWEEN :period_start AND :period_end;
```

That alone isn't sufficient, though. Once a plan is selected because *some* checkpoint of it falls in the target year, the decline calculation still needs the zero-month value to subtract from — and the zero-month test for that plan may well have happened the previous year. So the query doesn't stop at "select the matching timepoint row," it selects the matching *plan* and then returns that plan's full timepoint history, zero through twelve months, regardless of which year each individual checkpoint happened to land in. The selection is per-plan; the payload is per-plan-in-full. Filtering the payload down to only the rows dated within the evaluation year would just recreate the same bug one layer down.

One design decision made this safe to rely on without an extra date guard elsewhere: no synthetic test data gets generated past the report's fixed baseline date. There's no "future" stability test in this dataset, so the query doesn't need a second filter to exclude checkpoints that haven't happened yet relative to the baseline — a constraint enforced once, at generation time, instead of re-checked in every query that reads the result.

## Locking it in

A plan that opens late in one year and completes its year-long checkpoint early in the next is exactly the case a hand-picked test fixture tends to skip, because it's easy to only test plans that start and finish inside one tidy year. The regression test for this fix does the opposite on purpose: a plan with a start date deliberately placed near a year boundary, asserting that its twelve-month checkpoint gets selected into the review for the year it was *tested*, not the year the plan began.

## What generalizes past this one query

The underlying mistake is a common one and not specific to stability testing: a parent record spans time, one of its child events is the one you actually need, and it's tempting to select the parent by a date column sitting right there on the parent row because it's the obvious, no-join filter. That column answers "when did this begin," which is a different question from "did the event I need happen in my target window" the moment the process being modeled takes longer than the bucket you're filtering by. Any time a report bucket is "this year" and the underlying process is "this many months," the parent's own date column is the wrong filter almost by definition — the event you care about is offset from it by exactly the process length, and a year is rarely a clean multiple of that.

## What I haven't re-verified

The fix and its regression test cover the twelve-month case, which is the one that was structurally guaranteed to fail. I haven't gone back and audited whether the three, six, and nine-month timepoints were ever silently excluded too under specific plan start dates — the math says they're only excluded sometimes, depending on where in the year a plan started, not always, but "sometimes" is exactly the kind of gap a single passing test won't catch by itself.
