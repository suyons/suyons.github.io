---
title: "Windows Server Troubleshooting - A Verified Buffer Pool Fix Led to 4.6 GB of Memory Nobody Had Budgeted"
date: 2026-11-25
draft: false
tags: ["windows", "apache-ignite", "mysql", "memory", "capacity-planning", "production-debugging"]
categories: ["Infrastructure"]
description: "Auditing a week-old InnoDB buffer pool fix on a production Windows box confirmed it worked - and in the process of checking whether 3 GB fit in RAM, turned up a much bigger problem: an unrelated Java service quietly defaulting to 20% of physical memory for an off-heap cache."
showToc: true
---

A week and a half ago, a production Windows server running an internal document-management app got an emergency fix: raise the MySQL InnoDB buffer pool from its 128 MB default to 3 GB, because `SELECT` queries on a table with under 2,000 rows were taking seconds under load. That part is already written up. This session asked the follow-up question nobody had gotten to yet: did 3 GB actually solve it, and is the box healthy now? On a production machine, "the emergency fix worked" and "we checked" are not the same claim.

## Step one: verify the fix instead of trusting it

The InnoDB buffer pool is MySQL's in-memory cache of table and index pages. Too small, and every query goes to disk. So the first check was whether 3 GB is actually big enough to hold the working set:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_%';
SELECT SUM(data_length + index_length) / 1024 / 1024 AS mb
FROM information_schema.tables WHERE table_schema = 'app_db';
```

The answer was unambiguous. Buffer pool hit ratio: 99.997%. Of the 3 GB allocated, only 381 MB was actually in use, with `Innodb_buffer_pool_wait_free` sitting at zero - no thread had ever had to wait for a free page. Queries over 60 seconds, which had hit 414 in the five days before the fix, were at zero in the week after. The buffer pool is not the constraint anymore, and it has plenty of headroom to grow into.

One thing worth knowing if you do this audit yourself: on Windows, `SET PERSIST` writes to `mysqld-auto.cnf` in the data directory, and that file takes precedence over `my.ini`. The static config file still read `innodb_buffer_pool_size=128M` - checking the config file alone would have told me the fix never happened. Check `performance_schema.persisted_variables`, or the running value, not the file on disk.

## Step two: find what's still slow

With the buffer pool ruled out, the slow query log for the following eleven days told a cleaner story: 16 distinct query shapes logged a total of 40 executions over 10 seconds, all but one of them clustered in a single burst on one morning. Two shapes - a paginated list query and its matching `COUNT(1)` over the same view - accounted for 23 of those 40 by themselves.

Both share the same two problems:

- The view's date column is stored as a formatted string and its status column is a computed `CASE` expression. Neither can use an index, no matter what you add.
- The application wraps the search term in `LOWER(doc_name) LIKE '%term%'`. That's pure overhead: every table in this schema already uses the case-insensitive `utf8mb4_0900_ai_ci` collation, so `LOWER()` changes nothing except making the column unindexable and forcing a function call on every row.

There's also a cheap win sitting in the pagination logic itself: every page request re-runs the `COUNT(1)` over the same expensive view, even when only the offset changed. During the burst, the same search fired its count-and-list pair about a dozen times in one minute.

None of that is a server-infrastructure fix - it's a view rewrite and a query change, so it went to the application team as a ticket rather than something I touch. Two settings changes are still mine to make, and they're low-risk enough to flip whenever:

```sql
-- queued, not yet applied
SET PERSIST table_open_cache = 8000;  -- current cache is overflowing (62,853 misses) under view-heavy joins
SET PERSIST long_query_time = 2;      -- a 10 s threshold hid everything in the 5-9 s range from the slow log
```

Worth flagging on that second one: `long_query_time` is read per-session, so a connection pool's existing connections keep using the old threshold until they reconnect. Don't expect the new value to apply retroactively to long-lived pooled connections.

## Step three: the bigger problem was never MySQL

Checking whether a 3 GB buffer pool fits comfortably in RAM meant checking total memory pressure on the box - and that's where the real finding was. Windows reported commit charge at 18.8 of 23.8 GB, about 79%. Commit charge is the total memory every process has promised to use, backed by RAM plus the pagefile; when it hits the limit, new allocations start failing regardless of how much physical RAM shows as "free."

The biggest consumer on the box wasn't MySQL. It was an unrelated Java 11 service - a PDF conversion backend with an embedded Apache Ignite node - sitting at 8.7 GB committed, against MySQL's roughly 4.2 GB. Pulling the JVM heap apart with `jcmd <pid> GC.heap_info` showed the heap itself was fine: old-gen usage steady around 413 MB, well inside a 2 GB `-Xms`/4 GB `-Xmx` ceiling, no leak. The memory wasn't in the heap at all.

(Side note: `jstat` can't attach to this process - it runs as a `LocalSystem` service in session 0, and `jstat` reads performance counters that aren't visible across sessions. `jcmd` attaches differently and works fine. If `jstat` can't see a Windows service's JVM, reach for `jcmd` instead.)

## Apache Ignite's defaults are a hidden percentage of your RAM

Ignite keeps its working data in off-heap "data regions," and a region with no explicit `maxSize` defaults to **20% of physical RAM**. This service's persistent region (`DB_REGION`) had committed about 3.2 GB to hold 138 MB of actual data, plus a checkpoint page buffer that had grown to roughly 2 GB on top of that. The default (non-persistent) region added another 256 MB committed against 24 MB used. None of this shows up by inspecting the JVM heap - it's entirely outside it.

The plan, not yet applied, is to size the regions explicitly in the Ignite Spring XML instead of leaving them to the 20%-of-RAM default:

```xml
<!-- before: no maxSize on either region -->
<bean id="dbRegion" class="org.apache.ignite.configuration.DataRegionConfiguration">
    <property name="persistenceEnabled" value="true"/>
    <property name="name" value="DB_REGION"/>
</bean>
```

```xml
<!-- after: explicit caps, default region added -->
<property name="defaultDataRegionConfiguration">
    <bean class="org.apache.ignite.configuration.DataRegionConfiguration">
        <property name="maxSize" value="#{256L * 1024 * 1024}"/>
    </bean>
</property>
...
<bean id="dbRegion" class="org.apache.ignite.configuration.DataRegionConfiguration">
    <property name="persistenceEnabled" value="true"/>
    <property name="maxSize" value="#{512L * 1024 * 1024}"/>
    <property name="checkpointPageBufferSize" value="#{256L * 1024 * 1024}"/>
    <property name="name" value="DB_REGION"/>
</bean>
```

Because `DB_REGION` is persistent, capping it is safe in principle: if the working set doesn't fit, Ignite evicts pages back to disk instead of failing outright. The plan also drops the heap ceiling from `-Xmx4096m` to `-Xmx3072m` - still roughly double the observed live usage, just not quadruple it - and adds `-XX:NativeMemoryTracking=summary` temporarily so that `jcmd <pid> VM.native_memory summary` can confirm, after the next restart, where the committed memory actually went.

Doing the arithmetic on the configured caps (not a measured result - this hasn't shipped yet): capping the two regions and the checkpoint buffer at roughly 1 GB combined, down from about 5.5 GB today, projects to around 4.6 GB less committed memory for this one service - taking it from 8.7 GB down to somewhere near 4.1 GB. Applied server-wide, that would bring commit charge from 18.8 GB down toward roughly 14 GB, from 79% of the 23.8 GB limit to around 60%. Separately, the pagefile on the data drive is currently system-managed with almost no actual paging happening (254 MB observed max usage) - fixing it at a static 16 GB would raise the commit *limit* itself to around 32 GB, independent of the Ignite fix, as a safety margin rather than a cure.

None of this has shipped. The restart needed for the Ignite XML change means a maintenance window and a heads-up to whoever owns the PDF service, and I'd rather confirm the native-memory numbers on a quiet day than find out they're wrong during a real outage.

## What's actually done vs. planned

- **Done and confirmed:** the buffer pool resize. Verified via hit ratio, free pages, and the drop in 60-second-plus queries.
- **Diagnosed, ticketed, not mine to fix:** the view design and the redundant `LOWER()` filter behind the remaining slow-query cluster.
- **Queued, zero-risk, not yet flipped:** the two `SET PERSIST` changes above.
- **Planned, requires a restart, not yet applied:** the Ignite region caps, the heap ceiling cut, and the pagefile resize. The 4.6 GB figure is a projection from the before/after config values, not a measurement.

## Takeaways

1. Verifying a fix with its own metrics is what let me stop looking at MySQL and go find the actual remaining risk on the box. "It should be fine now" and "the hit ratio is 99.997%" are different levels of confidence.
2. A default that scales with available RAM is a hidden budget. 20% of physical memory per data region sounds reasonable in isolation, until two regions and a checkpoint buffer on one unrelated service eat more committed memory than the database you were actually worried about.
3. Commit charge, not "free RAM" in Task Manager, is the number that kills a Windows server under memory pressure.
4. A plan with config diffs and projected numbers is not the same thing as a fix. Say which one you're reporting - conflating them is how a "we already fixed it" story turns into a surprise outage later.
