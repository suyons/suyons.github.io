---
title: "SEO Migration - Redirecting Every Legacy URL to the Homepage Is a Soft 404 in Disguise"
date: 2026-07-26
draft: false
tags: ["seo", "http-redirects", "soft-404", "naver", "curl"]
categories: ["Backend"]
description: "Verifying a clinic's legacy-domain redirect with curl turned up a working 301 and a TLS gap nobody had checked — and confirmed a bigger problem: collapsing 37 indexed URLs onto one homepage looks like duplicate-content cleanup but reads as a soft 404 to search engines, discarding years of ranking signal instead of passing it forward."
showToc: true
---

The mechanics of getting a legacy domain to redirect to its replacement are their own story I've already written up. What that story left open was whether the redirect was actually *good* for search, or just harmless. Every request on the old domain collapses to the new site's homepage — no path, no query string survives. That's the simplest possible fix for duplicate content, and I assumed simplest meant safest. It doesn't. A blanket redirect to an unrelated page is exactly the pattern search engines treat as a soft 404, and a soft 404 doesn't pass along whatever ranking signal the old URLs had built up. It just drops it.

## First: what's actually indexed

Before deciding anything was wrong, I needed to know what search engines still had on file for the old domain. Naver's `site:` operator, paged through its web-results tab, returned 52 listings — with duplicates, since the same URL can surface under more than one query variant. Paging stopped being useful once results started repeating past the fourth page, which is as close to "this is the complete set" as the `site:` operator gets.

String-deduplicated, that's 37 distinct URLs, breaking down as:

- 16 static `.html` clinic-info pages
- 5 board list pages (gallery, reservation, notice, essay — notice appears twice)
- 7 individual notice posts
- 7 individual essay posts
- 2 consultation-board URLs

37 distinct strings isn't the same as 37 distinct documents, though. Two of the "notice list" URLs differ only in a `number=` query parameter that the list-rendering script ignores entirely — same page, indexed twice under different querystrings. One essay's list-view URL and its own article-view URL share a `number`, different scripts, same underlying `number=50` record. And two essay entries carried the *identical* title after translation — not a scraping bug, but a legacy-database quirk: the current site had already resolved that exact collision by renaming one of the two essays to disambiguate it. Worth noting for the obvious reason: if you build a redirect map by matching on title text instead of the stable path/ID, a real collision like this one will silently merge two different documents.

## The redirect works. The TLS story doesn't

With the actual URL list in hand, I verified the existing redirect against every category, not just the homepage:

```
http://old-clinic.example/                                 301 -> https://www.example.com/  -> 200
http://old-clinic.example/overview8.html                    301 -> https://www.example.com/
http://old-clinic.example/clinic2_04.html?cate=clinic2_04   301 -> https://www.example.com/
http://old-clinic.example/sw_board/view.php?...number=70    301 -> https://www.example.com/
http://old-clinic.example (bare apex, no www)                301 -> https://www.example.com/
```

One hop, 301 to 200, apex behaving the same as `www`. That part was already solid.

Then I checked HTTPS, mostly to rule it out:

```
$ curl -vI https://old-clinic.example/ 2>&1 | tail -3
* connect to 192.0.2.65 port 443 failed: Connection refused
* Failed to connect to old-clinic.example port 443
curl: (7) Failed to connect to old-clinic.example port 443: Connection refused
```

DNS resolves fine — there's an A record pointing at a real IP. There's just nothing listening on 443. All 37 indexed URLs are `http://`, so this isn't breaking anything *today*. But it's a live gap: any external link built as `https://` — a browser auto-upgrade, a copy-pasted link with the scheme "corrected" — hits connection refused before a redirect ever gets a chance to run. A redirect only executes after a successful connection; a missing TLS listener fails one layer earlier than anything `.htaccess` can touch.

## The bigger problem: collapsing to root is a soft 404

Here's the part that isn't a bug so much as a wrong assumption. Redirecting all 37 URLs to one homepage does solve duplicate content — Google and Naver won't index the same content twice under two domains. But "many unrelated URLs redirecting to one unrelated page" is the textbook shape of what Google's own documentation calls a soft 404: a response that looks like success (a 301 followed by a 200) but lands somewhere with no topical relationship to what was requested. Google has stated plainly that it treats this pattern as a soft 404 and passes little to no ranking signal through it. Naver doesn't publish an equivalent policy, so treat that half as an informed inference from observed indexing behavior, not a confirmed algorithm detail — but the practical result lines up: none of the accumulated relevance those 37 URLs built up over years is transferring to their modern equivalents. It's just gone.

The fix isn't a different redirect target — it's actually mapping paths.

## Building the real map

The rule I used: a legacy path only gets its own redirect target if a genuinely equivalent page currently exists — verified against the live `sitemap.xml`, not assumed from the old site's structure. Everything else keeps falling through to the homepage catch-all, which is a legitimate answer for "no such content anymore," not a lazy default.

That split three ways:

**Direct equivalents.** Most of the 16 static clinic-info pages map one-to-one onto a current page — same diagnosis or treatment topic, different URL shape. A few of the legacy pages were tab sub-pages of a single condition (`?sub_page=2`, `?sub_page=3`) that the current site merged into one page with conditional sections, so multiple legacy URLs collapse onto one modern URL. That's fine — it's still a real target, just not 1:1.

**No article-level equivalent, but a real fallback exists.** The 14 individual notice and essay posts have no surviving article page at all — the content itself moved off the site to a hosted blog platform, and the article templates were removed at that point. Redirecting those to the homepage would be defensible on its own, but redirecting them to the *board's list page* is strictly better: someone following an old notice link is looking for "notices," and the list page is the closest thing to that intent that still exists.

**No equivalent, period.** The consultation-board URLs (2 of the 37) point at a feature the redesign dropped entirely — no board, no list, nothing structurally similar. For these, the homepage catch-all isn't a compromise, it's genuinely the best available answer, unless there's a contact or booking flow worth routing that intent toward instead.

## The rules

Extending yesterday's single catch-all `.htaccess` with path-specific rules ahead of it:

```apache
RewriteEngine On

# Static clinic pages: direct path-to-path mapping
RewriteRule ^overview8\.html$ https://www.example.com/about/directions [R=301,L]
RewriteRule ^clinic2_01\.html$ https://www.example.com/clinic/digestive/indigestion [R=301,L]
RewriteRule ^clinic2_04\.html$ https://www.example.com/clinic/digestive/reflux-esophagitis [R=301,L]

# Board content: no article-level page survives, route to the board's list view instead
RewriteCond %{QUERY_STRING} cate=notice
RewriteRule ^sw_board/(board|view)\.php$ https://www.example.com/community/notice/ [R=301,L]

RewriteCond %{QUERY_STRING} cate=essay
RewriteRule ^sw_board/(board|view)\.php$ https://www.example.com/community/essay/ [R=301,L]

# Consultation board was dropped entirely - closest surviving equivalent
RewriteCond %{QUERY_STRING} cate=qna
RewriteRule ^sw_board/ https://www.example.com/reserve/ [R=301,L]

# Anything unmatched: the existing catch-all
RewriteRule ^ https://www.example.com/ [R=301,L]
```

Order matters: `mod_rewrite` stops at the first rule that matches and carries `[L]`, so the specific rules have to sit above the catch-all or it eats every request before the path-specific ones get a look. The `RewriteCond` checks work on query string, not path, because both the notice and essay boards run through the same `board.php`/`view.php` scripts and differ only by `cate=`.

## Worth double-checking

Google's soft-404 documentation is public and specific; I'm extending it to Naver by pattern-matching, not because Naver has published the same rule. If you're making the same call, treat the Naver half as a reasonable bet, not a cited fact. And this `.htaccess` file hasn't been deployed yet at time of writing — the mapping logic is verified against the live sitemap, but the file itself hasn't gone through the same FTP upload and post-deploy check as yesterday's catch-all version.

## Takeaways

- A redirect that resolves 301-to-200 can still be the wrong redirect. "Does it work" and "does it preserve ranking signal" are different questions with different verification methods.
- Many-URLs-to-one-unrelated-page is the shape of a soft 404, even when every individual hop is a clean 301. Solving duplicate content and preserving link equity are two separate problems; one fix doesn't automatically cover both.
- Build a path map against what currently exists — the live sitemap — not against what you remember the old site's structure. Content gets deleted, boards get merged, and a stale mental model of "the new site" produces wrong redirect targets.
- When there's no exact equivalent, route to the closest *functional* match (a list page, a contact flow), not reflexively to the homepage. The homepage catch-all should be for genuinely unmatched paths, not a default you stopped thinking about.
- DNS resolving is not the same as a service listening. A missing TLS listener fails before any redirect logic runs, and it won't show up unless you test the scheme you assume nobody uses.
