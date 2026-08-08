---
title: "ONLYOFFICE Integration Troubleshooting - When 'Saved Successfully' Means the Document Is Gone"
date: 2026-08-08
draft: false
tags: ["nodejs", "onlyoffice", "integration", "debugging", "security"]
categories: ["Backend"]
description: "A document-management integration was overwriting real documents with 404 error pages and reporting success — deleting its own last-resort backups in the process. The download bug, the network red herring, and the missing signature check that let it happen."
showToc: true
---

Users of a document management system started seeing a dialog they had never seen before: *"This file is opened from a server backup copy and is not saved to storage. Use the 'Save as...' option to save it to a drive."* Behind that message sat a month of quietly lost edits, a network rule nobody knew was there, and an integration bug that turned every failure into a reported success. This is how the whole chain came apart, and what fixed it.

## The architecture, and where it broke

The setup is a common one: a self-hosted collaborative document server (ONLYOFFICE Docs) sits alongside a business application. The application hands the editor a URL to download a document from; when editing finishes, the document server calls back to the application with a URL where the edited file can be fetched. The application downloads it and writes it to storage.

That callback is the fragile part. If it fails, the document server retries with backoff, and if the retries are exhausted it drops the file into a "forgotten" store on its own disk — a last-resort backup. The next time someone opens that document, the editor serves the forgotten copy and shows the backup-copy warning. So the dialog is not the bug; it's the symptom of a callback that failed several attempts ago.

## Reading the logs, and a gotcha that cost time

The document server's converter log showed the download side failing with HTTP 404, complete with stack traces. But the most recent entries were mysteriously truncated mid-string, with no error code at all. The cause turned out to be the logging configuration: a pattern of `%.10000m` caps each message at ten thousand characters. Because the download URLs carry a signed token in the query string, and one tenant's token was nearly ten kilobytes on its own, the actual error code was being sliced off the end of every line.

**Takeaway:** when a log line ends mid-token, suspect a formatter limit before you suspect the application. And if your URLs carry large tokens, your log truncation limit is effectively a limit on how much of the *error* you get to see.

## A network layer that lied convincingly

The callback side failed differently: `connect ETIMEDOUT`. The application server simply could not reach the document server.

The first instinct was that the document server's inbound rules were too strict. Counting distinct source addresses in the web server's access log turned up 799 of them, including internet-wide scanners from several clouds — which looked like proof that the port was wide open and the block had to be somewhere else.

That inference was wrong, and the error is worth dwelling on. The access log combines two listeners. Of those 799 addresses, the overwhelming majority had only ever reached **port 80** and received a redirect. Filtering to requests that could only have arrived over TLS cut the list to 29. The "wide open" conclusion had been drawn from traffic on the wrong port.

What settled it was a two-line test from the affected server:

```
Test-NetConnection -ComputerName <doc-server> -Port 80    →  TcpTestSucceeded : True
Test-NetConnection -ComputerName <doc-server> -Port 443   →  TcpTestSucceeded : False
```

Same source, same destination, seconds apart, opposite results. That is not routing and not a firewall on the host — it is a port-specific rule. Since the cloud provider's server-level security groups are allow-only and therefore cannot express "block this source," the culprit had to be a subnet-level network ACL, which does support explicit denies. Adding the right allow rule fixed connectivity immediately.

**Takeaway:** probe two ports before concluding anything about reachability. One reachable port and one unreachable port from the same source is a far stronger signal than any amount of log archaeology. And be careful drawing conclusions from aggregate log counts when the log mixes listeners.

## The real bug: a download that never checked its status code

With the network fixed, saves still didn't work — and the way they failed was much worse than failing.

The application's download helper looked like this:

**Before**

```js
const downloadToBuffer = (url) =>
  new Promise((resolve, reject) => {
    const client = url.startsWith("https") ? https : http;
    const chunks = [];
    client
      .get(url, (res) => {
        res.on("data", (chunk) => chunks.push(chunk));
        res.on("end", () => resolve(Buffer.concat(chunks))); // no status check
        res.on("error", reject);
      })
      .on("error", reject);
  });
```

Node's `http.get` does not treat a 4xx or 5xx as an error. The callback fires, the body streams, and `end` resolves the promise — with the *error page* as the buffer. An expired download link returns `410 Gone` with a short body; a bad signature returns `403` with another. Both resolved successfully.

The caller then did this:

```js
fs.renameSync(currentPath, path.join(versionDir, `prev${ext}`)); // real document → history
fs.writeFileSync(finalPath, data); // error page → document
```

So a failed download moved the real document into version history and wrote a 140-byte error page in its place — then answered the document server with `{"error": 0}`, which means *saved successfully*. On receiving that, the document server considers the work committed, closes the session, and **deletes its forgotten-store backup**. Every safety net removed, silently, with no log line anywhere.

**After**

```js
const DOWNLOAD_TIMEOUT_MS = 15000;

const downloadToBuffer = (url) =>
  new Promise((resolve, reject) => {
    const client = url.startsWith("https") ? https : http;
    const request = client.get(url, { timeout: DOWNLOAD_TIMEOUT_MS }, (res) => {
      if (res.statusCode !== 200) {
        res.resume(); // drain so the socket is released
        return reject(new Error(`download failed: HTTP ${res.statusCode}`));
      }
      const chunks = [];
      res.on("data", (chunk) => chunks.push(chunk));
      res.on("end", () => {
        const buffer = Buffer.concat(chunks);
        const expected = Number(res.headers["content-length"]);
        if (Number.isFinite(expected) && buffer.length !== expected)
          return reject(
            new Error(`truncated: ${buffer.length}/${expected} bytes`),
          );
        if (buffer.length === 0) return reject(new Error("empty response"));
        resolve(buffer);
      });
      res.on("aborted", () => reject(new Error("response aborted")));
      res.on("error", reject);
    });
    request.on("timeout", () => request.destroy(new Error("download timeout")));
    request.on("error", reject);
  });
```

Four guards where there were none: status code, content length, empty body, and a timeout. That last one mattered more than expected — without it the helper waited on the operating system's default, roughly 37 seconds, which consumed most of the document server's retry budget before the first retry even began.

Once this rejects properly, the existing `catch` returns a non-200, which is exactly what the document server needs in order to retry and, failing that, preserve the file.

## Two more problems in the same handler

**The callback accepted unsigned requests.** The other routes verified an application token, but the save callback verified nothing. An empty, unsigned POST returned `200`. Anyone who could reach the endpoint could overwrite any document in storage. The fix is a small middleware — with the wrinkle that the callback is signed with the *document server's* shared secret, not the application's own token key:

```js
const verifyCallbackToken = (req, res, next) => {
  const header = req.header("authorization") || "";
  const token = header.startsWith("Bearer ")
    ? header.slice(7)
    : req.body?.token;
  if (!token)
    return res.status(401).json({ error: 1, message: "missing token" });
  try {
    jwt.verify(token, DOC_SERVER_JWT_SECRET, { algorithms: ["HS256"] });
    return next();
  } catch (_) {
    return res.status(401).json({ error: 1, message: "invalid token" });
  }
};
```

**The save destroyed before it wrote.** The original moved the live document into history *first*, then wrote the new one. Any failure in between left nothing at the real path. Writing to a temporary file first and swapping afterwards keeps the original in place if anything goes wrong:

```js
const tmpPath = `${finalPath}.tmp-${process.pid}`;
fs.writeFileSync(tmpPath, data);
try {
  fs.renameSync(currentPath, path.join(versionDir, `prev${ext}`));
  fs.renameSync(tmpPath, finalPath);
} catch (error) {
  if (fs.existsSync(tmpPath)) fs.unlinkSync(tmpPath);
  throw error;
}
```

## A mistake worth publishing

While diagnosing, I replayed a real save callback against the running application to capture the response. It returned `200 {"error": 0}` and I nearly recorded that as success — but the download URL I had reused was already expired. Checking the web server's access log showed what actually happened: the application had received a `410`, written the error page over a real document, and reported success. I then ran a second "safe" probe with a deliberately invalid URL, describing it as one where no write could occur. That was wrong for the same reason, and it corrupted the same file a second time.

Two lessons. First, **never trust the integration's own success response when the integration's error handling is what you're investigating** — verify against an independent observer, which here was the access log on the other side. Second, a probe is only non-destructive if you have *read the code path* it will take. I had inferred it.

The damage was recoverable — the original sat in version history — and the file happened to be a disposable test document. That was luck, not design.

## Verifying across a load-balanced pair

The first verification run against the fixed code reported two failures out of six cases. Both failing responses were exactly what the *old* code produced. Repeating a single probe a dozen times explained it:

```
503 401 503 401 401 503 503 401 401 503 401 503
```

Two application servers sit behind a load balancer, and the rolling deploy had only reached one of them. The test suite had no server affinity, so each case landed on whichever host answered. Once both hosts reported the new behaviour consistently, all six cases passed.

**Takeaway:** when validating a deploy behind a load balancer, first establish that *every* instance is running the new build — repeat one cheap probe until the responses are uniform — and only then run the real assertions. Otherwise a passing suite means nothing and a failing suite sends you chasing ghosts.

## Outcome and takeaways

Connectivity was restored, the handler fixes shipped to a test environment and then to production, and both were verified with the same six-case suite: unsigned requests rejected, wrong-signature requests rejected, missing-URL saves failing loudly, and — the case that mattered — a save whose download returns `410` now answering with an error naming the refused status instead of a cheerful `{"error": 0}`. A real user edit was confirmed end to end: a clean `200` on the download, no errors in the logs, and the document server releasing its backup copy on its own.

An audit of the forgotten store found stranded editing sessions going back a month. Attributing each one to its originating tenant — the document server is shared across several — narrowed a scary-looking pile down to three documents that genuinely belonged to the production system, each verified as still stale by comparing the stored file's modification time against the time its editing session began.

One issue remains open by design: the editor saves legacy binary formats in their modern equivalents, so an old-format file edited through the browser now lands as a modern-format payload under its original extension. Previously the save failed before reaching storage, so nobody noticed. The decision was to preserve original formats, which means adding a conversion step before writing.

The lesson that generalises furthest is about the shape of the failure rather than any single bug. An integration that reports success when it failed is more dangerous than one that fails loudly, because every downstream safety mechanism — retries, backups, alerts — is keyed off the failure signal. Suppress that signal and you don't get a degraded system; you get a system that deletes its own evidence. Return "success" only once the bytes are durably stored, and let everything else be an error.
