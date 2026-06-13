---
title: "Algorithmic Trading - Every Uncorrelated Edge I Tested, and Why Only One Survived"
date: 2026-06-13T15:00:00
draft: false
tags: ["trading", "quant", "research", "intermarket", "statistical-arbitrage"]
categories: ["Trading"]
description: "Once I knew the gold strategy's edge was too thin to survive live, I went looking for an uncorrelated one: renko bricks, intermarket signals, pairs trading, FX carry, news sentiment, crypto trend. Fifteen-plus experiments, almost all honest negatives, and the specific landmine that fakes a positive in each."
showToc: true
---

## The premise: thin edge needs an uncorrelated friend

By this point I'd established the uncomfortable truth: my gold strategy had a real but thin edge — about two points of win rate over the 44.4% breakeven — and thin edges don't survive the [backtest-to-live gap](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/). Every internal lever I tried to widen it (wider stops, regime gates, consensus filters, tick-velocity gates) was refuted. The strategy was as good as it was going to get, and that wasn't good enough.

So I went looking for an *uncorrelated* edge. The math here is the one genuinely cheerful fact in the whole project: two strategies with modest Sharpe ratios that are uncorrelated combine better than one great strategy alone, because their drawdowns don't line up. A 0.5-Sharpe stream that's uncorrelated to my gold book is worth more to me than a 1.0-Sharpe stream that draws down at the same times. So the target wasn't "find a better strategy." It was "find a strategy whose PnL is uncorrelated with the one I have."

I ran fifteen-plus experiments across renko, intermarket macro, pairs, FX carry, news sentiment, and crypto. Almost all negative. Here's the honest scorecard, and — more useful — the specific way each one tries to fake a positive.

## Renko: every price-only lever refuted

Renko charts throw away time and plot fixed-size price "bricks," which is supposed to denoise trends. I tested three levers on gold renko:

- **ATR-scaled brick size.** Refuted. A fixed brick size beat every volatility-scaled variant, and a fixed brick was roughly breakeven over full history — no edge to scale.
- **A random-forest velocity filter.** This one is instructive. My *original* version showed a promising out-of-sample AUC. When I rebuilt it leak-free — proper time-ordered split instead of a shuffled one, no leaky holdout — the OOS AUC fell to **0.53**. That's noise. The "edge" was the shuffle-split letting the model see neighbors of its test points. (This is the same lesson as the lookahead post, wearing a machine-learning costume: a shuffled split on time-series data is lookahead bias.)
- **Continuation / trend-entry.** Refuted by an outlier. The strategy's entire profit came from **one week** that printed +$111k — about 137% of the total return. Strip that single week and the strategy is net-negative. One week is not an edge, it's a lottery ticket you already scratched.

Verdict: price-only renko on a liquid instrument like gold is arbitraged. No durable edge.

## Intermarket: it's beta, not alpha

The most respectable idea: gold's big macro drivers — the US dollar, real yields (via long-bond ETFs), risk sentiment (VIX, equities) — are all fetchable from the broker. Does cross-asset state, known at a bar's close, predict gold's *forward* return?

I ran the premise-first gate before building anything: measure the correlation of each lagged predictor with forward gold returns at lag 0 (contemporaneous, untradeable), lag 1 (tradeable), and lag 2 (robustness). The strongest survivor was a VIX z-score predicting gold's next 24 hours at a lag-1 correlation of **+0.106**, stable to lag 2. Real, but tiny — and its directional sign-agreement was **50.9%**, a coin flip.

The killer was the honest version of the test. When I built a market-neutral (demeaned) timing strategy from these signals — stripping out gold's own beta — the timing skill collapsed to nothing, and **buy-and-hold gold posted a Sharpe of +1.09 that beat every active intermarket strategy I could construct.** That's the verdict in one line: the intermarket relationship is *gold beta*, not alpha. The signals were predicting gold because gold goes up when the dollar goes down — but capturing that is just owning gold, not timing it. There was no incremental skill to harvest.

## Pairs trading: two landmines that manufacture fake edges

I tested gold-versus-silver as a pairs/statistical-arbitrage trade (their H1 correlation is +0.77, the best pair available). Under an honest backtest it was **gross-negative**: -5 basis points per trade, 33% win rate, -35% total, consistent out-of-sample. Dead.

The valuable part isn't the result, it's the two landmines I had to disarm to *get* an honest result, because both fake a beautiful equity curve:

1. **Revert-only exits manufacture a ~100% win rate.** If your pairs strategy only closes when the spread reverts to the mean, it never books the trades where the spread diverges and keeps diverging — those positions just sit open forever, unrealized. Your closed trades are all winners by construction. The fix: a stop-loss plus a timeout plus a forced mark-to-market at period end, so divergers get booked as the losses they are.
2. **Rolling-beta PnL manufactures fake convergence.** If you compute the spread's PnL using a beta that you re-estimate every bar, the beta drifts to fit recent prices and the spread looks like it's always reverting — because you keep redefining "the mean" to wherever price just went. The PnL must use the hedge ratio **fixed at entry**: `Δlog(gold) − β_entry · Δlog(silver)`, never a rolling residual.

Both landmines turn a losing strategy into a gorgeous fake. I'd built the gorgeous fake first. Disarming them turned +∞ into -35%.

## The quick graveyard

A run of ideas that died fast, for the record:

- **FX carry** — the broker skims the swap so hard that harvestable carry is under 2.8%/year. Too thin to bother.
- **News-tone sentiment (GDELT)** — no signal; correlation to forward gold under 0.04 and unstable.
- **Calendar / overnight-drift anomalies** — no robust edge across indices and crypto; arbitraged.
- **An ML crypto-direction forecaster** — overfit, and lost out-of-sample to naive time-series momentum. The fancy model couldn't beat "if it went up recently, bet it keeps going."
- **Short-horizon mean-reversion on crypto/indices** — no autocorrelation basis; hourly returns are close to a martingale.
- **Cross-sectional momentum across the universe** — weak, Sharpe ~0.2, OOS ~0.
- **A trailing stop instead of a fixed target on the gold strategy** — improved per-trade expectancy by a real +0.016 to +0.018 R and held in 9 of 10 half-years, but both the trailing and static versions stayed *negative* in expectancy. The best internal lever I found, and still below breakeven.

## The one survivor, and why it's not a savior

**Crypto trend-following (BTC + ETH)** was the lone genuine survivor. A simple time-series momentum strategy on crypto:

- Its PnL correlation to my gold strategy was **-0.03** — essentially uncorrelated, which is exactly the prize I was hunting.
- It showed real crisis alpha: in the 2022 drawdown it returned +24.5% while a long book bled, and a similar pattern repeated in 2026.
- Full-sample Sharpe was a healthy **+1.19**, and it survived realistic financing costs (-20%/year on the long side) and 4× cost stress.

And then the honest caveat that kept it on the bench. Out-of-sample — the most recent two years — its Sharpe collapsed to **+0.23**, with a -60% single-instrument drawdown. The textbook fix for single-instrument noise is to diversify into a basket of trend instruments. I tried that too; the basket diluted the Sharpe to ~0.33 (and ~0.19 OOS), because the trend edge was *concentrated in crypto*, not spread across the universe. Diversifying away from the one thing that worked made it worse.

So the survivor is real, uncorrelated, and crisis-friendly — but modest, regime-dependent, and OOS-weak. It earns a place as a small satellite sized at a few percent of account volatility. It does not earn a place as the thing that saves the project.

## The meta-lesson

Across renko, intermarket, pairs, carry, news, calendar, cross-section, and crypto, the same sentence keeps writing itself: **price-only signals on liquid instruments are arbitraged.** The one survivor that's genuinely uncorrelated is also genuinely modest. My gold strategy's own thin-edge conclusion didn't reverse when I looked elsewhere — it generalized across instruments, methods, and alt-data.

And every fake positive I found had a named mechanism: shuffle-split leakage, single-week outliers, revert-only survivorship, rolling-beta self-fulfillment. None of them looked like cheating. They looked like edges. The discipline that mattered wasn't generating ideas — it was knowing the specific way each class of idea lies to you, and testing for that lie first.

Which brings me to the last post: having found no edge worth betting the account on, how I actually decided to stop — using a kill rule I'd written down before the verdict, so I couldn't negotiate with myself when the moment came.
