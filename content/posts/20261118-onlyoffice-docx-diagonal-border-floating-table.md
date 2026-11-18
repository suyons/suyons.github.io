---
title: "OOXML Troubleshooting - 49 Diagonal Borders ONLYOFFICE Won't Draw, and a Table It Won't Move"
date: 2026-11-18
draft: false
tags: ["ooxml", "onlyoffice", "docx", "wordprocessingml", "document-rendering"]
categories: ["Backend"]
description: "A pharma customer's 52-page controlled form has to print the same from ONLYOFFICE as from Word. It didn't: 49 diagonal cell borders vanished, and a table landed on top of the floating table before it. Two independent bugs, one already known and unfixed since 2024, the other with a workaround that only works by accident."
showToc: true
---

Same customer, same kind of ticket as the last two: a controlled form authored in Word has to print identically out of ONLYOFFICE, because our document management system embeds ONLYOFFICE as the web editor for files the customer also opens in Word. This time the file is 52 pages, and two things don't match. Forty-nine diagonal cell borders that Word draws don't show up at all. And on page 18, a table that Word pushes to page 19 gets drawn directly on top of the table above it.

## Bug 1: a border type the editor doesn't have

The file uses the standard WordprocessingML diagonal border:

```xml
<w:tcBorders>
  ...
  <w:tr2bl w:val="single" w:sz="4" w:space="0" w:color="auto"/>
</w:tcBorders>
```

`w:tr2bl` draws a line from the bottom-left corner of the cell to the top-right. There are 49 cells using it across roughly 16 pages, with sizes 4 and 8 and colors `auto` and `BFBFBF`. No cell uses the mirrored `w:tl2br`, and no table style in `styles.xml` sets either one — every diagonal in the file is a direct per-cell border. Word draws all 49. ONLYOFFICE draws none.

As far as I can tell, this isn't a parsing bug — the Document Editor has no diagonal cell border implementation at all. Only the Spreadsheet Editor exposes one in its UI. That lines up with a thread on ONLYOFFICE's community forum where someone asked the same question and a staff reply confirmed it was logged as an enhancement suggestion in March 2024. It's still not in the product.

Filed as a suggestion is a reasonable place for a nice-to-have. For us it's a defect: a cell with a diagonal line in the approved Word original prints as an empty cell, and that's a controlled pharmaceutical form where "looks the same as the approved version" isn't optional. That's the framing I used in the vendor ticket — ask for it to be tracked as a defect against DOCX compatibility, not as a feature request.

There's no file-level workaround. Replacing each diagonal with a drawn line shape would change the approved document's structure, and the line stops following the cell if the table is ever edited — not acceptable for a document whose structure is under change control.

## Bug 2: a table drawn on top of the floating table before it

Numbering top-level tables from 0 in document order:

| Table | Word pages | Layout |
| ----: | ---------- | ------ |
| #16   | 16–17      | normal |
| #17   | 18         | floating, `<w:tblpPr w:vertAnchor="text" w:tblpY="67"/>`, 8 rows |
| #18   | 19–21      | normal |

#16 is followed directly by #17 with no paragraph in between. Between #17 and #18 there's exactly one paragraph — #17's anchor paragraph, which also happens to carry a continuous section break (`<w:sectPr><w:type w:val="continuous"/></w:sectPr>`).

Word lays #18 out below the floating #17, so it starts on page 19. ONLYOFFICE places #18 right after the anchor paragraph without checking whether the floating table in front of it is still occupying that space, so the two tables print on top of each other on page 18.

The same shape — a floating table immediately followed by another table — also shows up at tables #20 and #35 in this file. I haven't checked whether ONLYOFFICE mishandles those too; the reproduction above was enough to write the ticket.

## What changing the section break does, and why it doesn't generalize

A section break's `w:type` controls how the section *before* it starts. Changing both breaks around this pair from `continuous` to `nextPage` — #17's anchor paragraph and the one after #18 — puts every table in the file on the same page in both Word and, going by the one case I tested, ONLYOFFICE too:

| File | Word | ONLYOFFICE |
| --- | --- | --- |
| original | 52 pages, #17 on 18, #18 on 19–21 | overlap on page 18 |
| both breaks changed to `nextPage` | 52 pages, same pages for every table | not checked beyond this pair |

It works here only because #17 happens to land at the top of its page already — forcing a page break in front of it doesn't move anything, it just removes the ambiguity that was letting ONLYOFFICE place #18 wrong. It's not a general fix: a floating table that doesn't start at the top of a page would have its position changed by the same edit, and editing the section breaks of an approved, change-controlled form isn't something the customer can do to every file that has this pattern.

## What I'm asking the vendor for

Both bugs are in the report; neither is fixed yet. The ask:

1. Draw `w:tr2bl` and `w:tl2br`, and keep them on save — treat the gap against Word as a compatibility defect, not an enhancement request, since the March 2024 ticket has sat unaddressed for over two years.
2. When a table immediately follows a floating table's anchor paragraph and would overlap it, push it down the way Word does, instead of laying it out as if the floating table weren't there.

Neither has a fix on our side that doesn't compromise the document. That's the actual state of this one: reported, reproduced, not resolved.

## Takeaways

- "Already filed on the community forum" and "already fixed" are not the same thing. Check the date on the acknowledgment before assuming a gap is closed.
- A workaround that only works because of one file's specific layout (here, a floating table that happens to start at the top of a page) is not a fix — say so explicitly in the ticket, or someone will apply it to the next file and get burned when the geometry doesn't match.
- When a renderer drops a feature entirely rather than mis-drawing it, that's worth stating plainly in a vendor report: a missing capability and a layout bug need different fixes, and bundling them as one "doesn't render right" complaint makes both harder to triage.
