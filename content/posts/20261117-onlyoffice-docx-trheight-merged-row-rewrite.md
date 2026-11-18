---
title: "OOXML Troubleshooting - 184 Twips Became 260 After a Save Nobody Edited"
date: 2026-11-17
draft: false
tags: ["ooxml", "onlyoffice", "docx", "wordprocessingml", "document-generation"]
categories: ["Backend"]
description: "A pharma client's Word template has a header table with fixed row heights. One open-and-save in ONLYOFFICE grew two of those rows and shifted everything below them, even with no edits made. The cause: an `exact` row height rewritten to `at-least` because the editor measured a merged cell's empty continuation rows as if they were full rows."
showToc: true
---

A pharma client's Word template has a small header table with carefully fixed row heights. After one open-and-save round trip through ONLYOFFICE, two of those rows came out taller, pushing the table down and shifting text inside the merged cells below it. Our document management system embeds ONLYOFFICE Docs as its web editor, so every user who opens and saves this document hits the same thing. The question was what changed, why, and whether it could be stopped.

## Diffing the package first

A `.docx` is a zip of XML parts, so the first move was unzipping the original and the saved copy and diffing them. Three facts stood out immediately:

- The saved copy's document properties say it was written by ONLYOFFICE, not Word.
- ONLYOFFICE rewrote the whole package on save: header parts were renumbered, style IDs were replaced, and a few settings disappeared — including the page line grid and several compatibility flags.
- The only text change anywhere was the new word the user had typed. Nothing in the header table was touched by hand.

The dropped line grid and compatibility flags looked like the obvious suspect, and that was my first guess out loud. It was wrong — a reminder to measure before blaming whatever setting happens to look different first.

## Measuring instead of guessing

This document uses a different header on its first page, and it's only one page long, so the first-page header is the one that matters. Exporting both files to PDF from Word and comparing the horizontal rule positions of the table showed exactly what moved: the two header rows holding the revision-number and standard labels grew from about 9 pt to about 13 pt tall, and everything below them shifted down by about 8 pt. The title row and two other, already-taller rows were unchanged.

The table XML explained why:

```xml
<!-- Before: saved by Word -->
<w:trHeight w:hRule="exact" w:val="184"/>

<!-- After: saved by ONLYOFFICE -->
<w:trHeight w:val="260"/>
```

ONLYOFFICE changed both the value — 184 to 260 twips, a twip being 1/20 of a point — and the rule. `hRule="exact"` locks a row to that height regardless of content. No rule at all means "at least this tall, grow if needed." A fixed 9.2 pt row turned into a row that's 13 pt or taller.

Saving a second time with no further edits produced the identical result, which rules out anything the user did. This happens on any pass through the editor, full stop.

## Why the merged cells made it confusing

The table is four columns by five rows, with vertical merges offset by one row between the left and right halves. Each merged cell spans two physical rows, so its visible height is the *sum* of those rows' heights — 369 + 184 twips on the left, 184 + 184 in the middle-right cell, 184 + 369 below it. A merged cell has no height of its own; Word computes it from the rows underneath.

That matters because a row height of 184 twips (9.2 pt) doesn't mean a 9.2 pt-tall cell. The merged cell built from that row is actually about 18pt tall, which comfortably holds the 12 pt exact-height line used in every cell of this table. Word accounts for the merge and leaves the row alone.

My best explanation — unconfirmed, since I don't have ONLYOFFICE's source — is that it measures each row against the paragraphs physically inside it, *including* the empty continuation cells that a vertical merge leaves behind at the row level. Each of those continuation cells still carries a 12 pt exact-height paragraph. Seeing 12 pt of content in a 9.2 pt row, it grows the row and drops the `exact` rule to make room — ignoring that the merge already gives that row enough combined height. It's consistent with the data: the two rows that hold the same 12pt lines but are already 18pt+ tall on their own were left untouched, because by this theory there was never a mismatch to "fix" there.

## What the server configuration can't do

I checked the Document Server configuration reference for anything that controls how DOCX gets written back out on save. There's nothing. The closest knobs choose the save format, pass extra arguments to the converter, or point at a font folder — none of them touch row-height handling. Fonts were ruled out separately too, since the edit that triggered this was made in the desktop editor with the correct Korean fonts already installed locally.

## Outcome

The cause is identified and fully reproducible: ONLYOFFICE rewrites an `exact` row height to an `at-least` height whenever it measures more content in that row than the row's own height allows for — even when a vertical merge already gives the real, visible cell enough room. Nothing in the server configuration prevents it, and rewriting the client's template isn't on the table, since it's a controlled document that can't be restructured on our side.

The fix I'm proposing isn't written yet: a patch in the Document Server's save callback. When the server posts the saved file to our callback URL, the handler would diff the saved header table against the *original* file the client first supplied — not the previous saved copy, or a growth from an earlier save would get copied forward as if it were correct — and restore the `exact` row heights wherever the cell text still matches. This stays entirely outside the editor, which is the only place available to fix it without patching ONLYOFFICE itself. The one-row repro above is also going to ONLYOFFICE as a bug report, independent of whether we ship the callback patch.

## Takeaways

- Don't trust the first setting that differs between two file versions. Render both and measure where the layout actually moved before deciding what caused it.
- In a table with vertical merges, a row's stored height is not its visible cell height. Read the merge structure before reasoning about what a given height number means.
- When an editor round-trips a file, assume it rewrites anything it doesn't fully model, and design the surrounding pipeline — here, a save callback — to restore what actually matters rather than trying to stop the rewrite at the source.
