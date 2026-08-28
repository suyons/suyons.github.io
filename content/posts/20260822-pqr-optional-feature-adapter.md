---
title: "Legacy Integration - Design It So Deleting It Undoes Everything"
date: 2026-08-22
draft: false
tags: ["legacy-systems", "architecture", "adapter-pattern", "database-design", "pharma"]
categories: ["Backend"]
description: "Before any model or retrieval work could start on a pharma report generator, a database audit found that half the data it needs doesn't exist in queryable form — so the feature had to be built so it could be deleted, not just written."
showToc: true
---

Before I could write a line of the eval harness [I wrote about earlier](/posts/20260815-onprem-llm-regulatory-report-eval/) or the knowledge base [I wrote about after that](/posts/20260816-rag-knowledge-base-provenance-tagging/), I had to answer a duller question first: does the source data this report needs actually exist in a form a program can query? The proposal I was handed said yes, unconditionally — "the data's already in the system, just query it and fill the table." I opened the target system's production database and checked every item on the report against it, row by row. The answer was closer to half.

## What "already in the system" actually meant

The report is a Product Quality Review: twelve sections, most of them tables a program fills from existing records, a handful of narrative paragraphs an LLM writes from those same numbers. I went through the report spec line by line against the schema. Some of it held up — product master data, stability testing status, manufacturing lot counts. Some of it was half-true — lot counts existed, but yield and output volume didn't. And the four sections that matter most for a quality review — deviations, CAPAs, change control, and test results with process-capability scores — didn't exist as queryable data at all.

## The gap that actually blocks the report

Deviation records get logged when something goes wrong in manufacturing, and the system does store them — but what it stores per record is a title, a due date, and an approval chain. The grade, the product, the batch, the root cause, the linked CAPA: all of that lives in an attached document a person opens and reads. A program can't count what's inside a Word file.

The blocker underneath that: deviation records don't carry a product code at all. A quality review is inherently per-product — "how many deviations did this product have this year" — and there was no column, anywhere in the schema, that let me ask that question. CAPA and change-control records share the same table shape, so they're blocked the same way.

Test results had a different flavor of the same problem. The column meant to hold a measured value literally contained the string `"undefined"`; the real numbers sat one level down, inside a JSON blob, under form-specific labels that differ by which test sheet produced them. Spec limits weren't stored as numbers anywhere — the closest table held input-form metadata ("this field accepts free text"), not upper and lower bounds. Nothing about that JSON blob was hiding a bug; it's just not a shape a Cpk calculation can read.

## One exception, and it told me how the system actually works

One dataset — out-of-spec results — was already exposed cleanly, every field the report needed available as a plain column through an existing API. It turned out the system's own convention was exactly this: store the raw record as an unstructured JSON blob, same as everything else, but expose the handful of fields a report actually needs through a dedicated read-only view. Out-of-spec got that treatment because it flows through a test-request document that happens to carry a product code and batch number. Deviations never pass through that document, so they never got the same view — not a missing feature, just a gap nobody had reason to close yet.

That one working case became the template for the fix, instead of inventing a new pattern.

## Building a feature that doesn't need permission to exist

The report is optional — customers who don't use it shouldn't notice it's there, and I couldn't get write access to the production schema on any timeline that fit the project anyway. So the design constraint became: zero writes to the existing database, and every data source swappable behind one interface.

```js
// The report renderer calls this and never knows where the rows came from.
async function getDeviations(itemCode, year) {
  for (const adapter of adapters) {          // db → file → view, in that order
    const rows = await adapter.getDeviations(itemCode, year);
    if (rows) return rows;
  }
  return [];                                  // renders as "source data unavailable" — report still finishes
}
```

Today, the DB adapter covers what's already queryable (out-of-spec, stability, lot counts, product master); a file adapter reads generated JSON standing in for the four sections that aren't queryable yet; a future view adapter — modeled on the out-of-spec view — is the intended replacement once someone builds the equivalent for deviations. If none of them has data, the section renders as explicitly missing instead of failing the whole report — which turns out to already be the report format's own convention: one section note reads "if EBR integration isn't present, leave this field blank." I just wrote code that follows the rule the document already stated.

The payoff isn't elegance, it's what happens when something goes wrong: rollback is deleting a JSON file, not writing a migration. There's no "prove this had zero impact on production" conversation, because nothing touched production. And the report-assembly code doesn't change at all when a real view eventually replaces the file adapter — only the adapter does.

One near-miss worth flagging: I found a JSON column on the deviation table that looked unused and briefly considered writing into it. It wasn't unused — a different screen reads that exact column and expects a specific shape, so anything I wrote there would have broken that screen's query, not mine. A column that looks empty because nobody in the report's code path touches it isn't the same as a column nobody touches.

## What this doesn't fix

Deviations still don't carry a product code, and no adapter changes that. Every workaround here routes around the gap with synthetic data standing in for what should be a queryable field; it does not create one. The actual fix is a form field — capture product code and batch number at the point a deviation gets logged, the same way the out-of-spec form already does — and that's a change to how people enter data, not a change to code, which means it needs sign-off I don't have yet. Until it lands, the per-product deviation section of this report is only as complete as the synthetic data covers, and that's a limitation worth stating in the report's own methodology note, not hiding behind a clean-looking table.

The Cpk piece turned out to be the easy part once the numbers exist as numbers — the formula everyone in the industry actually uses:

```js
// values: measured results; lower/upper: spec limits. A missing bound means no constraint on that side.
function cpk(values, lower, upper) {
  const n = values.length;
  if (n < 2) return null;                 // too few points to trust a sigma estimate
  const mean = values.reduce((a, b) => a + b, 0) / n;
  const sd = Math.sqrt(values.reduce((a, b) => a + (b - mean) ** 2, 0) / (n - 1));
  if (sd === 0) return null;               // zero variance — nothing meaningful to score
  const upperIndex = upper == null ? Infinity : (upper - mean) / (3 * sd);
  const lowerIndex = lower == null ? Infinity : (mean - lower) / (3 * sd);
  const index = Math.min(upperIndex, lowerIndex);
  return Number.isFinite(index) ? Number(index.toFixed(2)) : null;
}
```

That's the same 1.33 threshold I wrote about [not being in the regulation text at all](/posts/20260816-rag-knowledge-base-provenance-tagging/) — the function computes the score, but whether 1.33 is the right bar to judge it against is still a knowledge-base question, not a code question. The one test worth keeping here is a plain regression check: feed it a fixed mean, a spec range, and a known standard deviation, assert on the exact number it returns, so a future refactor of the formula gets caught by a failing test instead of a silently wrong score in a shipped report.
