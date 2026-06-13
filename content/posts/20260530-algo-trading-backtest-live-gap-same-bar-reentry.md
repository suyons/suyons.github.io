---
title: "Algorithmic Trading - The Honest Backtest That Still Beat Live by 10 Points"
date: 2026-05-30
draft: false
tags: ["trading", "backtesting", "quant", "python", "execution"]
categories: ["Trading"]
description: "My backtest had no lookahead bias and reproduced every live trade's outcome exactly, yet it printed a 50.8% win rate while the live account ran at 41.1%. The dominant cause was a single sequence-generation artifact — same-bar re-entry — that almost no backtest controls for. Here's how I isolated it."
showToc: true
---

## A gap that wasn't supposed to exist

After I killed the lookahead bug (that's [the previous post](/posts/20260526-algo-trading-lookahead-bias-9pct-week/)), I had a backtest I trusted. Signals computed on closed bars, orders filled on the next bar's open, lag-robust under the one-bar check. The ML model on top of it was trained on clean out-of-sample data. I deployed it.

The live account ran at a **41.1% win rate**. The backtest, on the same period, printed **50.8%**.

Nine points. On a strategy whose breakeven is 44.4%, that's the difference between a comfortable winner and a steady loser — which is exactly what the two of them were. The backtest said I had edge. The account said I didn't.

The reflex is to blame execution: slippage, spread, fills. I checked all of it, and that's where this got genuinely confusing.

## Ruling out the obvious suspects

I built a reconciliation tool that replays the backtest engine on the *exact* entries the live account actually took — same signals, same entry prices, same stops — and compares outcomes trade by trade. If the engine and live disagree on how a given trade resolves, that's a fill-accuracy problem.

They didn't disagree. Replaying live's real entries reproduced **76 of 76 outcomes**, a 0-point win-rate gap. Per trade, the engine is faithful. I checked the inputs too:

- **Slippage:** a tick-accurate fill simulation produced the same win rate as the bar engine, exactly. Not a slippage problem.
- **Data:** live OHLC matched the fetched bars to 0.000. Not a data problem.
- **Signals:** live signals matched the backtest on 131 of 131 bars. Not a signal problem.
- **Entry price:** gap under 0.5 points. Not an entry problem.

So every individual trade, live and backtest, resolves identically. The fills are right. The data is right. The signals are right. And yet the two systems have a nine-point win-rate gap over the full period.

If each trade is faithful but the aggregate isn't, the gap can't be in *how* trades resolve. It has to be in *which trades happen.* The backtest and the live account are taking **different sets of trades.**

## The free-run sequence problem

Here's the mechanism, and it's subtle enough that I'd built three engines before I saw it clearly.

A backtest generates its own sequence of trades. It opens a position, follows it bar by bar until it hits a stop or target, books the outcome, and then looks for the next entry. The question is: *when* is it allowed to take that next entry?

My engine, like most, re-entered as soon as the previous trade closed. If a position closed at bar `j` — its stop got hit somewhere inside bar `j` — the engine would immediately look at the signal and open the next trade at bar `j`'s open. That feels innocent. It is not.

Walk the timing. A trade closes at bar `j` because price moved through the stop *during* bar `j`. To open the next trade "at bar `j`'s open," the engine uses the signal from bar `j-1`'s close. But it's making that entry decision with full knowledge that bar `j` is a bar where a stop got hit — and it enters at the open, before the move that hit the stop. The live account can't do this. Live, when a position closes mid-bar at `j`, the soonest it can act on a fresh signal is the *next* bar close. It waits a full bar. The backtest doesn't.

This is **same-bar re-entry**, and it hands the backtest a systematic luck advantage. The backtest gets to re-enter on bars it has implicitly pre-screened, at prices the live account will never get, because live always eats one full bar of delay after every close. Over tens of thousands of trades that compounds into a real, structural win-rate edge that exists only in simulation.

It is invisible to every out-of-sample test, because OOS runs on the same engine with the same artifact. You can split your data a hundred ways and the same-bar re-entry advantage is in every split. Just like the lookahead leak, a flaw that lives in the engine contaminates train and test equally.

## Isolating it: the bar-faithful engine

To measure the thing, I built a backtest engine with one switch: `same_bar_reentry`. Then I ran the full history — 73 weeks, 56,000+ trades — in three modes and watched the win rate walk down a staircase:

| Mode | Description | Win rate |
|------|-------------|----------|
| **A** — free-run | re-enter same bar a trade closed (the old default) | **50.8%** |
| **B2** | only change: no same-bar re-entry | 45.3% |
| **B** — bar-faithful | no same-bar re-entry + blackout windows + position cap | **45.4%** |
| — | actual live account | **41.1%** |

Turning off same-bar re-entry alone cost **5.5 points of win rate** — that one artifact accounts for **57% of the entire backtest-to-live gap.** The rest of the bar-faithful machinery (skipping the daily blackout window, respecting the concurrent-position cap) barely moved it, landing Mode B at 45.4%.

The residual from there to live — 45.4% down to 41.1%, about 4.3 points — I never fully closed. It's some mix of model sizing noise, the forced-close trades the engine doesn't perfectly mimic, and plain sample variance. But the headline was unambiguous: **most of the gap was one sequence-generation artifact, not slippage, not the model, not bad luck.**

## Why I couldn't just use tick data

The honest fix for "which trades happen" would be to simulate at tick resolution — every price change — so the engine's entries and exits match live timing exactly. I tried. The broker's tick feed (`copy_ticks_range` with `flags=1`) returns **quote-change ticks, not every trade print**, and they're sparse. The tick-level bid minimum sat *above* the bar low on 55% of sampled bars by more than 0.10 points. You can't resolve a stop that sits between the tick floor and the true bar low if your ticks never reach the low.

So tick simulation was less faithful than the bar engine, not more. The bar-faithful Mode B — no same-bar re-entry, blackout, cap — became the honest benchmark, and "Mode A is a ceiling, Mode B is the estimate" became a written rule in the project.

## The waterfall, end to end

Putting this together with the model issue from [the next post](/posts/20260602-algo-trading-ml-train-serve-stop-mismatch/), the full decomposition from optimistic backtest to live reality looks like this:

```
optimistic fixed-stop backtest      53.5%
  − ML train/serve mismatch          −5.4pp
  − free-run sequence gap            −8.9pp   ← dominated by same-bar re-entry
  ──────────────────────────────────────────
  live                               41.1%
```

Two of the three big subtractions are software artifacts, not market reality. Only after removing both do you see the strategy's true win rate — and the true win rate is below breakeven.

## The lesson worth keeping

The same-bar re-entry bias is the most general thing this project taught me, and I'd never seen it spelled out before I tripped over it. It's worth stating plainly:

> A backtest that re-enters on the same bar a position closed is giving itself trades the live account can never take. The flaw is in the trade *sequence*, not in any single trade's outcome, so per-trade reconciliation looks perfect and out-of-sample tests look clean. The only way to see it is to forbid same-bar re-entry and measure how far the win rate drops.

For me it dropped 5.5 points — more than the entire edge I thought I had over breakeven. Always run the bar-faithful version. Treat the free-run number as a ceiling you will never reach, not an estimate. And when someone shows you a strategy backtest, the first question isn't "what's the Sharpe," it's "when are you allowed to re-enter after a stop-out."
