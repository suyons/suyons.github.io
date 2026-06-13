---
title: "Algorithmic Trading - A Model Trained on One Stop, Traded on Another"
date: 2026-06-02
draft: false
tags: ["trading", "machine-learning", "lightgbm", "quant", "mlops"]
categories: ["Trading"]
description: "I switched my live stop-loss from a fixed distance to an ATR-scaled one and forgot the model was trained to predict win/loss under the old stop. The result was a classic train/serve skew: live predictions almost uncorrelated with outcomes. Plus the fantasy return number I had to retract."
showToc: true
---

## The bug that hides in the gap between two correct decisions

This one isn't exotic. It's the most ordinary production-ML failure there is — train/serve skew — and I want to write it up precisely *because* it's ordinary. Two reasonable decisions, each correct on its own, combined into a model that was quietly predicting nonsense in production for weeks.

The model is a LightGBM classifier. For each candidate trade it predicts the probability the trade wins, and that probability sets the position size: high-confidence trades get scaled up, low-confidence ones scaled down or skipped. The sizing function maps probability to a lot multiplier, roughly 0.2× at p=0.5 to the cap at p=0.9.

For this to mean anything, the model's notion of "win" at serve time has to be the same notion it learned at train time. That's the whole contract. I broke it without noticing.

## Two correct decisions

**Decision one (training):** I trained the model on historical trades labeled win/loss under a **fixed stop and target** — 8 points of stop, 10 points of target. Gold traded around 2,900 when I set those. The model learned "given these features, what's the probability this trade reaches +10 before -8."

**Decision two (serving):** Months later, for completely sound reasons covered in [the thin-margin post](/posts/20260613-algo-trading-pre-committed-kill-rule/), I switched the live stop and target from fixed points to **ATR-scaled** distances: stop = 3× the 14-period ATR, target = 1.25× the stop. Gold had run to around 4,900 by then, and a fixed 8-point stop had decayed into noise — it was getting tagged on random wiggles. ATR-scaling keeps the stop proportional to actual volatility. Correct call.

Each decision was right. Together they were a disaster, because **I never retrained the model.** The live system was now serving an ATR-stop reality to a model that had only ever seen fixed-8/10 outcomes. It was answering a question nobody was asking: "what's the probability this reaches +10/-8," for trades that would actually exit at ±3×ATR.

## How I found it: the predictions were uncorrelated with reality

The symptom didn't show up as a crash. It showed up as the model being *useless* — its probabilities had almost no relationship to whether trades won.

I built a reconciliation that takes the model's live predicted win-probability and correlates it against actual outcomes. A working model shows a clear monotonic relationship: trades it scored 0.7 win much more often than trades it scored 0.4. What I got was a correlation of **r = 0.047**. Essentially zero. The model's confidence and the trades' success were unrelated. It was sizing positions off noise.

That's the signature of train/serve skew. The model isn't broken — it's answering correctly for a world that no longer exists. The features at serve time encode an ATR-stop trade; the model's learned mapping encodes a fixed-stop trade; the two don't connect.

## The fix: retrain on the served reality

The fix is conceptually trivial and was a real amount of work to do faithfully: **retrain the model on ATR-stop outcomes, generated exactly the way they're generated live.**

The "exactly the way" part is where train/serve skew is usually re-introduced during the fix. Live, the model's rolling win-rate features update when each trade *closes*, in close-order, and a prediction is made at the *signal* bar using only the win-rate state known at that moment. If I trained on outcomes computed in signal-bar order instead of close order, I'd have rebuilt a subtler version of the same skew. So the training walk replays trades as an event stream — close events (which update the rolling state) and entry events (which emit a training row) interleaved and sorted so that at any equal timestamp, a close is recorded *before* an entry predicts. That's the live ordering, reproduced.

I retrained on 163,000 ATR-stop trades spanning 2021-07 to 2026-06. The calibration check tells the story. Before, the model's deciles were flat — its highest-confidence bucket won about as often as its lowest. After retraining on the served reality, the decile win rates spread from **0.6% to 99.7%**: the bottom-decile trades almost never win, the top-decile almost always do. The model could finally tell its good trades from its bad ones, because it was finally being asked the question it was trained on.

## The honest footnote: this didn't save the strategy

Fixing the skew roughly doubled the model's risk-adjusted contribution and the backtest win rate recovered by about 5 points on the bar-faithful engine. But — and this is the recurring theme of the whole series — it did not close the live gap. Bar-faithful win rate went to about 45.4%, live stayed at 41.1%. A correctly-served model on a thin edge is still a thin edge. The skew was costing me ~5 points I should never have lost, and recovering them still left the strategy below breakeven.

## The other thing I had to retract: fantasy returns

While I'm confessing modeling sins, here's the worst number I ever produced. During the ML breakthrough phase, one sweep printed a weekly return of **+35%**. I believed it for about a day.

It was fantasy, and the mechanism is worth knowing because it's a trap specific to multi-system portfolios. I had ~25 signal systems that fire concurrently and are *highly correlated* — by my own later analysis, only about 6.6 of them are effectively independent bets. The "+35%/week" came from **summing per-trade returns across roughly 930 concurrent correlated positions with no position cap.** On a real account you cannot hold 930 correlated positions; the pool cap and the correlation between them mean the portfolio return is nothing like the sum of the per-trade returns. I'd computed a number that assumes 930 independent bets when I had about 6.6.

The rule I wrote for myself afterward, verbatim in the project notes:

> Never report a portfolio % from per-trade sums without the pool-cap + correlation model. Per-trade expectancy (mean R), win rate, and R-tails are the only honest aggregates for an uncapped per-trade backtest.

When I redid the timeframe study later and caught myself about to print a portfolio percentage the same way, I retracted it in the commit message rather than ship it. "Mean R per trade" — the average outcome in units of risk — is honest because it doesn't pretend the bets are independent. A portfolio percentage from summed per-trade PnL is a lie scaled by however many correlated positions you pretended to hold.

## Two takeaways

For the ML engineering: **a model's labels are part of its interface.** Change what "success" means at serve time — a different stop, a different horizon, a different target — and you've changed the question, even though no code in the model changed. The features still compute, the model still predicts, nothing throws. The only way you'll catch it is to correlate live predictions against live outcomes and watch for the r ≈ 0 that says the model is answering a dead question.

For the metrics: **the units you report encode an assumption.** A portfolio percentage assumes a capital base and a position structure. If your backtest doesn't enforce that structure — the cap, the correlation — the percentage is invented. Report mean R. It can't lie about independence it doesn't assume.

The next and last post is the one this series has been walking toward: how I decided the edge was genuinely dead, using a rule I'd committed to in writing before the data came in, so I couldn't talk myself out of quitting.
