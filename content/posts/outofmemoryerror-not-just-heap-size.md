---
title: "OutOfMemoryError Is Not Always Fixed by Increasing Heap Size"
date: 2025-08-28
draft: false
tags: ["jvm", "java", "tomcat", "performance", "debugging", "springboot"]
categories: ["DevOps"]
description: "When a Spring Boot application throws OutOfMemoryError, the first instinct is to increase -Xmx. Often that is wrong — and sometimes it makes things worse. Here is how to diagnose which type of OOM you are dealing with and what to actually fix."
showToc: true
---

## The First Instinct Is Usually Wrong

An application throws `java.lang.OutOfMemoryError`. Someone doubles `-Xmx` from 2GB to 4GB. The error goes away for a week, then comes back. `-Xmx` is doubled again. This cycle continues until the server runs out of physical memory.

The mistake is treating OOM as "not enough memory" when it is often "memory that is allocated but never released." Adding heap to a leaking application does not fix the leak; it just slows down how quickly the leak fills the bucket.

Before changing any JVM flags, diagnose which type of OOM you are dealing with.

---

## Types of OutOfMemoryError

The error message tells you where the problem is:

### `Java heap space`

```
java.lang.OutOfMemoryError: Java heap space
```

This means the heap is full. Could be a memory leak (objects not being garbage collected because they are still referenced) or a genuine workload spike (a single request loaded too much data into memory).

### `GC overhead limit exceeded`

```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

The JVM spent more than 98% of its time doing garbage collection and recovered less than 2% of the heap. This is almost always a memory leak — the GC is continuously trying to clean up, but references are still held, so very little can be collected.

**Increasing heap size here delays the inevitable.** More heap means the leak takes longer to fill, so the GC runs slightly less frantically, but the underlying problem is not addressed.

### `Metaspace`

```
java.lang.OutOfMemoryError: Metaspace
```

Class metadata (loaded class definitions) has filled the metaspace. Common cause: dynamic class generation (CGLIB proxies, Groovy scripts, Javassist) or classloader leaks in applications that reload contexts (such as Tomcat hot-redeployment).

Increasing `-Xmx` does nothing here. Fix: `-XX:MaxMetaspaceSize=256m` (or larger), and investigate why class loading is unbounded.

### `unable to create new native thread`

```
java.lang.OutOfMemoryError: unable to create new native thread
```

The OS refused to create a new thread. Usually caused by too many threads being created without being returned to a pool, or by the OS's per-process thread limit being hit.

Fix: check thread pool configuration, look for `new Thread()` calls outside of thread pools.

---

## How to Diagnose a Heap Leak

### Step 1: Enable GC logging

```bash
# JDK 8
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/opt/app/logs/gc.log"

# JDK 9+
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:file=/opt/app/logs/gc.log:time,uptime:filecount=5,filesize=10m"
```

A healthy application shows a sawtooth pattern — heap rises, GC fires, heap drops back toward a stable baseline. A leak shows a sawtooth where the floor never returns to the same level:

```
Heap (GB)
4 |              /\
3 |         /\  /  \  /
2 |    /\  /  \/    \/   ← floor rising: leak
1 |   /  \/
0 |_____________________
  Time →
```

### Step 2: Take a heap dump

On the running JVM:

```bash
jmap -dump:format=b,file=/tmp/heap.hprof $(pgrep -f catalina)
```

Or configure automatic heap dump on OOM:

```bash
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/app/logs/"
```

The automatic option is better for catching the state right before the crash.

### Step 3: Analyze the heap dump

Open `heap.hprof` in Eclipse Memory Analyzer Tool (MAT). Run the "Leak Suspects" report. MAT identifies the top retained-memory accumulators — the objects holding the most memory transitively.

Common findings in Spring Boot applications:

| Object type | Likely cause |
|------------|-------------|
| `ArrayList` or `HashMap` in a singleton bean | In-memory cache with no eviction policy |
| HTTP session objects | Sessions not expiring; too many concurrent sessions |
| `byte[]` arrays | File or image data loaded entirely into heap |
| `String` objects | Log messages being built at INFO level even when INFO is disabled |

---

## Common Leak Patterns in Enterprise Applications

### In-memory accumulation in a singleton

A common anti-pattern in financial systems: a `@Service` bean accumulates records in a `List` that is never cleared.

```java
@Service
public class CaseAuditService {

    // This list lives for the lifetime of the application context
    private final List<AuditEvent> recentEvents = new ArrayList<>();

    public void record(AuditEvent event) {
        recentEvents.add(event);  // grows forever under load
    }
}
```

Fix: use a bounded structure with an eviction policy, or persist to the database and clear the in-memory buffer periodically.

### Session accumulation

If the application creates HTTP sessions for every request (the default in a non-stateless Spring Security configuration) and does not expire them, the session store grows without bound.

Check session configuration:

```java
// If you are using stateless JWT or SSO token authentication,
// sessions should not be created at all
http.sessionManagement()
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

If sessions are required, configure a maximum and timeout:

```xml
<!-- conf/web.xml -->
<session-config>
    <session-timeout>30</session-timeout>  <!-- minutes -->
</session-config>
```

### Large objects loaded per request

Loading an entire database table into a list for filtering in Java:

```java
// Don't do this
List<Case> allCases = caseRepository.findAll();
return allCases.stream()
    .filter(c -> c.getStatus().equals("FINAL"))
    .collect(toList());
```

Each request loads potentially thousands of rows into heap. Under load, multiple concurrent requests do this simultaneously.

Fix: push the filter to the database:

```sql
SELECT * FROM cases WHERE status = 'FINAL'
```

This is exactly why moving business logic into the database layer — through SQL — reduces heap pressure. The database filters and returns only the rows you need; the JVM never holds the full result set.

---

## When Increasing Heap Is the Right Answer

There are legitimate cases:

- A batch job that processes a large dataset once has a genuine peak memory requirement. If the peak exceeds the current heap and the usage drops after the batch completes, the heap genuinely needs to be larger for that peak.
- A new feature adds a legitimately large in-memory data structure that could not be avoided (a compiled regex pattern set, a large lookup table).

The test: after increasing heap, does the GC floor stabilize at a new level, or does it keep rising? If it stabilizes, the workload needed more headroom. If it keeps rising, the heap increase bought time but did not fix the problem.

---

## SQL to Monitor for Leak-Adjacent Issues

Database connection pool exhaustion is a related failure mode that presents similarly to an OOM — the application becomes unresponsive under load, but the culprit is not heap.

```sql
-- Check for long-running uncommitted transactions (sessions holding connections)
SELECT s.sid, s.serial#, s.username, s.status,
       TO_CHAR(s.logon_time, 'YYYY-MM-DD HH24:MI:SS') AS logon_time,
       ROUND((SYSDATE - s.logon_time) * 24 * 60) AS minutes_connected
FROM v$session s
WHERE s.username IS NOT NULL
  AND s.status = 'ACTIVE'
ORDER BY s.logon_time;
```

A large number of active sessions from the same application host indicates the connection pool is not releasing connections properly — often because an exception path skips the `finally` block that closes the connection.

With Spring's `JdbcTemplate` or MyBatis, connection management is automatic. With raw JDBC or a misconfigured `DataSource`, this is a genuine risk.

---

## Summary

`OutOfMemoryError` is a symptom, not a cause. The cause is either a genuine memory requirement exceeding the configured ceiling, or a memory leak where objects are being retained longer than their useful lifetime. Increasing heap size is the right fix for the first case and a temporary delay for the second.

The diagnostic sequence — GC log analysis, heap dump, MAT leak report — narrows the problem to a specific class and usually a specific code path. That is faster than guessing, and it produces a fix that actually holds.
