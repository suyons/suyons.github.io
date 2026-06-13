---
title: "Why OnlyOffice Renders a Numbered List as Bare Dots: The Missing w:numFmt Tag"
date: 2026-05-27
draft: false
tags: ["onlyoffice", "ooxml", "docx", "word", "xml", "troubleshooting"]
categories: ["DevOps"]
description: "A .docx whose multilevel list renders as 1. 1.1 1.1.1 in Microsoft Word but shows only bare dots in OnlyOffice. The cause is a missing w:numFmt tag, and the explanation is a difference in how lenient each renderer is about the OOXML spec."
showToc: true
---

## The Symptom

A document opens fine in Microsoft Word. The multilevel numbered list reads exactly as expected:

```
1.
1.1
1.1.1
1.1.1.1
```

The same file, opened in OnlyOffice, drops every number. What's left is the punctuation:

```
.
.
.
.
```

The text after each marker is intact. Only the generated numbers vanish, replaced by the literal separator (`.`) that was supposed to sit *after* the number. Word users see nothing wrong; OnlyOffice users see a broken document. Nobody changed the file in between.

When the rendering of a list depends on which application opens it, the disagreement is almost always in `word/numbering.xml`, and the resolution is almost always that one of the two applications is being lenient about something the other treats strictly.

---

## Cracking Open the .docx

A `.docx` is a ZIP archive. Rename it and unzip it:

```bash
cp problem.docx problem.zip
unzip problem.zip -d problem/
```

List definitions live in `word/numbering.xml`. Each level of a list is a `<w:lvl>` element keyed by its indent level (`w:ilvl`), nested under an `<w:abstractNum>`. Pull the level-0 definition from the broken file:

```xml
<w:lvl w:ilvl="0">
  <w:start w:val="1" />
  <!-- no w:numFmt here -->
  <w:lvlText w:val="%1." />
  <w:lvlJc w:val="left" />
</w:lvl>
```

Now the same level from a document that renders correctly everywhere:

```xml
<w:lvl w:ilvl="0">
  <w:start w:val="1" />
  <w:numFmt w:val="decimal" />
  <w:lvlText w:val="%1." />
  <w:lvlJc w:val="left" />
</w:lvl>
```

The difference is one line. The broken file has no `<w:numFmt>` element on levels 0 through 3. The working file has `<w:numFmt w:val="decimal" />` on every level.

`<w:lvlText w:val="%1." />` is the template for the marker: `%1` means "the value of level 1's counter," and the `.` is a literal. `<w:numFmt>` is what tells the renderer *how to format* that counter — decimal, roman, letter, and so on. With no `<w:numFmt>`, OnlyOffice has a placeholder (`%1`) but no instruction for what digits to substitute, so it substitutes nothing. The literal `.` is all that survives. That's your bare dot.

---

## Which Levels Break

Mapping the whole list against its rendering makes the pattern obvious:

| ilvl | lvlText        | w:numFmt present?              | OnlyOffice output |
| :--: | -------------- | :----------------------------: | :---------------: |
|  0   | `%1.`          | **missing**                    | `.`               |
|  1   | `%1.%2`        | **missing**                    | `.`               |
|  2   | `%1.%2.%3`     | **missing**                    | `.`               |
|  3   | `%1.%2.%3.%4`  | **missing**                    | `.`               |
|  4   | `%5`           | present (`decimalEnclosedCircle`) | renders          |
|  5   | `-`            | present (`none`)               | renders           |

Levels 4 and 5 carry a `<w:numFmt>` and render without complaint — including level 5, whose format is `none` (a bullet-style marker with no counter at all). The breakage tracks the missing tag exactly, not the list as a whole. That's the confirmation: it isn't a corrupt file or a font problem, it's four specific elements missing one specific child.

---

## Lenient vs. Strict, and Who's "Right"

The interesting question is why Word doesn't care.

When `<w:numFmt>` is absent, Microsoft Word falls back to `decimal` and renders `1, 2, 3`. OnlyOffice renders nothing. The note I'd scribbled during the investigation framed this as "OnlyOffice validates the OOXML spec more strictly." That's the intuitive read, but it's worth being precise about, because it's arguably backwards.

In ECMA-376, the `val` attribute of `<w:numFmt>` has a schema default of `decimal`. If that's correct, then a fully conforming consumer *should* treat a missing format as decimal — which would make Word's behavior the spec-compliant one and OnlyOffice's blank output the bug. (I'm flagging this rather than asserting it; the practical fix below is the same regardless of who's strictly correct.)

Either way, the engineering lesson holds: **a file that renders in one OOXML implementation is not proof the markup is complete.** Word's tolerance for missing-but-defaultable elements means it will happily *open* malformed numbering and, worse, *re-save* it still malformed — Word doesn't see a problem, so it doesn't inject the tag on save. The defect propagates silently through every edit until a stricter renderer like OnlyOffice surfaces it.

---

## A Bonus: Guessing Which Word Saved the File

While reading `numbering.xml`, the namespace declarations at the top give away the file's lineage. Word stamps each format generation with its own namespace:

- `w14` → Word 2010-era markup extensions
- `w15` → Word 2013-era
- `w16` → Word 2016 and later

The broken file declared only up to `w14`. The known-good file declared `w14`, `w15`, and `w16`. So the broken document was most likely last written by a Word 2010-vintage save path, which predates the era when the numbering markup got tightened up. That lines up with the symptom: older Word wrote lists without the explicit `<w:numFmt>`, relied on its own decimal fallback to display them, and never had a reason to add the tag later.

Treat the namespace-to-version mapping as a strong hint, not forensic proof — a file can carry namespaces from multiple editors over its life. But as a first guess for "why is this one file weird," it's fast and usually right.

---

## The Fix

Short term, patch the XML directly. Add `<w:numFmt w:val="decimal" />` to each level that's missing it, placed after `<w:start>` and before `<w:lvlText>` (element order matters in OOXML — the schema is a sequence):

```xml
<!-- before -->
<w:lvl w:ilvl="0">
  <w:start w:val="1" />
  <w:lvlText w:val="%1." />

<!-- after -->
<w:lvl w:ilvl="0">
  <w:start w:val="1" />
  <w:numFmt w:val="decimal" />
  <w:lvlText w:val="%1." />
```

Then re-zip and rename back to `.docx`. Zip the *contents* from inside the extracted folder, not the folder itself — the OOXML parts (`[Content_Types].xml`, `word/`, `_rels/`) must sit at the archive root:

```bash
cd problem/
zip -r ../fixed.docx . -x '.*'
```

Open `fixed.docx` in OnlyOffice and the numbers come back.

If you're doing this across many documents, script it: the edit is mechanical — find every `<w:lvl>` whose child set lacks `<w:numFmt>` and insert a `decimal` one. Be careful with a blunt text substitution, though; you want to match the absence of the element within each `<w:lvl>` block, not blindly insert before every `<w:lvlText>`, or you'll double-add it to the levels that already have a format.

The longer-term answer, if you control the documents, is to stop relying on the implicit fallback at all and make sure your templates carry an explicit `<w:numFmt>` on every level — so the file renders the same no matter who opens it.

---

## What This Generalizes To

The specific bug is a missing tag. The transferable habit is the diagnostic: when a document renders differently across two applications, unzip it and diff the relevant part against a known-good file from the same family. The two files in this case differed by a single element per level, and that element fully explained the symptom. You won't get that signal from staring at the rendered output — only from the markup underneath.

<!-- TODO: author — confirm the exact ECMA-376 clause for the w:numFmt default before citing it as fact -->
