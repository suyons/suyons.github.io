---
title: "Express Logging - The Failed Query Wasn't in the Error You Caught"
date: 2026-06-19
draft: false
tags: ["express", "knex", "logging", "nodejs", "morgan", "observability"]
categories: ["Backend"]
description: "A teammate kept seeing 403s that were really server-side failures, with nothing useful in the log — no failing query, sometimes no line at all. The fix wasn't a better catch block. It was capturing the SQL at the Knex layer that still has it, logging every HTTP outcome at response-finish, and a one-flag trick to stop double-logging."
showToc: true
---

## A 403 that swallowed its own cause

A teammate filed the kind of bug report that tells you exactly what's wrong without telling you how to fix it:

> "Sometimes it responds 403 though it's actually a server-side error, and I want all 4xx and 5xx logged with the raw queries."

This is an Express document-management backend talking to MySQL through Knex. The logging was already "in place" — a leveled logger, an access log, a `.catch` on every data-access call. And it was useless for this exact situation. When a query failed, one of two things happened:

1. The route caught the error, answered the client with a friendly `403`/`501`, and logged a database error **with no query attached** — the caught error's `.sql` field was empty.
2. The route answered directly and never reached the central Express error handler, so there was **no developer-facing log line at all.**

Either way, the one thing you need at 2 a.m. — *which query blew up* — was gone. The instinct is to fix the catch block: log harder, log `err.sql`, log the stack. That instinct is wrong, and understanding why is the whole post.

## You can't reconstruct the query from a swallowed error

Here's the catch site, roughly:

```js
// The query is already gone by the time you're here
const logDbError = (err, context) =>
  logger.error(`[DB] ${err.sqlMessage || err.message}`, {
    code: err.code,
    sql: err.sql, // ← frequently undefined
  });
```

The problem is structural, not a missing field. By the time a rejected query reaches your `.catch`, the error object you're handed is whatever the driver chose to throw. For a lot of failures that object simply doesn't carry the SQL — `err.sql` is empty or undefined. You're downstream of the place where the statement still existed, trying to rebuild it from an error that never had it.

So the rule that fixed this: **capture data at the layer that still has it.** Don't reconstruct the query at the catch site. Subscribe to the database driver where the statement is guaranteed intact.

Knex emits a `query-error` event on the connection. It hands you the original error *and* an object carrying the executed SQL and its bindings — the real statement, every time, no matter how some route downstream later mangles the response:

```js
const logQueryError = (err, obj) => {
  logger.error(err.sqlMessage || err.message || err, {
    code: err.code,
    errno: err.errno,
    sqlState: err.sqlState,
    sql: obj && obj.sql,           // the executed statement — reliable here
    bindings: obj && obj.bindings, // and the parameter values
  });
};

// One global listener is the single source of truth for failed queries
connection.on('query-error', logQueryError);
```

One listener, registered once at startup, becomes the authoritative record of every failed query in the app. It fires before any route's `.catch` gets a chance to flatten the error into a generic message. The catch block can answer the client however it likes; the query is already logged, correctly, with bindings.

## Log the HTTP outcome where every path converges

Capturing the query solves half the report. The other half — "all 4xx and 5xx logged" — has its own trap: some routes answer directly (`res.status(403).json(...)`) and never hit the central error handler, so a handler-based logger misses them entirely.

The fix is to stop logging at each `return` site and instead log once, at the moment the response finishes, where *every* path converges regardless of how it got there:

```js
app.use((req, res, next) => {
  // Capture the human-readable message out of the { status:false, message } body
  const sendJson = res.json.bind(res);
  res.json = (body) => {
    if (body && body.status === false && body.message) {
      res.locals.errorMessage = body.message;
    }
    return sendJson(body);
  };
  res.on('finish', () => logResponseError(req, res));
  next();
});

const logResponseError = (req, res) => {
  if (res.statusCode < 400 || req._errorLogged) return; // see dedup below
  const message = res.locals.errorMessage || res.statusMessage || '';
  logger.log(
    res.statusCode >= 500 ? 'error' : 'warn',
    `${res.statusCode} ${req.method} ${req.originalUrl || req.url}${message ? ` - ${message}` : ''}`,
  );
};
```

The `res.on('finish')` hook fires for every response — the ones that went through your error handler and the ones that answered directly. Wrapping `res.json` lets the hook recover the *real* reason from the response body (`{ status: false, message: '...' }`) instead of logging a bland "Forbidden". Now a direct `403` produces a single clean line with the message that actually explains it.

## The one flag that stops double-logging

Two mechanisms can now log the same response: the central error handler (which logs with a full stack trace) and the finish hook (lighter, no stack). For a handler-routed error, both would fire — you'd get the failure twice.

The fix is an explicit flag, and the rule for which logger wins is "the richer one." The error handler logs *with a stack* and sets the flag; the finish hook honors the flag and stands down:

```js
app.use((err, req, res, next) => {
  logRequestError(err, req); // logs with a stack trace
  req._errorLogged = true;   // tell the finish hook to stay quiet
  res.locals.errorMessage = err.message;
  res.status(err.status || 500).json({ message: err.message });
});
```

So: handler-routed errors log once, with a stack. Direct-response errors log once, via the finish hook. Nothing logs twice. When two paths can both record the same event, don't try to make them mutually exclusive by construction — let the one with more context mark the request and have the other defer.

## Bonus bug: not every caught error is a database error

While tracing this, the logs surfaced a second bug that the first fix made visible. The data-access catch tagged *everything* it caught as `[DB]` — including this:

```
[ERROR] [DB] EBUSY: resource busy or locked, unlink '\\<nas>\storage\document\<file>.docx' {"code":"EBUSY","errno":-4082}
```

That's a filesystem error trying to delete a file on a network share. Not SQL at all — yet it wore a `[DB]` badge and carried no query, which is exactly the kind of thing that sends you debugging the database for an hour when the problem is a locked file handle.

With genuine SQL errors now owned by the Knex listener, the catch block could be narrowed to only the errors it's actually positioned to explain — non-SQL failures — and made to include a stack so you can find where they came from:

```js
const isSqlError = (err) =>
  err.sqlMessage != null || err.sqlState != null || err.sql != null;

const logDbError = (err, context) => {
  if (!err || isSqlError(err)) return; // SQL errors are handled by the Knex listener
  logger.error(`${context ? context + ' - ' : ''}${err.message || err}`, {
    code: err.code,
    errno: err.errno,
    stack: err.stack, // non-SQL errors need a location; the listener can't give one
  });
};
```

A catch-all error logger that assumes a category will eventually mislabel something. Branch on the error's *shape* before you put a tag on it.

## One more, on identity in the access log

The same backend serves an in-browser document editor by redirecting to editor URLs. A browser navigation can't set custom headers, so those requests carry the JWT as a `?token=…` query parameter instead of an `Authorization` header. Two consequences: they bypass the auth middleware that sets `req.user`, so the access log showed `-` for the user, and the full token — hundreds of characters — sat in the logged URL on every request.

The fix decodes the token *for the log line only*, and strips it from the logged URL:

```js
const userIdFromTokenQuery = (req) => {
  const token = req.query && req.query.token;
  if (!token) return null;
  try {
    return jwt.verify(token, process.env.JWT_KEY).id || null;
  } catch {
    return null; // expired or forged → anonymous, never spoofed
  }
};

morgan.token('id', (req) => req.user || userIdFromTokenQuery(req) || '-');
```

`jwt.verify`, not `jwt.decode`: a tampered token degrades to `-` rather than letting someone forge an identity in your logs. A line that used to read `- … /editor/…?token=eyJhbGciOi…<300 chars>` now reads `u10427 … /editor/…` — identified, and compact.

(If you can avoid putting tokens in query strings at all, do — they leak into logs, proxies, and browser history. Here the embedded editor forced it, so the next best thing is to never persist the token to the log.)

## The transferable part

Strip away Express, Knex, and morgan and three rules remain, each one a place where the obvious fix is the wrong layer:

- **Capture data where it still exists, not where you happen to catch the failure.** A swallowed query can't be rebuilt downstream. Subscribe to the driver's error event where the statement is still attached.
- **Log outcomes at the point every path converges.** A `res.on('finish')` hook covers handler-routed *and* direct-response errors uniformly; per-return-site logging never will.
- **When two mechanisms can log the same event, let the richer one win with a flag.** Don't engineer mutual exclusion — mark the request from the stack-trace logger and have the lighter one defer.

The bug report asked for "the raw queries in the log." The actual lesson was that the query was never in the error to begin with — it was one layer up, in an event I wasn't listening to.
