---
title: "OOXML Troubleshooting - openpyxl Choked on a File Excel Opened Fine"
date: 2026-07-09
draft: false
tags: ["python", "openpyxl", "excel", "xlsx", "ooxml"]
categories: ["Backend"]
description: "A legacy xlsx built by a non-Microsoft office suite opened fine in Excel but made openpyxl throw on load. The fix wasn't to repair the file — it was to stop trying to read it and rebuild from raw XML instead."
showToc: true
---

## The setup

A battered legacy xlsx came my way: an event-planning checklist with dozens of rows, a budget sheet, some conditional formatting, still full of the previous owner's real numbers as a working example. The plan was simple — load it with `openpyxl`, strip the personal data, keep the structure, hand back a clean template. Fifteen minutes of work.

`openpyxl.load_workbook()` never got that far. It failed on the styles table before it read a single cell.

## Gotcha 1: a file Excel opens fine can still be unreadable to openpyxl

The file had been built in Hancom Cell (Hancom's Excel-compatible spreadsheet app), and Excel opened it without complaint — mostly. A handful of date formulas rendered `#NAME?`, a sign the file used a function or syntax variant Excel's formula engine didn't recognize, but the workbook itself opened. `openpyxl` was less forgiving: it walks the shared styles table (`styles.xml`) while loading, and this file's style references didn't resolve cleanly against it. `load_workbook()` raised before returning a workbook object at all — not a warning, not a partial load, a hard stop.

That ruled out the fifteen-minute plan. You can't "just strip a few cells" from a workbook object you were never handed.

Trying to repair the file's XML by hand — reindexing the styles table until openpyxl's loader was satisfied — was tempting for about five minutes. It's the wrong instinct: you'd be reverse-engineering a proprietary suite's serialization quirks with no spec to check your work against, to produce a file whose only purpose was to be read once and discarded. The actual goal was never "make this exact file loadable." It was "get the data out of it."

## Gotcha 2: when the library's parser won't cooperate, go around it, not through it

An xlsx is a zip archive of XML files. `openpyxl` is a convenience layer over that structure, not the only way to read it. When the convenience layer refuses to load, the raw structure is still sitting right there:

```python
import zipfile
import xml.etree.ElementTree as ET

NS = {"s": "http://schemas.openxmlformats.org/spreadsheetml/2006/main"}

with zipfile.ZipFile("legacy-checklist.xlsx") as z:
    shared = ET.fromstring(z.read("xl/sharedStrings.xml"))
    strings = [
        "".join(t.text or "" for t in si.iter(f"{{{NS['s']}}}t"))
        for si in shared.findall("s:si", NS)
    ]
    sheet = ET.fromstring(z.read("xl/worksheets/sheet1.xml"))
    for row in sheet.iter(f"{{{NS['s']}}}row"):
        for cell in row.iter(f"{{{NS['s']}}}c"):
            value = cell.find("s:v", NS)
            if value is None:
                continue
            text = (
                strings[int(value.text)]
                if cell.get("t") == "s"
                else value.text
            )
            print(cell.get("r"), text)
```

No styles table involved, because a shared-string lookup doesn't touch it. This is enough to recover cell values and structure — not formatting, not formulas' cached results, just the data — which was exactly what was needed to know what the old sheet contained before designing the new one from scratch. The lesson generalizes past this one file: when a high-level library's loader chokes on a real-world file, dropping to the format's actual on-disk structure is often less work than fighting the library, especially for a one-time read.

## Gotcha 3: the clean rebuild has its own trap — array formulas need to be written as array formulas

The new workbook was built from scratch with `openpyxl`, all dates as live formulas anchored to one input cell so the whole sheet recalculates if the event date changes. One feature needed more care than a plain formula: a "due soon" table on the dashboard that filters the checklist down to overdue-or-upcoming rows and makes each one clickable, jumping the reader to the matching row in the full checklist.

That's `SMALL(IF(...))` array-style row extraction feeding a `HYPERLINK` — a pattern that only works correctly in Excel if the extraction formula is a genuine array formula (legacy Ctrl+Shift+Enter style), not a normal one. Write it as a plain string starting with `=` through `openpyxl`, and it evaluates to a blank in any Excel that isn't running the newer dynamic-array engine — no error, just silently empty cells. The fix is `openpyxl`'s dedicated type for this:

```python
from openpyxl.worksheet.formula import ArrayFormula

ws["F4"] = ArrayFormula(
    "F4",
    '=SMALL(IF(($A$4:$A$61<=TODAY())*($D$4:$D$61<>"Done"),ROW($A$4:$A$61)),ROW()-3)',
)
```

`ArrayFormula` writes the cell with the internal marker that tells Excel to evaluate it as CSE, not as an ordinary formula. That distinction is invisible in the formula bar — a normal formula and an array formula can look identical as text — and only shows up as a wrong (blank) result on the specific Excel builds that need the marker.

## Gotcha 4: a freshly built workbook has no cached values, and some clients won't recalculate on their own

A workbook built entirely through `openpyxl` has formulas but no cached calculated results — `openpyxl` doesn't evaluate formulas, it just writes the formula strings. Most desktop Excel installs recalculate automatically the moment the file opens, so this is invisible most of the time. It stops being invisible when the file is opened by something that trusts the cached value instead of recalculating — a mobile Excel app, a preview pane, some read-only viewers — where every formula cell shows up as `0` or blank until someone forces a manual recalculation.

The fix is one property on the workbook, set before saving:

```python
wb.calculation.fullCalcOnLoad = True
```

That flags the file so any compliant reader recalculates everything on open, rather than trusting a cache that was never populated to begin with.

## A smaller detail: forcing a date format's locale independent of the viewer's Excel install

One more thing worth knowing if you're building anything for a mixed-locale audience: a custom number format string can pin its own locale with a `[$-<LCID>]` prefix, independent of whatever locale the viewer's copy of Excel is running under. A format like `[$-409]yyyy-mm-dd (ddd)` forces the weekday abbreviation to render in a specific target locale, even when the file is opened on an Excel install set to a different one — useful when a spreadsheet needs to display consistently for a specific audience regardless of whose machine it's opened on. This is deep, semi-documented territory (LCID codes aren't in the everyday Excel docs), so verify the code for your target locale against Microsoft's LCID reference rather than guessing.

## Takeaways

- A file opening cleanly in Excel says nothing about whether a library like `openpyxl` can load it — different parsers, different tolerance for a producer's quirks. When the loader fails before you get a workbook object, you have no partial win to build on.
- If a library's high-level loader won't cooperate with a specific file and you only need to read it once, `zipfile` plus a plain XML parser against the raw OOXML structure is usually less work than debugging the loader.
- Array formulas and normal formulas can be textually identical and behave completely differently depending on the Excel build reading them. If you're writing one programmatically, use the library's explicit array-formula type — don't rely on a leading `=` doing the right thing.
- A workbook built by a formula-writing library, not a calculating one, ships with no cached values. Force recalculation on load if anything downstream might trust the cache.
