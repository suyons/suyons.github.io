---
title: "Prometheus Instrumentation - The Scrape Interval That Made a Busy Queue Look Empty"
date: 2026-10-24
draft: false
tags: ["prometheus", "grafana", "nodejs", "observability", "nginx"]
categories: ["DevOps"]
description: "Adding metrics to a zero-dependency Node.js relay meant hand-writing the Prometheus text format, and a 15-second scrape interval that outran 2-12 second jobs forced a redesign of what a 'queue' dashboard even measures."
showToc: true
---

A small Node.js relay sits in front of an ONLYOFFICE Document Server and turns its DocBuilder API — a scripting endpoint that renders and merges documents — into a fire-and-forget job queue: a caller submits a job, the relay polls the Document Server until the conversion finishes, then POSTs the result to a callback URL. It was running in production as a black box. Nobody could answer "how many jobs are waiting right now, how many finished, how long do they take, which ones are in the queue?" without grepping logs. The goal was a Grafana dashboard that answers exactly those four questions. The interesting part turned out to be deciding what "queue" should even mean.

## Working with what already existed

The host already ran a Prometheus and Grafana stack with dashboards provisioned as code, so the task wasn't "pick a monitoring tool" — it was "get numbers out of the relay in a form Prometheus can scrape." The relay is deliberately a single file with zero npm dependencies, built only on Node's built-ins (`http`, `crypto`, `fetch`). Pulling in `prom-client` for this would have broken that property for what is, in fact, a simple text format. Prometheus exposition is just lines of `name{labels} value`. So the relay renders it by hand, in about 50 lines.

The relay already kept a `Map` of in-flight jobs — it needs that to send error callbacks on shutdown — so counting waiting jobs was free. What was missing was any record of jobs that had already finished, and how long they took.

## The scrape-interval trap

Before writing anything, I pulled the relay's logs for real job timings. That changed the design before a line of metrics code existed: typical PDF merges finished in **2 to 12 seconds**, and Prometheus scrapes every **15 seconds**. A naive "current queue" table — list whatever's in-flight right now — would almost always render empty. Most jobs are born and finished between two scrapes, invisible to any instant query.

So the per-job table exposes **in-flight jobs plus the 20 most recently finished ones**, each row carrying a `state` label of `waiting`, `ok`, or `error`. A waiting-count gauge still exists for the time-series panel, but the dashboard text says plainly that it under-samples short jobs and points readers at the table instead.

```js
// Before: the only job state was the in-flight map
const jobs = new Map();

// After: aggregates plus a bounded ring of recently finished jobs
const BUCKETS = [1, 2, 5, 10, 30, 60, 120, 300, 600];
const RECENT_JOBS = 20;
const stats = Object.fromEntries(['async', 'sync'].map((mode) =>
  [mode, { ok: 0, error: 0, sum: 0, buckets: BUCKETS.map(() => 0) }]));
const recent = [];

// Processing time = submit until output is ready; callback delivery isn't included.
function finish(job, { bytes, error }) {
  const seconds = (Date.now() - job.started) / 1000;
  const s = stats[job.mode];
  s[error ? 'error' : 'ok']++;
  s.sum += seconds;
  BUCKETS.forEach((le, i) => { if (seconds <= le) s.buckets[i]++; });
  recent.unshift({ ...job, seconds, bytes, state: error ? 'error' : 'ok' });
  recent.length = Math.min(recent.length, RECENT_JOBS);
}
```

The exported metric families:

- `docbuilder_jobs_in_flight{mode}` — the "waiting" number.
- `docbuilder_jobs_total{mode,result}` — the "done" (and failed) counter.
- `docbuilder_job_duration_seconds` — a histogram for processing-time percentiles.
- `docbuilder_recent_job_seconds`, `..._started_timestamp_seconds`, `..._output_bytes` — each labelled by `key`, `mode`, and `state`: the rows of the queue table.

Putting a job key in a label is normally a cardinality red flag — labels are supposed to have a small, known set of values, and a job key is neither. Here it's bounded by construction: at most ~20 live series for the recent ring, plus whatever is currently in flight. I still added a `sample_limit` on the scrape job as a backstop in case that assumption ever breaks, because "bounded by how I wrote the code" is not the same guarantee as "bounded by the protocol."

## What "size" is allowed to mean

The original ask was a table with "key + size." The relay can't know an output's size until the conversion finishes, and in synchronous mode it never downloads the files at all — the caller does that directly. So "size" became **total output bytes for finished async jobs only**; every other row leaves that column blank. That's a narrower claim than "size," but it's an honest one, and the panel description says so rather than implying a number that isn't there.

## Three details that would have quietly broken the scrape

**Label escaping.** In sync mode the caller picks the job key. One stray `"` or `\` in a label value makes the entire scrape response unparseable — not just that metric, the whole payload Prometheus is trying to read. Values get escaped per the exposition spec before they're written. The relay's offline selftest uses a deliberately hostile key, `` k"1\ ``, specifically to exercise that path.

**Duplicate series.** If a caller reuses a sync key, two rows would end up sharing the same label set, and Prometheus rejects the entire scrape rather than picking one. A `Set` dedupes rows by key at render time, right before the text gets assembled.

**An unauthenticated endpoint on the public route.** The edge nginx proxies everything under `/document-builder/` straight through. Without a change, `/document-builder/metrics` — complete with a list of job keys — would have been reachable from the internet. An exact-match `location` beats the prefix block in nginx, so it was the fix:

```nginx
# /metrics is unauthenticated and lists job keys; Prometheus scrapes it on loopback, not through here.
location = /document-builder/metrics { return 404; }

location ^~ /document-builder/ {
    proxy_pass http://127.0.0.1:10003/;
    ...
}
```

I checked the obvious bypasses before trusting it: URL-encoding (`/document-builder/%6Detrics`) and doubled slashes (`/document-builder//metrics`). nginx normalizes both before location matching, so both still return 404. A single blocked path with no bypass test is just a path you haven't broken yet.

## The dashboard layout

Four stat tiles — waiting now, done in 24h, failed in 24h, p90 processing time — and three time series: waiting jobs, finished jobs per 5 minutes, and p50/p90/mean processing time. Below them, the job table, built from three instant PromQL queries in table format, joined on the shared `key`/`mode`/`state` labels with Grafana's *merge* transformation. An *organize* transformation then hides Prometheus's bookkeeping columns and renames `Value #A/B/C` to *Seconds*, *Started*, and *Output size*. The start-timestamp column gets multiplied by 1000 in the PromQL expression itself, because Grafana's time-field units expect milliseconds and the metric is in Unix seconds.

## What's actually verified, and what isn't

- The relay's offline selftest passes with the new metrics assertions — run inside the relay's own pinned Node container, since the host itself has no Node installed.
- `promtool check metrics` reports no errors against the hand-written exposition output.
- The scrape target shows healthy in Prometheus's target list, and every PromQL expression on the dashboard executes against live data.
- Grafana picked up the provisioned dashboard without manual steps.
- The edge nginx returns 404 for the metrics path and still serves its health check.

What's still open: the table's rendering with real rows. The relay restart that shipped this reset the in-memory counters, so the next real PDF merge will be the first actual data point — I haven't seen the table populate with production traffic yet. Alerting on a growing queue or a rising error rate is also still on the list, not built.

## Takeaways

1. **Look at real job durations before designing a "queue" view.** If jobs finish faster than your scrape interval, a snapshot of the current queue tells you almost nothing — most of its life happens between two scrapes. Keep a short history instead of trusting the instant value.
2. **The Prometheus text format is simple enough to write by hand.** For a zero-dependency service, 50 lines of string building beats pulling in a client library — as long as you respect the escaping rules and never emit duplicate series, both of which will silently take down the entire scrape, not just one metric.
3. **Every new endpoint on a proxied service is public until proven otherwise.** Check exactly what the edge forwards before adding a debug or metrics route, then test the bypass variants — encoded paths, doubled slashes — not just the happy-path request.
