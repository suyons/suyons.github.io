---
title: "RAG Knowledge Base Design - Don't Let Practice Look Like Regulation"
date: 2026-08-16
draft: false
tags: ["rag", "llm", "knowledge-base", "retrieval", "embeddings", "compliance"]
categories: ["Backend"]
description: "Curating a RAG knowledge base for a regulated document generator means the retrieval mechanics are the easy part — the hard part is tagging every chunk with how strong its source actually is, before a model can quote industry convention as if it were law."
showToc: true
---

I'm building the retrieval side of a pipeline that drafts a pharmaceutical Product Quality Review — the annual compliance report I wrote about [designing the eval harness for](/posts/20260815-onprem-llm-regulatory-report-eval/). That post was about the model: what it's allowed to generate, what code computes instead. This one is about the knowledge base the model retrieves from, and the problem that only shows up once you start writing the source documents down: regulation, official guidance, and plain industry habit all read exactly the same once they're a paragraph of text sitting in a vector index. Nothing distinguishes them unless you make something distinguish them.

## The chunking and hybrid-search half is a solved problem here

I already worked through chunk sizing, dense-vs-BM25 retrieval, and Korean tokenization for a different corpus in [an earlier post](/posts/20260804-rag-retrieval-eval-hybrid-search/), and I'm reusing that setup as-is — paragraph-aware chunking, `bge-m3` embeddings, Chroma, BM25 with character-bigram tokenization, fused with RRF. None of that is what's interesting about this corpus. What's interesting is what happens *before* any of it: deciding what earns a place in the index at all, and what gets said about it once it's there.

## Every claim needs a strength label, not just a citation

The corpus mixes three kinds of source material, and they carry very different weight:

- **Primary text** — an actual quoted clause from the regulation itself.
- **Secondary citation** — a clause I know exists in an official guidance document, but haven't gotten the primary PDF for yet, so it's quoted from a source that quotes it (in this case, an industry blog post citing the guidance).
- **Industry practice** — a number or procedure with zero textual basis in any regulation, but that's how the field actually does it, because the regulation states the requirement without stating a method.

The report requires exactly this kind of number. The regulation says a manufacturer must show, through statistical trend analysis, that its process remains "consistent" and its specifications remain "appropriate" — and stops there. It never says which statistic, and it never gives a passing threshold. The number everyone in the industry actually uses for a process capability index is a target of 1.33. That number appears nowhere in the regulation text. If a chunk carrying "Cpk target: 1.33" sits in the same index as a chunk carrying an actual quoted regulatory clause, with no marker distinguishing them, the retriever hands both to the model with identical confidence — and a fluent model will write "the regulation requires a Cpk of at least 1.33" without knowing that's false. That sentence is exactly the kind of thing a regulatory reviewer would ask you to point to in the text, and you'd have nothing to show them.

The fix is a tag on every section, not just a source note in a README:

```
### 2.2 Judgment threshold `[industry practice]`

A Cpk target of 1.33 is the conventional benchmark...
> **Caveat** — not a regulatory requirement. Must be stated in the report
> as an internal target, confirmed against internal SOPs before use.
```

Tagging changes what review means for that chunk. A `[primary text]` chunk gets checked for transcription typos against the source PDF. A `[secondary citation]` chunk needs the actual primary document tracked down and diffed against it — three clauses in this corpus are sitting in that state right now, cited only through a blog post because I haven't obtained the original guidance PDF, and they stay flagged until I do. An `[industry practice]` chunk needs something else entirely: cross-checking against whatever internal procedure the target audience actually operates under, because there's no regulatory text to fall back on if that internal procedure disagrees.

None of this is enforced by the pipeline yet — it's a convention in the markdown source. The obvious next step, not yet built, is a check that runs before indexing and refuses to admit any section without one of the three tags:

```python
import re

TAGS = ("[primary text]", "[secondary citation]", "[industry practice]")

def untagged_sections(markdown_text):
    """Return the heading of every ##/### section missing a provenance tag."""
    sections = re.split(r"\n(?=#{2,3} )", markdown_text)
    return [
        s.splitlines()[0]
        for s in sections
        if s.startswith("#") and not any(tag in s for tag in TAGS)
    ]

sample = "## 1. Scope `[primary text]`\nQuoted clause.\n\n## 2. Threshold\nNo tag here.\n"
assert untagged_sections(sample) == ["## 2. Threshold"]
```

A lint gate like this is cheap and it closes off an entire failure mode at write time instead of relying on a reviewer catching an untagged claim after the fact.

## Indexing both the full text and the excerpt creates false duplicates

The regulation runs to well over a thousand lines; the report only draws on a fraction of it. My first instinct was to index the full regulation text so nothing gets missed, and separately keep a curated file of the clauses that actually matter for this report type. That's wrong: indexing both means the same clause exists in two chunks with near-identical wording, and near-duplicate chunks dilute retrieval — a query that should return one clean hit instead splits its relevance score across two chunks that say almost the same thing. The resolution was to index only the curated excerpt, with each clause carrying its own citation back to the exact section number in the source, and keep the full regulation text out of the index entirely, on disk as the excerpt's paper trail rather than as retrievable content.

## Two unrelated documents, coincidentally identical headings

The corpus also includes a self-inspection (internal audit) guideline alongside the quality-review guideline, because the two processes are related — a self-inspection plan is supposed to take prior quality-review findings into account. Both guidelines happen to number their final section "5.4," and both sections are titled "follow-up action management," with wording similar enough that reading them side by side feels like a copy-paste. They are not the same process. A quality review is a per-product annual analysis; a self-inspection is a facility-wide audit on its own separate schedule, and a review of my draft KB confirmed neither the regulation's list of required review items nor the guideline's own item list for a quality review mentions self-inspection at all.

If I'd indexed both guidelines in full, a query about quality-review follow-up actions could easily retrieve the self-inspection guideline's near-identical section instead of — or alongside — the correct one, and a generated report could quietly cite the wrong governance process's procedure as if it belonged to the one it's actually writing about. The fix here wasn't a tag, it was an exclusion: the self-inspection portion of that source document doesn't go into the index at all. The relationship between the two processes (review findings feeding into inspection planning) is real and worth documenting, but it belongs in a paragraph written by a person, not in two independently retrievable chunks that an embedding model has no way to keep apart.

## Trap documents, kept on purpose

The retrieval eval set for this corpus needs a category that tests for false positives — queries the retriever should return *nothing* relevant for, because the answer isn't in scope. For that I kept a handful of unrelated internal documents (HR leave-policy samples, left over from an earlier unrelated retrieval exercise) sitting alongside the pharmaceutical corpus instead of deleting them. They're never meant to be cited in a real report; they exist so a query like "how many CAPAs were overdue" can be checked against whether the retriever stays quiet, or worse, confidently returns a leave-policy paragraph because the embedding space put "overdue" near "leave" for some incidental lexical reason. A corpus that only contains material that's supposed to match every query can't tell you anything about what happens when a query shouldn't match.

## Draft status is a gate, not a label

Every file in the knowledge base carries a review-status field, and right now every one of them reads **draft — needs review**. None of it is indexed yet. That's deliberate: writing the knowledge base and indexing it are two different acts with two different failure costs. A wrong sentence in a draft markdown file costs nothing until someone reads it. A wrong chunk in a live index gets retrieved, quoted, and shipped in a report before anyone notices, because nothing in the retrieval path distinguishes a reviewed chunk from an unreviewed one once it's embedded. Treating "reviewed" as a precondition for indexing, not a task to get to eventually, is the same instinct as not merging code without review — except a bad merge here doesn't fail a build, it produces a plausible paragraph in a compliance document that nobody flagged as needing another look.

## What's still open

The lint check above is sketched, not wired into anything — there's no pre-commit hook or indexing script enforcing it yet, and it's only been run against the one hand-written sample in this post. The three secondary-citation clauses are still waiting on the primary guidance PDF; until that shows up, they stay marked `[secondary citation]` and out of any report language stronger than "per secondary reference." And the industry-practice numbers, including the 1.33 Cpk target, still need a pass against actual internal procedure documents before I'd trust the corpus to hand them out as anything other than clearly-labeled convention.
