---
title: "Your File Upload Has No Size Limit"
date: 2026-06-06
draft: false
tags: ["nodejs", "express", "multer", "validation", "uploads", "security"]
categories: ["DevOps"]
description: "A 227 MiB document went through an upload endpoint that was supposed to cap files at 50 MB. The cap existed in the code — it just never ran. Two things conspired: multer's default file-size limit is Infinity, and the guard meant to enforce the cap read a property that doesn't exist yet when it runs."
showToc: true
---

## A file that shouldn't have fit

A client uploaded a 227 MiB `.docx` through a document-management endpoint and it went straight in. The endpoint was supposed to cap uploads at 50 MB. There was code that looked like it enforced that cap. The file went in anyway.

Two separate things had to be true for a quarter-gigabyte file to land in a folder that was, on paper, limited to 50 MB. One is a default nobody thinks to override. The other is a guard that was written, reviewed, merged — and never executed a single time. The second one is the interesting one, but you have to see the first to understand why nothing caught it.

## `express.json({ limit })` does not touch your uploads

The app had this near the top, and everyone pointed at it as the size limit:

```js
app.use(express.json({ limit: "50mb" }));
```

That line is real and it works. It just has nothing to do with file uploads. `express.json` parses `application/json` request bodies. A file upload is `multipart/form-data`, and the JSON body parser ignores it completely — wrong content type, not its job. So the `50mb` there caps the size of a JSON payload and is silent about every byte that arrives as multipart. The limit that governs uploads lives entirely in whatever handles multipart, which here was multer, configured per route.

If you remember one thing: the body-parser limit and the upload limit are different limits in different middleware. Setting one tells you nothing about the other.

## multer's default file-size limit is `Infinity`

Here's the multer setup, lightly cleaned up:

```js
const upload = multer({
  storage: multer.diskStorage({ /* ... */ }),
  limits: { fieldSize: 20 * 1024 * 1024 }, // 20 MB
  fileFilter(req, file, cb) {
    if (file.size > 50 * 1024 * 1024) {       // intended 50 MB cap
      return cb(new Error("File too large"));
    }
    cb(null, true);
  },
});
```

There's a `limits` object. It's even got a megabyte figure in it. So at a glance, limits are configured. They aren't — not for files.

`limits.fieldSize` is the cap on the size of a **non-file text field** in the multipart body — a form field's value. It has nothing to do with the uploaded file's bytes. The key that caps the file is `limits.fileSize`, and it isn't set. When `fileSize` is absent, [multer defaults it to `Infinity`](https://github.com/expressjs/multer#limits). No cap. A 227 MiB file is under `Infinity`, so multer is happy to stream all of it to disk.

This is the same shape as a bug I wrote about on the front end, where `<input type="number">` happily accepts `1e999` and hands you `Infinity`. Different layer, same punchline: the default for "how big is too big" is "there is no too big," and the absence is invisible because nothing errors. You only find it when something large enough shows up.

## The guard ran before the bytes existed

But there *was* a check. `fileFilter` explicitly compares `file.size` against 50 MB. Why didn't it fire?

Because `file.size` is `undefined` every time that function runs.

`fileFilter` is multer's hook for deciding whether to accept a file **before it reads the body**. It runs at the moment the file part's headers are parsed — you get `fieldname`, `originalname`, `encoding`, `mimetype`. You do not get `size`, because not a single byte of the file has been read yet. Size is something you only know *after* you've consumed the stream; the disk-storage engine attaches `file.size` once the file is fully written. At `fileFilter` time it doesn't exist.

So the comparison is `undefined > 52428800`, which is `false`, every time, for every file. The guard accepts everything. It's not a weak check or an off-by-one — it's dead code that type-checks, reads like enforcement in review, and has never once returned `true` for that `if`. The 5 MB file and the 227 MiB file take the exact same path through it.

That's the trap. A missing limit is at least honestly absent. A check that references a field that isn't populated yet *looks* like coverage. It passes code review because the reader supplies the meaning the code doesn't have — "of course `file.size` is the file's size" — without asking whether it's set at that point in the lifecycle.

## The fix: cap in `limits`, gate in `fileFilter`

Split the two responsibilities by what information is available when.

Size is enforced in `limits.fileSize`, because multer is the only thing that can act on size — it's counting the bytes as they stream:

```js
const MAX_FILE_BYTES = 50 * 1024 * 1024; // 50 MiB

const upload = multer({
  storage: multer.diskStorage({ /* ... */ }),
  limits: {
    fieldSize: 20 * 1024 * 1024,
    fileSize: MAX_FILE_BYTES, // the cap that actually fires
  },
  fileFilter(req, file, cb) {
    // Only decide on things known before the body is read.
    const ok = /\.(docx?|pdf|xlsx?)$/i.test(file.originalname);
    cb(ok ? null : new Error("Unsupported file type"), ok);
  },
});
```

`fileFilter` keeps the job it can actually do: rejecting by extension or mimetype, both of which arrive in the headers before the bytes. Anything that depends on the content — size, a virus scan, parsing — cannot live here, because the content isn't here yet.

When `fileSize` is exceeded, multer stops reading and emits a `MulterError` with code `LIMIT_FILE_SIZE`. You have to catch it, or it surfaces as an unhandled error:

```js
app.post("/api/documents/upload", (req, res) => {
  upload.single("file")(req, res, (err) => {
    if (err instanceof multer.MulterError && err.code === "LIMIT_FILE_SIZE") {
      return res.status(413).json({ message: "Each file may be at most 50 MB." });
    }
    if (err) return res.status(400).json({ message: "Upload failed." });
    res.json({ ok: true });
  });
});
```

`413 Payload Too Large` is the honest status here, not a generic 400.

The property that makes this the *right* fix and not just a working one: multer aborts mid-stream. It doesn't buffer all 227 MiB to disk and then measure — it counts as it goes and kills the request the moment the count crosses the line. So the cap is also a defense against someone filling your disk or memory with a deliberately huge upload, which a post-hoc `file.size` check (even one that worked) wouldn't give you, because by the time you can read `file.size` the bytes are already written.

## Two caveats worth knowing

**MB vs MiB.** The localized error message in the original said "max 50 MByte per file," but the constant was `50 * 1024 * 1024` — that's 52,428,800 bytes, which is 50 *mebibytes*, not 50 megabytes (which would be 50,000,000). Nobody's going to file a bug over the 4.86% difference, and I left the label alone. But if your limit ever has to match a number in a contract, a spec, or another system's check, decide which unit you mean and be consistent. `1024`-based math labeled "MB" is the default way to be quietly off.

**Set it per route, and on every upload factory.** multer limits are configured on the multer instance, so if you have several — one for avatars, one for documents, one for bulk imports — each needs its own `fileSize`. The codebase here had three upload configs and the missing limit was missing in all three; fixing one would have left two endpoints wide open. Grep for every `multer(` call, not just the one in the bug report.

## The check to actually run

The thirty-second version for a review: if you use multer, confirm `limits.fileSize` is set, because the default is `Infinity` and nothing warns you. Treat any `fileFilter` that reads `file.size` as a bug on sight — that property is `undefined` at filter time, so the check is dead. And remember that `express.json({ limit })` says nothing about multipart uploads; the upload cap is a different setting in different middleware.

Then upload something absurd. Not a 51 MB file that nudges the boundary — a 200 MB one that no honest user would send. A limit you haven't watched reject something is a limit you're only assuming exists.
