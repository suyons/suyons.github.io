---
title: "PDF Generation Troubleshooting - The Base-Href Fix That Broke My Fonts"
date: 2026-09-17
draft: false
tags: ["python", "pdf-generation", "css", "headless-browser", "markdown"]
categories: ["Backend"]
description: "Adding a <base> tag to resolve a markdown document's relative image paths silently broke the @font-face rules sitting in the same inline stylesheet, because both resolve against the same base once one is set."
showToc: true
---

My markdown-to-PDF pipeline renders a document by converting it to HTML, writing that HTML into a temp cache folder next to some bundled fonts, and printing it to PDF with headless Edge. It had worked fine for text-only documents. The moment a document referenced an image with a relative path — `![](diagram.png)`, sitting next to the source markdown file, not next to the render cache — the image broke. The obvious fix is a `<base href>` pointing at the markdown file's own folder, so relative image paths resolve against the right place. That fix worked. It also broke every font on the page.

## Same mechanism, different intent

The stylesheet is generated inline and written straight into the rendered HTML — not linked from an external `.css` file:

```python
CSS = """
@font-face { font-family: "NotoKR"; font-weight: 400;
             src: url("NotoSansKR-Regular.ttf") format("truetype"); }
@font-face { font-family: "NotoKR"; font-weight: 700;
             src: url("NotoSansKR-Bold.ttf") format("truetype"); }
...
"""
```

This is the detail that matters and is easy to get wrong: a `url()` inside a *linked* stylesheet (`<link rel="stylesheet" href="...">`) resolves relative to that stylesheet's own location, `<base>` or no `<base>`. A `url()` inside an *inline* `<style>` block — which is what this pipeline writes — resolves relative to the document's base URI instead. Before any `<base>` tag existed, the document's base URI was wherever the HTML file itself was written: the render cache folder, right next to the font files. The relative font paths worked by coincidence — because the HTML and the fonts happened to live in the same directory, not because the CSS was doing anything base-URI-aware.

Add `<base href="{markdown_folder}">` to fix the image paths, and the *same* inline stylesheet's font `url()`s now resolve against the markdown file's folder instead — a directory that has never had a `.ttf` file in it. Fonts silently fall back to whatever the renderer substitutes, and nothing throws an error; a print-to-PDF pass doesn't fail loudly when a font 404s, it just renders with the wrong typeface.

## Two paths, two resolution rules, one fix

The actual requirement is that images and fonts need to resolve against *different* base directories — images against the markdown source folder, fonts against the render cache. A single `<base>` tag can't express two different bases at once, so the fix is to stop asking it to: let `<base>` handle images (the common case, and the one relative markdown links are written for), and force font URLs to bypass base-URI resolution entirely with an absolute `file:` URI substituted in at render time.

```python
CACHE_DIR = pathlib.Path(tempfile.gettempdir()) / "note-md2pdf"

# __FONTDIR__ gets replaced with the font cache folder's file: URI. <base>
# points at the document's folder, so only the font paths need to be absolute.
CSS = """
@font-face { font-family: "NotoKR"; font-weight: 400;
             src: url("__FONTDIR__/NotoSansKR-Regular.ttf") format("truetype"); }
@font-face { font-family: "NotoKR"; font-weight: 700;
             src: url("__FONTDIR__/NotoSansKR-Bold.ttf") format("truetype"); }
...
"""
```

```python
work = prepare_fonts()
html = work / "render.html"
html.write_text(
    f"<!doctype html><meta charset='utf-8'><title>{src.stem}</title>"
    f"<base href='{src.parent.as_uri()}/'>"
    f"<style>{CSS.replace('__FONTDIR__', work.as_uri())}</style>{body}",
    encoding="utf-8",
)
```

`pathlib.Path.as_uri()` does the file-path-to-`file://`-URI escaping correctly (spaces, non-ASCII characters in a path — both show up in practice once documents live in folders with real names), which is the reason to reach for it instead of hand-building the string with an f-string and hoping the path never has a character that needs escaping.

## The print-CSS detail riding along in the same diff

Two smaller fixes landed in the same change, worth a mention since they're easy to miss if you're only looking for the font bug:

```css
/* Markdown's --- means a forced page break here, not a horizontal rule. */
hr { border: 0; margin: 0; height: 0; break-after: page; }
img { max-width: 100%; height: auto; border: 1px solid #ddd; }
```

Repurposing `<hr>` as a page-break signal is a deliberate hijack, not an accident — markdown has no native "force a page break here" syntax, but every markdown renderer already turns a bare `---` line into an `<hr>`, so `break-after: page` on that element gets a free page-break control without touching the markdown source at all. It costs the ability to render an actual horizontal rule from markdown, which this document format apparently doesn't need. And `img { max-width: 100% }` is the unglamorous fix for the failure mode that shows up the moment the first fix works: an image sized for a screen is often wider than an A4 page, and without a max-width it prints past the page margin instead of scaling down.

## What I didn't verify

This pipeline shells out to `msedge.exe` for the actual print-to-PDF step, which means it only runs on Windows with Edge installed — I couldn't execute this code in the environment I wrote this post in. The base-URI resolution rules (inline `<style>` resolves against the document base, linked stylesheets resolve against their own URL) are standard CSS behavior and not specific to Edge's print renderer, but the end-to-end fix — including the `as_uri()` escaping — is unverified beyond reading it. If you're building something similar, I'd confirm the font rendering with an actual PDF output before trusting this pattern on a document with non-ASCII paths.

## Takeaways

- **`<base href>` changes resolution for every relative URL on the page, not just the one you added it for.** A fix aimed at markdown image links reached into an unrelated `@font-face` block because both live in the same document.
- **Inline `<style>` and linked `<link rel="stylesheet">` resolve `url()` differently.** Only the inline case follows `<base>`; a linked stylesheet's relative URLs stay pinned to its own location regardless. Know which one you're writing before debugging why a path "should" have worked.
- **When two kinds of relative path need two different base directories, pick one to make absolute instead of fighting `<base>` for both.** The images were the common case worth keeping relative; the fonts became the one exception.
