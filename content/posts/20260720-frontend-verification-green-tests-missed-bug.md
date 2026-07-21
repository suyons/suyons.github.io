---
title: "Frontend Testing - Nine Widths of Green Checks and the Bug Was Still There"
date: 2026-07-20
draft: false
tags: ["testing", "css", "browser-automation", "headless-chrome", "verification"]
categories: ["Web Development"]
description: "A CSS specificity bug survived a nine-viewport, four-assertion verification pass I was pleased with, plus two smaller episodes — a screenshot that lied and a regex that could have eaten a client's prose — that all trace back to one habit: trusting the check instead of reading back the value you actually changed."
showToc: true
---

The second half of a day migrating a clinic's site to static HTML was feature work: social share images, a readability pass on ten essays, and restoring a click-to-expand card UI. All three shipped. But the part worth writing down is a CSS bug that survived a verification pass I was genuinely pleased with — nine viewport widths, four assertions each, every one green — and the thing was still visibly broken on a phone.

## Rendering an image when you have no image library

The site had no `og:image` on any of its 75 pages, so every share on KakaoTalk, Naver or Facebook fell back to whatever the scraper guessed. Fixing that means producing a 1200×630 card, which turned out to be the harder half of the task, because the machine had no ImageMagick, no Pillow, and no PyObjC — and the card needed Korean text that the standard library can't rasterize.

The way out was to treat a headless Chromium browser as the image renderer: lay the card out in HTML and CSS, screenshot it at exact pixel dimensions, then convert to JPEG with the OS image tool.

```sh
"…/Brave Browser" --headless --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=1 --window-size=1200,630 \
  --virtual-time-budget=12000 --screenshot=og-raw.png file://…/card.html
sips -s format jpeg -s formatOptions 70 og-raw.png --out og-main.jpg
```

Two details matter. `--force-device-scale-factor=1` is not optional on a Retina host — without it you silently get a 2400×1260 file. And Chrome itself wasn't installed on this machine; Brave is Chromium underneath and takes the identical flags, worth remembering when a machine seems to lack a headless browser.

Then a false alarm that nearly caused a real change. The card's tagline uses a Korean serif webfont, and in the rendered output the Hangul looked sans-serif to me. The obvious conclusion was that the font had failed to load — plausible, since a genuinely dead font endpoint had turned up on this same site earlier that day. Before swapping it out, I measured instead, using a canvas text-width probe read back via `--dump-dom`, since `--screenshot` gives you no way to get JavaScript results out:

```js
const probe = (f) => {
  const c = document.createElement("canvas").getContext("2d");
  c.font = "100px " + f;
  return c.measureText(SAMPLE).width;
};
// Nanum Myeongjo 1085.9 · generic serif 1124.0 · sans-serif 1004.0
```

Three distinct numbers means the requested face really was applied. The font was fine; that particular serif simply reads almost sans at small sizes. I had been about to "fix" a non-bug on the strength of a glance.

## Reflowing prose with regular expressions, safely

Ten essays had been migrated such that every line the author typed in the old editor became its own paragraph. That produced two opposite defects at once: sentences split across paragraph boundaries where the author had wrapped a line, and sentences glued together with no space where the author had *not* broken, because the migration dropped the space.

The transform is simple: join a paragraph onto the previous one when the previous doesn't end in sentence-final punctuation, restore the missing space after `.?!,`, emit one sentence per paragraph. The interesting part is that paragraph counts moved in *both* directions, which is the signal that both defects were real — one essay went from 48 paragraphs to 28 as fragments merged, another went from 5 to 20 as blobs split apart.

Running regular expressions over someone's prose is exactly the kind of change that can quietly eat a sentence, so the guard was a content diff: strip all tags and whitespace, then compare before and after.

```
OK   essay-1: identical
OK   essay-2: identical
…
FAIL essay-6: 1546 -> 1530 chars
```

That failure was the check working. Sixteen characters, precisely the length of that page's title, which had been duplicated as an echo in the opening paragraph and was deliberately removed. Every other file was character-identical. Without that comparison I'd have been trusting a regex with the client's writing on faith.

One classical-Chinese quotation needed exempting: it carries no sentence-final punctuation, so the join rule swallowed it into the following sentence. Detecting it as "contains CJK ideographs, contains no Hangul" keeps it standing alone.

## The bug that passed every test

The last feature restored a grid of clickable cards — icon, title, a "see details" button — that expand in place. I built it, then verified it across nine viewport widths from 360px to 1400px, asserting four things at each: column count, horizontal overflow, whether the buttons aligned across each row, and the expanded card's padding. All green.

The site owner reported that it lacked left padding. I measured the expanded card, found 40px, and concluded it was a matter of taste — so I raised it to 64px. They reported it again. I looked at a different screenshot, still of the expanded desktop state, and again found the padding present. Only when they said plainly *mobile viewport, collapsed state* did I measure the one combination I had never looked at:

```
360  { summaryPadLeft: '0px', icon: 1, title: 59, exp: 1 }
390  { summaryPadLeft: '0px', icon: 1, title: 59, exp: 1 }
```

Zero. The card summary's padding had never applied — at any width, from the very first commit.

The cause is ordinary specificity. An older rule in the same stylesheet targeted the same element:

```css
/* pre-existing — (0,1,2) */
details.more summary {
  padding: 16px 0;
}

/* mine — (0,1,1), loses */
.detail-cards summary {
  padding: 30px 20px;
}
```

One class plus two type selectors beats one class plus one type. My padding lost every cascade it entered. The fix was to write the selector as `.detail-cards details.more summary`, at `(0,2,2)`.

What makes this worth recording isn't the CSS — it's why the verification sailed past it. **Not one of my four assertions touched padding.** Column count, overflow, and button alignment are all satisfied by a card with zero padding. And on desktop the summary was a centered flex column, so zero horizontal padding was invisible: the content was centered either way. Only mobile's horizontal row layout exposed it, putting the icon one pixel from the card border.

So the check was thorough along three axes and completely blind along the fourth — the one I had actually changed. Worse, I let the green result override a human telling me twice that it was broken, and each time I re-measured a state they weren't describing.

## A screenshot is not a measurement

A smaller instance of the same theme, from the essay work. I screenshotted a page at `--window-size=390,900` to check the mobile layout, and the result looked like text overflowing the right edge — a convincing rendering of a bug that didn't exist. Headless Chromium doesn't honor the viewport meta tag without device emulation, so the page had laid out at desktop width and the screenshot simply cropped it.

Measuring the same page inside a same-origin iframe, whose width genuinely drives the media queries, reported horizontal overflow of exactly 0 at all eight widths tested. The screenshot was not evidence of anything.

There was a happier surprise in the same area. Rolling the card UI out to the remaining ten pages looked like it would need each page mapped back to the legacy site for icons and summary text. It didn't: every page already carried that data in a list sitting directly above its collapsed sections — the migration had split one legacy component into a visual list plus a set of unlabelled disclosures. The generator could read the card data off the page itself, and the count match between list items and disclosure blocks became the safety assertion. All eleven pages lined up exactly; a page that didn't would have been skipped rather than guessed at.

## Outcome

Everything shipped and was verified against production rather than locally: the share image serves byte-identical to the local file, the reflowed essays contain zero glued sentences, the corrected stylesheet is live, and the files that must never be public still return 404.

## Takeaways

- **Assert on the property you changed.** A test suite can be thorough along every axis except the one that matters. If the change was about padding, something in the check must read padding — adjacent properties passing tells you nothing about it.
- **Specificity fails silently, and centering hides it.** A losing CSS rule produces no error and no warning; it simply never applies. When a new rule shares a target with older CSS, read back the computed value rather than assuming your rule won. Centered layouts mask horizontal spacing bugs completely — they only surface once something switches to a left-aligned row.
- **When someone reports the same bug twice, they're describing a state you haven't looked at.** My instinct both times was to re-verify the configuration I'd already verified, which could only ever reconfirm what I already believed. The report was information about *where I wasn't looking*, and treating my green checks as more authoritative than a human's direct observation cost several rounds to correct.
