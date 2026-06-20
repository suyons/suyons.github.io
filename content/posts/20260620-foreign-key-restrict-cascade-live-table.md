---
title: "Database Troubleshooting - A Blocked DELETE Was the Database Doing Its Job"
date: 2026-06-20
draft: false
tags: ["mysql", "knex", "foreign-keys", "referential-integrity", "database-design", "nodejs"]
categories: ["Databases"]
description: "A DELETE failed with a foreign key error, and the manual SQL fallback failed the same way. The bug wasn't the constraint that fired — it was the constraint that was missing, and a delete route that pretended children didn't exist. Here's how I decided restrict over cascade, added foreign keys back into a live table without breaking it, and why the follow-up timeout was a lock wait, not a logic bug."
showToc: true
---

## The error was the feature

A `DELETE` against a workflow-management endpoint failed. Fine, bugs happen — so someone tried the manual SQL fallback, and that failed too, with the same message:

```
Cannot delete or update a parent row: a foreign key constraint fails
```

The instinct in the room was that something was broken. It wasn't. That error is the database refusing to create orphaned rows, which is exactly what you want it to do. The real bug was upstream: a delete route that only ever deleted the parent and pretended the children didn't exist, plus — more dangerously — a place where the constraint *should* have existed and didn't.

This is the backend of an Electronic Document Management System. The interesting work wasn't writing the fix; it was deciding *where* referential integrity belongs (database, application, or both) and *what behavior* it should enforce (cascade or restrict). Those two questions come up in every CRUD backend, and the default answers people reach for are usually wrong.

## The data model, and the orphan-shaped trap

Three tables matter here. I'll use illustrative names:

- **`workflow_master`** — one row per workflow process (an approval/revision pipeline).
- **`workflow_detail`** — a mapping table: which document numbers belong to each workflow.
- **`doc_revision`** — document-revision history, which carries a `workflow_reg_id` column pointing back at the workflow.

`workflow_detail` had a proper foreign key onto `workflow_master`. So when the delete tried to remove a master row that still had detail rows referencing it, the database correctly refused. Good.

`doc_revision` did not. Its `workflow_reg_id` column pointed at the workflow, but **no database-level foreign key enforced it.** That's the trap. A delete that managed to remove a workflow would silently leave revision history pointing at a workflow that no longer existed — and nothing would stop it. The missing constraint was more dangerous than the one that fired, because the one that fired at least failed loudly. The missing one fails silently, months later, when someone joins revision history back to a workflow and gets nulls.

## Restrict, not cascade

The fast way to make the delete "just work" is to remove the children, then the parent, inside a transaction. That's the **cascade** reflex, and for master/reference data it's almost always the wrong call.

Cascade means a single delete on the workflow quietly destroys its document mappings, and (if you wired it up that way) its revision history too. That's a lot of data loss hidden behind a convenient API call. For data that other records depend on, **restrict** is the right default: refuse the delete and make the caller clean up the dependencies explicitly. The friction is the point — it forces an intentional decision instead of a silent one.

So the route enforces restrict semantics by hand, inside a transaction: block if revisions reference the workflow, block if detail rows still exist, and only then delete the master.

```js
await knex.transaction(async (trx) => {
  const master = await trx(tableName).select('reg_id').where(req.params).first();
  if (!master) return res.json({ status: false, message: 'not found' });

  // Block if document revisions reference this workflow
  const inUseByRevisions = await trx('doc_revision')
    .where('factory_code', req.params.factory_code)
    .where('workflow_reg_id', master.reg_id)
    .first();
  if (inUseByRevisions) {
    return res.json({ status: false, message: 'in use by revisions' });
  }

  // Block if the workflow still has documents mapped to it
  const inUseByDetail = await trx('workflow_detail')
    .where('factory_code', req.params.factory_code)
    .where('reg_id', master.reg_id)
    .first();
  if (inUseByDetail) {
    return res.json({ status: false, message: 'has member documents' });
  }

  await trx(tableName).where(req.params).del();
  return res.json({ status: true });
});
```

## App checks aren't the guarantee — the constraint is

Here's the part people skip. The route above gives friendly error messages, but it is not a guarantee. A second code path, a background job, or a human in a SQL client can bypass every one of those checks. App-level validation is a UX layer on top of the real rule; it is not the rule.

The durable fix was to add the foreign keys the schema was missing, with `ON DELETE RESTRICT ON UPDATE RESTRICT`:

```sql
ALTER TABLE `workflow_detail`
  ADD CONSTRAINT `wf_detail_doc_fk`
  FOREIGN KEY (`factory_code`, `doc_no`)
  REFERENCES `document_master` (`factory_code`, `doc_no`)
  ON DELETE RESTRICT ON UPDATE RESTRICT;
```

A second constraint went on `doc_revision` to reference the workflow detail, closing the orphan trap from earlier. Now the integrity rule lives in the one place nothing can route around. The app checks stay — but their job changed. They no longer *enforce* anything; they exist to turn a raw `errno 1451` into "this document is registered in a workflow," which is the message a user should see.

That division is the whole lesson: **enforce at the database, present at the application.** The constraint is the guarantee. The app check is the translation.

## Adding a foreign key to a live table without breaking it

`ADD FOREIGN KEY` on an empty table is trivial. On a table with existing rows, it fails the instant any row already violates the constraint you're adding — and a production table almost always has a few. So each `ALTER` got an orphan check first:

```sql
-- Find detail rows whose referenced document doesn't exist.
-- If this returns anything, ADD FOREIGN KEY will fail.
SELECT d.factory_code, d.doc_no
FROM workflow_detail d
LEFT JOIN document_master m
  ON d.factory_code = m.factory_code
  AND d.doc_no = m.doc_no
WHERE m.doc_no IS NULL;
```

Empty result, then add the key. If it's not empty, you have to decide per row whether it's data to repair or data to delete — and that's a decision, not a migration step.

Two more things make `ADD FOREIGN KEY` fail in ways the error message barely explains, both worth memorizing:

1. **The referenced columns need a PRIMARY KEY or UNIQUE index, in the same column order.** A foreign key onto `(factory_code, doc_no)` requires a unique index on exactly `(factory_code, doc_no)` in the parent — not `(doc_no, factory_code)`, and not just an index on each column separately.
2. **The column types and collations must match exactly on both sides.** A `VARCHAR(20)` referencing a `VARCHAR(30)`, or `utf8mb4_general_ci` referencing `utf8mb4_unicode_ci`, fails with the famously unhelpful `errno: 150 "Foreign key constraint is incorrectly formed."` Nine times out of ten, errno 150 is a type or collation mismatch, not a logic error.

## The follow-up timeout was a lock, not a bug

After deploying, a delete request hung and timed out. No error, no response — just nothing, until the request gave up.

The reflex is to suspect the code you just shipped. Resist it, because the *shape* of the failure rules that out. A logic bug throws an exception. A **timeout** — a request that hangs and returns nothing — is a different animal: it points at a lock wait, not a code path.

The cause was external to the new code entirely. Someone had run a manual `DELETE` earlier in a SQL client with autocommit off, and never committed it. That open transaction still held row locks on the table. The application's transaction queued politely behind it and waited until the request timed out. Committing the stray client transaction released the locks and everything flowed again.

The transferable habit: **distinguish a timeout from an error before you start debugging.** An error points at your code. A timeout points at concurrency — open transactions, held locks, a deadlock victim that isn't yours. They send you to completely different places, and confusing the two costs an afternoon.

## What carries over

Strip away the EDMS specifics and four things generalize:

- **For reference and master data, prefer restrict over cascade.** Make deletion of depended-upon rows an explicit act, not a silent side effect.
- **Enforce integrity at the database; present it at the application.** The constraint is the guarantee no code path can bypass. The app check exists to produce a human-readable message, not to be the rule.
- **Before adding a foreign key to a live table, run the orphan check** — `LEFT JOIN ... WHERE parent IS NULL` — and confirm a matching unique index plus identical column types. Those are the three things that make `ADD FOREIGN KEY` fail, and errno 150 almost always means a type or collation mismatch.
- **A timeout is not an error.** When a request hangs with no response, look for lock waits and stray uncommitted transactions before you blame the code you just shipped.
