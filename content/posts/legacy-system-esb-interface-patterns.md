---
title: "Integrating with Legacy Systems via ESB: Patterns and Pitfalls"
date: 2025-05-22
draft: false
tags: ["esb", "integration", "springboot", "java", "enterprise"]
categories: ["DevOps"]
description: "What Enterprise Service Bus (ESB) integration looks like in a real financial IT project: message format negotiation, request logging, validation queries, and the coordination overhead you don't see in the architecture diagram."
showToc: true
---

## What ESB Means in a Korean Financial System

An Enterprise Service Bus in a Korean financial institution is not a message queue in the modern sense. It is the routing and transformation layer that sits between the core banking system and any downstream service that needs data from it. Every interaction goes through a formally defined **message format** — a fixed-field or structured XML/JSON document with a header and a body that must match the specification to the byte.

You do not call the core banking system's API directly. You send a message to the ESB, the ESB routes it to the right backend, and eventually a response message comes back. The message contract is negotiated upfront between teams and signed off in a document. Changing it later requires going back through the same approval chain.

This is the environment where I built a new WAS that needed to receive real-time case update events from the core banking system.

---

## Message Format Negotiation

The first step is not writing code. It is a meeting — usually several — to agree on the message structure with the ESB team.

A typical request message structure:

```
[HEADER]
service_id     : string(10)   -- identifies the ESB routing rule
request_id     : string(20)   -- unique per request, for deduplication
timestamp      : string(14)   -- YYYYMMDDHHMMSS
system_code    : string(5)    -- identifies the caller

[BODY]
case_id        : string(20)
incident_type  : string(4)
fault_rate     : decimal(5,2)
assigned_to    : string(10)   -- employee ID
org_code       : string(8)    -- department code
status         : string(2)    -- A0=draft, A1=final
```

Key negotiation points that came up:

- **Transmission frequency**: does the ESB send on every save, or only on final submission? The answer turned out to be both — a draft save triggers a transmission, and a final submission triggers another. The consumer (my WAS) had to handle duplicates with different `status` values for the same `case_id`.
- **Required fields**: `org_code` was initially listed as required but later declared optional. The code had to handle both absent and present values without failing.
- **Timing**: the ESB sends regardless of whether the consumer is up. A consumer outage means messages accumulate on the ESB side and arrive in burst when the consumer comes back up. Rate limiting on the consumer side matters.

---

## Request Logging Before Anything Else

The first thing to deploy to the test environment is a filter that logs every incoming request, header and body included. Do not deploy any business logic until you have confirmed what the ESB actually sends.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpReq = (HttpServletRequest) request;
        ContentCachingRequestWrapper wrappedReq = new ContentCachingRequestWrapper(httpReq);

        chain.doFilter(wrappedReq, response);

        String body = new String(wrappedReq.getContentAsByteArray(), StandardCharsets.UTF_8);
        log.info("METHOD={} URI={} BODY={}", httpReq.getMethod(), httpReq.getRequestURI(), body);

        Enumeration<String> headers = httpReq.getHeaderNames();
        while (headers.hasMoreElements()) {
            String name = headers.nextElement();
            log.info("HEADER: {}={}", name, httpReq.getHeader(name));
        }
    }
}
```

The reason for `ContentCachingRequestWrapper`: once `getInputStream()` is called, the stream is consumed. Wrapping it caches the bytes so you can read them in the filter after the chain has processed the request.

Deploy this first. Ask the ESB team to send a test transmission. Read the log. Only then write the deserialization logic.

---

## Handling the Actual vs. Specified Message

The specification and the actual transmission almost never match perfectly on the first test.

Common discrepancies I encountered:

| Spec said | ESB actually sent |
|-----------|-------------------|
| `org_code` present | `org_code` absent |
| `fault_rate` as decimal | `fault_rate` as string `"0.00"` |
| `status` values: A0, A1 | Occasionally sent empty string |
| `case_id` max 20 chars | Sent 21-char values in edge cases |

None of these cause problems if your deserialization is lenient and your validation happens explicitly in code rather than relying on type coercion.

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class CaseUpdateMessage {

    private String caseId;
    private String incidentType;
    private String faultRateRaw;  // accept as string, convert in business logic
    private String orgCode;       // nullable
    private String status;

    public BigDecimal getFaultRate() {
        if (faultRateRaw == null || faultRateRaw.isBlank()) return null;
        try {
            return new BigDecimal(faultRateRaw);
        } catch (NumberFormatException e) {
            return null;
        }
    }
}
```

Explicit null checks and conversion methods are not defensive programming theater — they are the documented handling for real discrepancies observed in test.

---

## SQL Validation After Integration Testing

Once the ESB is sending real test traffic and the WAS is persisting records, the most effective quality check is SQL.

### Verify message deduplication

The core banking system may send the same case update twice (draft + final). Check that your deduplication logic is working:

```sql
-- Count cases with more than one FINAL status record
SELECT case_id, COUNT(*) AS final_count
FROM case_updates
WHERE status = 'FINAL'
GROUP BY case_id
HAVING COUNT(*) > 1;
```

A non-empty result means the deduplication logic missed at least one duplicate.

### Verify all transmitted messages were persisted

Ask the ESB team for the count of messages sent to your endpoint during a test window. Cross-check against your table:

```sql
SELECT COUNT(*) AS received_count
FROM case_updates
WHERE received_at BETWEEN TO_TIMESTAMP('2025-03-27 09:00:00', 'YYYY-MM-DD HH24:MI:SS')
                      AND TO_TIMESTAMP('2025-03-27 18:00:00', 'YYYY-MM-DD HH24:MI:SS');
```

If the counts do not match, either some messages were dropped (consumer error), rejected (validation failure), or the ESB has a transmission retry that created duplicates.

### Check for records with null required fields

After a day of test traffic:

```sql
SELECT case_id, incident_type, status, received_at
FROM case_updates
WHERE case_id IS NULL
   OR incident_type IS NULL
   OR status IS NULL
ORDER BY received_at DESC;
```

Even if the spec says these are required, run this. The spec has been wrong before.

---

## The Coordination Cost

Architecture diagrams make ESB integration look clean. In practice, the friction is almost entirely in coordination, not in code.

You are blocked until the ESB team has set up the routing rule pointing to your test endpoint. You are blocked until they fix a mismatched field type. You are blocked waiting for a security review to allow your server to be reachable from the ESB network segment.

The response to this is two things:

1. **Stub the ESB locally**. Write a unit test that sends a POST request with the agreed message body to your controller. You can develop and test your entire handling logic without the real ESB being available.

```java
@Test
void handlesCaseUpdateWithNullOrgCode() throws Exception {
    String payload = """
        {
          "caseId": "C20250327001",
          "incidentType": "AUTO",
          "faultRateRaw": "0.35",
          "status": "A1"
        }
        """;
    mockMvc.perform(post("/api/case-updates")
            .contentType(MediaType.APPLICATION_JSON)
            .content(payload))
        .andExpect(status().isOk());
}
```

2. **Log everything from day one**. When the ESB team says "we sent it," you need evidence of what arrived. Request logs with full body and headers are that evidence.

---

## Summary

ESB integration in a financial system is more about coordination and validation than code. The code is straightforward; the challenge is ensuring it handles every variation the ESB actually sends rather than just what the specification says it should send. Full request logging, lenient deserialization, and SQL-based validation after each test session are the three practices that catch problems before they reach production.
