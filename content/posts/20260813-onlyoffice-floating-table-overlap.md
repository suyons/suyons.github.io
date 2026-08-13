---
title: "OOXML Troubleshooting - Proving a Word-vs-OnlyOffice Layout Bug with Single-Element Control Files"
date: 2026-08-13
draft: false
tags: ["ooxml", "onlyoffice", "docx", "documents", "debugging"]
categories: ["Backend"]
description: "Two Word documents rendered fine in Microsoft Word but overlapped their own content in OnlyOffice. Isolating the one XML element responsible — and pre-empting the vendor's obvious rebuttal — is what turned a screenshot complaint into a ticket a developer could act on."
showToc: true
---

A customer reported that two Word documents — pharmaceutical QC test-record forms — rendered correctly in Microsoft Word but came out visibly broken in OnlyOffice: tables and shapes painted on top of other tables. The goal was not just to confirm the bug but to produce a vendor ticket strong enough that it could not be deflected back as "your document is malformed." That distinction shaped the whole session.

## Reading the document instead of guessing

A `.docx` file is a ZIP archive of XML parts, so the first move was to unpack both files and inspect `word/document.xml` directly rather than reasoning from the visual symptom. Counting structural elements immediately narrowed the field. Both documents contained floating tables — tables carrying a `<w:tblpPr>` positioning property — and the larger document additionally contained 133 paragraphs carrying `<w:framePr>`, a text-frame positioning property.

Two positioning mechanisms, both plausible culprits. The floating tables looked like this:

```xml
<w:tblPr>
  <w:tblpPr w:leftFromText="142" w:rightFromText="142"
            w:vertAnchor="text" w:tblpY="1"/>
  <w:tblOverlap w:val="never"/>
</w:tblPr>
```

Mapping the document body structure revealed the shape of the collision: in three places a floating table was immediately followed by another table with **no separating paragraph**. That adjacency turned out to matter enormously, for reasons covered below.

## Reproducing, then isolating

The editor was available locally, so the bug was reproduced directly and captured with screenshots — driving the GUI from PowerShell, which produced its own detour worth recording (see takeaways).

Reproduction alone proves a bug exists; it does not prove _what causes it_. The decisive step was building control files that differ from the original by exactly one element type, with everything else — namespaces, relationships, every other package part — byte-identical:

```python
# Strip exactly one element type from word/document.xml, repackage everything else untouched
def drop_tblp(xml):
    xml = re.sub(r"<w:tblpPr\b[^>]*/>", "", xml)
    return re.sub(r"<w:tblOverlap\b[^>]*/>", "", xml)

def drop_frame(xml):
    return re.sub(r"<w:framePr\b[^>]*/>", "", xml)
```

The results formed a clean truth table: removing `framePr` changed nothing, removing `tblpPr` fixed both documents. The 133 `framePr` paragraphs — the most eye-catching finding of the initial analysis — were a **red herring**. Had that gone into the ticket unqualified, it would have pointed the vendor's developers at the wrong subsystem.

## Locating the defect precisely

Comparing broken and fixed screenshots produced the sharpest observation of the session: **the floating table itself sits at exactly the same position in both.** Only the content _after_ it moves. That single comparison splits a vague "floating tables are broken" complaint into a specific claim — position calculation works, reserving vertical space does not — which is the difference between a ticket a developer can act on and one they cannot.

Geometry sealed the argument. Measuring the table widths against the page's text column showed every floating table occupying **94.0% of available width**, leaving 5.5 mm after the mandated wrap distances. Nothing can fit beside them, so "push following content below" is the only layout that satisfies the spec — not a judgment call where implementations might reasonably differ.

## Anticipating the brush-off

The obvious vendor response to "floating tables break your document" is "then stop using floating tables." The control file pre-empted it by accident: with `tblpPr` removed, the two adjacent tables **merged into one**. In this document format, adjacent tables with no intervening paragraph merge by rule, and marking the first as floating is the standard mechanism for keeping them apart. The property is load-bearing, not decorative — and the control file that proves the bug also proves the workaround is unacceptable. Both facts went into the report explicitly.

## The report that shipped

### Summary

OnlyOffice places a floating table (`<w:tblpPr>`) at the correct position, but **does not reserve its area in the text flow**. Every block that follows the anchor — inline tables, paragraphs, and paragraph-anchored shapes — is laid out as if the floating table were not there, so it paints directly on top of the table. Microsoft Word renders the same files correctly.

One root cause, two symptoms:

| Symptom                | Colliding objects                                      |
| ----------------------- | ------------------------------------------------------- |
| Table drawn over table  | floating table → next **inline table**                  |
| Shape drawn over table  | floating table → next paragraph's **anchored textbox**  |

**Severity:** these are pharmaceutical QC test-record forms. The overlap makes measurement grids and approval fields unreadable, so the documents cannot be used or printed from OnlyOffice at all.

### Environment

|                                           |                                       |
| ----------------------------------------- | ------------------------------------- |
| ONLYOFFICE Desktop Editors                 | **9.4.0.129**                         |
| OS                                         | Windows 11 Home 10.0.28000, x64       |
| Reference application (renders correctly)  | Microsoft 365 Word for Windows, 16.x  |
| Document `compatibilityMode`               | `15`                                   |

### Root cause

The trigger is `w:tblpPr`, confirmed by control files differing by a single element type:

| Control file        | `w:tblpPr` | `w:framePr` | Rendering                |
| -------------------- | :--------: | :---------: | ------------------------- |
| Document 1 (as-is)   |  present   |   present   | **Overlap**                |
| `framePr` removed    |  present   |   removed   | **Overlap — unchanged**    |
| `tblpPr` removed     |  removed   |   present   | **Correct**                |
| Document 2 (as-is)   |  present   |      —      | **Overlap**                |
| `tblpPr` removed     |  removed   |      —      | **Correct**                |

`w:framePr` is not involved despite appearing 133 times.

**Why nothing can sit beside these tables.** Page geometry (`w:pgSz w=11906`, `w:pgMar left=754 right=1225`):

| Quantity                          |   Twips |      mm |
| ---------------------------------- | ------: | ------: |
| Text column width                  |    9927 |   175.1 |
| Floating table width                |    9330 |   164.6 |
| `leftFromText` + `rightFromText`    |     284 |     5.0 |
| **Remaining horizontal space**      | **313** | **5.5** |

The table occupies **94.0%** of the text column. After the mandated wrap distances, 5.5 mm remains — too narrow for any text run or table. Pushing following content below is the only valid layout.

**Defect localization.** Comparing broken and control renders, the table itself is drawn at the same Y in both; only following content moves. Therefore:

- Positioning of the floating table itself: **correct**
- Advancing the flow cursor past the floating table's height: **missing**

The following object's anchor (`<wp:positionV relativeFrom="paragraph">`) resolves against a paragraph whose Y was computed without the floating table's height. Additionally, `<w:tblOverlap w:val="never"/>` is present on every such table and appears to have no effect.

**Spec references (ECMA-376 Part 1):** §17.4.57 `tblpPr` (floating table positioning; `leftFromText`/`rightFromText`/`topFromText`/`bottomFromText` define required clearance), §17.4.65 `tblOverlap`.

### Why the documents are authored this way (not user error)

Two adjacent `<w:tbl>` elements with no intervening paragraph **merge into a single table**. Document 1 contains three such adjacencies. Marking the first table as floating is the standard mechanism for keeping adjacent tables distinct. The property is load-bearing and cannot simply be ignored by the consumer, nor removed by the author without restructuring the form.

### Expected result

Content following a floating table's anchor should be laid out below the table when insufficient horizontal space remains beside it, matching Word's rendering.

### Workaround (until fixed)

In Word: set Table Properties → Text Wrapping → None, **and** insert an empty paragraph between the adjacent tables — omitting the second step merges them.

## Outcome and takeaways

The report shipped with the original files, three control files, and five annotated screenshots. Both symptoms trace to one root cause. The user-facing workaround above was documented for use until a fix ships.

A few lessons generalize:

- **A reproduction is not a diagnosis.** Minimal single-variable control files convert "this is broken" into "this element causes it," and cost very little once the file format is understood.
- **Actively disprove your best suspect.** The 133 `framePr` paragraphs were the most conspicuous anomaly and were entirely irrelevant. Volume is not evidence.
- **Notice what _doesn't_ move.** The table staying put across broken and fixed renders localized the defect better than any amount of studying the thing that did move.
- **Write the counter-argument into the report.** Knowing the likely dismissal and refuting it up front is worth more than additional evidence for the original claim.
- **A tool detour worth remembering:** `SetForegroundWindow` is refused when called from a background process, so screenshot automation kept capturing the wrong window. Switching to `PrintWindow` with the `PW_RENDERFULLCONTENT` flag captures a specific window by handle without touching focus at all — far more reliable for GUI-driven verification.
- **One genuine misstep:** the first attempt at minimal repro files round-tripped the XML through a parser, which rewrote namespace prefixes and silently corrupted the documents. Reverting to plain string edits on the raw XML kept the packages byte-identical apart from the intended change. When the _format_ is what's under test, avoid tools that normalize it.
