---
title: "RAG Retrieval Troubleshooting - When Your Eval Metric Hits the Ceiling Before You've Learned Anything"
date: 2026-08-04
draft: false
tags: ["rag", "retrieval", "chromadb", "bm25", "embeddings", "python"]
categories: ["Backend"]
description: "Building a small RAG pipeline surfaced two lessons that only show up once you look past the top-line hit rate: the metric you pick can hide the thing you're trying to measure, and a benchmark can crown a winner for the wrong reason."
showToc: true
---

I built a small retrieval-augmented generation pipeline against a handful of internal-policy text files — the kind of thing meant to answer "how many vacation days do I get" from a handful of source documents instead of hallucinating an answer. The pipeline itself is unremarkable: paragraph-aware chunking, `BAAI/bge-m3` embeddings, a Chroma collection, a prompt that tells the model to say "not in the documents" when nothing relevant comes back. What was worth writing down is what happened when I tried to actually measure whether the thing was any good.

## The first metric was useless before I noticed

The obvious way to check retrieval quality is a hit rate: for each test question, does the correct source document show up in the top-k results? I wrote ten questions, ran them, and got 10/10 on the first try.

That result meant nothing, and it took a minute to see why. There were only five source documents. With `k=3`, the retriever only has to avoid being wrong about which *file* an answer lives in among five candidates — a bar so low that almost any embedding model clears it immediately. A file-level hit rate on a five-document corpus is ceilinged from the start; it can tell you the pipeline is *broken*, but it can't tell you it's *good*, and it can't distinguish one retrieval configuration from another once both are past the ceiling.

## Switching to a metric that can actually fail

The fix was to stop measuring at the document level and start measuring at the chunk level: does the *specific chunk* that got retrieved actually contain the sentence that answers the question — not just the right file, but the right slice of it. I hand-picked a short answer-key phrase for each test question (a phone number of a fact, effectively) and checked whether it appeared verbatim inside the retrieved chunk's text:

```python
def hit_rate(coll, k):
    return sum(
        any(answer_keys[q] in doc
            for doc in coll.query(query_embeddings=emb.encode([q]).tolist(),
                                   n_results=k)['documents'][0])
        for q, _ in test_questions
    )
```

This metric is sensitive to chunk size in a way the file-level one never could be, because a chunk boundary landing in the wrong place cuts the answer sentence in half. That sensitivity is the point — it's the first metric in the whole exercise that can actually distinguish a good configuration from a mediocre one.

## Chunk size, and a result that looked backwards until I checked why

Rerunning the same ten questions across three chunk sizes:

| max_chars | chunks | avg length | hit@1 | hit@3 |
|---|---|---|---|---|
| 400 | 16 | 327 | 9/10 | 10/10 |
| 800 | 10 | 525 | 8/10 | 10/10 |
| 1500 | 5 | 1052 | 10/10 | 10/10 |

`hit@3` is still ceilinged across all three — five documents, top 3, same problem as before, just one layer down. `hit@1` is where it gets interesting, and the 1500-char row looks like the best result in the table. It isn't. At `max_chars=1500`, the chunker produced exactly five chunks — one per document — so "pick the right chunk" and "pick the right document" collapsed into the same question again. The 10/10 isn't evidence that giant chunks retrieve better; it's the ceiling problem reappearing in a different column, and it comes with a real cost: doubling chunk size roughly doubles the context handed to the generation model on every query.

The real signal is that 400 beats 800: smaller chunks keep the answer sentence from getting diluted by surrounding paragraph text that isn't relevant to the query. At production scale — hundreds or thousands of chunks, not five — the 1500-char row's illusion goes away, because chunk count stops tracking document count. The 400–800 character range is the one that's actually informative here, and it's the one I'd trust to generalize.

## Dense retrieval, BM25, and a benchmark that flattered the wrong thing

The second question was whether adding lexical search on top of the dense embeddings helps. I indexed the same chunks with BM25 and combined the two rankings with Reciprocal Rank Fusion — summing `1 / (k + rank)` across both rankers, a way to merge two differently-scaled ranking systems without having to normalize their scores against each other:

```python
def hybrid_ranking(q, rrf_k=60):
    ranks = [{doc: rank for rank, doc in enumerate(rank_fn(q))}
             for rank_fn in (dense_ranking, sparse_ranking)]
    return sorted(range(len(all_chunks)),
                  key=lambda i: -sum(1 / (rrf_k + r[i]) for r in ranks))
```

One tokenization detail mattered more than expected: BM25 over whitespace-split tokens does badly on Korean, because grammatical particles attach directly to the noun with no space — "annual leave" followed by a topic-marker particle and "annual leave" followed by an object-marker particle are two different whitespace-delimited tokens, even though the word that actually matters for matching is identical in both. Splitting into overlapping character bigrams instead of words sidesteps that without needing a real morphological tokenizer:

```python
def tokenize(text):
    text = re.sub(r'\s+', '', text)
    return [text[i:i + 2] for i in range(len(text) - 1)]
```

That's a deliberate simplification with a known ceiling — it has no notion of word stems or parts of speech, so noise grows as the corpus grows. A proper Korean tokenizer (`kiwipiepy` or similar) is the upgrade path once the corpus is bigger than a toy.

Results, same chunk-level metric, `max_chars=800`:

| retriever | hit@1 | hit@3 |
|---|---|---|
| dense (bge-m3) | 8/10 | 10/10 |
| sparse (BM25, char bigrams) | 10/10 | 10/10 |
| **hybrid (RRF)** | **10/10** | 10/10 |

Read naively, this says BM25 beat a purpose-built multilingual embedding model outright, which would be a strange result to trust. It's an artifact of the eval set: all ten test questions reuse the source document's own vocabulary almost verbatim, so there's no question in the set that actually tests the thing dense retrieval is supposed to be good at — paraphrase, synonymy, a question phrased nothing like the source text. An eval set built entirely from the document's own words rewards lexical overlap by construction, before either retriever runs a single query.

The hybrid approach still earns its place, but not because it "won" this benchmark — it won by construction, since RRF over two rankings can't score worse on `hit@1` than whichever single ranker is best. The real case for defaulting to hybrid is that the two questions the dense-only ranker got wrong (an overtime-request procedure and a probation-period length) are ones where the answer chunk also contains competing numeric details from other policies, and BM25's exact-token matching cut through that noise where dense similarity didn't. That's a narrow, specific case for hybrid, not a general one — and a text-only eval set built from a handful of toy documents doesn't have any examples of the case hybrid is *usually* pitched for: paraphrased or synonym-heavy queries. The next step, not yet done, is rebuilding the question set with deliberate paraphrases and typos so the eval can actually discriminate between the two retrievers instead of assuming it does.

## A generation backend built out of a coding CLI

One implementation note, since it might save someone a subprocess call: with no local LLM server running, I wired the "generate an answer from retrieved context" step through the Claude Code CLI (`claude -p "<prompt>" --model claude-sonnet-5`) instead of standing up a real inference server for a one-off eval. The one thing that matters if you try this — invoke it from an empty directory, not from inside a project repo. Called from within a repo, Claude Code picks up ambient project context (`CLAUDE.md`, surrounding files) the same way it would in an interactive session, which is exactly the kind of uncontrolled context injection an isolated eval can't afford. A throwaway temp directory as the working directory keeps each call scoped to only the prompt you actually sent it.

## Takeaways

- **A hit-rate metric that's ceilinged tells you the pipeline isn't broken, not that it's good.** Check whether your top-k is actually smaller than your candidate pool before trusting a perfect score.
- **Measure at the granularity that can actually fail.** File-level hit rate couldn't fail on five documents; chunk-level answer-in-context could, and that's what made it useful.
- **A perfect score at large chunk sizes can be the ceiling problem wearing a different hat** — check whether chunk count still exceeds document count before reading it as a real result.
- **A retriever "winning" a benchmark it structurally can't lose doesn't validate the retriever — it validates that the benchmark has a construction bias.** Build the eval set to contain the failure mode you actually care about.
