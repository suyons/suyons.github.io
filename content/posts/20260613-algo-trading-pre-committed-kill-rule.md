---
title: "Algorithmic Trading - The Kill Rule I Wrote Before I Needed It"
date: 2026-06-13
draft: false
tags: ["trading", "quant", "risk-management", "statistics", "decision-making"]
categories: ["Trading"]
description: "The hardest part of a losing trading strategy isn't diagnosing it — it's deciding to stop. I wrote a pre-committed kill rule in statistical terms before the verdict came in, so I couldn't talk myself out of quitting when it triggered. Here's the rule, the math, and the number that killed the project."
showToc: true
---

## The problem with deciding to quit

Every post in this series has been a diagnosis: a [lookahead leak](/posts/20260613-algo-trading-lookahead-bias-9pct-week/), a [sequence artifact](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/), a [model skew](/posts/20260613-algo-trading-ml-train-serve-stop-mismatch/), an [edge hunt](/posts/20260613-algo-trading-uncorrelated-edge-hunt-negatives/) that came up nearly empty. Diagnosis is the easy part. The hard part is the decision the diagnosis implies: stop.

It's hard for a specific reason. A losing strategy and an unlucky winning strategy look identical in the short run. When you're down, you can always tell yourself it's variance — and sometimes it is. So you keep going, and "I'll give it a bit more data" becomes the phrase that funds every blown account in the world. The person deciding whether to quit is the same person who spent seven months building the thing. That person is not objective.

The fix isn't willpower. It's removing yourself from the decision by writing the rule down *before* the data arrives, in terms specific enough that you can't wriggle out. I did that. Here's the rule, and the number that triggered it.

## Breakeven is a real number, not a vibe

First you need to know what you're measuring against. With ATR-scaled stops at a 1.25 risk-reward ratio (risk 1 to make 1.25), the win rate at which the strategy exactly breaks even — after spread and commission — is **44.4%**. Above that, positive expectancy. Below, you bleed.

That number is the bar. Everything reduces to a single question: is the *true* win rate above 44.4%, and by enough of a margin to survive the gap between backtest and live?

## The kill rule, pre-committed

Here's what I wrote down, before the live sample was large enough to judge:

> Pool the live trades. Compute the win rate and its confidence interval, treating trading days as blocks (so correlated same-day trades don't inflate the sample). If the **upper** bound of the confidence interval on the win rate is below **47%**, the edge is not worth running, and the strategy is killed.

Three design choices in that sentence, each one closing an escape hatch I knew I'd reach for:

1. **Day-blocks, not raw trades.** My systems fire concurrently and correlated — a good day produces a cluster of wins that are really one bet, not twenty. Counting each trade as independent would make a lucky day look like overwhelming evidence. Blocking by day deflates the sample to something honest, roughly the 6.6 effective independent bets I actually have rather than the hundreds of raw trades. This stops me from cherry-picking a hot streak as proof.
2. **The upper bound, not the point estimate.** I don't kill on "the win rate is below breakeven." I kill on "I'm confident the win rate is below 47% *even being generous*." Using the CI upper bound gives the strategy the benefit of every doubt. If even the optimistic end of the interval can't clear a 47% bar — itself set above the 44.4% breakeven to demand a real margin — there's nothing to argue about.
3. **47%, not 44.4%.** The breakeven is 44.4%, but the backtest-to-live work taught me that even a clean bar-faithful backtest runs a few points optimistic versus live. So I demanded the edge clear breakeven by a margin — 47% — before I'd believe it was real. A strategy that's "probably just barely above breakeven" is a strategy I should not be risking a funded account on.

The point of writing all three down in advance is that when the moment came, there was no decision left to make. I'd already made it. All that remained was to read a number off a script.

## The number

The script — `edge_verification.py`, run after every session — pooled the live record and reported it. The pooled win rate came in at **41.2%**. Below breakeven outright. And the day-block confidence interval's **upper** bound was **45.3%** — below the 47% bar I'd pre-committed to.

That's the trigger. Not "it's losing and I feel bad." A specific, pre-registered condition: the optimistic end of the interval, computed with correlation-aware blocking, sat under the worth-it threshold. The pre-committed kill rule fired.

I added it up across every preserved live session. All-in, the strategy was net **-3.2% on the $100k account.** No preserved session was net profitable. The verdict in my project notes is a single word: **DEAD.**

## What "dead" actually means here

It's worth being precise, because the strategy isn't *fraudulent* — it's just thin. The edge is genuine in the sense that it's regime-durable: across 2021-2024, broken into half-years, the win rate cleared breakeven in every single one (46-49%) and the profit factor was above 1 everywhere. It's not curve-fit garbage. It's a real ~2-point edge over breakeven.

But a 2-point edge does not survive a 5.5-point sequence gap. The same-bar re-entry artifact alone is bigger than the entire edge. That's the whole tragedy in one sentence: the strategy is real, and the gap between simulation and reality is larger than the strategy. There's no internal lever left to pull — I tested them all and wrote up the [negatives](/posts/20260613-algo-trading-uncorrelated-edge-hunt-negatives/). Making the stop wider lowers the margin. Gating by regime deletes winners. The only honest move is to stop risking money on it.

## Why I keep the demo running anyway

I didn't delete the system. The demo accounts still run, at a reduced risk setting, in pure data-accumulation mode with the drawdown halts switched off. Not because I think the verdict will reverse — the kill rule is the kill rule — but because a forward live stream is the only fully honest test data that exists. Everything else is a backtest, and this whole series is a catalog of how backtests flatter you. If I ever build the next thing, the cleanest possible benchmark is "did it actually beat breakeven on a real account, day-blocked, CI-upper above 47%." That bar now has a verified failing example behind it, which makes it a sharper bar.

## The takeaway

The single most useful thing I built in this entire project wasn't a signal or a model. It was a sentence I wrote before I needed it, defining failure in numbers I couldn't argue with later. Pre-commitment is the only defense against the version of you that's down money and certain it's just variance.

If you take one thing from the whole series: **decide what failure looks like, in falsifiable numbers, while you're still calm.** Define the breakeven. Demand a margin above it. Account for the correlation in your own data so a lucky streak can't masquerade as proof. Use the optimistic bound so you can never claim you were robbed by bad luck. And then, when the number prints, do the thing you already decided to do.

I built an algorithmic trading system for seven and a half months. It works exactly as designed and loses money, because the design's edge is smaller than the unavoidable gap between testing and trading. Writing the kill rule first is what let me say that plainly instead of feeding it another month. That's the skill the project was actually for.
