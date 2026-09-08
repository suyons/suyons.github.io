---
title: "Report Generation Troubleshooting - Every Table Said No Source Data While the Prose Quoted Their Numbers"
date: 2026-09-08
draft: false
tags: ["python", "document-generation", "data-quality", "code-review", "pharma"]
categories: ["Backend"]
description: "A generated compliance report claimed no source data was connected in section after section while its own prose confidently cited numbers from those same tables — the flag behind that contradiction was reading the wrong signal entirely."
showToc: true
---

A generated compliance report came back from review with one verdict: "totally a mess." The document is an annual Product Quality Review draft — a regulated pharmaceutical document where [a tool I've been building](/posts/20260822-pqr-optional-feature-adapter/) queries a database, fills in what it can, and leaves the rest to a human. Nothing had crashed, no test had failed, and every number in the report was correct. The document was still wrong in a way that mattered more than a wrong number would have.

## One boolean, derived from the wrong signal

The assembler decides, per section, whether a data source was ever connected. That decision matters because the tool distinguishes three kinds of empty and renders each one differently:

- **no source data** — nothing to query was ever connected
- **query returned no results** — the query ran and found nothing
- **query path exists but is unimplemented**

The distinction is the whole point. In a regulated document, an empty table reads as "nothing happened that year." If nothing happened is a lie, the document is a lie. So the tool refuses to print a bare empty table and instead states which of the three situations it's in.

Here's how the flag was derived:

```python
# Before
sourced = adapter != 'none'
db = source_module.pick(adapter, **adapter_options)
```

`adapter` is how a command-line caller names its source. But every internal caller opens a database connection itself and passes it straight in as `db=`, naming no adapter at all. So `sourced` came out `False` everywhere, in every internally generated report — the flag was checking a parameter nobody in this call path ever populated.

The result: every queried section announced *no source data was connected*, while, further down, the generated narrative sections confidently discussed the numbers those very tables hold. Both halves came from the same run, against the same live database. That contradiction sitting in a document a regulator might read is what "totally a mess" meant.

```python
# After — handing over a database connection IS having a source.
sourced = db is not None or adapter != 'none'
if db is None:
    db = source_module.pick(adapter, **adapter_options)
```

Two tests now pin both halves, because either half alone passes trivially — one call path with a pre-opened `db`, one with only an `adapter` name. This is the classic shape of a bug that survives a long time: the flag was plausible (an adapter literally named `none` really does mean no source), it never raised an exception, and the only symptom was text that no test ever read.

## Deleting the scaffolding that outgrew the content

The next complaint was simpler, and the fix was pure deletion. Every one of the report's 38 sections was headed by two or three lines explaining what kind of section it was and who was responsible for filling it. Across a full report, that guidance ran longer than the actual content — and it duplicated, word for word, a format spec document that already exists and a section table in code that already encodes the same thing.

What a report section actually has to state is whether a slot is filled and, if not, why. That belongs in the body placeholder, not the header. So the headers collapsed to a single line naming who fills the section and in what shape — table or prose — and since that shape is derivable from the section's own definition, changing a section now updates its label automatically instead of leaving a stale note behind. One exception survived: an unfilled manual section still names its owner, because that's the reader's next action, not format documentation.

## When the schema has no column, the honest answers are narrow

The bulk of the review was a list of "this field is empty and it shouldn't be" complaints. Each one had to be answered by reading the actual schema, not by inventing a column — table and column names in this project have to match the production system exactly, because the query paths were confirmed against a development database that no longer exists. A plausible-looking guess is the expensive failure mode here, not a harmless one.

Three different answers came out of that list, and the variety is the lesson:

**Sometimes the data is there under a name nobody looked at.** A discarded-lot count had been a manual entry field, which meant a prior-year figure could never be filled — nobody re-enters last year's numbers by hand. The batch-status table genuinely has no discard code. But the *progress*-status table has a "stopped" code, and a discarded lot by definition never reaches final approval. So the count became a query for both years, with a test tying the generator's discard flags to the expected totals so the two can't drift apart again.

**Sometimes there is no link, and the honest move is to make it an input.** Two sections needed a product's raw materials, and another needed the trend reports covering that product — the schema has no table joining a finished product to either. Inventing a join or pattern-matching on codes would have been a fabrication dressed as a query. Instead these follow a pattern the report already used elsewhere: the codes are entered once, and everything downstream of them is queried. The section is honest about being half-entered, and the reviewer types a handful of codes instead of transcribing dozens of rows by hand.

**Sometimes the answer is that there is no answer.** The format asks for a rework-lot count. Nothing in the schema records a rework state. That cell now prints "unconfirmed," with the reason named in a source column — never a dash, because a dash in a table reads as zero or as not applicable, and neither of those is a claim this tool can back up.

## Changing the answer key, carefully, exactly once

Scoring here is anchors-first: the correct figures get written down before any data exists, then data is generated to produce them, then the generator is checked against them. Pulling expected values out of already-generated data would make every score perfect and verify nothing.

Which means changing an expected value is a serious act, and it happened exactly once this round. One test scenario has a lot manufactured in December of one year and recalled in the next. Production performance is reported by *manufacturing* year, so once the discard count started coming from the database instead of a hand-entered form, that lot landed in the earlier year — and the later year's expected figures became unreachable. The fix was to move those two expected values, add one to the earlier year, write the reasoning down next to them, and flag it explicitly back to the reviewer as the one change to their answer key. When a definition genuinely moves, the expected values move with it — but that has to be a visible, argued decision, never a quiet edit to make a check go green.

## Outcome and takeaways

All 37 review annotations from this round are applied across six scenarios. The full test suite passes, all six scenario datasets pass their consistency checks, the four deliberate-error cases are still caught by the checks that own them, and rule-based scoring is unchanged — which was the goal, since none of this was supposed to move the measurement.

Three things worth carrying forward:

- **A boolean derived from a proxy will eventually disagree with reality.** "Was a source connected" was inferred from a name, when the actual object was sitting right there in the arguments. Prefer the direct evidence; if you must infer, test both branches.
- **Contradictions between machine-filled sections and machine-written prose are invisible to unit tests.** Both halves passed their own checks. Nothing compared a table's emptiness against the paragraph discussing it. That's a real gap in how generated documents get verified.
- **"There is no column for this" is a finding, not a failure.** Three complaints about missing data produced three different honest answers — query it under its real name, take it as an input, or state plainly that it's unconfirmed. The one answer never available is inventing a source.
