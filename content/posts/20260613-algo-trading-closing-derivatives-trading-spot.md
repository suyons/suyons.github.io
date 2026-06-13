---
title: "Algorithmic Trading - Why I'm Closing My Derivatives Accounts and Trading Spot"
date: 2026-06-13T13:00:00
draft: false
tags: ["trading", "quant", "cfd", "futures", "retrospective", "investing"]
categories: ["Trading"]
description: "Seven months of building an algorithmic trading system led me to one conclusion: a retail trader cannot consistently extract edge from liquid derivatives with a price-only strategy, because the people on the other side already do everything I hoped to discover, faster and cheaper. So I'm closing the CFD and futures accounts and trading spot."
showToc: true
---

## The conclusion the whole project was for

This is the last post in the [series](/posts/20260613-algo-trading-mt5-rest-prop-firm-bot/), and it's the one that changes what I do next rather than just explaining what went wrong. Everything before this was mechanism — a [lookahead leak](/posts/20260613-algo-trading-lookahead-bias-9pct-week/), a [sequence artifact](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/), a [model skew](/posts/20260613-algo-trading-ml-train-serve-stop-mismatch/), an [edge hunt that came up empty](/posts/20260613-algo-trading-uncorrelated-edge-hunt-negatives/), a [kill rule that fired](/posts/20260613-algo-trading-pre-committed-kill-rule/). This post is the lesson I actually take with me.

It's this: **a retail trader cannot extract edge from a liquid derivatives market with a price-only strategy. Full stop.** Not "it's hard." Not "you need a better model." Not "most people fail." There is no edge there to extract — the institutions took all of it long before I showed up, and they keep it taken. It's the wrong fight, against opponents who already won it, on their home field. So I'm closing my CFD and futures broker accounts and moving to spot only.

Let me defend that, because it's a strong claim and I spent seven months and 392 commits earning the right to make it.

## What "price-only" means and why it loses

A price-only strategy uses nothing but the price history of the instrument — opens, highs, lows, closes, volume, and everything you can derive from them. Every signal I built falls in this bucket: momentum, mean-reversion, channel breakouts, ADX regimes, ATR scaling, the ML model that sat on top of all of it. The only inputs were the chart.

Here's the problem with that, stated as plainly as I can. The price history of a liquid instrument is the *most* examined dataset on earth. Every pattern I could find in gold's 5-minute bars has been found, by people with more capital, better data, lower latency, and direct economic incentive to trade it away the instant it appears. When I "discovered" that trend alignment improves win rate, or that an ATR-scaled stop holds margin across price regimes, I wasn't discovering anything. I was rediscovering things that were priced in before I woke up.

That's not a metaphor — it's literally what the data kept telling me. My own findings, across the whole project, converge on one sentence I ended up writing over and over in different words:

> Price-only signals on liquid instruments are arbitraged.

The gold strategy had a real edge of about two points of win rate over breakeven — and that edge was smaller than the unavoidable [gap between backtest and live execution](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/). When I went looking [elsewhere](/posts/20260613-algo-trading-uncorrelated-edge-hunt-negatives/) — intermarket macro, pairs trading, FX carry, calendar effects, cross-sectional momentum — the same wall. The intermarket "edge" was just gold beta. The pairs trade was gross-negative once I stopped letting it cheat. The one genuinely uncorrelated survivor, crypto trend-following, was modest and fell apart out-of-sample. Different instruments, different methods, same answer.

The people on the other side of my trades aren't a fair fight. In a liquid derivatives market, my counterparty is the most sophisticated and best-capitalized participants in finance, and they have already taken every price-only edge there is to take. Any pattern I can compute from the chart, they computed first, priced in, and now arbitrage in microseconds at costs I can't approach. There is no price-only edge left in a liquid market for a retail trader to find — not a small one, not a hidden one, none. The ones that look like edges are leftovers too thin to be worth an institution's time, which means they're also too thin to survive my costs and my execution gap. Bringing a price-only retail strategy to that market is bringing a calculator to a fight against the people who own the calculator factory. You do not win that fight. You were never going to.

## The structure of derivatives makes it worse for me specifically

Beyond the edge problem, the *instrument type* was working against me, and that's the part that points to the concrete decision.

CFDs and futures are leveraged derivatives. Three things follow from that, all of which I measured directly:

- **The broker takes a cut on every dimension.** Spread, commission, and — for anything held overnight — financing or swap. When I tested whether FX carry was harvestable, the broker skimmed the swap so hard that the available carry collapsed to under 2.8%/year. The house edge is baked into the instrument. A thin strategy edge has to clear all of that before it earns a cent, and mine couldn't.
- **Leverage turns volatility into ruin risk.** My whole [prop-firm setup](/posts/20260613-algo-trading-mt5-rest-prop-firm-bot/) existed because the binding constraint wasn't return, it was *never breaching a drawdown floor.* That's a derivatives problem. With leverage, a normal adverse move doesn't just lose money, it can end the account. I spent enormous effort — circuit breakers, weekly governors, rolling-peak governors, position caps — building machinery whose only job was to stop leverage from killing me. That's effort spent surviving the instrument, not finding edge.
- **The financing clock runs against a holding strategy.** Crypto trend-following, my best survivor, had to overcome -20%/year in long financing before it made anything. The instrument charges you rent for patience.

Spot doesn't have these properties in the same way. You own the thing. There's no overnight financing eating a long position, no leverage converting a drawdown into a margin call, no swap for the broker to skim. The downside is bounded by what you put in. The game changes from "extract a thin timing edge faster than institutions, while leverage and financing try to kill you" to "own an asset and be right about it over a horizon where my disadvantages matter less."

## Why spot is a different game, not the same game with a smaller account

I want to be careful not to overclaim. Going spot doesn't mean I've found a way to beat the market. It means I'm *changing the objective* to one where my structural disadvantages stop being decisive.

Algorithmic intraday trading rewards exactly the things I don't have and can't buy: latency, data depth, capital, and an information edge over the price. On a five-minute bar, the only thing that matters is being faster and better-informed than the counterparty, and I am neither. Spot investing over longer horizons rewards different things — patience, position sizing, not needing to be right this minute, surviving to be right eventually. Those I can actually compete on, because they don't require beating an institution at its own microsecond game. I'm not trying to out-trade the calculator factory anymore. I'm trying to own good assets and not get shaken out.

That's the honest reframing. I'm not retreating to spot with a bruised ego and the same losing plan at lower stakes. I'm leaving a game I was structurally guaranteed to lose for one where the odds aren't stacked against my specific weaknesses.

## The decision

So, concretely:

- **Close the CFD account.** It exists to offer leverage and synthetic exposure with the spread and financing built in. I have no edge that survives those costs.
- **Close the futures account.** Same logic — leverage and a clock, against counterparties I can't beat on price.
- **Trade spot only.** Own the asset, no leverage, no financing, downside bounded by capital. Compete on horizon and patience, not latency and information.

The algorithmic system stays archived and the demo keeps logging forward data, because a real live stream is the only test set that doesn't [flatter me](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/). But I'm done risking money on price-only timing in leveraged derivatives. The seven months weren't wasted — they bought me the one conclusion that actually protects capital: knowing precisely which game not to play.

## If you're starting down this road

The thing I'd tell anyone about to build a retail algo-trading system on a liquid derivative: the question isn't "can I find a pattern in the price." You can. Patterns are everywhere. The question is "is there any reason this pattern would still pay *me*, retail, price-only, after the people whose job is to arbitrage it have had it for years." For a liquid market the answer is no. It has already been taken. What's left for you is the residue too thin to interest the people who took it — and your own [execution gap](/posts/20260613-algo-trading-backtest-live-gap-same-bar-reentry/) will eat that residue before you ever bank it.

I had to build the whole thing to believe that. Maybe you do too. But if this saves you a few months, the project paid off better for you than it did for me.
