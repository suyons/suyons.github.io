---
title: "Python DOCX Automation - The Constant That Assumed Every Image Was Landscape"
date: 2026-08-26
draft: false
tags: ["python", "docx", "document-generation", "python-docx", "ooxml"]
categories: ["Backend"]
description: "A fixed image width worked fine for every screenshot in the script's test set until a portrait one came in taller than the page it was supposed to fit on."
showToc: true
---

[The same script I wrote about before](/posts/20260825-python-docx-cjk-fonts/) inserts images with one line: `document.add_picture(str(path), width=IMAGE_WIDTH)`, where `IMAGE_WIDTH` was a flat `Cm(16)`. Every screenshot I'd tested it with was a wide browser or terminal capture, so a fixed width across the body area looked like the right call — until a tall, narrow screenshot went in and came out taller than the page.

## Do the arithmetic before blaming the library

The page is A4, 29.7cm tall, with 1.8cm top and bottom margins, leaving about 26.1cm of usable body height. A portrait image at 1080×1920px, scaled to a fixed 16cm width, comes out at `16 * 1920 / 1080 ≈ 28.4cm` tall — over two centimeters past the entire page height, margins included. Word doesn't error on that. It just renders an image that overflows its own page, pushed onto whatever page break happens to land nearest.

The library didn't do anything wrong; the constant baked in an assumption — every image is wider than it is tall — that only held for the images I happened to test with. Any tool that hardcodes one dimension without checking orientation has this bug waiting somewhere in its input space.

## Sizing by orientation instead of by convention

The fix reads each image's pixel dimensions before deciding: wide images fill the body width, tall images cap at a fraction of the body height instead, with a second check in case that height-based width still overflows the body width for something extremely narrow.

```python
PORTRAIT_MAX_HEIGHT_RATIO = 0.5  # cap portrait images at half the body height

def add_image(document, base_dir, src):
    section = document.sections[0]
    body_width = section.page_width - section.left_margin - section.right_margin
    body_height = section.page_height - section.top_margin - section.bottom_margin

    image = DocxImage.from_file(str(base_dir / src))
    px_width, px_height = image.px_width, image.px_height

    if px_width >= px_height:
        width, height = body_width, int(body_width * px_height / px_width)
    else:
        height = int(body_height * PORTRAIT_MAX_HEIGHT_RATIO)
        width = int(height * px_width / px_height)
        if width > body_width:
            width, height = body_width, int(body_width * px_height / px_width)

    document.add_picture(str(base_dir / src), width=width, height=height)
```

`DocxImage.from_file` is `python-docx`'s own internal image inspector — the same one `add_picture` already calls to read a file's native DPI before scaling it. Reaching for it directly here just means the sizing decision happens before the insert instead of being left to the library's own default, which doesn't take orientation into account at all.

Half the body height for portrait images is a judgment call, not a derived constant — chosen so a tall screenshot stays clearly readable without dominating a page that also needs a heading and a caption around it. It's a knob, not a law, and it's the first thing to revisit if a document type turns up where portrait images need to run larger.

## A second bug in the same function, unrelated to sizing

While rewriting `add_image`, the table-row parser nearby had its own gap: the check for whether a line is a Markdown table delimiter row (the `|---|---|` line under a header) required at least three dashes per cell —

```python
return all(re.fullmatch(r":?-{3,}:?", c) for c in cells)
```

— which rejects a perfectly valid delimiter row for a narrow column that only uses two dashes. Loosening the count fixes that, but the more interesting bug was already sitting in the surrounding `all()` call: `all()` over an empty sequence returns `True`, so if `cells` ever ends up empty — an edge case that shouldn't happen given how rows get split, but costs nothing to guard against — the function would call an empty line a valid delimiter row instead of correctly saying no.

```python
return bool(cells) and all(re.fullmatch(r":?-+:?", c) for c in cells)
```

Neither line was wrong in a way any of my test documents would have hit. Both were wrong in the general sense that matters once the script runs on markdown files someone else wrote.

## What this doesn't cover

The orientation check trusts `px_width`/`px_height` as reported by the file's own encoded dimensions. It doesn't read EXIF orientation metadata, so a landscape photo tagged to display rotated 90° would size as landscape and then render sideways at the wrong aspect ratio — a real gap for phone photos, not for screenshots, which is all this script currently handles. And the 0.5 height ratio is untested against anything taller than a typical mobile screenshot; a genuinely long portrait image — a full scrolled webpage capture, say — would still fit the ratio math but might read as uncomfortably small at half a page's height. I haven't hit that case yet, so I haven't tuned for it.
