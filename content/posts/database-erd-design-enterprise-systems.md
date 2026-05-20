---
title: "Database ERD Design in Enterprise Projects: Designing for Validation"
date: 2025-04-24
draft: false
tags: ["oracle", "database", "sql", "erd", "design"]
categories: ["DevOps"]
description: "How to approach ERD design in a multi-integration enterprise project, and why writing validation SQL queries against the schema is just as important as the design itself."
showToc: true
---

## Why ERD Design is Harder in Integration Projects

Standalone application databases are relatively forgiving. You control the data lifecycle end to end. But in a system that receives data from multiple upstream sources — a core banking system, a document management platform, an organization directory — the ERD carries a different burden. It has to express not just the shape of your data but the contracts you have with every caller.

In my experience, the design phase is where most integration bugs originate, not the code. A misunderstood field, a nullable column that is always expected to carry a value, a foreign key that only holds for one of two message types — these will not show up in unit tests. They show up in production at 2 AM.

The countermeasure is deliberate: **write validation queries as part of the design process, before a single line of application code exists**.

---

## Starting with Business Questions, Not Tables

A common mistake is starting with tables. Tables are an answer; they should come after the question.

Start with the business operations that need to be audited:

- How many cases were received today, and how many are still unresolved?
- Which employee submitted the most recent update on a given case?
- How many uploads arrived via the document system but could not be matched to a case?

Each of these is a query. Write it out in pseudo-SQL first:

```sql
-- Unresolved cases as of today
SELECT count(*) FROM cases WHERE resolved_at IS NULL AND received_at < SYSDATE;

-- Latest submitter per case
SELECT case_id, employee_id, submitted_at
FROM case_updates
WHERE submitted_at = (
    SELECT MAX(cu2.submitted_at)
    FROM case_updates cu2
    WHERE cu2.case_id = case_updates.case_id
);

-- Uploads with no case match
SELECT d.document_id, d.uploaded_at
FROM document_uploads d
WHERE NOT EXISTS (
    SELECT 1 FROM cases c WHERE c.case_id = d.case_id
);
```

Now look at what tables and columns these queries imply. That is your first ERD sketch.

---

## Schema Design Example

### Case Management Tables

```sql
CREATE TABLE cases (
    case_id         VARCHAR2(20)    NOT NULL,
    received_at     TIMESTAMP       NOT NULL,
    resolved_at     TIMESTAMP,
    fault_rate      NUMBER(5,2),
    PRIMARY KEY (case_id)
);

CREATE TABLE case_updates (
    update_id       NUMBER          GENERATED ALWAYS AS IDENTITY,
    case_id         VARCHAR2(20)    NOT NULL,
    employee_id     VARCHAR2(20)    NOT NULL,
    update_type     VARCHAR2(10)    NOT NULL,  -- 'DRAFT' or 'FINAL'
    submitted_at    TIMESTAMP       NOT NULL,
    PRIMARY KEY (update_id),
    FOREIGN KEY (case_id) REFERENCES cases(case_id)
);
```

### Document Upload Tracking

```sql
CREATE TABLE document_uploads (
    document_id     VARCHAR2(50)    NOT NULL,
    case_id         VARCHAR2(20),   -- nullable: may arrive before case is created
    uploaded_by     VARCHAR2(20),   -- nullable: uploader may not be mappable
    uploaded_at     TIMESTAMP       NOT NULL,
    PRIMARY KEY (document_id)
);
```

Notice that `case_id` in `document_uploads` is nullable. This is a deliberate design decision: the document management system sends uploads with a shared service account rather than a real employee ID, so the case match has to happen asynchronously by joining on case number from the core banking feed. If you make it NOT NULL with a FK constraint, inserts will fail until the case record arrives. If the systems are decoupled in time (and they always are), the nullable column with a deferred match is the safer design.

---

## Validation SQL as a Development Discipline

Once the schema exists in the test environment and data starts flowing, switch from building features to auditing data. The pattern I follow:

### 1. Verify cardinality assumptions

Each case should have at most one FINAL update. If this breaks, the business logic is wrong.

```sql
SELECT case_id, COUNT(*) AS final_count
FROM case_updates
WHERE update_type = 'FINAL'
GROUP BY case_id
HAVING COUNT(*) > 1;
```

If this returns rows, there is either a bug in the submission logic or the business rule was misunderstood.

### 2. Verify join completeness

How many documents are still unmatched after 24 hours?

```sql
SELECT COUNT(*) AS unmatched_old
FROM document_uploads
WHERE case_id IS NULL
  AND uploaded_at < SYSDATE - INTERVAL '1' DAY;
```

An unmatched document that is more than a day old is almost certainly a data quality problem — either the upstream system sent an unknown case ID, or the matching logic has a bug.

### 3. Trace data lineage under business rules

The business rule: if employee A submits a DRAFT and employee B later submits a FINAL, the case is attributed to B.

Validate this is consistent in the data:

```sql
SELECT
    c.case_id,
    latest.employee_id AS final_owner,
    earliest.employee_id AS original_submitter,
    CASE WHEN latest.employee_id != earliest.employee_id THEN 'REASSIGNED' ELSE 'SAME' END AS ownership
FROM cases c
JOIN (
    SELECT case_id, employee_id
    FROM case_updates
    WHERE update_type = 'FINAL'
) latest ON latest.case_id = c.case_id
JOIN (
    SELECT case_id, employee_id,
           ROW_NUMBER() OVER (PARTITION BY case_id ORDER BY submitted_at) AS rn
    FROM case_updates
) earliest ON earliest.case_id = c.case_id AND earliest.rn = 1;
```

This query is not production code. It is a diagnostic tool you run against the test environment after deploying the submission logic, to confirm the data reflects the rule rather than something else.

---

## Schema Versioning in a Closed Network

Database changes in a closed network cannot be applied via a CI/CD migration pipeline. The process is manual: write the DDL, hand it to the DBA, wait for confirmation.

A few practices that reduce friction:

**Provide idempotent scripts.** Use `CREATE TABLE IF NOT EXISTS` equivalents and `ALTER TABLE ADD COLUMN IF NOT EXISTS` where the DBA may run scripts more than once:

```sql
-- Oracle does not have IF NOT EXISTS natively; use this pattern
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM user_tab_columns
    WHERE table_name = 'CASE_UPDATES' AND column_name = 'REVIEWED_BY';

    IF v_count = 0 THEN
        EXECUTE IMMEDIATE 'ALTER TABLE case_updates ADD reviewed_by VARCHAR2(20)';
    END IF;
END;
/
```

**Attach a validation query to every schema change.** After the DBA applies the DDL, they (or you via a controlled query session) run a simple check:

```sql
-- Verify column was added
SELECT column_name, data_type, nullable
FROM user_tab_columns
WHERE table_name = 'CASE_UPDATES';
```

This makes the deployment step verifiable without needing application logs.

---

## What Gets Missed Without Validation Queries

The single most common silent failure in integration projects is **mismatched nullability assumptions**. The API spec says a field is required; the database allows NULL; the code never checks because unit tests use mock data that always provides it; and in production the first edge case sends a record without it, which inserts a NULL, and six months later a report produces wrong numbers.

The cure is simple: run a NULL audit on every column that should not be NULL, one week after go-live:

```sql
SELECT
  'cases' AS table_name,
  'fault_rate' AS column_name,
  COUNT(*) AS null_count
FROM
  cases
WHERE
  fault_rate IS NULL
UNION ALL
SELECT
  'case_updates',
  'employee_id',
  COUNT(*)
FROM
  case_updates
WHERE
  employee_id IS NULL;
```

If any row in this result has `null_count > 0`, go investigate before the business team finds it in a dashboard.

This is not automated testing. It is manual discipline, and it catches things that automated testing does not.
