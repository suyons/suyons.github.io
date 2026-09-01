---
title: "Process Capability Troubleshooting - The Zero-Variance Guard That Real Data Always Defeats"
date: 2026-09-01
draft: false
tags: ["javascript", "floating-point", "statistics", "data-quality", "pharma"]
categories: ["Backend"]
description: "Two silent bugs in a Cpk calculation only showed up once real database values replaced hand-picked test fixtures: a blank field parsed as a measurement of zero, and a floating-point residual that made the zero-variance guard never fire."
showToc: true
---

I [published a `cpk()` function](/posts/20260822-pqr-optional-feature-adapter/) a couple weeks ago and it passed every test I wrote for it. Then I ran it against actual measured values pulled from the dev database instead of the small hand-picked arrays in my test file, and it produced numbers that made no sense — a process with four samples that were basically identical scored as wildly, impossibly capable. Tracing that back turned up two separate bugs, both silent, both the kind that puts a confident wrong number in a document nobody double-checks.

## Bug one: a blank field is not a measurement of zero

The measured values sit in a JSON blob per test result, one field per test sheet, and the parsing code looked like this:

```js
// Before
function parseValue(raw) {
  return Number(raw);      // Number('') is 0, not "no value"
}
```

`Number('')` is `0`. Not `NaN`, not an error — `0`. Every test sheet has fields nobody filled in, and every one of those came out of `parseValue` as a real, valid-looking measurement of zero. For a spec like "survival rate, 80.7 or higher," a single silent zero is enough to make the whole batch look like it's failing when the field was just never populated.

The fix only needed to special-case the blank string; `Number()` already handles trailing whitespace and scientific notation correctly, so the numbers that are real numbers didn't need touching:

```js
// After
function parseValue(raw) {
  if (String(raw).trim() === '') return null;   // unfilled field, not a measurement of zero
  return Number(raw);
}
```

The distinction matters because a *real* zero is a valid, important data point in this dataset — a test that failed outright is recorded as `0` on purpose, and that's exactly the kind of result a process-capability score is supposed to catch. The bug wasn't "zero is scary," it was "an absent value and a real zero were indistinguishable after parsing." Fixing it means going back to the raw string before any numeric coercion happens, because once you've called `Number()` on `''`, the information that it was ever blank is gone.

## Bug two: the guard against zero variance never fires

The second bug lives in the function I already showed once:

```js
// Published two weeks ago
function cpk(values, lower, upper) {
  const n = values.length;
  if (n < 2) return null;
  const mean = values.reduce((a, b) => a + b, 0) / n;
  const sd = Math.sqrt(values.reduce((a, b) => a + (b - mean) ** 2, 0) / (n - 1));
  if (sd === 0) return null;               // zero variance — nothing meaningful to score
  const upperIndex = upper == null ? Infinity : (upper - mean) / (3 * sd);
  const lowerIndex = lower == null ? Infinity : (mean - lower) / (3 * sd);
  const index = Math.min(upperIndex, lowerIndex);
  return Number.isFinite(index) ? Number(index.toFixed(2)) : null;
}
```

`if (sd === 0)` looks like exactly the right guard for "there's no spread in this data, don't pretend to score it." It just never fires. Feed it `[0.09, 0.09, 0.09]` — three identical readings — and the computed standard deviation isn't `0`. It's about `1.2e-17`. Mathematically the values are identical; in floating point, the mean isn't exactly representable, so subtracting it back out leaves a residue instead of a clean zero.

That residue then gets used as a divisor. `(upper - mean) / (3 * sd)` with `sd` at `1.2e-17` doesn't throw, doesn't return `Infinity`, doesn't fail any obvious check — it returns something like `7.8e14`. That's comfortably above the 1.33 threshold this pipeline uses for "sufficient," so a process with zero real variance gets reported as not just capable but essentially perfect. A `null` would have been an honest answer. This was a wrong one dressed up as a confident one, which is worse.

The fix drops the derived statistic entirely and compares the raw inputs instead:

```js
// After — compare the raw values, not a statistic computed from them
function cpk(values, lower, upper) {
  const n = values.length;
  if (n < 2) return null;
  if (Math.max(...values) === Math.min(...values)) return null;  // exact input comparison, immune to fp residue
  const mean = values.reduce((a, b) => a + b, 0) / n;
  const sd = Math.sqrt(values.reduce((a, b) => a + (b - mean) ** 2, 0) / (n - 1));
  const upperIndex = upper == null ? Infinity : (upper - mean) / (3 * sd);
  const lowerIndex = lower == null ? Infinity : (mean - lower) / (3 * sd);
  const index = Math.min(upperIndex, lowerIndex);
  return Number.isFinite(index) ? Number(index.toFixed(2)) : null;
}
```

`Math.max(...values) === Math.min(...values)` never touches a mean or a sum of squares. It asks whether the inputs themselves are identical, which is the question the guard was always supposed to answer. Any derived quantity — mean, variance, standard deviation — is fair game for floating-point residue the moment it comes from a division; the raw values you started with aren't, because there's no arithmetic between them and the comparison.

## The zero that was never the bug

One real result from the gate is worth keeping next to this, because it's the case that would make the fix look wrong if I hadn't checked it: a survival-rate spec (≥80.7) with four measured values — `94.8, 94.8, 0, 94.8` — scored `cpk = -0.07`, correctly judged not capable. That `0` isn't a parsing artifact. It's a test that was recorded with a failing status and a nonconforming judgement in the source data. A negative Cpk is the right answer there, and neither fix touches it — the blank-field fix only intercepts values that were empty strings before parsing, and the variance guard only fires when every value in the group is identical, which four values including a real outlier are not.

That's the actual discipline both bugs point at: don't let a single check stand in for "is this data trustworthy." A zero can be absence or it can be a real observation, and a computed statistic can be zero-in-spirit without ever being zero-in-bits. Neither distinction survives contact with a fixture array you typed by hand — both of these only surfaced once real values, with real gaps and real repeated readings, went through the same function I'd already called done.

## One more thing the same pass caught

A smaller finding from the same gate, worth a paragraph rather than its own section: two of the test specs in this dataset share an identical name but have different bound sets — one process's spec runs roughly -0.08 to 0.13, another using the same label runs 0.53 to 1.18. Grouping measured values by spec *name* to compute a Cpk series silently merges those two populations into one. The grouping key has to be the name plus the bounds plus the comparison type, not the name alone — otherwise the series a Cpk trend line is drawn from is measuring two different things and calling them one.

## What I didn't get to verify

Both fixes are covered by a self-check test file, and the full suite passes. What I haven't done is re-run the *entire* historical dataset through the corrected function and diff every score against what the buggy version produced — I fixed the two cases I could isolate and confirmed the specific example above by hand, not the whole set.
