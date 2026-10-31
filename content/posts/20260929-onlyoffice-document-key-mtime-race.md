---
title: "ONLYOFFICE Troubleshooting - The Warning That Was Already the Fix"
date: 2026-09-29
draft: false
tags: ["onlyoffice", "nodejs", "race-condition", "filesystem", "document-server"]
categories: ["Backend"]
description: "Users occasionally saw ONLYOFFICE's 'file version has changed' warning on open. The tempting fix was a unique document key per request — that would have quietly traded a harmless reload for silent data loss."
showToc: true
---

Once in a while, a user opening a document in our self-hosted ONLYOFFICE Docs integration saw: *"The file version has been changed. The page will be reloaded."* Rare, and only ever right after opening. A forum thread on the symptom said the document key has to be regenerated on every edit and save, which made it tempting to just issue a brand-new key on every request. I want to walk through why that would have been the wrong fix, and what the warning actually meant.

## How ONLYOFFICE uses `document.key`

Every time our backend serves the editor page, it passes a `document.key` in the editor config. ONLYOFFICE treats that key as the identity of one *version* of a file, not of one browser session:

- Everyone who opens the same version with the same key joins the same co-editing session.
- Once a version has been saved back to storage, its key is closed and must never be reused.

Our key generator already respected that contract. It combines a storage ID, a file ID, and the file's last-modified time:

```js
const getDocumentKey = (storageId, fileId) => {
  const filePath = resolveFilePath(storageId, fileId);
  try {
    const stat = fs.statSync(filePath);
    return generateRevisionId(`document_${storageId}_${fileId}_${stat.mtime.getTime()}`);
  } catch (_) {
    // File is missing: fall back to a one-off key
    return generateRevisionId(`document_${storageId}_${fileId}_${Date.now()}`);
  }
};
```

`generateRevisionId` shapes the string to ONLYOFFICE's rules for `document.key`: at most 128 characters, using only `0-9 a-z A-Z - . _ =`. Anything too long gets replaced by a 32-bit hash, and any other character becomes `_` — so a filename with non-ASCII characters collapses to a run of underscores. That's harmless here, because the storage ID and `mtime` still keep the key distinct. When the save callback writes the new file, `mtime` changes, and the next open gets a new key. The design was already doing what the forum thread was asking for.

## Where the warning actually comes from

The warning traces back to a short race around saving:

1. User A closes the editor. A few seconds later — typically around ten — the document server finishes assembling the final file and calls our save callback.
2. The callback downloads the new file and renames it into place. Until that rename completes, the old file and its old `mtime` are still on disk.
3. User B opens the document inside that window, gets the old `mtime`, and therefore the old key.
4. The callback returns success, and ONLYOFFICE marks that key as saved and closed.
5. B's editor notices its key belongs to a version that's already been closed, shows the warning, and reloads. The reload fetches a fresh config with the new `mtime` and a new key, and B lands on the current content.

That fits the symptoms exactly: rare, and only during editor startup. The warning is ONLYOFFICE catching its own race condition. B loses nothing — they had only just opened the file and hadn't typed anything yet.

## Why a key generated per request would be worse

It doesn't matter whether the per-request value is a UUID or an HMAC over some nonce. What matters is whether the key identifies a version or a request. An HMAC over `file + mtime` is still one key per version — it just gives you a tidier fixed-length string, with no security benefit, because the key isn't a secret once JWT signing between the editor and the backend is enabled. A key that's actually unique per request has three real costs:

- **Silent data loss.** Someone who opens the file during the save window gets a fresh key. The document server then downloads the *old* file, because the new one isn't written yet. When that user later saves, they overwrite the edits that were just committed. We'd be trading a harmless reload message for lost work.
- **No co-editing.** Two people — or one person in two tabs — end up in separate sessions, and whichever saves last wins silently.
- **No cache reuse.** Every open re-downloads and re-converts the file, which is slower and fills the document server's cache until entries expire.

My own first instinct was to fold the file size into the key too, "for robustness." That's wrong for the same reason: during the race window the size is exactly as stale as the `mtime`, and a copy that preserves the old `mtime` can change the size by nothing more than luck. The only way to genuinely close the window is to block or delay opens while a save is still being written to disk — extra code whose sole purpose is to hide a message that never cost anyone data, so we left it alone.

## Checking `mtime` resolution, since Explorer lies about it

Windows Explorer displays modification times rounded to the minute, which raised a real question: is `stat.mtime.getTime()` actually precise enough to tell two saves apart? A quick loop on NTFS with Node 16, writing the same file five times back to back, answered it:

```js
for (let i = 0; i < 5; i++) {
  fs.writeFileSync(f, "x" + i);
  const s = fs.statSync(f);
  const b = fs.statSync(f, { bigint: true });
  console.log(i, s.mtime.getTime(), b.mtimeNs.toString());
}
```

```
0 1790575877879 1790575877879012400
1 1790575877883 1790575877883058700
2 1790575877884 1790575877884057300
3 1790575877884 1790575877884057300   <- identical to write 2
4 1790575877885 1790575877885058700
```

NTFS stores timestamps in 100-nanosecond ticks, and `getTime()` returns real millisecond values — Explorer's minute display is only a formatting choice. But writes 2 and 3 landed on the exact same tick, because Windows advances its system clock in discrete steps (roughly 1 ms on this machine, up to ~15.6 ms on others), not continuously. For a document key that doesn't matter here, since saves are seconds apart, not milliseconds. It would matter on a filesystem with coarser granularity — FAT-style volumes round to 2 seconds, and I haven't yet checked what the production network share actually gives us, which is the one open item from this investigation.

## Outcome and takeaways

No code changed. The key design was already correct; the occasional warning is expected, self-healing behavior during a narrow save window, not a bug to engineer away.

- A document key identifies a *version*, not a request. Making it unique per request trades a cosmetic warning for co-editing breakage and silent overwrites.
- Before "fixing" a warning message, check whether it's the system's own safety net catching a race it already knows about.
- Before adding a field to a cache key "for robustness," check whether it's actually current during the failure window you're worried about. If it goes stale alongside the field you already have, it buys you nothing.
- A filesystem's timestamp *precision* (how finely it's stored) is not the same as its update *granularity* (how often it actually advances). Measure both on the real filesystem — don't trust what a file browser shows you.
