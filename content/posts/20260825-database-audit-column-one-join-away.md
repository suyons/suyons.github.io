---
title: "Database Audits - The Column That Existed One Join Away"
date: 2026-08-25
draft: false
tags: ["database-design", "legacy-systems", "data-audit", "pharma", "sql"]
categories: ["Databases"]
description: "A schema audit that called four report sections impossible turned out to be wrong about two of them — the data existed, just not in the table I checked, and finding it changed what 'this column doesn't exist' is allowed to mean."
showToc: true
---

I [audited a legacy database](/posts/20260822-pqr-optional-feature-adapter/) for a report generator and called four data categories impossible to query: deviation records, CAPAs, change control, and test results with process-capability scores. A reviewer pushed back on two of those with concrete evidence, I reopened the audit, and the reviewer was right both times. Half of what I'd called "doesn't exist" was one join away from existing.

## What "doesn't exist" actually meant the first time

For test results, I'd checked the table that looked like it should hold spec limits — upper bound, lower bound, the numbers a Cpk calculation needs — and found only input-form metadata: field types, whether a value is required, nothing numeric. I concluded the data wasn't there. That conclusion was about the one table I'd opened, not about the system.

The actual spec limits live in a sibling table — the master record for the test method itself, inside a JSON blob, under a key that has nothing to do with the table I'd checked. I hadn't looked at the wrong system. I'd looked at the wrong table within the right system, and stopped there.

## The join that unlocked it

The pattern that fixed it was one I'd already half-documented: the one dataset that *had* worked cleanly on the first pass — out-of-spec results — worked because it flows through a "test request" document that carries a product code and batch number, and the report-facing view joins through that document instead of reading the raw event table directly.

The reviewer's evidence was that the same join path, followed one step further, surfaces two more things I'd written off:

- **Which specific test item failed**, not just that a test batch produced an out-of-spec result. The system stores a pass/fail flag per item, and a background process flips the batch to out-of-spec the moment any item in a completed group comes back failing. The per-item detail was never missing — I'd only queried the summary level.
- **The measured value and its spec bounds together**, once you follow the join from the test result to the test method's master record instead of stopping at the result row. With both present, Cpk is a five-line function, not a research problem.

Two of four "impossible" categories flipped to "queryable" without a single schema change, because the data had been reachable the whole time through a path I hadn't tried.

## The catch that a clean join can hide

The second pass surfaced a subtler bug in my own reasoning before I could ship it: the join field that makes the product/batch link possible is populated by an automated integration, and it's populated *only* on records that came through that integration. Records a person created by hand in the same screen, for the same event type, have the exact same column — sitting empty.

In dev data, that split wasn't rare. Most existing records in the category I re-checked were manually entered test cases, and only a couple had come through the automated path with the link field actually filled in. A query that treats "the column exists" as "the column is populated" will silently undercount every hand-entered record and never throw an error while doing it — the join just returns fewer rows than there are real records, and nothing about that looks like failure.

That's a different bug class than the one I started the audit chasing. "This field doesn't exist" fails loud — every query against it errors. "This field exists but is null for a path I didn't think to check" fails quiet, and it only shows up if you go looking at whether the row count matches what you expected, not just whether the query ran.

## What didn't flip

Deviations, CAPAs, and change control stayed exactly as blocked as the first pass found them. They're a different table shape entirely — no test-request document to join through, no product code anywhere in their lineage, because the workflow that creates them was never designed to require one. The join trick that rescued test results doesn't generalize to them; there's no join to make, because the field the join would need was never captured at the point of entry. That gap still needs a form change, not a smarter query, and no amount of re-auditing the schema was going to find data that was never written down.

## What actually changes in the design

The [adapter interface from the earlier post](/posts/20260822-pqr-optional-feature-adapter/) doesn't change at all — it was already built to not care where a section's data comes from. What changes is which adapter handles which section: two report sections that were going to be served from generated stand-in files move to the database adapter instead, and the file adapter's job shrinks to exactly the categories that are still genuinely unwritten anywhere — not "everything I couldn't confirm on the first pass."

The lesson I'm taking forward isn't "look harder." It's that a negative finding from a schema audit is a claim about the tables you queried, not about the system, until you've also checked whether an existing feature already solved the same lookup and just never told you which path it used to do it.
