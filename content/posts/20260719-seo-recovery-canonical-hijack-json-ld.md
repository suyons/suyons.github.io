---
title: "SEO Recovery - The Canonical URL Google Picked Wasn't Mine"
date: 2026-07-19
draft: false
tags: ["seo", "structured-data", "json-ld", "google-search-console", "naver"]
categories: ["Backend"]
description: "After an outage tanked a corporate site's search rankings, Search Console's canonical-selection data caught a hijacked product page, and a JSON-LD Product schema with no public price turned into a lesson about not gaming structured data to pass validation."
showToc: true
---

The homepage migration itself — Cloudflare Custom Domains, a stale CNAME, a 522 that turned out to be a `workers.dev` gotcha — is its own story I've already written up. What that migration didn't fix was three days of downtime that Google and Naver had already indexed as a symptom. Rankings dropped, and getting them back turned into a separate, slower project with its own set of surprises.

## Proving to crawlers the site is healthy again, with curl not a browser

The first pass was server-level correctness. Three checks, run with `curl` specifically because browser caching hides the exact redirect behavior being tested:

```bash
curl -sI http://example.com/           # expect 301 -> https
curl -sI https://example.com/          # expect 301 -> www
curl -sI https://www.example.com/      # expect 200
curl -sI https://www.example.com/sitemap.xml  # expect 200
```

Three things were actually broken: HTTP was serving 200 instead of redirecting to HTTPS, the bare apex domain wasn't consolidating onto a single canonical host (`www`), and — separately — Cloudflare's bot-fight mode needed checking, because aggressive bot-protection settings are known to challenge Naver's Yeti crawler along with the malicious traffic they're meant to stop. A "healthy" site that quietly CAPTCHAs half its search engines isn't healthy.

## The canonical URL Google picked wasn't mine

With the redirects fixed, Search Console turned up something I wouldn't have found by reading the page's own code: one product page had been canonical-hijacked during the outage window. Google's crawler had hit outage garbage on that URL and, in the confusion, selected a completely unrelated third-party domain as the canonical URL for it. That single misattribution explained a disproportionate drop in that page's search impressions — Google wasn't ranking my page under my URL at all.

The fix isn't code. It's requesting recrawl and waiting for Google to re-evaluate. But the diagnosis came from one specific place: the URL Inspection tool's "Google-selected canonical" field. It's worth checking any time a page's traffic craters with nothing wrong visible in the page itself — the problem can live entirely in what the crawler decided about the URL, not in the URL.

## A JSON-LD fix that would've been easier to fake

Search Console also started surfacing structured-data errors that hadn't shown up before, likely because pages weren't being actively re-crawled during the outage. The first was a product snippet and merchant-listing error on a shielded test-enclosure product page — quote-based B2B hardware with no public price. The JSON-LD looked like this:

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "RF Shielded Test Enclosure",
  "offers": {
    "@type": "Offer",
    "priceCurrency": "USD"
  }
}
```

`Offer` without a `price` fails validation — reasonably, since the whole point of an `Offer` is a price. My first attempt was to strip the `offers` block and keep `Product`:

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "RF Shielded Test Enclosure"
}
```

Still invalid, from a different angle: Google's structured-data guidelines require a `Product` to carry at least one of `offers`, `review`, or `aggregateRating`. Remove the only one you have and you fail the same check for a different reason.

The tempting shortcut was a placeholder `price: 0` — that would pass validation and I considered it for about a minute. I rejected it because publishing a fabricated price on a real product listing carries manual-action risk from Google; it's misrepresenting the product to satisfy a schema, not describing it. The actual fix was to drop the `Product` JSON-LD block entirely from every quote-based page and keep only `BreadcrumbList` and `Organization` markup, neither of which carries a pricing requirement:

```json
[
  {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": [
      { "@type": "ListItem", "position": 1, "name": "Home", "item": "https://example.com/" },
      { "@type": "ListItem", "position": 2, "name": "RF Shielded Test Enclosure" }
    ]
  },
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Example Corp",
    "url": "https://example.com"
  }
]
```

Less rich-snippet eligibility, but a valid, honest one. If the product ever gets a public price list, `Product` and `offers` go back in — the fix was about matching the schema to reality, not permanently giving up the snippet.

The other recurring structured-data issue was narrower and Naver-specific: its URL Inspection tool enforces a 40-character limit on `<title>` and an 80-character limit on meta and Open Graph descriptions, flagging every page over. All pages got rewritten to fit, and it's now a standing constraint for anything new.

None of this shipped straight to production. The team's release rule — change, self-validate in a real browser in both supported languages, commit to a working branch, wait for the site owner's manual sign-off, only then deploy — exists because of an earlier incident where a CSS class reused on a new page silently turned a feature checklist into white-on-white invisible text. Syntactically fine, visually broken, and no linter would have caught it. Structured-data changes are low-risk by comparison, but they went through the same gate anyway.

## Forcing the issue: 19 URLs, by hand, against a deadline

The automated fixes address root causes, but none of them force a search engine to re-evaluate a page on any particular timeline — and there was a real deadline: prospective clients who'd been met in person expected to look the company up within days. Waiting for Google and Naver's own recrawl cadence wasn't good enough.

So it was manual: Google Search Console's URL Inspection tool and Naver's Search Advisor, one page at a time, for all 19 URLs in the sitemap — not just the one confirmed canonical-hijack victim. For each: run the inspection, confirm current indexing/canonical status, submit an explicit recrawl request. Neither tool has a "recheck my whole sitemap" button. Covering all 19 instead of just the known-bad one was deliberate — the outage could plausibly have left similar crawl artifacts on pages that hadn't shown a visible symptom yet, and a full sweep costs time, not risk.

## Where it stands

Redirect and crawler-access fixes are curl-verified and done. Structured-data cleanup is mostly shipped, pending final pages. Recrawl requests are submitted for all 19 URLs on both engines; actual re-indexing is out of my hands from here — it depends on each engine's own crawl queue, and the signal to watch for is the canonical URL and indexing status flipping back to correct on the next inspection pass.

## Takeaways

- A traffic drop with no visible cause on the page itself might not be about the page — check the URL Inspection tool's "Google-selected canonical" field before you go looking for anything else.
- Structured-data requirements and real-world facts sometimes conflict — no public price for a quote-based product is a real case, not an edge case. The fix is removing the schema block that doesn't apply, not inventing a value that passes validation and misrepresents the product.
- Automated checks don't force a recrawl timeline. When there's a real deadline, manually working every URL through Search Console and Search Advisor is slower than waiting but it's the only lever that actually moves the timeline.
- Cover every URL in the sitemap when you suspect crawl damage, not just the one you've already confirmed. An outage doesn't leave symptoms on a predictable schedule.
