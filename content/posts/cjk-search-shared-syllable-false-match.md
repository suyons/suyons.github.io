---
title: "A Search for One Word That Kept Returning Another"
date: 2026-06-01
draft: false
tags: ["search", "i18n", "korean", "tokenization", "elasticsearch", "qa"]
categories: ["DevOps"]
description: "On a QA pass, a product search for one Korean word kept surfacing a different word that only shared its first syllable. The cause was syllable-level tokenization with OR matching, and the reason my first fix did nothing was the reindex trap. Here's why CJK search breaks this way and how to actually fix it."
showToc: true
---

## The test that was supposed to show nothing

One line on the QA checklist looked trivial:

> Search a term with no matching products → show the empty-state ("no results") message.

It failed. Not with an error — with results. I searched for a word the catalog had no product for, and the page filled with cards for a *different* product. The empty state never rendered because, as far as the search index was concerned, there were matches.

The two words, romanized: I searched `paenti` (panties) and got back `paencheu` (pants). In Korean they're written `paen-ti` and `paen-cheu` — two syllable blocks each, sharing the first block, `paen`. That shared syllable is the whole bug. The search wasn't matching the word I typed. It was matching a piece of it.

This is the kind of failure you only catch with a query that's *supposed* to return nothing. Every test where the search correctly finds something looks identical whether your relevance is good or terrible. You have to search for an absence to see it.

## Why CJK search breaks where English doesn't

English search gets a free gift it never acknowledges: spaces. "panties" and "pants" are atomic tokens because whitespace already told the tokenizer where the word ends. A naive `WHERE name LIKE '%' || query || '%'` even mostly works, because the full query string is a contiguous run of bytes.

Korean (and Chinese, and Japanese) hand you no such gift. Words aren't space-delimited, and the script is built from syllable blocks that are themselves meaningful units. `paenti` is two blocks. So is `paencheu`. A tokenizer that doesn't *understand* Korean has no way to know `paenti` is one word rather than two syllables that happen to sit next to each other — so the common fallback is to index at the syllable (or character n-gram) level and let the ranking sort it out.

That fallback is where the false match is born. Walk it through:

```
query "paenti"      → tokens: ["paen", "ti"]      (syllable-level)
product "paencheu"  → tokens: ["paen", "cheu"]

default match semantics: OR across query tokens
  → a document matches if it contains "paen" OR "ti"
  → "paencheu" contains "paen"  →  it matches
```

Two things combine to make this bad. First, the query got shredded into syllables, so a single shared syllable is enough surface area to match on. Second — and this is the part people miss — the default matching across those tokens is **OR**, not AND. The engine asks "does this document contain *any* of the query's tokens?" and a product that shares one syllable out of two answers yes. There's no relevance floor saying "one of two syllables isn't enough." So it ranks, it surfaces, and your "no results" page shows a wall of the wrong product.

You'll see the exact same shape with character n-gram indexes (Postgres `pg_bigm`, an Elasticsearch n-gram tokenizer): the index is built to match *fragments*, which is great for recall and terrible for precision the moment two unrelated words share a fragment.

## The fix that did nothing (the reindex trap)

My first instinct was right and my execution was incomplete, and the gap between those two is worth more than the fix itself.

The right instinct: stop tokenizing by syllable. Use a real Korean morphological analyzer — [nori](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html) for Elasticsearch/OpenSearch, `mecab-ko` underneath most Korean NLP — which segments text into actual words. A good analyzer tokenizes `paenti` as the single word *panties*, not as `paen` + `ti`, so it never matches *pants* on a shared syllable in the first place. If you can't add an analyzer, the cheaper patch is to require all query tokens to match (`minimum_should_match` in Elasticsearch terms) or set a relevance cutoff — turn the implicit OR into something closer to AND.

So I changed the analyzer. Searched again. **Same wrong result.**

Here's the trap, and it's a general one, not a Korean one: changing the analyzer changes how *new* documents get tokenized. It does nothing to the documents already in the index, which were tokenized by the *old* analyzer and are still sitting there as syllable tokens. The mapping is updated; the data is stale. An analyzer change is a no-op until you **reindex** — re-run every existing document through the new analyzer.

```
1. update the index mapping / analyzer        ← I did this
2. reindex existing documents through it       ← I skipped this
   (the old syllable tokens live on until you do)
```

My note to myself at the time was blunt: *unverified after the code change; the production index cache needs to be cleared.* "Cleared" was the wrong word for it — the operation isn't a cache flush, it's a reindex — but the symptom that taught me I needed it was real: the fix worked nowhere until the existing rows were rebuilt. On a small catalog that's a one-shot reindex into a fresh index and an alias swap. On a large one it's a background job, and you live with mixed old/new tokenization until it finishes, which is its own source of "why does it work for this product but not that one."

## What I'd actually check

This bug is invisible to most of a test suite, so the checks are specific:

- **Search for something that should return nothing.** A query for a word not in the catalog. If results come back, your tokenizer is matching fragments, not words. Every "search finds the right thing" test passes regardless of this defect — only the absence test exposes it.
- **Search two real products that share a leading syllable** and confirm each returns only itself. That's the minimal reproduction of the CJK false-match, and it's a thirty-second manual check.
- **After any analyzer or mapping change, confirm you reindexed.** If the fix "didn't take," that's the first suspect, not the last. Updated mapping plus stale documents looks exactly like a fix that doesn't work.

The deeper lesson is the one about what tests can't see. A search box that confidently returns the wrong product looks, to every happy-path test, identical to one that works — both return results, both render cards. The defect only shows up when you ask the system for something that isn't there and watch it answer anyway.
