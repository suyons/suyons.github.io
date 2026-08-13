---
title: "Node.js Troubleshooting - The Upload That Hung Forever, and Why the File Was Never the Problem"
date: 2026-08-13
draft: false
tags: ["nodejs", "multer", "busboy", "http", "debugging"]
categories: ["Backend"]
description: "A multipart upload hung indefinitely for one user and only one user. The file was never the cause — an oversized authorization header was silently stalling the parser, with a parser-library upgrade trap and a fall-through hang waiting on the other side."
showToc: true
---

A user reported that uploading one particular Word document to a workflow-management endpoint timed out after 30 seconds, while a different document uploaded fine. The obvious reading — "this file is broken" — turned out to be completely wrong, and chasing it would have wasted the day. The real cause was that large HTTP request headers can stall an old multipart parser mid-body, leaving the request hanging with no response, no error, and no log line anywhere.

## Starting from the wrong premise

The bug report came with two files: one that "always fails" and one that "always works." The tempting move is to diff the two files. Instead, the first useful step was reading the server's own access log, which had a telling detail: the failing requests were recorded with a status of `-`. The logging middleware emits `-` when response headers were never sent. So the server had not merely errored — it had never answered at all.

Two more facts came from the logs and the filesystem:

- The uploaded file was written to disk **at its full, correct size** every time, byte-identical to the source.
- The database row the request was supposed to update was **unchanged**.

So the upload was parsed far enough to write the file, and then the request evaporated.

The 30 seconds turned out to be the front end's own client-side timeout, not a server limit. One request that was left alone longer hit the reverse proxy's 300-second upstream read timeout and returned a gateway timeout — proving the server was hanging indefinitely, not for 30 seconds.

Then the premise collapsed entirely. Searching the historical logs showed the "broken" file had uploaded **successfully** on an earlier day, for two different users, minutes apart. And comparing request sizes across all recorded failures and successes showed them interleaving: one request of 178,448 bytes succeeded while one of 178,487 bytes hung; a 238,726-byte request succeeded while a 202,786-byte one hung. Neither the file nor the payload size explained anything.

## Eliminating the middle of the stack

With the premise gone, the remaining suspects were the proxy, the parser, the database, and the transport. Rather than guess, each was tested directly:

- **Reverse proxy truncation.** Holding a partial upload open produced _no file at all_ on the app side, and a body that cleanly ended short of its declared `Content-Length` was rejected outright. With request buffering enabled, the proxy never forwards a partial body — so whenever a file appeared on disk, the app had received the complete body.
- **Transport.** Three hundred requests carrying the exact failing payload all returned success, over both HTTP/1.1 and HTTP/2, including deliberately paced small writes.
- **Parser abort paths.** Every rejection path in the upload middleware — bad extension, missing extension, too many files, oversized field — returned a proper error response.
- **Chunk boundaries.** Feeding the same body split at 121 different byte offsets, including every position inside the multipart boundary markers, parsed cleanly every time.

Everything passed. That is the frustrating middle of a debugging session: a growing list of things that are _not_ the cause, and no reproduction.

## Instrumenting instead of guessing

The breakthrough came from adding temporary instrumentation rather than forming another hypothesis. The key measurements were deliberately chosen to split the remaining possibilities in half:

- Compare the request's declared `Content-Length` against how many bytes the socket had actually read.
- Record whether the request stream ever emitted `end`, `aborted`, or `close`.

One important detail: you must **not** attach a `data` listener to the request to count bytes. Doing so switches the stream into flowing mode and breaks the upload middleware you are trying to observe. The socket's own cumulative `bytesRead` counter gives the same answer for free.

When the user reproduced the failure, the log said everything at once:

```
multer start   contentLength=180877  socketBytesRead=65536   marks=[]
STILL RUNNING after 5000ms   socketBytesRead=200585  marks=[]
STILL RUNNING after 15000ms  socketBytesRead=200585  marks=[]
STILL RUNNING after 60000ms  socketBytesRead=200585  marks=[req:aborted@29997ms, req:close@29997ms]
```

The socket had read _more_ bytes than the body was long — the whole request had arrived. Yet the stream never emitted `end`, and the parser's callback never fired. The parser had simply stopped consuming the request.

That surplus was the clue that cracked it. The difference between bytes read and body length was about 19.7 KB — the request headers. This application carries an authorization token containing a role array, which makes every request header block enormous. Large enough, it turns out, to matter: the deployment already ran with a raised maximum header size, and the proxy with enlarged header buffers, precisely because of this token.

## The root cause

The upload library in use (multer 1.4.4, built on busboy 0.2.x) stalls while parsing a multipart body when the request headers are large. The parser stops draining the request stream, so the stream never ends, the middleware never calls `next()`, the route handler never runs, and the framework never responds.

The threshold measured on this runtime, with the real payload, was sharp: headers up to about 19,300 bytes parsed fine, and from about 19,400 bytes upward the request hung. The real token produced 19,708 bytes of headers — barely over the line.

Header size and body size interact, which is what produced the misleading symptom. With the real token in place:

```
11 KB attachment, 114,383-byte body  ->  200 OK
74 KB attachment, 177,098-byte body  ->  hangs forever
same two bodies with a 1 KB header   ->  both 200 OK
```

That single table explains the entire bug report. The "broken" file was only broken for _this user_, because their token pushed the header block over the threshold; colleagues with smaller role arrays uploaded the same file without trouble. And every synthetic reproduction attempt had passed because the test token was short.

## The fix, and the trap that came with it

Upgrading the upload library to the maintained 1.4.x LTS release (which uses busboy 1.x) fixed it outright: a sweep of header sizes from 17,000 to 21,900 bytes passed completely, including the exact real-world value.

The upgrade carries a trap that is easy to ship by accident. busboy 1.x decodes filename parameters as latin1 rather than UTF-8, so non-ASCII filenames arrive mangled — and this application stores the original filename in the database and displays it. A non-ASCII filename came through as mojibake.

The correction is one function, applied in the single place every upload route already funnels through:

**Before** — filenames arrived correctly decoded, so nothing was needed:

```js
function withCreatedWho(multipartUpload) {
  return function (req, res, next) {
    multipartUpload(req, res, function (err) {
      if (err) return next(err);
      // ...
      next();
    });
  };
}
```

**After** — restore UTF-8 from the latin1 bytes the new parser hands back:

```js
/* busboy 1.x decodes the filename parameter as latin1. Non-ASCII filenames would be
   stored mangled, so convert them back to UTF-8 here. No-op for ASCII names. */
function decodeOriginalname(file) {
  if (file && file.originalname) {
    file.originalname = Buffer.from(file.originalname, "latin1").toString(
      "utf8",
    );
  }
}

function withCreatedWho(multipartUpload) {
  return function (req, res, next) {
    multipartUpload(req, res, function (err) {
      if (err) return next(err);

      if (req.files) [].concat(req.files).forEach(decodeOriginalname);
      decodeOriginalname(req.file);
      // ...
      next();
    });
  };
}
```

`Buffer.from(name, 'latin1').toString('utf8')` restores the original exactly and is a no-op on ASCII names — both verified before shipping.

The investigation also surfaced an unrelated latent bug in the same handler: a branch with no `else`, which fell off the end of the function without ever answering. Any request reaching it would hang exactly like the parser bug, with no log line at all.

**Before:**

```js
if (regIdQuery) {
  // ... insert detail rows, respond in every inner branch
}
// falls through here with no response at all
```

**After:**

```js
if (regIdQuery) {
  // ... unchanged
}

// If the lookup came back empty we used to fall through and hang. Always answer.
if (fileIdList.length > 0) await deleteUploadedFiles(fileIdList);
return res
  .status(501)
  .json({
    status: false,
    message: "Could not re-read the registered process.",
  });
```

## Outcome and takeaways

The fix is deployed and confirmed in production, not just in a test harness: the user's retry of the byte-identical request that had hung twice that morning returned 200. The continuous integration pipeline ran the full test suite, rebuilt, and restarted the service cleanly. All temporary instrumentation was removed; the `else`-branch fix was kept.

Lessons worth carrying:

- **Distrust the reported correlation.** "It fails with file A and works with file B" was true and completely misleading. Two variables were confounded; only the log history showed file A had succeeded before.
- **A status of `-` is information.** It means headers were never sent, which is a fundamentally different failure from a 500, and it immediately rules out entire categories of cause.
- **Check whether the event loop is alive.** During a 300-second hang, the server happily served a dozen other requests. That single observation eliminated every CPU-bound explanation and pointed at an unsettled promise or an uncalled callback.
- **Instrument to bisect, not to confirm.** The winning measurements — bytes read versus body length, and whether the stream ended — were chosen because each possible answer eliminated half the remaining suspects. Two rounds of that beat a day of hypotheses.
- **Request headers are part of the payload.** Authorization tokens that carry permission structures grow without anyone noticing, and they can push a request past limits nobody associates with headers. If a deployment needs raised header limits to function, that is a standing risk, not a solved problem.
- **Scope cleanup commands to what you created.** Clearing test artifacts with a time-window match on a live upload directory caught a real user's file. It was restorable, but the correct approach is to delete the specific names you generated, never a pattern that a live system can also match.
