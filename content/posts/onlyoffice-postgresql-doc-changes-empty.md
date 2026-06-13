---
title: "Why OnlyOffice Needs Its Own PostgreSQL — and Why doc_changes Is Almost Always Empty"
date: 2026-05-28
draft: false
tags: ["onlyoffice", "postgresql", "architecture", "self-hosted", "edms", "troubleshooting"]
categories: ["DevOps"]
description: "When you integrate OnlyOffice Docs with a system that already stores the files, it still insists on its own PostgreSQL. Here's what that database actually does, why its main table sits empty, and the Windows Server gotcha when you try to inspect it remotely."
showToc: true
---

## The Question That Started This

I was integrating OnlyOffice Docs into an EDMS that already had file storage solved. Files land on disk under a `storage/` tree, and the application database maps each one back to its real name, owner, and permissions. That part was done.

Then OnlyOffice's install requirements asked for a PostgreSQL instance, a Redis instance, and a RabbitMQ broker. Three stateful services for a thing that — as far as I could tell — was just going to render documents the EDMS already owned.

So I installed it, wired it up, and out of curiosity connected to the OnlyOffice PostgreSQL with DBeaver to see what it was storing. The schema had two tables. One of them, `doc_changes`, the table that sounds like it holds the actual document edits, had zero rows. Every time I looked. Mid-edit, after edit, didn't matter — usually empty.

That combination bugged me enough to dig in: why does a stateless document renderer need an ACID database, and why is the table that should be busiest almost always blank? The answer is a clean little lesson in separation of concerns.

---

## Two Storage Systems, Two Jobs

The confusion comes from assuming "storage" is one responsibility. In this setup it's two, and they don't overlap.

**Your EDMS owns the documents.** When a user uploads a `.docx`, the application writes the bytes to its `storage/` path (typically under an opaque key, not the original filename) and records everything *about* the file — true name, folder, owner, ACLs — in its own database. That's the system of record. It survives forever.

**OnlyOffice owns the editing session.** When a user opens a document for editing, OnlyOffice Document Server takes over the live collaborative session. By design it tries to stay stateless across the long term: it does not want to be the durable home for your files, and it delegates persistent storage back to the integrator via the callback URL. When everyone closes the document, OnlyOffice should be able to forget it ever existed.

But "stateless long-term" and "no state at all" are different things. While N people are typing into the same paragraph simultaneously, *something* has to hold the in-flight changes with real guarantees — ordering, atomicity, durability against a crash. You cannot do real-time co-editing on hope. That short-lived, high-integrity workspace is what the PostgreSQL is for. It is not a document archive. It is a transaction buffer.

---

## The Division of Labor: Postgres vs. Redis vs. RabbitMQ

Once you frame Postgres as the durable scratchpad, the other two services fall into place. Each of the three handles a different class of state:

- **RabbitMQ — message transport.** Carries operational events between Document Server's internal services: a user connected, a conversion finished, a forced save was requested. These are fire-and-forget. Once delivered, they're gone.
- **Redis — volatile session state.** Holds the ephemeral stuff that's fine to lose on restart: presence, cursor positions, active-room bookkeeping. It lives in memory because it's meant to be cheap and disposable.
- **PostgreSQL — the durable buffer.** Holds the one thing you must *not* lose mid-session: the unsaved edits. If the box reboots while three people are editing, Redis evaporates, but Postgres has every keystroke that hadn't yet been flushed back to the EDMS.

So all three are "stateful" in some sense, but only Postgres is the part whose loss corrupts a user's work. That's the one that has to be ACID.

---

## The Schema Is Tiny

Here's what surprised me most. For all that ceremony, the OnlyOffice database is minimal — on the version I inspected, two tables:

```sql
CREATE TABLE public.doc_changes (
    tenant varchar(255) NOT NULL,
    id varchar(255) NOT NULL,
    change_id int4 NOT NULL,
    user_id varchar(255) NOT NULL,
    user_id_original varchar(255) NOT NULL,
    user_name varchar(255) NOT NULL,
    change_data text NOT NULL,
    change_date timestamp NOT NULL,
    CONSTRAINT doc_changes_pkey PRIMARY KEY (tenant, id, change_id)
);

CREATE TABLE public.task_result (
    tenant varchar(255) NOT NULL,
    id varchar(255) NOT NULL,
    status int2 NOT NULL,
    status_info int4 NOT NULL,
    created_at timestamp DEFAULT now() NULL,
    last_open_date timestamp NOT NULL,
    user_index int4 DEFAULT 1 NOT NULL,
    change_id int4 DEFAULT 0 NOT NULL,
    callback text NOT NULL,
    baseurl text NOT NULL,
    "password" text NULL,
    additional text NULL,
    CONSTRAINT task_result_pkey PRIMARY KEY (tenant, id)
);
```

`change_data` in `doc_changes` is where the actual edit deltas go. `task_result` tracks the per-document task: its status, the callback URL OnlyOffice will POST the assembled file to, and where the baseline document came from. Both are keyed by `(tenant, id)`, where `id` is the document key OnlyOffice assigns to an editing session.

<!-- TODO: author — the exact table set varies by Document Server version; confirm against the version you ship rather than treating "two tables" as universal. -->

---

## Why doc_changes Sits at Zero Rows

This is the part that looked like a bug and isn't. `doc_changes` is a delta log with a deliberately short life. Walk the lifecycle:

1. **Editing is live.** OnlyOffice does *not* rewrite the on-disk `.docx` on every keystroke — that would be slow and a corruption risk. Instead each change is appended as a delta row in `doc_changes`. During an active multi-user session, this table genuinely fills up.
2. **Everyone leaves.** When the last editor disconnects, OnlyOffice waits a configured cooldown, then assembles: it takes the baseline document, replays every delta row in order, produces one updated file, and POSTs it back to your EDMS callback URL.
3. **The EDMS acknowledges.** The moment your callback returns success, those deltas have a permanent home — in *your* storage, not OnlyOffice's. OnlyOffice immediately deletes the session's rows from `doc_changes`.

So the steady state is empty, because empty *is* the success condition. Rows in `doc_changes` mean either an edit is in flight right now, or — more interestingly — an assembly failed to hand off. If you ever find that table stubbornly non-empty with no active editors, that's your signal that the save callback to your EDMS is failing and documents are silently not persisting. It's a genuinely useful health check: **`doc_changes` should drain to zero shortly after sessions end; rows that linger are stuck saves.**

The cooldown before assembly is a configurable delay (in `local.json`), not instant-on-disconnect.

<!-- TODO: author — verify the exact config key for the save delay before naming it; behavior described above is correct, the key name is what I'm unsure of. -->

---

## The Windows Server Gotcha: Inspecting It Remotely

Connecting to that database from a workstation, on a Windows Server install, is where I lost time. Point DBeaver at it and PostgreSQL refuses the handshake:

```text
FATAL: no pg_hba.conf entry for host "203.0.113.10", user "onlyoffice", database "onlyoffice", no encryption
```

That's host-based authentication doing its job — there's no rule permitting your address. The OnlyOffice bundle ships its own PostgreSQL with config under the Document Server tree, not the usual Linux locations. On the install I had:

```
C:\Program Files\ONLYOFFICE\DocumentServer\Data\PostgreSQL\
```

Add a rule for your specific address at the bottom of `pg_hba.conf` (open the editor as Administrator — the file is under `Program Files`):

```text
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    onlyoffice      onlyoffice      203.0.113.10/32         scram-sha-256
```

The `/32` is doing real work: it grants exactly one host, not a range. Don't widen it to a whole subnet for convenience — this database is reachable from wherever the rule allows, and it holds in-flight document content.

Then confirm the server actually listens off-loopback, in `postgresql.conf` in the same folder:

```text
listen_addresses = '*'
```

And reload. On Windows the bundled service restarts via PowerShell (Administrator):

```powershell
Get-Service -Name *postgres* | Restart-Service -Verbose
```

One thing to leave alone while you're in there: the `dbHost` value in OnlyOffice's own `local.json`. OnlyOffice and its PostgreSQL are installed side by side on the same box and talk over loopback. Opening Postgres to your workstation is only about *your* read-only inspection; OnlyOffice itself should keep talking to `localhost`. Repointing `dbHost` makes OnlyOffice's own traffic take the long way around for no benefit.

---

## The Backup Consequence

The split that explains the empty table also dictates how you back this up. Your durable document state lives in two places that reference each other: the file bytes under the EDMS `storage/` path, and the rows in *your* application database that map keys to real files. (OnlyOffice's own Postgres, by contrast, you don't need to back up — by design it's transient; a clean instance is fine.)

The trap is backing those two up on independent schedules. Snapshot the storage directory at 02:00 and dump the database at 02:30, and any file created in between exists in one backup but not the other — an orphaned blob, or a database row pointing at a file that isn't in the snapshot. Restore from that pair and you get mapping corruption. Storage snapshot and database dump have to be taken as one atomic unit, or close enough that no writes happen between them.

---

## What It Comes Down To

OnlyOffice asking for a PostgreSQL instance isn't redundant with your existing file storage, because they're solving different problems: your storage is the permanent record, and OnlyOffice's database is a crash-safe buffer for edits that haven't been handed back yet. Once you see it that way, the empty `doc_changes` table stops looking broken and starts looking like a monitoring signal — and the rule for backups (atomic across storage and metadata) writes itself.
