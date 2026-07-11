---
title: "OOXML Troubleshooting - A Label Only ONLYOFFICE Thought Belonged on Top of a Table Cell"
date: 2026-07-11
draft: false
tags: ["ooxml", "docx", "onlyoffice", "microsoft-word", "wordprocessingml", "document-rendering"]
categories: ["Backend"]
description: "A client-supplied docx rendered fine in Word and broken in ONLYOFFICE. The XML wasn't wrong — the bug was that two independent layout engines compute a floating object's position instead of reading it off an attribute, and they disagreed."
showToc: true
---

## The bug

A client-supplied `.docx` looked perfectly normal in Microsoft Word. Opening the same file in ONLYOFFICE's web editor showed a small text label — roughly "Font: Malgun Gothic/8/Bold" — sitting directly on top of a table cell that should have read "Non-conforming" instead. Easy to dismiss as "just a font issue." It turned out to be a well-known class of interoperability problem: two engines computing a *derived* position independently, and disagreeing.

## Why the position isn't actually stored in the file

The first instinct when a shape overlaps something it shouldn't is to look for a hard-coded X/Y coordinate that's wrong. That's not what was happening. A `.docx` is just a zip archive of XML files, like every modern Office format, so unzipping it and reading `word/document.xml` directly showed the real picture.

The table was a **floating table** — WordprocessingML lets a table opt out of normal in-line flow with a `tblpPr` element, positioning it independently so text can wrap around it, the same way an image or shape can float:

```xml
<w:tblpPr w:leftFromText="142" w:rightFromText="142" w:vertAnchor="text" w:tblpY="1"/>
```

`vertAnchor="text"` means "position this relative to wherever it lands in the surrounding text flow" — not a fixed page coordinate.

The overlapping textbox was likewise a floating shape, anchored the same way:

```xml
<wp:positionV relativeFrom="paragraph">
  <wp:posOffset>272415</wp:posOffset>
</wp:positionV>
```

This says "put me 272415 EMUs below wherever this specific paragraph ends up" — again, not an absolute coordinate.

Neither of those is a bug by itself. The problem is that "wherever this paragraph ends up" isn't written down anywhere in the file — it's computed at layout time, by reflowing every preceding paragraph, table, and floating shape above it. Word's layout engine and ONLYOFFICE's layout engine each implement their own version of that reflow calculation, and small differences between them are enough to make the same XML resolve to two different Y-coordinates.

## The specific trigger: a "clear all" text-wrapping break

One more element sitting between the table and the textbox made the reflow ambiguity worse: a text-wrapping break with a "clear" instruction.

```xml
<w:br w:type="textWrapping" w:clear="all"/>
```

This tells the layout engine "don't just start a new line — skip down past *all* currently-wrapping floating objects first." Reasonable to want, but it means the engine has to know the full rendered extent of the floating table and any floating shapes above it before it can decide where the next paragraph begins. That's exactly the kind of computation where two independently-written layout engines are most likely to diverge, and it's a good illustration of why floating-object positioning is one of the most fragile parts of Office document interoperability.

A simple test made this concrete without touching the XML at all: pressing Enter above the table in Word visibly shifted the table *and* the textbox downward together, proving both were tracking the live text-flow position rather than sitting at a fixed spot on the page.

## Landing on client-friendly guidance, not a technical mandate

The next question was what to actually tell the client, who authors these documents in Word with no interest in XML or "anchors." A few rounds of guidance were tried and reconsidered.

"Just remove the clear-all break" turned out to be bad advice for a non-technical audience — finding an invisible formatting mark and swapping it for a plain paragraph break, then manually re-checking spacing, is a fiddly, error-prone ask for someone who isn't comfortable with Word's internals.

The practical, low-friction fix that was settled on: select each floating shape (table, arrows, textbox) in Word, open its Position dialog, and set the position to **absolute, relative to the page** rather than relative to the surrounding paragraph or column. Four clicks per object, and it completely removes the ambiguous reflow calculation, because a page-relative coordinate is a fixed number both engines agree on immediately.

The honest tradeoff to disclose: once an object is page-anchored, it stops automatically shifting if the client later edits the text above it — they'd need to manually reposition it after such edits. For a fixed, stamp-like layout block that doesn't need to reflow with prose, that's a reasonable price for reliability across both editors.

As a fallback for a client who finds even that frustrating, the simplest, if inelegant, mitigation is just leaving a wider empty gap between the objects in Word, so a small cross-engine positioning discrepancy doesn't visually overlap.

Since the underlying divergence is a genuine rendering-engine bug rather than something fixable from application code or editor configuration, the recommended long-term path is also to file a report with ONLYOFFICE describing the specific combination — floating table, paragraph-anchored shape, wrapping break with "clear all" — that triggers it.

## Takeaways

- When two document renderers disagree on layout, look for values that are computed from context rather than stored literally in the file: reflowed paragraph positions, wrapped-text extents, anything derived by a layout algorithm rather than read directly off an attribute. Those are the exact spots where independently-implemented engines are most likely to part ways.
- The "correct" root-cause fix — delete this specific invisible break — is often worse advice than a slightly less pure but far more operable fix, like repositioning via a GUI dialog. Matching the fix to who has to carry it out matters as much as matching it to the root cause.
