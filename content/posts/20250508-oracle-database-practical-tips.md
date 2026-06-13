---
title: "Oracle Database Practical Tips for Enterprise Projects"
date: 2025-05-08
draft: false
tags: ["oracle", "database", "sql", "sqlplus", "enterprise"]
categories: ["Databases"]
description: "Practical Oracle Database tips accumulated from financial IT projects: client compatibility issues, useful SQL for auditing schema and sessions, and common gotchas that cost time."
showToc: true
---

## The Oracle Client Compatibility Trap

The first thing that trips up anyone connecting to Oracle from Windows is the client version mismatch. Oracle client DLLs are not backward-compatible in the ways you might expect. If the database server runs 11g and you install a 19c client locally, tools like SQLGate or older JDBC thin drivers may fail with cryptic errors.

The safe rule: **match the client major version to the server major version**. If the database is 11g, install an 11g client. If it is 19c, install 19c.

For 11g on Windows x64, use the **Oracle Instant Client** rather than the full client installer — it is smaller, does not require an Oracle Home installation, and is easier to manage in a closed network where downloads are restricted:

```
instantclient-basic-windows.x64-11.2.0.4.0.zip
instantclient-sqlplus-windows.x64-11.2.0.4.0.zip
```

Extract to a directory (e.g., `C:\oracle\instantclient_11_2`), then add it to your `PATH` and set `ORACLE_HOME` to the same path. Tools like DBeaver and SQLGate discover the client by scanning `PATH`.

If SQLGate fails to launch with a DLL load error, the root cause is almost always this mismatch or `PATH` not being updated in the correct order (user path vs. system path).

---

## Useful SQL for Schema Inspection

In a financial project, you often inherit a schema from another team or a DBA who is not always reachable. These queries let you understand the schema without documentation.

### List all tables and their row counts

```sql
SELECT table_name, num_rows
FROM user_tables
ORDER BY num_rows DESC NULLS LAST;
```

`num_rows` is stale unless the DBA runs `ANALYZE TABLE` or `DBMS_STATS.GATHER_TABLE_STATS`. For a live count, use:

```sql
SELECT 'SELECT ''' || table_name || ''', COUNT(*) FROM ' || table_name || ' UNION ALL'
FROM user_tables
ORDER BY table_name;
```

Take the output, strip the trailing `UNION ALL`, and run it. It is slow but accurate.

### List columns for a given table

```sql
SELECT column_name, data_type, data_length, nullable, data_default
FROM user_tab_columns
WHERE table_name = 'YOUR_TABLE_NAME'
ORDER BY column_id;
```

### List foreign keys and their referenced tables

```sql
SELECT
    a.constraint_name,
    a.table_name,
    a.column_name,
    c_pk.table_name AS referenced_table,
    b.column_name   AS referenced_column
FROM user_cons_columns a
JOIN user_constraints c ON a.constraint_name = c.constraint_name AND c.constraint_type = 'R'
JOIN user_constraints c_pk ON c.r_constraint_name = c_pk.constraint_name
JOIN user_cons_columns b ON b.constraint_name = c_pk.constraint_name AND b.position = a.position
ORDER BY a.table_name, a.constraint_name;
```

This is invaluable when you need to know the deletion order for test data cleanup.

### Find indexes on a table

```sql
SELECT index_name, column_name, column_position, descend
FROM user_ind_columns
WHERE table_name = 'YOUR_TABLE_NAME'
ORDER BY index_name, column_position;
```

---

## Checking Active Sessions and Locks

In a shared test environment, multiple developers may be connected and one may have an uncommitted transaction holding a lock. This blocks everyone else.

### Find active sessions

```sql
SELECT sid, serial#, username, status, machine, program, last_call_et
FROM v$session
WHERE username IS NOT NULL
ORDER BY last_call_et DESC;
```

`last_call_et` is the number of seconds since the last SQL call — a session with a large value may be abandoned.

### Find lock holders

```sql
SELECT
    s.sid,
    s.username,
    s.machine,
    o.object_name,
    l.locked_mode
FROM v$locked_object l
JOIN dba_objects o ON l.object_id = o.object_id
JOIN v$session s ON l.session_id = s.sid;
```

If you identify a stale session, have the DBA kill it:

```sql
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

Do not do this in production without authorization — in a test environment it is usually fine.

---

## Validation SQL Patterns for Data Quality

Data quality checks are not a one-time activity. In a system that receives records from external sources, run these routinely.

### Duplicate primary key candidates

```sql
SELECT case_id, COUNT(*) AS cnt
FROM cases
GROUP BY case_id
HAVING COUNT(*) > 1;
```

If your primary key is a natural key (a business ID from an upstream system), duplicates indicate the upstream system sent the same record twice — which is normal, so your deduplication logic must handle it.

### Orphaned child records

```sql
SELECT cu.update_id, cu.case_id
FROM case_updates cu
WHERE NOT EXISTS (
    SELECT 1 FROM cases c WHERE c.case_id = cu.case_id
);
```

Orphaned updates mean either the parent was deleted without cascading, or a child record arrived before the parent. Both are worth investigating.

### Range and format validation

For a column that should always be a non-negative percentage:

```sql
SELECT case_id, fault_rate
FROM cases
WHERE fault_rate < 0 OR fault_rate > 100;
```

For a column expected to match a specific format (e.g., employee IDs):

```sql
SELECT employee_id
FROM case_updates
WHERE NOT REGEXP_LIKE(employee_id, '^[A-Z][0-9]{7}$');
```

These queries are fast to write and catch data quality problems that unit tests miss entirely, because unit tests use controlled fixture data that always looks correct.

---

## Date Handling Gotchas

Oracle's `SYSDATE` returns the database server's local time, which may differ from your application server's timezone. In a multi-server environment, **always store timestamps as UTC** and convert at the display layer.

```sql
-- Store UTC
INSERT INTO cases (case_id, received_at) VALUES ('C20250401', SYS_EXTRACT_UTC(SYSTIMESTAMP));

-- Query in local time for reporting
SELECT case_id, FROM_TZ(CAST(received_at AS TIMESTAMP), 'UTC') AT TIME ZONE 'Asia/Seoul' AS received_kst
FROM cases;
```

If you mix `SYSDATE` (local) and `SYSTIMESTAMP` (timestamped with timezone info) in the same schema, comparisons between them silently produce wrong results. Pick one and enforce it.

---

## Exporting and Importing Data Between Schemas

When you need to copy data from the production schema to test for debugging, and a full `expdp`/`impdp` is too heavy, insert-select across a database link is faster:

```sql
-- Assuming a DB link "TEST_LINK" pointing to the test database
INSERT INTO cases@TEST_LINK
SELECT * FROM cases
WHERE received_at >= DATE '2025-03-01';

COMMIT;
```

For a closed network where DB links are not allowed, export to CSV and import:

```bash
# Export with sqlplus spool
SELECT /*csv*/ case_id, received_at, resolved_at FROM cases WHERE ROWNUM <= 1000;
```

Use SQL Developer's export feature for structured CSV output when sqlplus spool formatting is too messy.

---

## Summary

Most Oracle issues I have encountered in practice come down to three things:

1. **Client version mismatch** — always match client to server major version.
2. **Nullability assumptions** — validate them with SQL, not documentation.
3. **Timezone inconsistency** — store UTC, convert at the boundary.

The SQL patterns here are not exotic. They are the queries you reach for repeatedly when something unexplained appears in the data, and having them ready saves the time you would otherwise spend constructing them from scratch at the wrong moment.
