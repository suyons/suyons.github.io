---
title: "OOXML Troubleshooting - The Bug Was Never the Table: Chasing a DOCX That Only Renders in Word"
date: 2026-09-04
draft: false
tags: ["ooxml", "onlyoffice", "docx", "wordprocessingml", "document-rendering"]
categories: ["Backend"]
description: "A client's Word form rendered fine in Word and lost text from every table cell in ONLYOFFICE. The suspect was a floating-table property that turned out not to exist in the file — the real cause was a line-height override that Word was silently overriding right back."
showToc: true
---

A customer's Word form rendered perfectly in Microsoft Word and fell apart in ONLYOFFICE: text missing from table cells, lines overlapping the cell borders. The working theory going in was floating tables — `<w:tblpPr>`, the property Word writes when you drag a table around the page — and a support ticket had already gone out saying so. The job was to write a fix guide for a non-technical client. It turned into a root-cause hunt, because the theory was wrong.

## The first guide was built on an unverified premise

I wrote the client guide first — "set Text wrapping to None on every table," genuinely the right advice *for floating tables*. Then I unzipped the DOCX to confirm which tables actually needed it.

There were none. `grep` for `tblpPr` across `word/document.xml`, `word/header1.xml`, `word/footer1.xml`, and `word/styles.xml` returned zero hits. So did `framePr`, `txbxContent`, `v:shape`, and floating `wp:anchor`. All five body tables were plain in-flow tables with consistent grid and cell widths.

I said so, and the client tried the guide anyway, out of caution — cutting and re-pasting every table for good measure. Nothing changed, which is exactly what you'd expect when the fix targets a property that was never there.

## Reproducing the bug locally instead of guessing again

The breakthrough was realizing I didn't need to guess how ONLYOFFICE renders the file — ONLYOFFICE ships its own converter binary, `x2t`, inside the desktop app bundle. Pointing it at the DOCX produces a PDF through the same layout engine the editor uses.

It takes a parameter file rather than plain CLI arguments, and it needs to be told where the generated font index lives — the app writes `AllFonts.js` into its user data directory on first launch, not into the app bundle itself:

```xml
<TaskQueueDataConvert>
  <m_sFileFrom>orig.docx</m_sFileFrom>
  <m_sFileTo>orig.pdf</m_sFileTo>
  <m_nFormatTo>513</m_nFormatTo>          <!-- PDF -->
  <m_sFontDir>/path/to/data/fonts</m_sFontDir>
  <m_sAllFontsPath>/path/to/data/fonts/AllFonts.js</m_sAllFontsPath>
  <m_bIsNoBase64>true</m_bIsNoBase64>
</TaskQueueDataConvert>
```

That reproduced the customer's screenshot on the first run. It also gave me something a screenshot never could: a text layer. Running `pdftotext -layout` on the broken PDF returned **all** the missing strings, laid out in roughly the right positions. The glyphs weren't absent — they were being drawn into a line box so short that the cell's clipping path swallowed them.

That reframed the whole investigation. Not a positioning bug. A line-height bug.

## One attribute, in every single paragraph

Every body paragraph in the document carried the same property:

**Before**

```xml
<w:pPr>
  <w:spacing w:line="24" w:lineRule="auto"/>
</w:pPr>
```

With `w:lineRule="auto"`, the `w:line` value is measured in 240ths of a line — `240` means single spacing. So `24` means Multiple 0.1: a line box one tenth of its natural height. Eighty-five paragraphs, all of them.

**After** (what Word writes once you set line spacing to Single — it drops the override entirely, since single spacing is already the style default)

```xml
<w:pPr>
</w:pPr>
```

The obvious follow-up question is why Word doesn't show the same collapse on the original file. The answer was two elements away, in the section properties and the compatibility block:

```xml
<w:docGrid w:type="lines" w:linePitch="360"/>
<w:adjustLineHeightInTable/>
```

That's an East Asian typography feature: a document-wide line grid, 360 twips (18pt) per line, with the compat flag extending grid snapping into table cells. Word snaps every line to that grid, so a 0.1 multiple gets silently rounded back up to a full 18pt line. ONLYOFFICE doesn't apply the grid, takes the 0.1 at face value, and everything collapses.

## Four experiments, because "I found a suspicious attribute" is not a root cause

A plausible-looking culprit is exactly how the session got burned the first time. So each claim got its own test:

1. **Original through `x2t`** — reproduces the reported breakage. Confirms the conversion harness is faithful to what the customer saw.
2. **Change `w:line` from `24` to `240`, touch nothing else, re-convert** — renders correctly. Confirms the attribute is sufficient to cause the collapse.
3. **Export both the original and the patched file to PDF from Word itself** (driven over AppleScript), then diff the `pdftotext -layout` output — identical, same page count. Confirms the fix is invisible in Word, which is what makes it safe to hand a client whose form is under document control and can't have its Word appearance change.
4. **Keep `w:line="24"` but strip `docGrid`, then render in Word** — Word breaks in exactly the same way ONLYOFFICE does. Confirms the grid is the mechanism doing the masking, not some other Word behavior.

Experiment 4 is the one that turns a correlation into an explanation, and it took about ninety seconds to run once the other three were in place.

## Verifying the human instruction, not just the XML

The deliverable was never an XML patch — the client edits in Word. So the last check was to drive Word through the actual UI action, not just simulate its effect in the XML:

```applescript
set line spacing rule of paragraph format of text object of doc to line space single
```

Save, re-unzip, confirm all 85 `w:spacing` elements are gone, push the result back through `x2t`. Clean render. "Select all, set line spacing to 1.0, save" is verified end to end, not inferred from reading the XML.

## Outcome and takeaways

The client guide got rewritten around the real cause: select all, set line spacing to 1.0, save. Word's own output is provably unchanged by that action. The vendor ticket needed correcting too — the request isn't about table positioning, it's that the renderer should honor `w:docGrid` when computing line height, especially inside tables where `w:adjustLineHeightInTable` is set. The minimal repro is one paragraph with `w:line="24" w:lineRule="auto"` inside a document that also declares a line grid.

Three things worth carrying forward:

- **Verify the premise before building on it.** A guide written against an unconfirmed diagnosis costs the client real time. Thirty seconds of `grep` on the unzipped file would have caught it before the first draft went out.
- **Look for the vendor's own engine before reasoning about its behavior.** Desktop office suites ship headless converters. Reproducing a rendering difference locally beats reading spec prose about how it *should* behave, and it turns "the customer says it looks wrong" into an artifact you can bisect.
- **Text present but invisible is a clipping signal, not a font signal.** When a PDF's text layer contains strings that don't appear in the raster, stop debugging fonts. Something collapsed the box the text was drawn into.

The general shape generalizes past this one file: when a document renders correctly in exactly one application, look for a setting that's broken *and* a second setting elsewhere that happens to compensate for it. The compensation is usually the thing the other application doesn't implement.
