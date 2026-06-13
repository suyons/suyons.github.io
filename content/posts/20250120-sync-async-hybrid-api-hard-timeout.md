---
title: "Designing a Sync/Async Hybrid API When I had a Hard Timeout"
date: 2025-01-20
draft: false
tags: ["api", "async", "springboot", "architecture", "timeout"]
categories: ["Backend"]
description: "When a customer-facing API must respond within 60 seconds but the backend operation can take up to several minutes, a pure sync or pure async model both fail in different ways. Here is how we designed a hybrid approach and the architectural decisions that shaped it."
showToc: true
---

## The Constraint

An insurance claims system needed a video analysis feature: customers upload dashcam footage, and an AI model classifies the accident. The non-negotiable requirements were:

- The customer-facing API **must** respond within 60 seconds. This was a hard corporate policy for all externally-facing services, not a guideline.
- Some videos — particularly long recordings — can take the AI inference service **3 to 5 minutes** to process.
- The system runs **dual redundant WAS instances** behind a load balancer.

A purely synchronous API where the WAS blocks waiting for AI results immediately violates the 60-second rule for long videos. A purely asynchronous API where the caller polls for results was rejected because the customer-facing mobile app did not want to implement polling logic.

---

## The Architecture Decision

We settled on a **fire-and-wait hybrid with a defined fallback**:

1. The WAS accepts the video upload and immediately dispatches the analysis job to the AI server asynchronously (non-blocking thread).
2. The WAS waits up to 60 seconds for the AI result.
3. If the result arrives within 60 seconds, it is returned directly in the same HTTP response.
4. If not, the WAS returns a "processing" acknowledgment to the caller with a job ID, and stores the in-progress state internally.
5. The caller can optionally poll a separate `/result` endpoint using the job ID.

This satisfies the 60-second response guarantee (the caller always gets an HTTP 200 within the window) while gracefully handling the long-tail processing cases.

---

## API Design

### Upload and analyze

```
POST /api/v1/video/upload
Content-Type: multipart/form-data

Fields:
  request_id   : string (UUID, client-generated)
  video_file   : binary

Response (complete within 60s):
{
  "request_id": "...",
  "status": "COMPLETE",
  "results": [
    { "class": "REAR_END",        "confidence": 0.91 },
    { "class": "LANE_DEPARTURE",  "confidence": 0.04 }
  ]
}

Response (timeout — still processing):
{
  "request_id": "...",
  "status": "PROCESSING"
}
```

### Retrieve result (polling fallback)

```
POST /api/v1/video/result

Body:
{
  "request_id": "..."
}

Response:
{
  "request_id": "...",
  "status": "COMPLETE" | "PROCESSING" | "FAILED",
  "results": [...],
  "elapsed_ms": 12400,
  "error_code": null
}
```

---

## Implementation Pattern

The WAS side uses a `CompletableFuture` with a timeout:

```java
@PostMapping("/upload")
public ResponseEntity<AnalysisResponse> upload(
        @RequestParam("request_id") String requestId,
        @RequestParam("video_file") MultipartFile videoFile) {

    // Store the upload state immediately so polling works even if we timeout
    analysisStore.initiate(requestId);

    CompletableFuture<AnalysisResult> future = aiClient.submitAsync(requestId, videoFile);

    try {
        AnalysisResult result = future.get(55, TimeUnit.SECONDS); // 5s margin before the 60s wall
        analysisStore.complete(requestId, result);
        return ResponseEntity.ok(AnalysisResponse.complete(requestId, result));

    } catch (TimeoutException e) {
        // Don't cancel the future — let the AI server finish and store the result
        future.whenComplete((result, ex) -> {
            if (result != null) analysisStore.complete(requestId, result);
            else analysisStore.fail(requestId, ex);
        });
        return ResponseEntity.ok(AnalysisResponse.processing(requestId));
    }
}
```

Key decisions embedded in this code:

- **55-second wait, not 60.** The 5-second margin leaves room for serialization, network overhead, and response transmission before the caller's timeout fires.
- **Don't cancel the future on timeout.** The AI server has already started processing the video. Cancelling the future would abandon work that is already consuming GPU resources. Let it finish; the result will be stored and retrievable via the polling endpoint.
- **`analysisStore.initiate()` before the future.** If the WAS returns "PROCESSING" and the client immediately polls `/result`, the poll must find a record. Initiating the store entry before starting the async work prevents a race condition where the poll arrives before the entry exists.

---

## The Redundancy Problem

The `analysisStore` object above hides a significant complexity: with two WAS instances behind a load balancer, a poll request from the client may arrive at the instance that does **not** hold the in-memory state.

Options considered:

| Option | Approach | Trade-off |
|--------|---------|-----------|
| Sticky sessions | Load balancer routes each client to the same instance | Works, but defeats the purpose of redundancy on instance failure |
| Shared store | Redis or DB-backed `analysisStore` | Correct but adds dependency |
| AI server as truth | Query the AI server directly on poll | Couples the WAS poll path to AI server availability |

For phase 1, we used sticky sessions because it was the fastest to implement without adding infrastructure. The polling fallback is used by less than 5% of requests (those exceeding 55 seconds), so sticky-session failure events are rare and tolerable.

The correct long-term solution is to back `analysisStore` with Redis, which both WAS instances share. The `CompletableFuture` callback stores the result in Redis; the polling endpoint reads from Redis regardless of which WAS instance handles the request.

---

## Error Response Design

The AI server returns structured error responses that include machine-readable codes:

```json
{
  "request_id": "...",
  "status": "FAILED",
  "error_code": "AI_GPU_OVERLOAD",
  "error_detail": "GPU utilization exceeded threshold; request rejected",
  "elapsed_ms": 1200
}
```

Error codes are prefixed by their origin (`AI_` = AI server, `WAS_` = upload server, `ASYNC_` = async processing failures). This convention makes it trivial to filter logs by error origin and assign ownership during incident triage.

The `elapsed_ms` field is included even in error responses because it helps distinguish capacity exhaustion (fast failure, short elapsed time) from genuine timeouts (slow failure, elapsed time near the limit).

---

## What Not to Do

**Don't return HTTP 202 Accepted for everything and force polling.** When the operation typically completes in 10–30 seconds (the common case), making clients implement polling just to handle the rare 3-minute case is poor API design. The hybrid approach serves the common case as a synchronous response and gracefully degrades to polling for the outliers.

**Don't set the timeout exactly at the client's limit.** A 60-second timeout in the WAS plus 60-second client timeout means a single millisecond of overhead causes the client to time out before it receives the "PROCESSING" acknowledgment. Always subtract a margin.

**Don't cancel the AI job on WAS timeout.** GPU resources were already committed. Cancelling wastes the work done so far and forces a retry that will consume additional GPU time. Let it complete; just decouple the response path from the result.

---

## Summary

When a hard response-time SLA meets a variable-duration backend operation:

1. Set the WAS wait slightly below the client SLA (55s for a 60s limit).
2. Return a structured "processing" acknowledgment — not an error — when the wait expires.
3. Continue the backend operation and store the result asynchronously.
4. Provide a polling endpoint for clients to retrieve results for long-running jobs.
5. For multi-instance deployments, back the result store with a shared cache (Redis) rather than instance-local memory.

The result is a system that meets the SLA for the common case, degrades gracefully for outliers, and never loses work in progress.
