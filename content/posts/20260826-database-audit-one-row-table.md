---
title: "Database Audits - The View That Only Ever Had One Row"
date: 2026-08-26
draft: false
tags: ["database-design", "legacy-systems", "data-audit", "pharma", "sql"]
categories: ["Databases"]
description: "A schema audit confirmed every source table for a pharma report was reachable, then a row-count check found the table the whole design leaned on had exactly one row in it."
showToc: true
---

Two audits into this report generator, I'd confirmed which tables exist, which columns hold real values, and which joins reach data I'd first written off as missing. What I hadn't done, before greenlighting the next stage, was run `SELECT COUNT(*)` against any of it. [The schema audit](/posts/20260822-pqr-optional-feature-adapter/) answered "can I query this." [The follow-up](/posts/20260825-database-audit-column-one-join-away/) answered "is this column actually populated." Neither one answers "is there enough of it to write a report section from." That third question turned out to have a much worse answer than either of the first two.

## Reachable is not the same claim as populated, and populated is not the same claim as enough

The report needs fifteen product-years of history for its out-of-spec review sections — how many out-of-spec results a product had, what caused them, whether the pattern is getting better or worse. The very first audit had already flagged one out-of-spec detail table as the good case: every field the report needs, exposed cleanly through an existing view, because out-of-spec results flow through a request document that carries a product code and batch number. That view became the template the rest of the design copied.

Running an actual query against it in the dev database returned one row.

Not "sparse." Not "missing recent months." One row, for the one dataset the entire out-of-spec section was designed around. Everything downstream of that table — the review narrative, the trend line, the anchor counts a grader checks the generated text against — has nothing to read from.

## What checking the neighboring table confirmed

The same audit pass also queried the shared ticket table that tracks deviations, CAPAs, change controls, and out-of-spec cases by category, since that's the second place an out-of-spec record could plausibly show up. Nineteen tickets were tagged as out-of-spec-related. Only two of those carried the product/batch link that would make them queryable per-product — the same link-only-on-automated-entries pattern the previous audit had already found on deviation records. Seventeen tickets with no way to attribute them to a product.

Two independent tables, two different query paths, the same conclusion from both: whatever out-of-spec activity happened in this environment, it didn't happen through the path this report needs to read from.

Contrast that with the two categories the second audit had rescued: the measured-test-value table backing them held 4,749 rows, all with real recorded values, against 1,186 defined test specs with actual bound payloads for all but one. That's a populated table by any reasonable bar. The out-of-spec table sitting at a single row isn't a data-quality issue on the same axis — it's a different failure mode entirely, and the query being syntactically valid doesn't tell you which one you're looking at.

## Why this had to be checked before the next stage, not during it

The scenarios, the anchor numbers a grader checks generated report text against, and the training pairs for the generation step all get written against whichever bucket a data source falls into: queried from the database, or synthesized as a stand-in. Write fifteen out-of-spec scenarios assuming real historical data backs them, discover the row count afterward, and every one of those scenarios needs rewriting along with everything downstream of them. Running the count query first costs a few minutes. Running it after the scenarios are written costs the scenarios.

| Category | Rows checked | Verdict |
|---|---|---|
| Test specifications | 1,186 total, 1,185 with real bound values | populated — query it |
| Measured test values | 4,749 rows, 100% with real recorded values | populated — query it |
| Out-of-spec detail | 1 row | not populated — synthesize it |
| Out-of-spec-tagged tickets | 19 rows, 2 with product/batch link | not usable per-product — synthesize it |
| Deviation-tagged tickets | 7 rows, 0 with product/batch link | already known — synthesize it |
| Change-control-tagged tickets | 12 rows, 0 with product/batch link | already known — synthesize it |
| Supplier-tagged tickets | 17 rows, 8 with product/batch link | best-linked non-test category, still partial |
| Manufacturing yield specs | 40 rows, no recorded yield values in any | not populated — synthesize it |

## What actually changed

Out-of-spec moves from "query it" to the generated/manual-form bucket, alongside deviations, CAPAs, and change control — the same bucket [the adapter design from the first audit](/posts/20260822-pqr-optional-feature-adapter/) already built for exactly this situation. Nothing about the adapter interface changes; only which adapter gets tried first for that section does. That's the actual payoff of designing the swap to be a config change instead of a rewrite: the design decision from two posts ago absorbed a finding that inverted the assumption it was originally built around, and it absorbed it for free.

The dev database isn't necessarily representative of what a real customer's production instance will hold — this environment may simply predate the out-of-spec workflow being used in earnest, and a live deployment could look completely different. That's worth stating plainly rather than treating a one-row dev table as proof the feature is unused everywhere. But a report generator has to work against the data that's actually there on day one, and "the query path exists" was never a substitute for checking what's behind it.

One more thing this pass surfaced, worth a flag on its own: there's no dedicated read-only database account for any of this. The connection this audit used has full write privilege at the account level — "read this data, never write it" is a rule the query code has to follow on its own, not something the database enforces. That's a gap between what the design assumes and what the infrastructure actually guarantees, and it's the kind of gap that a schema audit won't catch because it isn't a schema question.
