---
title: "CSS Troubleshooting - Five Bugs a Legacy Stylesheet Had Been Hiding"
date: 2026-07-20
draft: false
tags: ["css", "responsive-design", "specificity", "legacy-code", "cloudflare-workers"]
categories: ["Web Development"]
description: "Migrating a small clinic's site from PHP + jQuery to static HTML/CSS/JS on Cloudflare Workers Static Assets and reusing the old stylesheet verbatim turned up five bugs that had been silently broken in production for years."
showToc: true
---

A small medical clinic's website went from a PHP + jQuery codebase to a static site of plain HTML, CSS and vanilla JavaScript, deployed on Cloudflare Workers Static Assets. The interesting part wasn't the migration itself — copying content is grunt work — it was what turned up once the legacy stylesheets got reused verbatim on a modern browser. Several of them had been broken in production for years, invisibly. Five of those bugs are worth writing up, because each one generalizes past this particular site.

## The webfonts had been dead the whole time

The legacy CSS loaded its Korean body font, Noto Sans KR, through five hand-written `@font-face` blocks pointing at `fonts.gstatic.com/ea/notosanskr/v2/…` — Google's old "early access" endpoint. That endpoint has been retired. Every one of those requests returned 404 or 503, fifteen failures per page load, and the browser silently fell back to `Malgun Gothic` / `Dotum` / generic `sans-serif`. The site had been rendering in a different typeface on every operating system for years, and nobody noticed, because a fallback font is still *a* font.

**Before**

```css
@import url(//fonts.googleapis.com/earlyaccess/nanummyeongjo.css);

@font-face {
  font-family: "Noto Sans KR";
  font-weight: 300;
  src:
    url(//fonts.gstatic.com/ea/notosanskr/v2/NotoSansKR-Light.woff2)
      format("woff2"),
    …;
}
/* …four more blocks for 400 / 500 / 600 / 900 */
```

**After**

```css
@import url("https://fonts.googleapis.com/css2?family=Nanum+Myeongjo&family=Noto+Sans+KR:wght@100..900&display=swap");
```

The trap here is worth stating plainly: **deleting the dead blocks was mandatory, not tidiness.** They sat *after* the `@import` in source order, so their broken faces for weights 300–900 would have overridden the working ones from the new import. Swapping the URL alone would have produced a no-op fix that looks correct in the diff.

Verifying it also needed care. "The network tab shows 200s" only proves a file downloaded, not that the text is painted with it. The check that actually settles the question is measuring rendered text width on a canvas: body copy measured 152px with Noto Sans KR loaded, versus 165px for the Malgun fallback and 146px for generic sans. A number that differs per font is a test; a green network panel is not.

## A percentage margin is not a margin on a phone

The layout wrapper used `width: 94%` at every viewport below 1260px. That reads fine on a laptop — 36px of gutter at 1200px — but a percentage shrinks with the viewport. At 390px it left 12px. At 360px, 11px. On a real phone the text looked like it was touching the screen edge, because effectively it was.

```css
/* Before — gutter scales with the viewport */
.res_wrap {
  width: 94%;
}

/* After — inside the ≤768px media query */
.res_wrap {
  width: 100%;
  padding: 0 20px;
}
```

That change alone produced an *asymmetric* 0-left / 20-right gutter on two pages. The cause was specificity: both the main container and the footer wrapper carried `padding-left: 0` from **id** selectors elsewhere in the cascade, and an id beats a class no matter how late the class rule appears. The fix has to match specificity, not just come later:

```css
#container,
#footer_wrap {
  padding-left: 20px;
}
```

There's a measurement gotcha buried in this one too. The browser automation available for this work silently ignored window-resize commands — it reported success while the viewport width never actually changed. Media queries were instead exercised by loading the page inside a same-origin iframe whose width was set directly, since media queries evaluate against the iframe's own layout viewport. Chrome also caches stylesheets for iframe subresources aggressively enough that a hard reload isn't sufficient; the dev server had to be restarted on a different port to force a refetch. If your responsive verification "passes" suspiciously easily, confirm the viewport actually changed.

## Nesting a layout wrapper inside itself doubles the inset

The hero slider was full-bleed — its background spanned edge to edge — while every section below it sat inside a 1200px content column. The image edges never lined up with the cards underneath.

The lazy fix is to hand the hero the same width math. The better fix is to make the hero *be* the wrapper, since that class already encodes the entire gutter contract across three breakpoints:

```html
<!-- Before -->
<div id="main_visual">
  <div class="res_wrap">
    <div class="typo">…</div>
  </div>
</div>

<!-- After -->
<div id="main_visual" class="res_wrap">
  <div class="typo">…</div>
</div>
```

The per-slide wrapper had to be **deleted**, not left alone. A `.res_wrap` inside a `.res_wrap` applies 94% of 94% — an 88% content column below 1260px, visibly narrower than everything else on the page. Because the text block is absolutely positioned and the hero was already `position: relative`, removing the wrapper simply re-anchored it one level up; nothing else depended on those divs.

The slider dots turned out to be a pre-existing bug that surfaced as collateral damage:

```css
/* Before — geometry hard-coded to a full-bleed hero */
#main_visual ul.slick-dots {
  left: 50%;
  width: 1200px;
  margin-left: -600px;
}

/* After */
#main_visual ul.slick-dots {
  left: 60px;
}
```

The old rule centered a fixed 1200px strip on the viewport. At a 1000px viewport that put the dots at -100px — clipped off the left edge — which had been true long before this change. Re-anchoring the hero would have shifted them again either way. One nice property of absolute positioning here: below 768px the wrapper switches from a width to horizontal padding, and absolute offsets resolve against the *padding box*, so `left` stays measured from the visible image edge at every breakpoint without a special case.

## Four buttons is where inline-block layout falls apart

The clinic consolidated its contact channels to exactly four. The home page card holding them styled the buttons as `inline-block` pills with margins, letting them wrap naturally. With three items that had been fine. With four it broke differently at nearly every width: 2+2 at 1400px, 1+1+1+1 at 900px (a media band makes that card only ~200px wide), all four in a row at 768px, and 3+1 at 430px — which is what the client actually saw on their phone.

```css
/* Before */
.btns a {
  display: inline-block;
  padding: 0 18px;
  margin: 0 8px 0 0;
}

/* After */
.btns {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.btns a {
  display: block;
  text-align: center;
  padding: 0;
  margin: 0;
  white-space: nowrap;
}
```

The subtle part is that the pills had to *lose* their own padding and margin. Under a grid the column decides the width, so horizontal padding no longer creates space — it eats into the available text width, and at a 200px card the old 14–18px side padding clipped the longest label. Centering the text does the same visual job with none of the width cost. A narrow-viewport override for these buttons was also deleted rather than adjusted: its font size duplicated the base rule and its padding and margin were exactly what the grid now replaced. Dead code that *looks* live is worth grepping for after any layout-model change.

Verification was mechanical: at eleven widths from 1400px down to 360px, assert the grid is 2+2 and that no pill has `scrollWidth > clientWidth`. That comparison is the cheapest reliable "is this text clipped?" check in the browser.

## Fixed table layout, hidden columns, and the colgroup that outlives its cells

The post-list tables use `table-layout: fixed` so long titles truncate with an ellipsis instead of stretching the column. Below 640px the leftmost "number" column is hidden with `display: none`. Column widths had been declared in a `<colgroup>`, and the result was that titles collapsed to zero width on mobile while the date column took 280px.

The reason is a rule that's easy to forget: **a `<col>` keeps defining its column even when the matching cells are `display: none`.** The column slots don't renumber. Hiding the first column therefore shifted the title cells into the first `<col>`'s zero-ish width. Dropping the colgroup and setting widths on `th:nth-child(1)` / `th:nth-child(3)` restores the mapping, because those selectors stop matching once the cell disappears.

## Deploying: everything in the directory is public

The site deploys as Workers Static Assets, where the deploy directory is the repository root. That means every file in the repo is served — confirmed by measurement rather than assumption: with the Wrangler config file absent from the ignore list, requesting it returned **200**. Internal documentation, the local dev server script, and the config itself all needed explicit exclusion entries.

Two things about that verification are worth carrying forward. First, the dry-run's file count is misleading — it reported reading roughly 1,790 files against 244 real ones, because it counts version-control internals before filtering. That number is not evidence the ignore list is being honored. The only trustworthy check is requesting the URLs from a running local server and confirming 404s. Second, the URL contract depends on one config value that's already the default but was pinned explicitly anyway: directory index files serve *with* a trailing slash, individual files *without*. Every canonical tag, Open Graph URL and sitemap entry on the site assumes exactly that, so it's worth stating in config rather than inheriting silently.

The apex domain redirects to the `www` host in a single 301 hop, including for plain HTTP, because the redirect rule runs ahead of the HTTPS upgrade. The apex has no origin at all — it points at the IPv6 discard prefix and exists purely for the proxy to intercept. It must stay proxied for the redirect rule to run, and must never be attached to the Worker as a second custom domain, which would serve identical content on both hostnames and create precisely the duplicate-content problem the redirect exists to prevent.

## Outcome

The site is live. All 75 pages serve, fonts render as authored for the first time in years, there is no horizontal overflow at any width tested from 1400px down to 360px, and the excluded files return 404 on the real domain.

## Takeaways

- **Silent fallbacks are the most expensive bugs.** A missing font, a stale endpoint, a percentage that quietly shrinks — none of them throw. They just render *something*, and something is enough to stop anyone looking. Assert on the observable result (measured text width, measured gutter, measured overflow), not on the absence of errors.
- **When you fix a cascade bug, delete the loser.** A dead `@font-face` after a working `@import`, an override whose values the new layout model already supplies — leaving them in place turns a real fix into a no-op or a landmine for the next person.
- **Changing layout model invalidates the old tuning.** Padding under inline-block creates space; padding under a fixed grid column consumes it. Every dimension declared for the old model needs re-justifying, not carrying over.
- **Trust your measuring instrument last.** A resize command that no-ops while reporting success, a cached stylesheet, a file count that includes what it claims to exclude — each of these produces a confident green result for a broken change. Prefer checks whose failure mode is loud.
