---
title: "JMeter Load Testing for a Spring Boot WAS: Setup, Scenarios, and Interpretation"
date: 2025-08-14
draft: false
tags: ["jmeter", "testing", "springboot", "performance", "java", "tomcat"]
categories: ["DevOps"]
description: "How to set up Apache JMeter for load testing a Spring Boot WAS in a financial project: test plan structure, validating results with SQL, interpreting thread pool behavior, and common pitfalls."
showToc: true
---

## Why Load Test Before Production

In financial enterprise projects, load testing is often the last item on the schedule before go-live. Teams treat it as a checkbox — run some requests, see that the server stays up, declare success. This misses the point.

Load testing is the only practical way to answer three questions before go-live:

1. Under the expected concurrent user count, does the system stay within acceptable response times?
2. Does the system fail gracefully under overload, or does it cascade?
3. Does the JVM behave stably over time, or does memory or thread usage drift?

The answers to these questions are not visible in unit tests or in manual walkthroughs. They require sustained concurrent load against a realistic deployment.

---

## JMeter Test Plan Structure

A minimal JMeter test plan for a WAS endpoint:

```
Test Plan
└── Thread Group
    ├── HTTP Request Defaults
    ├── HTTP Cookie Manager
    ├── CSV Data Set Config (for variable inputs)
    ├── HTTP Request (your endpoint)
    │   └── Response Assertion
    └── Listeners
        ├── View Results Tree (debugging only)
        ├── Summary Report
        └── Aggregate Report
```

### Thread Group Configuration

For a financial WAS with moderate expected load:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Number of threads | 10–20 | Representative concurrent user count |
| Ramp-up period | 60 seconds | Gradual load buildup, not a spike |
| Loop count | 100–200 | Enough samples to see stable averages |
| Duration (if using duration-based) | 10 minutes | Long enough to see GC cycles |

Start conservatively. A thread count that is too high tells you the system breaks; a thread count that matches expected production load tells you whether it handles the expected load.

---

## Realistic Request Scenarios

Testing with a single hardcoded request body is not useful. Financial systems handle many different case types, employee IDs, and status combinations. Use a CSV data set:

### Create the CSV file

```csv
caseId,incidentType,status,employeeId
C20250801001,AUTO,A1,E1000001
C20250801002,AUTO,A0,E1000002
C20250801003,PROPERTY,A1,E1000003
C20250801004,PROPERTY,A0,E1000001
C20250801005,INJURY,A1,E1000002
```

### Configure CSV Data Set Config in JMeter

- Filename: path to your CSV
- Variable names: `caseId,incidentType,status,employeeId`
- Recycle on EOF: `true`
- Stop thread on EOF: `false`

### Use variables in the HTTP Request body

```json
{
  "caseId": "${caseId}",
  "incidentType": "${incidentType}",
  "status": "${status}",
  "submittedBy": "${employeeId}"
}
```

This ensures the test exercises different code paths and data combinations, not the same cached response path repeatedly.

---

## Assertions

Every HTTP request should have a response assertion to mark failed requests correctly:

**Response code assertion:**
```
Response Code equals: 200
```

**Response body assertion (for API endpoints that return a status field):**
```
Response Body contains: "success"
```

Without assertions, JMeter marks all responses as successful as long as the server returns any HTTP response. A 500 error with a response body counts as a pass unless you assert otherwise.

---

## Validating Results with SQL

After running a load test against the test environment, the JMeter summary tells you response times and error counts. It does not tell you whether the data is correct. For that, use SQL.

### Verify all requests were persisted

JMeter reports 200 OK for every request in the test. Cross-check against the database:

```sql
-- Count records inserted during the test window
SELECT COUNT(*) AS inserted_count
FROM case_updates
WHERE received_at BETWEEN TO_TIMESTAMP('2025-08-14 10:00:00', 'YYYY-MM-DD HH24:MI:SS')
                      AND TO_TIMESTAMP('2025-08-14 10:15:00', 'YYYY-MM-DD HH24:MI:SS');
```

If JMeter sent 200 requests and this returns 185, 15 requests returned 200 but did not persist — a silent failure in the service layer.

### Verify no duplicates were created

```sql
SELECT case_id, COUNT(*) AS cnt
FROM case_updates
WHERE received_at BETWEEN TO_TIMESTAMP('2025-08-14 10:00:00', 'YYYY-MM-DD HH24:MI:SS')
                      AND TO_TIMESTAMP('2025-08-14 10:15:00', 'YYYY-MM-DD HH24:MI:SS')
GROUP BY case_id
HAVING COUNT(*) > 1;
```

Under concurrent load, race conditions in insert logic can create duplicates that are invisible in unit tests. The load test surface this.

### Verify data integrity for each record

```sql
-- Check for records with null required fields inserted during the test
SELECT case_id, incident_type, status, received_at
FROM case_updates
WHERE received_at >= TO_TIMESTAMP('2025-08-14 10:00:00', 'YYYY-MM-DD HH24:MI:SS')
  AND (case_id IS NULL OR incident_type IS NULL OR status IS NULL);
```

---

## Monitoring JVM During the Test

Run VisualVM (bundled with JDK) or connect JConsole to the Tomcat process during the test. Watch:

- **Heap used** — should follow a sawtooth pattern (rise, GC, drop). A flat rise with no drops indicates a memory leak or GC being suppressed.
- **Thread count** — should be bounded by `maxThreads` in `server.xml`. A continuously rising thread count indicates threads are not being returned to the pool.
- **CPU usage** — should spike during request processing and settle during GC. Sustained 100% CPU often indicates a GC thrashing problem, not a request processing bottleneck.

Enable JMX on Tomcat to allow remote monitoring:

```bash
# In setenv.sh
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.port=9010"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.authenticate=false"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
```

This is acceptable for a test environment. Do not use unauthenticated JMX in production.

---

## Interpreting Error Distributions

A meaningful load test should produce some errors. If every single request succeeds under maximum thread count, either:
- The test is not stressing the system enough, or
- The system has capacity far beyond the expected load (both are fine, but confirm which)

When errors do appear, categorize them:

| Error type | Likely cause | Where to look |
|-----------|-------------|--------------|
| HTTP 500 | Exception in service layer | `catalina.out`, application log |
| HTTP 503 | Thread pool exhausted | `server.xml` `maxThreads`, `acceptCount` |
| Connection refused | Tomcat not running or port wrong | Server process check |
| Timeout (JMeter timeout) | Slow response, not a server error | Response time distribution, GC logs |

Do not treat all errors as the same. An HTTP 500 from a NullPointerException in business logic and an HTTP 503 from thread pool exhaustion require completely different fixes.

---

## Common JMeter Mistakes

**Viewing Results Tree in production load tests.** This listener logs every request and response body. Under load, it consumes significant memory and skews results. Use it only during debugging; disable it for the actual test run.

**Not warming up the JVM.** The first 10–20 requests are slower because the JVM JIT-compiles the code paths. Add a brief warmup period (30 seconds at low thread count) before the main test run, then reset the statistics.

**Not accounting for test environment differences.** The test server almost certainly has fewer CPU cores and less memory than production. Load test results from test are a relative measure (before vs. after a fix, or under different configurations) not an absolute prediction of production behavior.

---

## Summary

JMeter load testing is most valuable when it is part of a feedback loop: run the test, identify the bottleneck (from JVM metrics or server logs), fix it, run again. A single load test run with no follow-up is just a number. The SQL validation step — cross-checking persisted records against JMeter's reported request count — bridges the gap between HTTP-level testing and data correctness, and often surfaces the most important bugs.
