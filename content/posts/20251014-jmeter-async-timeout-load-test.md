---
title: "JMeter Load Testing an Async Video Upload API: Results from a Real Timeout Fix"
date: 2025-10-14
draft: false
tags: ["jmeter", "testing", "springboot", "async", "jvm"]
categories: ["Backend"]
description: "After patching an async timeout handling bug in a video analysis API, we ran a JMeter load test to quantify the improvement. This post covers the test setup, what changed in the error distribution, what didn't change, and what the JVM heap graph revealed."
showToc: true
---

## Background

The system under test is a Spring Boot WAS that accepts video uploads, forwards them to a GPU-backed AI inference service, and returns analysis results. The AI service has variable processing times depending on video length — short clips return in seconds; long clips can exceed the client's timeout budget.

Before the patch, any analysis request that timed out at the AI server caused an `ASYNC_ERROR` to cascade all the way back to the client, even when the AI server did eventually finish. The fix introduced proper async error tracking so that timeout events are absorbed and retried without escalating to the caller.

---

## Test Configuration

| Parameter | Value |
|-----------|-------|
| Concurrent users | 4 |
| Request rate | 10 requests / user / minute |
| Total requests | 400 |
| Test duration | ~10 minutes |
| Test video pool | 200 video files, lengths from 1 s to 360 s |
| AI server timeout threshold | 180 seconds (configured in the AI service) |

The video pool deliberately includes clips well above the 180-second threshold to force timeout conditions. This is the scenario the patch targets.

---

## Results: Error Code Distribution

### Before the patch

| Error code | Count | Cause |
|-----------|-------|-------|
| `ASYNC_ERROR` | 228 | AI server timeout cascaded to client |
| `AI_ANALY_CPU_MAX` | 17 | AI server rejected request (CPU/queue limit) |

### After the patch

| Error code | Count | Cause |
|-----------|-------|-------|
| `ASYNC_ERROR` | 17 | WAS thread pool exhaustion (not AI timeout) |
| `AI_ANALY_CPU_MAX` | 17 | AI server rejected request (CPU/queue limit — unchanged) |

The patch reduced `ASYNC_ERROR` from 228 to 17 — a **92.5% reduction**. Critically, the remaining 17 `ASYNC_ERROR` codes have a different root cause: they stem from the WAS's own thread pool being exhausted under concurrent load, not from AI server timeouts. These are a separate issue from what was patched, and they showed up only under the 4-user concurrency scenario.

`AI_ANALY_CPU_MAX` remained constant at 17 in both runs because it is a hard limit inside the AI inference service — it rejects requests when its internal queue exceeds capacity. The patch has no effect on this path.

---

## Response Time Distribution

Response times are measured from the moment the WAS receives the upload to the moment it sends a response to the client. This includes the HTTP multipart upload transmission time itself, not just the AI processing time.

For the 400-request run:

- Majority of requests: under 3 seconds (upload transmission + quick WAS processing)
- Long-tail requests: bounded by the video transmission time for large files, not by AI analysis time (because the WAS does not block waiting for AI results — it returns immediately after queuing the analysis job)

This is expected behavior for the async model: clients get a fast acknowledgment and poll for results separately.

---

## JVM Heap Behavior

Monitoring JVM heap during the test revealed a **sawtooth pattern**:

```
Heap usage (%)
90% |     /\        /\
80% |    /  \  /\  /  \
70% |   /    \/  \/    \
60% |  /
50% | /
40% |___________________
     Time →
```

- Heap climbs as video upload buffers accumulate in memory during burst.
- At roughly 80% heap usage, the GC triggers (Parallel GC configured).
- After collection, heap drops back to ~40%.
- The pattern repeats as new requests arrive.

This is normal behavior for a workload that creates large byte-array objects (video multipart data). The 80% trigger threshold for GC is intentional — the JVM heap is configured at 4 GB (`-Xmx4096m`) so there is headroom before OOM conditions.

The sawtooth becoming steeper or not recovering to the baseline would indicate a memory leak. In this run it was stable.

---

## What the Remaining 17 `ASYNC_ERROR` Codes Mean

After the patch, the `ASYNC_ERROR` codes that remain are triggered by WAS thread pool exhaustion, not AI server timeouts. Under 4 concurrent users hammering 10 uploads per minute (40 uploads/minute total), the upload processing thread pool fills up before all uploads can be dispatched.

Two options for addressing this:

1. **Scale out** — add a second WAS instance behind the load balancer.
2. **Tune the thread pool** — increase the executor's core/max pool size and queue capacity.

Option 1 is the standard DevOps answer; option 2 can buy time but doesn't scale indefinitely.

The thread pool exhaustion is a separate bug from the AI timeout issue. Running the test helped us separate them clearly — before the patch, the AI timeout errors dominated the error distribution and masked the thread pool issue entirely.

---

## Lessons

**Load testing after a targeted fix validates scope.** Before the patch, 228 `ASYNC_ERROR` codes made it look like the AI integration was fundamentally broken. After the patch, the error count collapsed to 17 with a completely different root cause — the fix did exactly what it was designed to do and nothing more.

**Keeping video files of varying lengths in the test set is essential.** If all test videos were short, the AI timeout path would never fire and the test would give a false clean signal.

**The JVM heap graph is free context.** Even without a dedicated APM tool, exporting JVM metrics during the test takes two minutes of setup. The sawtooth pattern here confirmed the GC is working as intended; a flat or ever-rising line would have flagged a problem that code review alone would not catch.
