---
title: "Database Migration - The COUNT(*) That Had to Match Before a Blind Replace Could Run"
date: 2026-10-27
draft: false
tags: ["mysql", "sql", "database-migration", "json", "production-safety"]
categories: ["Databases"]
description: "Fixing one row's height inside a JSON-shaped blob stored as TEXT meant a global string replace across production tables — safe only if a COUNT(*) could first prove every matching record contained exactly one occurrence."
showToc: true
---

A document management system stores each cover-page layout as a blob of JSON-shaped text in a `TEXT` column — an array of page elements, one of them a table row with a `"size"` field for its height. One row needed its height changed from 49 to 28. The column isn't a native `JSON` type and the array isn't addressable by a stable path (different templates don't necessarily lay their rows out in the same order), so the only practical fix was a plain string replace: find `{"size": 49},`, write `{"size": 28},`. Across two tables, in production, in one pass.

`REPLACE()` doesn't replace "the first match" or "the one at a known position" — it replaces every occurrence in the string. If even one row's layout happened to contain the height-49 fragment twice, a blind replace would silently flip both, not the one row that needed it. The task was proving that couldn't happen before running anything, not after.

## Write the rollback before the forward query

The script's first move after snapshotting the table isn't the fix — it's the undo, written and ready before anything risky runs:

```sql
create table app_docs_test.doc_workflow_master_backup_20261027
as select * from app_docs_test.doc_workflow_master;

-- find rows that would need restoring, by comparing live vs. backup
select t.process_code, t.cov_template as live, b.cov_template as backup
from app_docs_test.doc_workflow_master t
left join app_docs_test.doc_workflow_master_backup_20261027 b
  on t.process_code = b.process_code
where not (t.cov_template <=> b.cov_template);

-- the restore itself, ready to run unmodified if anything looks wrong
update app_docs_test.doc_workflow_master t
left join app_docs_test.doc_workflow_master_backup_20261027 b
  on t.process_code = b.process_code
set t.cov_template = b.cov_template
where not (t.cov_template <=> b.cov_template);
```

That `<=>` is MySQL's NULL-safe equality operator, not a typo for `<>`'s negation. Plain `<>` returns `NULL` — not `TRUE` — whenever either side is `NULL`, so a diff check built on `where t.cov_template <> b.cov_template` would silently drop exactly the rows where a template went missing or got nulled out, the cases you'd most want the diff to catch. `<=>` treats `NULL <=> NULL` as true and compares everything else normally, so `not (... <=> ...)` catches every real difference, including the ones involving `NULL`.

Having the rollback sitting right there, already written, already using the exact same join key as the forward change, means reverting under pressure is "run this block" instead of "improvise a query while something's on fire."

## Proving the substring is unique before trusting a blind replace

The actual safety gate is a count. MySQL has no built-in "count occurrences of a substring," but you can derive it: strip every occurrence and compare the resulting length to the original.

```sql
with target as (
  select
    process_code,
    (length(cov_template) - length(replace(cov_template, '{"size": 49},', '')))
      / length('{"size": 49},') as hit_count
  from app_docs_test.doc_workflow_master
  having hit_count = 1
)
select count(*) from target;
```

`replace()` removes all occurrences of the substring; the difference in length, divided by the substring's own length, is the occurrence count per row. That's the standard trick. The less obvious part is the `having hit_count = 1` with no `group by` above it. A `where` clause can't reference `hit_count` — the alias doesn't exist yet when `where` is evaluated, since the select list hasn't been computed. `having` runs after the select list, so in MySQL you can use it as a row filter on a computed alias even without aggregation. It reads like it should need a `group by` and doesn't.

That query's result then has to equal a second, independent count — every record that contains the substring at all, regardless of how many times:

```sql
select count(*)
from app_docs_test.doc_workflow_master
where cov_template like '%{"size": 49},%';
```

If the two counts match, every row containing the target substring contains it exactly once, and a blind `replace()` is safe. If they don't, some row has two or more hits, and running the replace would be silently wrong on at least one record — the script stops there, full stop, no forward update until that's resolved record by record.

## Running it

With both counts equal, the forward change ran inside an explicit transaction (autocommit off), on both tables:

```sql
update app_docs_test.doc_workflow_master
set cov_template = replace(cov_template, '{"size": 49},', '{"size": 28},')
where cov_template like '%{"size": 49},%';
```

Before any of this touched production, the layout change itself was verified visually: render the cover PDF in a dev environment with height 49, confirm it's the problem, flip it to 28, render again, confirm the fix. The SQL pattern only answers "is this safe to run," not "is this the right value" — that's a separate check, and skipping it would mean efficiently automating the wrong change.

Once the row counts and the rendered output both checked out, the backup table was the last thing to go — not before.

## Takeaways

- A `TEXT` column holding JSON-shaped data with no reliable path addressing leaves you with string matching as the only tool. That's fine, but it inherits every risk of string matching: it can't tell "the occurrence I mean" from "an occurrence that looks the same."
- Turn "is this substring unique per row" into a number you can compare, twice, independently, before trusting a global replace. `(length(s) - length(replace(s, needle, ''))) / length(needle)` is the whole trick.
- `having` can filter on a select-list alias without `group by`; `where` can't reference that alias at all in the same query.
- Use `<=>` over `<>` for any diff check that has to be trustworthy in the presence of `NULL` — `<>` quietly excludes exactly the rows most worth catching.
- Write the rollback query immediately after the backup, before the risky statement, using the same join key you'll need under pressure. The time to write an undo is before you need one.
