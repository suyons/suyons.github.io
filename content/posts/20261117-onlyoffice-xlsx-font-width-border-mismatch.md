---
title: "OOXML Troubleshooting - 451.3 pt of Excel Printed as 418.1 pt Somewhere Else"
date: 2026-11-17
draft: false
tags: ["ooxml", "onlyoffice", "xlsx", "spreadsheetml", "document-rendering"]
categories: ["Backend"]
description: "A pharma customer's controlled QA spreadsheet has to print identically from Excel and ONLYOFFICE. It didn't: Korean text fell back to the wrong font, the printed table came out 7% narrower, and a border that's one line in Excel broke into nine. Three independent bugs, found by measuring points instead of eyeballing screenshots."
showToc: true
---

A pharmaceutical customer's controlled QA form is authored in Excel and has to print the same way out of ONLYOFFICE, since our document management system embeds ONLYOFFICE as the web editor for files the customer also opens in Excel. It didn't. The PDF ONLYOFFICE produced measured 33 pt narrower than Excel's, the Korean column headers rendered in the wrong font, and a border that's a single line in Excel broke into nine pieces. Three bugs, found only because the comparison was done in points, not by eyeballing two screenshots side by side.

## Three numbers, and nothing else moved

Exporting the same file to PDF from both applications and measuring every page element in points instead of looking at it ruled out most of the obvious suspects fast: left/top margins, row heights, and page count all matched. Three things didn't:

| What | Excel | ONLYOFFICE |
| --- | --- | --- |
| Korean text in `scheme="minor"` fonts | Malgun Gothic | Batang (a font the file never names) |
| Printed table width (A4, 595 pt page) | 451.3 pt | 418.1 pt |
| Top border of the header row | one continuous line | 9 segments, 0.75 pt gap at every column |

Because the table is narrower, it also sits further left, and the right margin ends up 34 pt wider than Excel's. All three turned out to be independent bugs with separate causes, confirmed by testing each one in isolation.

## Bug 1: a font the file never names

Eight of the eleven font definitions in `xl/styles.xml` look like this:

```xml
<font>
  <sz val="10"/><color theme="1"/>
  <name val="Malgun Gothic"/><family val="3"/><charset val="129"/>
  <scheme val="minor"/>
</font>
```

The other three have the same `name` but no `<scheme>`. In the header row, cells using the no-scheme font render in Malgun Gothic in both applications. Cells using the `scheme="minor"` font render in Malgun Gothic in Excel and in Batang — a typeface that appears nowhere in the file — in ONLYOFFICE.

The file's theme defines the minor font like this:

```xml
<a:minorFont>
  <a:latin typeface="Calibri"/>
  <a:ea typeface=""/>
  <a:cs typeface=""/>
  <a:font script="Hang" typeface="Malgun Gothic"/>
  ...
</a:minorFont>
```

Deleting every `<scheme val="minor"/>` from the file (and nothing else) makes ONLYOFFICE render all of it in Malgun Gothic, which looks like confirmation that the theme's empty `<a:ea>` is the trigger. But that's not what Excel does with the same file. Two one-line probes against `xl/theme/theme1.xml` settled where Excel actually gets its Korean font from:

| Probe | Change | Excel's output |
| --- | --- | --- |
| Set `script="Hang"` to Gulim | theme's Hangul entry | unchanged — still Malgun Gothic |
| Set `<a:latin>` to Malgun Gothic | theme's Latin entry | unchanged — still Calibri for Latin text |

Excel's rendering doesn't move no matter what the theme says. For a `scheme="minor"` font, it draws Korean text using the cell's own `<name>`, and Latin text using the theme's Latin font (Calibri) — it never consults the theme's `Hang` entry at all. ONLYOFFICE isn't reading the cell's `name` either, since if it were, this bug wouldn't exist. The working theory — unconfirmed, we don't have the source — is that it falls back on the theme's empty `<a:ea>` to a system default instead of falling back to the cell's own font name. The theme here is the stock "Office 2007 - 2010" theme with an empty `<a:ea>`, which is exactly what Excel writes under a Korean locale, so this isn't a one-off malformed file — it's what any Korean user's default spreadsheet looks like.

Removing `<scheme>` isn't a usable fix, either. It makes ONLYOFFICE render everything in Malgun Gothic, but it also changes Excel's own output on the same file: wider columns, shorter rows, Latin text switching to Malgun Gothic too. For a controlled document where the approved Excel rendering is the thing being matched, a "fix" that changes the reference is worse than the bug.

## Bug 2: the columns are 7% narrower in print

Column widths in this format are stored in units of the default font's maximum digit width, and in this file that's Calibri 11 — a digit advance of 0.5068 em, 5.575 pt. Measuring every column edge in both PDFs and fitting the numbers against two models:

- ONLYOFFICE: digit width rounded to 7 px at 96 dpi (7.43 px rounded down to a whole screen pixel).
- Excel: digit width at 47/600 inch — 46.46 dots rounded up to a whole dot at 600 dpi, the resolution of the "Microsoft Print to PDF" driver used for the export. Inferred from the measurements, not confirmed against Excel's actual source.

| Column | Width (chars) | Excel measured | 47/600-in model | ONLYOFFICE measured | 7 px model |
| --- | ---: | ---: | ---: | ---: | ---: |
| A | 4.625 | 26.0 | 25.8 | 23.8 | 24.0 |
| C | 16.5 | 91.9 | 92.1 | 85.4 | 85.8 |
| F | 12.25 | 68.5 | 68.4 | 63.8 | 63.7 |
| Total | 80.75 | 451.3 | 450.9 | 418.1 | 419.7 |

Every column fits its own model within 0.4 pt. Both applications use the same digit width as input; the only difference is which resolution they round it to before multiplying by 80.75 character-widths. Rounding to a 96 dpi screen pixel throws away about 6% of the digit's true width, and the whole table loses that fraction in print. This is independent of the font bug — swapping which font actually renders doesn't move the width numbers at all.

## Bug 3: one border, nine segments

Row 1 carries a `medium` top border on every cell A1 through I1. Excel draws it as one continuous line. ONLYOFFICE draws it as 9 separate segments, one per column, with a 0.74–0.75 pt gap between each pair — visible under zoom as thin white breaks exactly at the column boundaries of the table below.

Every other horizontal border on the page is also drawn in 9 segments, but there the neighboring segments overlap by 0.75 pt instead of leaving a gap, so they still look continuous. Only the top edge of the first row — where there's no border above it to overlap with — shows the seams. The bug is a border-painting detail, not a geometry detail: it shows up in every PDF we generated regardless of which font or theme variant produced it.

## What doesn't fix it

| Change | Excel's output | ONLYOFFICE's output | Verdict |
| --- | --- | --- | --- |
| None | 451.3 pt, Malgun Gothic | 418.1 pt, Batang | both bugs present |
| Remove `<scheme>` | 488.4 pt, shorter rows, all Malgun Gothic | 479.7 pt, all Malgun Gothic | fixes the font in ONLYOFFICE, but changes Excel's approved output — not usable |
| Theme Latin font → Malgun Gothic | 451.3 pt, unchanged | 479.7 pt, all Malgun Gothic | Excel ignores the change; ONLYOFFICE moves further from correct |

Nothing we changed inside the file closes the gap on both sides at once, which is the whole reason this went to ONLYOFFICE as a vendor ticket rather than a file fix: for a font with `scheme="minor"` or `"major"`, Korean text should be drawn with the cell's own `name` when the theme's `<a:ea>` is empty, column widths shouldn't round the digit advance to a 96 dpi pixel before computing print width, and a border spanning several cells should be drawn without gaps at the cell boundaries. All three are open with the vendor as of this writing — this is a report of what we found, not of a fix that shipped.

## Takeaways

- Measure in points, not pixels, and not by eye. All three bugs here are invisible until you measure the same geometry in both outputs and subtract.
- A theory about where a renderer gets a value isn't proven until you've shown the *other* renderer ignoring the same knob. Deleting `<scheme>` looked like proof until checking what it did to Excel's own output.
- A fix that changes the reference application's rendering isn't a fix when the reference rendering is the thing under contract. "Match Excel" rules out any change that also changes Excel.
- Write "we think, unconfirmed" into a vendor report when you don't have the source to prove it. ONLYOFFICE's exact font-fallback mechanism and Excel's exact width-rounding formula are both still guesses here, clearly labeled as such.
