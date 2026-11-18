---
title: "MySQL Troubleshooting - The 128 MB Buffer Pool Behind a Seven-Second SELECT on 1,800 Rows"
date: 2026-11-14
draft: false
tags: ["mysql", "innodb", "performance", "slow-query-log", "production-debugging"]
categories: ["Databases"]
description: "A plain SELECT * on a table with fewer than 2,000 rows was taking 7 seconds under load. Tracing it through the slow query log led to an InnoDB buffer pool still at its 128 MB default, a wide view dragging a 107 KB JSON column through every page load, and a connection pool amplifying contention on a 4-core box."
showToc: true
---

Users of an internal document-management app started seeing timeouts whenever more than a few people opened a list screen at the same time. A plain `SELECT *` on a table with fewer than 2,000 rows was taking 7 seconds. That's the kind of number that makes no sense until you stop assuming row count is the thing that matters.

## Start with the logs, not the query

First question: is the table actually big? No — around 1,800 rows. But it occupied 222 MB on disk, roughly 138 KB per row. Checking column types found the culprit: one `JSON`-shaped column, a stored document cover-page template, averaged 107 KB per row and accounted for about 187 MB of the table by itself. Every list screen read through a view joining around ten tables, and that view selected the big column along with everything else — so a page showing five rows still dragged five 107 KB blobs through the entire join, every time.

Row count was never the problem. A handful of oversized columns behind a wide view and a `SELECT *` habit is enough to make a tiny table behave like a large one.

## Separating "slow" from "timing out"

The slow query log (threshold: queries over 10 seconds) gave a clean list to work from. Filtering to executions over 60 seconds and grouping by normalized SQL — literals swapped for `?` so that otherwise-identical queries count as one shape — turned 51 distinct shapes into a ranked list: two shapes, both filtering the same view with patterns like

```sql
SELECT * FROM document_revisions
WHERE factory_code = ? AND LOWER(owner) LIKE ? AND status IN (?)
LIMIT ?
```

accounted for 262 of 414 total executions over 60 seconds in one week — about 63%. `LOWER(col) LIKE '%text%'` can't use an index no matter what you put on the column; every one of those scanned the view's full result set.

## The timeline mattered more than the totals

The same query shapes ran in about 12 seconds on quiet days and 100 to 220 seconds during bursts at the start of the work day. That 8x–18x swing isn't what a bad index produces on its own — a missing index makes a query consistently slow, not slow-sometimes. Looking at timestamps, identical queries started within the same second of each other, meaning a single page load was firing several of them in parallel rather than one at a time. Even trivial lookups, like loading a short reference list, took over a minute during a burst.

That's the signature of saturation, not of one bad query: several heavy, unindexed scans landing at once on a 4-core box, with a 50-connection application pool happy to let all of them run concurrently and fight over the same CPU and disk. A bigger pool doesn't help saturation — it just lets more queries queue for the same scarce resource instead of one.

## Ruling things out instead of guessing

Before touching any settings, I checked the usual suspects and crossed each one off with a number, not a hunch:

- Lock waits: negligible, longest observed was 57 ms. Not a locking problem.
- Disk latency measured outside a burst: normal. The disk itself wasn't failing.
- Data directory: on its own drive with plenty of free space.

The one thing that did matter: MySQL's temp directory sat on the system drive, which had only about 4 GB free, and sort-spill temp files from those unindexed scans were piling up by the hundreds of thousands. A nearly-full system drive wasn't a database problem by itself — it became one only because the temp directory happened to live there.

And the number that explained the rest: InnoDB's buffer pool — the in-memory cache of table pages — was still sitting at its 128 MB default, while the actual data plus indexes ran around 400 MB. Roughly a third of all page reads were missing that cache and going to disk, on every query, all day.

## What I changed

First round, deliberately small, reversible, and applied online with `SET PERSIST` — takes effect immediately and survives a restart:

```sql
-- Before: 128 MB buffer pool, no execution cap, 256 KB join buffer
SET PERSIST innodb_buffer_pool_size = 3221225472;  -- 3 GB
SET PERSIST max_execution_time = 60000;            -- 60 s, SELECT only
SET PERSIST join_buffer_size = 1048576;            -- 1 MB
```

60 seconds for the execution cap matches the clients' own request timeout — past that point the result is thrown away anyway, so letting the query keep running only holds a connection and burns CPU for nobody's benefit. I didn't go lower, like 30 seconds: some legitimate dashboard queries had already reached 36 seconds under load, and a cap that kills real traffic is worse than no cap. `join_buffer_size` only moved up modestly, because it's allocated per unindexed join per connection — a generous value here multiplies fast across 50 connections, which would have traded one memory problem for another. A fourth fix, moving the temp directory off the nearly-full system drive, needs a restart; it's staged as a config change for the next maintenance window rather than applied live.

## What's actually confirmed, and what isn't

The buffer pool resize applied without a restart, and all three values showed up correctly afterward in MySQL's persisted-variables table (`performance_schema.persisted_variables`). That part is verified.

What isn't verified yet: real-world impact. The cache starts cold after a resize — it has to warm up by actually serving traffic before the hit rate improves — so the honest status is applied and confirmed, impact not yet measured. I haven't watched it through a real morning burst.

## Takeaways

- Row count isn't row size. A table with a couple thousand rows can behave like a much bigger one if a few columns hold large values, especially behind a wide view and a `SELECT *`.
- Averages lie, timelines don't. The same query at 12 seconds and at 220 seconds isn't "sometimes slow" — look at when the slow executions cluster, and whether they start in the same second as each other.
- A bigger connection pool doesn't fix contention on a small number of cores; it just gives contention more ways to happen at once.
- Check the defaults before anything else. A 128 MB buffer pool against several hundred MB of actual data is one of the cheapest, highest-leverage things you can find in a slow-query investigation.
- Still open on this one: stop selecting the oversized JSON column in list queries that don't need it, replace the `LOWER(col) LIKE '%...%'` filters with something indexable, and move the temp directory off the system drive.
