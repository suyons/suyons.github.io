---
title: "Nginx Troubleshooting - The CI Runner Heartbeat Behind a 4.59 GB Access Log"
date: 2026-11-28
draft: false
tags: ["nginx", "gitlab-ci", "windows", "powershell", "logging", "log-rotation"]
categories: ["Infrastructure"]
description: "An nginx access log on a shared Windows dev server had grown to 4.59 GB in four months; the obvious suspect accounted for 0.01% of it, and the real cause was a CI runner polling for work every few seconds - which took a map-based filter, a Windows-safe rotation script, and a three-line gitignore trick to actually fix."
showToc: true
---

An nginx access log on a shared Windows development server had grown to 4.59 GB in about four months. The obvious suspect was a flight-data playback app that fetches its data in many small chunks - plausible on its face, and wrong.

## Measure before blaming

The log format recorded neither host nor port, and every site behind the proxy wrote to the same file. So the only honest way to find the source was grouping requests by URL path and summing bytes across all 28.7 million lines in one pass:

| Traffic | Share of the log |
|---|---|
| GitLab Runner job polling (`POST /api/v4/jobs/request`) | 95.91% |
| All other apps and paths | 4.05% |
| The suspect's chunk requests | 0.01% (980 lines) |

A GitLab Runner is the agent that picks up and runs CI jobs, and it keeps asking the server whether there's work. Most of the time the answer is `204 No Content`. A handful of runners polling every few seconds adds up to millions of lines that say nothing happened - about 60 MB a day of pure heartbeat.

## Filtering the boring lines, not the endpoint

Turning off logging for the whole endpoint wasn't an option - a failed poll (a 403, a 502) is worth seeing, only the successful no-op isn't. nginx can make logging conditional with a `map`, keyed on method, path, and status together:

Before:

```nginx
access_log logs/access.log masked;
```

After:

```nginx
map_hash_bucket_size 64;   # must come before the first map block

map "$request_method $uri $status" $loggable {
    "POST /api/v4/jobs/request 204"  0;
    default                          1;
}

access_log logs/access.log masked if=$loggable;
```

The first `nginx -t` failed with `could not build map_hash`: the new map key didn't fit in the default 32-byte hash bucket. Testing before reloading meant the live proxy never saw the broken config - after bumping the bucket size and reloading, about 30 seconds of traffic produced 2 log lines, where the prior rate would have logged over 100 polls in that window. That window was too short to measure the new daily growth rate, and I didn't watch a failed poll or an actual job pickup happen live during the check.

One side effect worth writing down for whoever reads this log next: a missing poll line no longer means the runner went offline. It means the poll succeeded and found nothing.

## Rotating a log a service still has open

There was no rotation in place either. The fix is a PowerShell script scheduled daily under the SYSTEM account - required to signal a service-mode nginx - that renames the current logs, runs `nginx -s reopen` (lighter than a full reload), compresses the renamed files, and deletes archives older than 90 days.

The detail that matters is how it opens a file before compressing it:

```powershell
# FileShare Read makes this fail while any process still has the file open for writing.
$in = [IO.File]::Open($file.FullName, 'Open', 'Read', 'Read')
```

If something still holds the file open for writing, the open throws, the original is left alone, and the next scheduled run tries again. The script also reads the date for cleanup from the file name rather than the last-write timestamp, and refreshes the file size right before checking it - Windows can report a stale cached size for a file something still has open.

That safety check mattered on the very first real run. The rename succeeded and new requests went to a fresh log, but compression on the old one failed. Asking the Windows Restart Manager which process held the file pointed to a single nginx worker that had started twelve days earlier, outlived that day's reload, and was holding roughly 1.3 GB of memory. My guess is it was still waiting on old client connections to close, though I didn't confirm that - a second run later proved the rest of the pipeline on the newer logs the old worker didn't hold: compressed, and decompressible again. A dummy archive dated 2020 was deleted on schedule; a recent one was kept.

## Keeping a server script in a config repo that ignores everything

The config repository ignores everything by default except an explicit allowlist, including the whole folder of server scripts. Git won't look inside an ignored directory, so re-including a single file inside one needs the un-ignore applied at each level:

```gitignore
!/shims/
/shims/*
!/shims/rotate-logs.ps1
```

The other scripts in that folder stay ignored on purpose. The scheduled task itself isn't a file the repo can track, so its settings got written down separately instead.

## What's actually confirmed

The map filter, the rotation script, and the notes are committed and pushed. Still open at the time of writing: the old worker that held the original 4.59 GB log was still running, and I chose not to kill it without approval, since that would cut whatever connections it still had open. The plan is for it to let go and get compressed on the next scheduled run - and since that will land it under the 90-day cleanup window, it needs a different name if the history is worth keeping past that.

## Takeaways

1. Measure before fixing. The obvious suspect accounted for 0.01% of the problem; the actual cause was a CI agent's idle heartbeat.
2. Filter on status as well as path - drop the no-op lines, keep anything that failed.
3. Test a shared proxy's config before reloading it. A bucket-size error caught at `nginx -t` is free; the same error caught after a reload isn't.
4. On Windows, don't assume a log is free to compress just because you renamed it and told the server to reopen its handle. Make the compression step fail safely and retry, rather than assuming success.
5. An old worker can survive a reload and keep an old log file open indefinitely. `worker_shutdown_timeout` bounds how long, at the cost of cutting long-lived connections - left unset here, on purpose, pending approval.
