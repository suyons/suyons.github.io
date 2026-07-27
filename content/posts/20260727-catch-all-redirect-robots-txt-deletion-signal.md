---
title: "SEO Troubleshooting - Blocking a Crawler Doesn't Remove a URL, It Preserves It"
date: 2026-07-27
draft: false
tags: ["seo", "robots-txt", "http-redirects", "naver", "web-crawlers"]
categories: ["Backend"]
description: "A legacy domain's indexed URL count dropped from 37 to 6 to 1 in a single day after a removal request, then one URL came back — chasing the bounce-back showed the catch-all redirect that fixed duplicate content also makes it structurally impossible for the origin to ever say a page is gone, only that it moved."
showToc: true
---

Yesterday's fix for a client's legacy domain was a set of path-specific 301s to replace a single catch-all redirect, because collapsing 37 indexed URLs onto one homepage reads as a soft 404 and throws away their ranking signal. The path map wasn't deployed yet. In the meantime I filed a removal request through Naver Search Advisor for the legacy domain and started re-checking the index size daily, mostly to confirm the request was doing anything at all. It was — dramatically. Then one URL came back that had never been on the original list, and tracing that turned out to be the more useful finding of the two.

## The numbers, and the one that shouldn't be there

Same query, same paging method as yesterday, re-run within the same day:

| | First pass | Re-check |
|---|---|---|
| Listings (with duplicates) | 52 | 25 |
| Distinct URLs | **37** | **6** |
| Dropped | — | 32 |
| Remaining | — | 5 |
| **New appearance** | — | **1** |

Paging confirmed this wasn't a fluke: `start=1/16/31/46/61` all returned the same handful of results, meaning the result set was now smaller than one page. A targeted query for one specific legacy page returned zero hits, confirming that page was genuinely out of the index, not just off-screen.

Five of the six survivors were `sw_board/view.php` board entries — a notice page, two board posts, a Q&A entry, an essay. Expected: board-script URLs were always the slowest-moving category. The sixth was `overview3.html`, and that one was new — it hadn't been in yesterday's 37-URL list at all.

A URL appearing *while the total shrinks* only makes sense one way: it was already indexed, just buried below where paging stopped surfacing results. Naver's web-results tab was cutting off around the same point every time I paged it, and `overview3.html` sat just under that cutoff. As the visible count fell, previously-hidden entries floated up into view. The practical consequence: don't treat "0 results from a `site:` search" as proof of a fully clean index. Treat a shrinking count as a process that can still reveal more of what was always there — confirm removal with several consecutive zero-count days, not one.

## Why one page keeps resurfacing

`overview3.html` is the "why choose us" page — no rewritten equivalent existed in the old site's structure, but the new site's sitemap has a direct replacement at `/about/strengths`, so it slots into yesterday's redirect map cleanly. The interesting part wasn't the page. It was why a removal request would ever let an indexed URL come back at all.

I checked what the origin actually returns for a few requests that should behave differently from each other:

```
http://old-clinic.example/robots.txt                          301 -> https://www.example.com/
http://old-clinic.example/robots.txt          (apex, no www)   301 -> https://www.example.com/
http://old-clinic.example/robots.txt          (Yeti UA)        301 -> https://www.example.com/
http://old-clinic.example/this-path-does-not-exist-12345.html  301 -> https://www.example.com/
http://old-clinic.example/overview3.html      (Yeti UA)        301 -> https://www.example.com/
http://old-clinic.example/overview3.html      (Googlebot UA)   301 -> https://www.example.com/
```

Every single one is a 301 to the new site's root. Not just the real pages — `robots.txt` itself, and a path I made up on the spot that has never existed. That's the catch-all doing exactly what it was built to do: collapse everything, unconditionally, to one target.

Two consequences fall out of that, and neither is about the redirect being broken. It works perfectly — that's the problem.

**There is no way to serve `robots.txt` for this host.** A crawler that requests `/robots.txt` and gets redirected to a different domain has no rules file to apply to the *original* host — a robots.txt fetched from `example.com` doesn't govern `old-clinic.example`. My reading is that RFC 9309-following crawlers treat a host with no reachable robots.txt of its own as "no rules, crawl freely," which would mean no robots.txt-based exclusion can ever take effect here regardless of what anyone configures — there's no mechanism left to configure. I want to flag this specific claim rather than assert it as settled: I haven't found an authoritative statement of how Naver's crawler specifically handles a robots.txt fetch that redirects cross-origin, so treat it as a working theory that matched what I observed, not a cited spec detail.

**There is no way for the origin to say a page is gone.** A nonexistent path gets the same 301 as a real one. Every response this host can produce means "moved," never "gone." A crawler doing a scheduled re-check of a suppressed URL doesn't see a 404 or a 410 — it sees the same redirect it always saw, which reads as "still here, still moved," not "confirmed removed." A removal request built on top of that is asking for a temporary suppression that expires the next time the crawler happens to re-verify, because re-verification can only ever return the one answer the origin is capable of giving.

That matches the pattern exactly: a URL count that fell hard within hours, and one entry that came back the same day.

## The fix that would have made it worse

The obvious next move is "block the legacy host in robots.txt, then." Don't. If the crawler is disallowed from fetching a URL it has already indexed, it can't refetch that URL at all — which means it can never see the 301 (or a 410, if one existed) that would tell it the page moved or was removed. The URL doesn't drop out of the index. It gets stuck in it, as a bare URL-only listing with no title or snippet the crawler is permitted to re-derive, because re-deriving anything requires a fetch that robots.txt now forbids. Blocking crawl access to an already-indexed URL doesn't clean the index — it fossilizes whatever's already in it.

## What actually resolves it

The choice is between two signals, and they serve opposite goals:

- **410 Gone**, served directly from the origin for legacy paths (carved out ahead of the catch-all), is the unambiguous "delete this" signal. Fast, no bounce-back, but it discards whatever ranking equity those URLs had built up.
- **Path-specific 301s** — yesterday's map — tell the crawler "this content still exists, here's where," and consolidate the old URL's signal into the new one instead of throwing it away.

With the count already down to one, and that one URL slotting cleanly into the existing map (`overview3.html` → `/about/strengths`), there's no case left for burning the signal with a 410. The right move is to finish deploying yesterday's path map, stop filing further removal requests, and let consolidation run its course — a shrinking `site:` count is the natural result once the crawler sees real redirect targets to consolidate onto instead of "still moved, still to nowhere."

Either path requires the same prerequisite, though: carve `/robots.txt` out of the catch-all so the origin serves an actual file instead of redirecting it away, and make sure the file allows crawling rather than blocking it —

```
User-agent: *
Allow: /
```

— because the crawler has to be able to *see* the 301 (or the 410, if that path is chosen instead) for either signal to register. A robots.txt that blocks a host currently mid-migration doesn't protect it. It just makes sure nothing that host does ever gets noticed.

## Worth double-checking

The RFC 9309 reasoning about a cross-domain robots.txt redirect voiding rule application for the original host is my own inference from observed behavior, not a quoted spec passage or a Naver-published policy — verify against Naver's or Google's current documentation before treating it as settled if this matters for a real migration. Everything else here — the index counts, the curl responses, the URL-only-listing behavior for blocked-but-linked pages — is observed or matches documented crawler behavior, not inferred.

## Takeaways

- A shrinking `site:` count isn't proof of a clean index by itself — a result set that's still shrinking can still reveal previously-hidden URLs as it does. Confirm with consecutive zero-count checks, not one good number.
- A catch-all redirect that sends every path, including `robots.txt` and paths that never existed, to the same target is a redirect with exactly one vocabulary word: "moved." It structurally cannot say "gone," and a removal request built on top of it is only ever a temporary suppression.
- Blocking a crawler from an already-indexed URL doesn't remove it — it prevents the crawler from ever re-deriving that it should be removed. Crawl blocking preserves stale index entries; it doesn't clean them.
- Decide what you're optimizing for before picking a signal: 410 discards ranking equity fast, a path-specific 301 preserves it slowly. They're not two ways to reach the same outcome.
- Whatever signal you choose, the origin has to be reachable enough to emit it — a robots.txt that's itself swallowed by the catch-all blocks the crawler from ever seeing whichever answer you configured.
