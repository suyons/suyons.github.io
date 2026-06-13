---
title: "What a Security Review Actually Looks Like in a Financial IT Project"
date: 2025-07-31
draft: false
tags: ["security", "devops", "compliance", "enterprise", "financial"]
categories: ["DevOps"]
description: "The security review process in Korean financial IT projects: what gets reviewed, what gets blocked, how long it takes, and how to prepare so you are not the bottleneck."
showToc: true
---

## The Concept of Security Review

In Korean financial institutions, deploying new software or connecting systems across network segments requires formal security approval. This is not a code review or a penetration test. It is a compliance gate: a committee or security team reviews the proposed architecture, data flows, and access permissions and decides whether to approve, require modifications, or reject.

Nothing crosses a network boundary without this approval. Your WAS cannot reach the database server. The database cannot be queried from your development workstation. The ESB cannot send messages to your test endpoint. All of these require network access rules, and network access rules require a passing security review.

This is not optional overhead. It is a legal and regulatory requirement for systems that touch financial data. Understanding how it works changes how you plan your project timeline.

---

## What Goes Into a Security Review

The review covers:

### 1. Network topology and data flow

You submit a diagram showing which systems communicate with which, what protocol they use, and what data flows between them. The security team checks whether the proposed communication paths cross network segments that should be isolated from each other, and whether the data classification (public, internal, confidential, restricted) is appropriate for the channel.

For a typical WAS project, this means documenting:
- Which IP ranges and ports your WAS needs to reach (database, ESB endpoint, SSO server, document management API)
- Which systems send requests to your WAS and from which IP ranges
- Whether any data leaving the internal network is encrypted

### 2. Authentication and authorization

The review confirms that every endpoint is protected. Common questions:
- Does the system authenticate using the organization's SSO?
- Are there any anonymous endpoints, and if so, why?
- How are service accounts (for batch jobs, scheduled tasks) authenticated?
- Are credentials stored securely, not in plaintext config files?

### 3. Logging and audit trail

Financial systems are required to log access to sensitive data. The review checks:
- Is there an audit log for data access operations?
- Are logs stored in a tamper-resistant location?
- Do logs capture enough information to reconstruct what happened if an incident occurs?

### 4. Vulnerability assessment (for production deployments)

For production systems, a vulnerability scan of the WAR or the server is sometimes required. This includes checking for known CVEs in dependencies and verifying that the server does not have unnecessary services running.

---

## How Long It Takes

This varies by institution and urgency, but realistic expectations:

| Stage | Typical duration |
|-------|-----------------|
| Submission of architecture document | 1–3 days to prepare |
| Initial review and feedback | 1–2 weeks |
| Revision based on feedback | 1–3 days |
| Final approval | 3–5 business days |
| Network access rule implementation (after approval) | 1–5 business days |

End to end, expect 3–6 weeks from first submission to having the firewall rules in place. For an urgent project, it is possible to expedite to 1–2 weeks with strong project management support, but plan for the longer estimate.

The key takeaway: **submit the security review package before you need the access, not when you need it**. If you start developing and realize you need the database connection rule three weeks in, and then submit the review, you are looking at 3–6 more weeks of blocked work.

---

## What Gets Blocked During Review

From practical experience, these are the most commonly blocked items in test environments while a review is pending:

- **Database connection from the WAS** — you cannot run integration tests against the real test database until the firewall rule is approved.
- **Log file access** — viewing application logs on the server may require going through a jump host or bastion, and those access paths also require approval.
- **External system integration** — ESB connections, SSO validation, document management API calls are all blocked until the relevant network rules are in place.

This means your test environment deployment can succeed (the WAR loads) but the application cannot be fully functional. Work around it by:

1. **Stub integrations locally** — mock the database, ESB, and SSO for unit and integration tests that run on your own machine.
2. **Deploy to test early** — even with some features blocked, deploying early confirms the WAR loads, the Spring context initializes, and the application starts without errors. These are useful data points.
3. **Test what you can** — endpoints that do not require the blocked resources can be tested. Write smoke tests that hit those endpoints and confirm they return expected status codes.

---

## Preparing the Review Package

The review submission requires a structured document. What I typically include:

### Architecture diagram

A network diagram showing:
- All servers involved (application server, database, ESB, external systems)
- Their network zones (DMZ, internal, restricted)
- Arrows for each communication path with protocol, port, and direction

Tools: draw.io or a simple spreadsheet works. Fancy diagrams are not required; accuracy is.

### Port and protocol list

| Source | Destination | Protocol | Port | Purpose |
|--------|-------------|----------|------|---------|
| App server (WAS zone) | Database server (DB zone) | TCP | 1521 | Oracle JDBC |
| ESB (integration zone) | App server (WAS zone) | HTTPS | 8443 | ESB callback |
| App server (WAS zone) | SSO server (auth zone) | HTTPS | 443 | Token validation |

### Data classification

List which tables or APIs contain sensitive data (employee personal information, financial transaction data) and confirm they are only accessible via authenticated paths.

### Authentication mechanism

Describe how each communication path is authenticated: SSO token for human users, service account certificate for machine-to-machine, API key for internal services.

---

## The Human Side

The security review is not adversarial. The security team is not trying to block your project. They are managing risk on behalf of the institution, and they have seen enough incidents to know what to look for.

Effective engagement:

**Be specific in your submissions.** Vague diagrams produce more questions and longer review cycles. A diagram that precisely identifies which server (by hostname or role) connects to what, on which port, gets approved faster than one that says "app connects to database."

**Respond to feedback quickly.** When the review comes back with questions or required changes, a fast response (same day if possible) keeps the review moving. A slow response means your item falls to the bottom of the queue.

**Use the review as a forcing function.** The requirement to document network flows and data classification surfaces architectural decisions you might otherwise defer. "Where does employee personal information go?" is a question you should be able to answer before you deploy, not after.

---

## SQL to Prepare for the Audit Log Requirement

One frequent requirement from security reviews is an audit log of access to sensitive data. If your application queries employee personal information, case details, or financial data, you need a log of who accessed what and when.

A simple audit table:

```sql
CREATE TABLE access_audit (
    audit_id        NUMBER          GENERATED ALWAYS AS IDENTITY,
    accessed_by     VARCHAR2(20)    NOT NULL,
    accessed_table  VARCHAR2(50)    NOT NULL,
    record_id       VARCHAR2(50),
    access_type     VARCHAR2(10)    NOT NULL,  -- SELECT, INSERT, UPDATE, DELETE
    accessed_at     TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
    PRIMARY KEY (audit_id)
);
```

Validation query — verify the audit log is capturing access during a test session:

```sql
-- Count of access events per user in the last hour
SELECT accessed_by, access_type, COUNT(*) AS event_count
FROM access_audit
WHERE accessed_at >= SYSTIMESTAMP - INTERVAL '1' HOUR
GROUP BY accessed_by, access_type
ORDER BY event_count DESC;

-- Check for any access without a logged user (should be zero)
SELECT COUNT(*) AS anonymous_access
FROM access_audit
WHERE accessed_by IS NULL OR accessed_by = '';
```

If the review requires audit logs, having this table and being able to demonstrate populated data during a test session is significantly more persuasive than promising it will be implemented later.

---

## Summary

The security review is a fixed cost of working in financial IT, not a one-time obstacle. The engineers who navigate it smoothly are the ones who submit complete documentation early, respond quickly to feedback, and treat the security team as a collaborator rather than a blocker. The engineers who struggle are the ones who treat it as a surprise and submit late. At 3–6 weeks per review cycle, one late submission can delay a project by a full sprint.
