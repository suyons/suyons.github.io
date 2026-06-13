---
title: "MSSQL Used `+` to Join Strings. MySQL Read It as Math."
date: 2026-06-09
draft: false
tags: ["mysql", "mssql", "sql", "database-migration", "json", "views"]
categories: ["DevOps"]
description: "Migrating a backend from SQL Server to MySQL, the schema and the application code were the easy part. The bugs that survived the migration were hiding inside the database views — where MSSQL's `+` string concatenation silently became MySQL arithmetic, and a column holding an empty string blew up JSON_TABLE."
showToc: true
---

## The migration that looked done

I moved a document-management backend off SQL Server onto MySQL 8.0. The schema was already there — 323 tables, dumped and loaded, `lower_case_table_names=1`, all green. The application code went through a dialect-abstraction pass so a single `.env` switch picks the driver. Module load passed. Direct-to-DB smoke tests passed. The app booted and listened.

Then a real HTTP request hit the dashboard and the server returned `501`. The log said:

```
Incorrect arguments to JSON_TABLE
```

A different endpoint returned garbage numbers, with this in the MySQL log:

```
Truncated incorrect DOUBLE value: ' > '
```

Neither bug was in the schema and neither was in the application code. Both were inside the **views** — the part of the migration that no tool touched, that doesn't live in your git repo, and that runs SQL written for a dialect you no longer use.

That's the thing I want to pull apart. When you migrate databases, you instinctively audit the tables and the queries your code runs. The views are SQL too, they were written against the old engine's semantics, and they fail at runtime in ways that look nothing like a migration problem.

## `+` is the trap, because it doesn't error — it lies

In SQL Server, `+` is overloaded. Given two strings, it concatenates them:

```sql
-- MSSQL: this returns the string '5 > 10'
SELECT CAST(a AS varchar) + ' > ' + CAST(b AS varchar)
```

A lot of view code leans on this. Breadcrumb paths, composite labels, revision strings — all built by gluing fragments together with `+`.

MySQL does not overload `+`. It is the arithmetic addition operator, full stop. Hand it strings and it doesn't refuse — it *coerces them to numbers* and adds them. `' > '` has no leading digits, so it converts to `0`, and MySQL emits a warning, not an error:

```sql
-- MySQL: this returns 0, with "Truncated incorrect DOUBLE value: ' > '"
SELECT 'foo' + ' > ' + 'bar';
```

A warning. The query succeeds. The view returns rows. Your app gets a column full of `0`s or partial sums where it expected text, and nothing in the stack raised its hand. This is strictly worse than an outright error, because an error stops you at deploy time and a silent coercion ships to production and waits.

The most insidious case wasn't even the obviously-broken `' > '` paths. It was a revision-number column built like `'0' + revision` to zero-pad single digits. In MSSQL that produced `"05"`. In MySQL, `'0' + '5'` is `5` — a number, padding gone. The column still had a plausible value. It was just quietly wrong, `"5"` where every downstream consumer expected `"05"`, and you only notice when something does a string compare or a sort and the ordering goes sideways.

The fix is to stop relying on operator overloading and say what you mean:

```sql
-- portable across MySQL/MSSQL/Postgres: CONCAT is concatenation, always
SELECT CONCAT(CAST(a AS CHAR), ' > ', CAST(b AS CHAR));
```

`CONCAT` is unambiguous in every dialect that matters here. There's no version of it that means "add." If you're porting views, every `+` that touches a string literal or a character column is a bug, and the ones joining a literal like `' > '` or `'/'` are the easy ones to grep for.

The part that made this tedious rather than trivial: I couldn't blindly replace `+` with `CONCAT`, because plenty of `+` in those views is honest arithmetic — `tree_level + 1` in a recursive walk, numeric offsets, real math. So the rewrite had to be a balanced-paren transform that converts only `+` chains containing at least one string literal and leaves numeric expressions alone, then `CREATE OR REPLACE VIEW`, then a `COUNT(*)` to confirm the view still resolves, with an automatic restore of the prior definition on any failure. Nine views had string-concat `+`; after the pass, zero did.

## The second trap: `JSON_TABLE` on a column that isn't JSON

The `501` was a cousin of the same problem — old SQL meeting a stricter engine. The views used SQL Server's `OPENJSON` to expand a JSON column into rows; the MySQL equivalent is `JSON_TABLE`. The mechanical translation is clean enough:

```sql
JSON_TABLE(col, '$[*]' COLUMNS (...))
```

It works right up until `col` contains an empty string `''` instead of `NULL` or valid JSON. SQL Server's `OPENJSON` tolerates the empty/legacy value. MySQL's `JSON_TABLE` requires the first argument to be valid JSON, and `''` is not valid JSON, so the whole query dies:

```
Incorrect arguments to JSON_TABLE
```

One bad row in the source data takes out the entire view, and therefore the entire endpoint. The guard is to normalize the not-JSON sentinel to `NULL`, which `JSON_TABLE` accepts (it just yields zero rows for that row):

```sql
JSON_TABLE(NULLIF(col, ''), '$[*]' COLUMNS (...))
```

`NULLIF(col, '')` returns `NULL` when the column is an empty string and the value otherwise. `JSON_TABLE` on `NULL` produces no rows instead of throwing, so a row with no payload contributes nothing instead of detonating the query. I'd already added exactly this guard in the application-code path that does the same expansion; the lesson is that the views needed the identical guard and nobody had thought to apply it there. There were 41 views doing JSON expansion. All 41 needed the `NULLIF`.

## Why the tests didn't catch either one

Here's the detail worth internalizing. I had two layers of tests passing before these bugs surfaced: the modules loaded, and a direct-DB harness ran the dialect helper functions and even executed recursive CTEs and `JSON_TABLE` calls against the live database. Green across the board.

Both view bugs only fired on an actual HTTP request through the real endpoint. The reason is that the views are exercised by the route handlers, not by the unit-level checks — the smoke tests hit the building blocks and the function outputs, not the composed query path that a controller runs when a request comes in. A view is lazy: it's just a stored `SELECT`, and its SQL is not evaluated until something selects from it with real data flowing through. So a `JSON_TABLE` that throws on one empty-string row, or a `+` that coerces text to `0`, is invisible to everything except a query that actually reads those rows.

If you take one operational thing from this: after a database migration, the test that matters is an end-to-end request through every endpoint that reads a view, against data that includes the ugly legacy rows — the empty strings, the nulls, the values someone wrote before the column had a constraint. Clean fixture data hides exactly the bugs a migration introduces.

## These fixes don't live in your repo

A closing caveat that bit me and will bite you. The application-code changes — the dialect abstraction, the operator rewrites in code — are committed and travel with the deploy. The **view fixes are DDL that lives inside the database**. `CREATE OR REPLACE VIEW` changes the dev database and nothing else. There is no migration file, no commit, no diff that carries it to test or production.

So every fix described here has to be re-applied to each environment's database as its own step, and if you stand up a fresh environment from the old schema dump, all of these bugs come back, because the dump contains the original view definitions. Treat the corrected views as migration artifacts you script and version deliberately, or you will rediscover `Truncated incorrect DOUBLE value: ' > '` in production three weeks after you thought you'd fixed it.

## The thirty-second version for a review

If you're porting SQL Server views to MySQL: every `+` that touches a string is a silent bug — MySQL reads it as arithmetic, coerces your text to `0`, and only warns. Replace it with `CONCAT`, but only on the chains that actually involve strings; leave the real math alone. Any `OPENJSON`/`JSON_TABLE` over a column that can hold an empty string needs `NULLIF(col, '')`, or one legacy row kills the whole view. And test it by sending a real request through the endpoint against real, messy data — your unit tests and your `COUNT(*)` checks will pass while the composed query path is broken. Finally, remember the fixes are DDL: they don't ride along in your git history, so script them per environment or they won't survive the next fresh deploy.
