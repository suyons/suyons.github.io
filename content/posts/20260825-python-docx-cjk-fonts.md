---
title: "Python DOCX Automation - The Font Override That Only Half Applied"
date: 2026-08-25
draft: false
tags: ["python", "docx", "document-generation", "python-docx"]
categories: ["Backend"]
description: "A script that converts Markdown design docs to Word looked done as soon as it ran without errors — until every CJK character in the output ignored the font it was told to use, because python-docx's font API only ever touches half the font model."
showToc: true
---

I wrote a small script to turn Markdown design docs into `.docx` — headings, tables, images, bold text, page breaks — using `python-docx`. It ran cleanly on the first try, produced a document, and looked correct at a glance. Then I checked the font. Every Latin character was set correctly. Every Korean character was rendering in whatever the reader's default font happened to be, completely ignoring the font I'd set. The script hadn't half-failed silently — it had fully succeeded at a job that was only half of what I actually needed.

## Why `run.font.name` doesn't do what it looks like it does

`python-docx` exposes `run.font.name` as if there's one font per run. There isn't. OOXML's `<w:rFonts>` element carries up to four separate font slots — ASCII, East Asian, complex script, and high-ANSI — and a run picks its rendering font per character based on which Unicode range that character falls in. `run.font.name` writes exactly one of those slots (`w:ascii`), and leaves `w:eastAsia` however it was before, which for a freshly created document means "unset," which means "fall back to the reader's theme default." Setting `run.font.name` on a run full of Korean text is a no-op for every character that matters.

The fix is writing the East Asian slot explicitly, through the raw XML, because there's no higher-level API for it:

```python
from docx.oxml.ns import qn
from docx.shared import Pt, RGBColor

FONT_NAME = "Malgun Gothic"

def style_run(run, size, bold=False):
    """Apply font, size, weight, and color to a run.

    run.font.name only sets the Latin-script font slot. For CJK glyphs to
    actually render in FONT_NAME, the w:rFonts eastAsia attribute has to be
    set too, or they silently fall back to the reader's default font.
    """
    run.font.name = FONT_NAME
    r_pr = run._element.get_or_add_rPr()
    r_fonts = r_pr.find(qn("w:rFonts"))
    if r_fonts is None:
        r_fonts = r_pr.makeelement(qn("w:rFonts"), {})
        r_pr.append(r_fonts)
    r_fonts.set(qn("w:eastAsia"), FONT_NAME)
    run.font.size = Pt(size)
    run.font.bold = bold
```

The same gap exists at the document level. Setting `document.styles["Normal"].font.name` only rewrites the default style's ASCII slot too, so any paragraph that doesn't get an explicit run-level override — which, in a naive implementation, is most of them — still renders CJK text in the template's original font. I set `w:eastAsia` on the `Normal` style's `rPr` the same way, once, at document setup, so it's a real fallback instead of a fallback that only covers half the alphabet.

## Two smaller gaps in the same API

**Cell shading has no method.** Table styling built into `python-docx` covers borders and a handful of named table styles, but there's no `cell.shading = ...`. Header-row background color is another raw-XML insert:

```python
def shade_cell(cell, hex_color):
    """Set a table cell's background fill — python-docx has no API for this."""
    tc_pr = cell._tc.get_or_add_tcPr()
    shd = tc_pr.makeelement(qn("w:shd"), {qn("w:val"): "clear", qn("w:fill"): hex_color})
    tc_pr.append(shd)
```

**A trailing `---` shouldn't emit a trailing blank page.** Treating every standalone `---` line as `document.add_page_break()` is correct until the last line of the file happens to be one, at which point it produces a document that ends on an empty page. The fix is checking whether anything non-blank still follows before inserting the break, not just matching the line:

```python
if stripped == "---":
    if any(line.strip() for line in remaining_lines):
        document.add_page_break()
    continue
```

## What this doesn't cover

The font issue is specific to CJK — an all-Latin document would never have surfaced it, because `w:ascii` was always the slot doing the work. Anyone adapting this for Cyrillic or another non-Latin script that isn't classified as East Asian by OOXML needs to check which of the four `rFonts` slots Word actually routes that script through; I haven't verified that mapping outside Korean and don't want to guess at it here. And the shading and page-break fixes above are the two gaps I hit — `python-docx`'s OOXML coverage is thin enough that any table or paragraph formatting beyond the common cases is worth checking against the actual XML output before assuming the high-level API supports it.
