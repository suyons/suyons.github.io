---
title: "PDF Generation Troubleshooting - A Font File That Quadrupled Every PDF It Touched"
date: 2026-08-04
draft: false
tags: ["python", "pdf", "fonts", "headless-chrome", "uv"]
categories: ["Backend"]
description: "A markdown-to-PDF script rendered correctly but produced bloated files, and the cause turned out to be a single variable font file that a headless browser couldn't subset-embed."
showToc: true
---

I wanted a small script to turn a markdown file into a clean, print-ready PDF — no LaTeX, no Pandoc dependency chain, just something that runs on a bare Windows machine. The approach was: convert markdown to HTML, style it with CSS, print it to PDF with a headless browser. It worked on the first try. The PDFs it produced were just needlessly huge, and finding out why led to a font-tooling rabbit hole I hadn't expected.

## The setup

The whole thing is one file, run with `uv run`, using PEP 723's inline dependency metadata so there's no separate `requirements.txt` or virtualenv to manage:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = ["markdown", "fonttools"]
# ///
```

`uv run md2pdf.py input.md` builds an ephemeral environment with exactly those two packages and runs the script. No project state, nothing to clean up afterward.

The rendering path itself is simple: convert markdown to an HTML string, wrap it in a small stylesheet, write it to a temp file, and shell out to a headless browser to print it:

```python
subprocess.run(
    [
        str(edge),
        "--headless",
        "--disable-gpu",
        "--no-pdf-header-footer",
        "--allow-file-access-from-files",
        f"--print-to-pdf={dst}",
        html.as_uri(),
    ],
    check=True,
    capture_output=True,
    timeout=120,
)
```

`msedge.exe --headless --print-to-pdf` is effectively a free, no-dependency PDF renderer — it's already installed on any Windows machine, and Chromium's print pipeline handles page breaks, tables, and code blocks better than most lightweight PDF libraries. This is the part that worked immediately.

## The font that didn't behave

The documents needed Korean text, so the stylesheet points at Noto Sans KR:

```css
@font-face { font-family: "NotoKR"; font-weight: 400;
             src: url("NotoSansKR-Regular.ttf") format("truetype"); }
@font-face { font-family: "NotoKR"; font-weight: 700;
             src: url("NotoSansKR-Bold.ttf") format("truetype"); }
```

The text rendered fine. The file size didn't. A one-page document was coming out several times larger than it had any right to be.

The cause is specific to how this font ships on Windows: Noto Sans KR installs as a single **variable font** (`NotoSansKR-VF.ttf`), which encodes an entire weight axis — thin through black — in one file via interpolation, rather than shipping a separate static file per weight. That's a great format for a browser rendering live web text. It is not a format Edge's print-to-PDF pipeline knows how to subset and embed. When it hits a variable font, it falls back to drawing each glyph as a Type 3 outline instead of embedding the font as a proper resource — the page still renders correctly, but every character becomes its own little vector shape instead of a glyph reference, and the file balloons to more than four times the size it should be.

Nothing in the render step raised an error. The PDF looked right on screen. The bug only shows up when you look at the output file's size, which is exactly the kind of defect that stays in production the longest — nobody's checking file size on a document that opens and reads fine.

## Instancing static weights instead of shipping the variable font

The fix is to stop asking the browser to deal with the variable font at all. `fontTools` can instantiate a single static weight out of a variable font's design space, which is exactly what a `@font-face` block wants:

```python
from fontTools import ttLib
from fontTools.varLib import instancer

font = ttLib.TTFont(variable_font_path)
instancer.instantiateVariableFont(
    font, {"wght": weight}, inplace=True, updateFontNames=True
)
font.save(out_path)
```

Run once for weight 400 and once for 700, and the CSS above can point at ordinary static TTFs. Edge embeds those the normal way — actual glyph outlines referenced by index, not redrawn per character — and the bloat disappears.

The rest of the function is just making that a one-time cost:

```python
def prepare_fonts():
    CACHE_DIR.mkdir(parents=True, exist_ok=True)
    targets = {"Regular": 400, "Bold": 700}

    if all((CACHE_DIR / f"NotoSansKR-{n}.ttf").exists() for n in targets):
        return CACHE_DIR

    for name, weight in targets.items():
        out = CACHE_DIR / f"NotoSansKR-{name}.ttf"
        if out.exists():
            continue
        # find a pre-existing static file first; only instance if none exists
        ...
```

Two details here matter more than the happy path:

**It checks for an existing static file before instancing anything.** Some machines already have static Regular/Bold TTFs installed alongside the variable font; instancing is only a fallback for machines that only have the variable one.

**Nothing gets written into the project.** The generated fonts land in the OS temp directory, not the repo, and the script never bundles or redistributes the font file itself — it only reads whatever's already installed on the machine it runs on and derives a local, disposable cache from it. Noto Sans KR is OFL-licensed and redistribution would probably be fine, but "don't ship a copy of a font you didn't write" is a cheap habit that avoids ever having to check.

## Where this breaks

This is a narrow tool, deliberately. It hardcodes Windows font directories and `msedge.exe` install paths — it has no fallback for a machine without Edge installed, and no Linux or macOS path at all. If the variable font is missing entirely (no static fallback, no VF to instance from), it exits with a clear error rather than silently falling back to a system default, which is the right failure mode for a document meant to look a specific way. It doesn't handle any font family other than the one it's told about, and it doesn't try to detect *which* fonts a given document actually needs — you tell it, or it doesn't look right.

## The takeaway

A rendering pipeline can be functionally correct — every glyph in the right place, every page break in the right spot — and still hide a resource-handling failure that only shows up as a number you weren't watching. The general shape of the bug is worth remembering past this one font: a variable font is a single file standing in for a whole family, and any tool downstream that expects to embed *one* weight can silently do something much more expensive than intended rather than failing outright. When a renderer's output is correct but its resource usage looks wrong, check what it's doing with fonts before anything else.
