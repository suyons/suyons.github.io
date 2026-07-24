---
title: "Web Performance - One display:none Was Wrecking Both CLS and LCP"
date: 2026-07-24
draft: false
tags: ["css", "core-web-vitals", "performance-optimization", "lcp", "cls"]
categories: ["Web Development"]
description: "A clinic homepage's Core Web Vitals looked like two separate problems — a layout shift and a slow paint — until both traced back to one CSS rule hiding the hero slider until JavaScript ran."
showToc: true
---

A clinic homepage was showing poor Core Web Vitals in Cloudflare Web Analytics: 11% of Cumulative Layout Shift samples in the "Poor" bucket, and Largest Contentful Paint at 3.5s (P90) and 6.8s (P99). The two metrics looked like separate problems — one a layout bug, one a network/image problem. They turned out to be the same single line of CSS.

## Reading the report

Cloudflare's Web Vitals panel gives you element attribution, which is the useful part. It named:

- **CLS 0.318** attributed to `#section1` — the row of treatment-category icons directly below the homepage hero slider.
- **LCP 6,836ms** attributed to `#slick > div.bg.on`, resolving to the *second* hero slide's image.
- **LCP 3,488ms** attributed to `article.content > figure > picture > img` on an interior condition page.

One detail worth flagging before drawing conclusions: those rows had counts of 1 and 2. On a small business site, a single visitor on a bad mobile connection is 6% of your sample. The percentages are noise; the element attribution is signal. I treated the report as a pointer to *where* to look, not as a measurement to optimize against.

## The hero slider never existed until JavaScript ran

The slider is a hand-rolled crossfade — four stacked `div`s, each carrying its photo as an inline `background` with `image-set()` for AVIF/WebP/JPEG negotiation. Visibility is class-driven:

```css
/* Before */
#slick .bg    { display: none; }
#slick .bg.on { display: block; animation: vfade 1.2s; }
```

```html
<!-- Before -->
<div class="bg" style="background:url(/img/main/main_bg01.jpg) ...">
```

Nothing in the markup carries `on`. The class is applied exclusively by the slider's initializer, which runs from a `defer`red script:

```js
document.addEventListener('DOMContentLoaded', () => {
  autoSlider([...document.querySelectorAll('#slick .bg')], /* ... */, 5000);
});
// ...and inside autoSlider:
show(0);   // the ONLY thing that ever adds `on` to the first slide
```

So between first paint and the deferred script executing, every slide is `display:none` and the hero occupies zero pixels.

To confirm this rather than reason about it, I rendered the live page inside an iframe with scripts blocked — which is exactly the DOM the browser paints before a `defer` script runs:

```js
const frame = document.createElement('iframe');
frame.setAttribute('sandbox', 'allow-same-origin');  // same-origin read, no script execution
frame.src = '/';
```

Measuring inside that frame against the normal page:

| | scripts blocked | after JS |
|---|---|---|
| slider height | 0px | 655px |
| `#section1` top | 90 | 745 |

A 655px downward jolt of the entire page below the header (210px at mobile widths). That is the 0.318, and it explains why `#section1` — not the slider — got the blame: CLS attributes the element that *moved*, and the slider didn't move, it inflated.

The same line explains the LCP. A `background-image` on a `display:none` element is never fetched, so the hero photo wasn't even requested until after `DOMContentLoaded`. And because the slider advances every five seconds, each new slide painted a fresh LCP candidate — which is why the reported LCP resource was the *second* slide's image rather than the first. The metric was still climbing minutes into the page's life.

## Four fixes

**1. Put the first slide's visibility in the markup.** The initializer's `show(0)` performs `remove('on')` then `add('on')` on the same element within a single task, so the browser sees no net change — no flicker, no restarted animation.

```html
<!-- After -->
<div class="bg on" style="background:url(/img/main/main_bg01.jpg) ...">
```

**2. Preload the hero image explicitly.** This is the non-obvious part: the browser's preload scanner reads markup, not computed style. An image referenced from an inline `style` attribute's `image-set()` is invisible to it. Only an explicit hint gets it into flight early:

```html
<link rel="preload" as="image" href="/img/main/main_bg01.avif"
      type="image/avif" fetchpriority="high">
```

Only the AVIF variant is preloaded, deliberately. The `type` attribute makes browsers without AVIF support skip the hint entirely and fall back to the normal path; adding a second WebP preload would cause browsers supporting *both* formats to download both files.

**3. Stop lazy-loading the above-the-fold image.** An earlier performance pass had swept `loading="lazy"` across the site somewhat indiscriminately. Measuring the first content image's position on an interior page:

- Desktop (1280×900): top at 717px
- Mobile (390×844): top at 493px, fully visible

Both are inside the first viewport, and both were lazy. A lazy LCP element is doubly penalized — deferred until layout is computed, then fetched at low priority.

```html
<!-- Before -->
<img alt="..." loading="lazy" src="..." width="1200" height="540">

<!-- After -->
<img fetchpriority="high" alt="..." src="..." width="1200" height="540">
```

Applied to the first content image only, on each of 51 interior pages. The remaining ~209 lazy attributes are genuinely below the fold and were left alone.

**4. Promote the web font from `@import` to `<link>`.** The site loaded two Korean font families through an `@import` at the top of its main stylesheet:

```css
/* Before — in the site's base stylesheet */
@charset "utf-8";
@import url('https://fonts.googleapis.com/css2?family=...&display=swap');
```

```html
<!-- After — in each page's <head> -->
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&amp;display=swap">
```

An `@import` is only discovered after the importing stylesheet has been downloaded *and* parsed, creating a serialized render-blocking chain that a `preconnect` hint cannot shorten — you can warm the connection, but you can't discover the resource any sooner. After the move, both stylesheets begin at the same millisecond instead of one waiting on the other.

## Outcome & takeaways

Verified against production after deploying: the scripts-blocked layout now matches the post-JavaScript layout exactly (655/745 desktop, 210/270 mobile), so the shift is structurally impossible rather than merely smaller. The hero image is fetched with `initiatorType: "link"`, confirming the preload is the request that lands, with no duplicate download. The font stylesheet and the site stylesheet now start simultaneously. Every page in the sitemap returns 200, and several were byte-compared against the repository to confirm the deploy was complete.

Four things worth carrying forward:

- **Test the pre-JavaScript DOM directly.** A sandboxed same-origin iframe with scripts disabled is a precise, deterministic reproduction of what the browser paints before a `defer` script runs. It turns "this might be a hydration-ish layout jump" into two numbers you can diff. Far more reliable than throttling and hoping to catch the shift.
- **CLS blames the element that moved, not the element that caused it.** The report pointed at the icon row; the bug was in the slider above it. Always look *up* the document from the attributed element.
- **An auto-advancing carousel keeps generating LCP candidates.** LCP finalizes on first user interaction, so a rotating hero on a page nobody clicks can report a "load time" of several seconds that has nothing to do with load. If your LCP resource is a later slide, that's the tell.
- **`display: none` defers the fetch, and the preload scanner doesn't read inline styles.** Two separate blind spots that compound viciously when a JS-driven slider hides its own LCP image behind both of them.

One meta-lesson about validating a deploy: my first content check against production came back empty because my own 52-URL sweep, run seconds after publishing, had re-cached the *pre-deploy* HTML at the edge. It resolved on its own moments later, but it's a good reminder that an eager verification sweep can poison the cache it's trying to verify.

Finally, on expectations: this is field data, so the dashboard repopulates over the following day as real visitors arrive. With single-digit sample counts, the right success signal is P75 and P90 trending down over time — not the percentage snapping to 100%.

### Postscript: the optimization I talked myself out of

The obvious next item on the list was self-hosting the Korean font subsets instead of loading them from Google Fonts. It removes a third-party dependency from the critical path, it's the textbook recommendation, and I had already flagged it as the remaining work. Before committing to it I measured what it would actually buy — on a warm-cache load of the live page, since the interesting question is where the font stylesheet sits relative to the element that determines LCP:

| Resource | Start | Done |
|---|---|---|
| Font stylesheet (render-blocking) | 140ms | 157ms |
| `domInteractive` | — | 159ms |
| Hero image (the LCP element) | 140ms | 436ms |
| 20 font subset files | — | 559ms |

The font stylesheet costs about 18ms and finishes roughly 280ms before the LCP element does. It is no longer on the critical path — moving it out of `@import` had already taken the serialization hit off the table, which was the expensive half of the win. Self-hosting would compete for a slice of those 18 milliseconds plus a connection handshake that `preconnect` already warms. The 20 subset files finishing at 559ms look alarming until you remember `display=swap` means they never block rendering at all.

There's also a real downside risk that the textbook advice glosses over for CJK. Google splits Korean into roughly 92 `unicode-range` subsets per family and fetches only the ones a page needs — which is why 20 files load here rather than one. Naive self-hosting means shipping a monolithic Korean font file, which for these families runs to megabytes. Done carelessly it's a straight regression; done properly you are rebuilding Google's subsetting pipeline to win 18ms.

So the honest verdict: self-hosting is a resilience improvement, not a performance one. A render-blocking stylesheet on a third-party origin does mean first render stalls if that origin is slow or blocked for a given visitor — a genuine tail risk, and a fair reason to do the work eventually. It is not a Core Web Vitals lever, and it should not be sold as one.

The decision was to change nothing further and let field data accumulate for a day before touching anything else. Two reasons. First, the images on this page are already AVIF with explicit dimensions, and the hero image that was the LCP element is 14KB — there is no obvious structural lever left. Second, and more importantly, with single-digit sample counts you cannot distinguish a remaining defect from one visitor on a bad connection. Optimizing against that is how you end up doing a font-subsetting project for 18 milliseconds.

The general lesson: "what's the next best practice on the list" and "what is actually slow" are different questions, and only the second one deserves engineering time. Measure the candidate optimization's position on the critical path *before* you build it, not after. A profile that says "this costs 18ms and finishes 280ms before the thing you care about" is a complete argument for not doing the work — and knowing when to stop is as much a result as any of the four fixes above.
