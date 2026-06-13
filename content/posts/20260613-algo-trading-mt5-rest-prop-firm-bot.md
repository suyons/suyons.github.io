---
title: "Algorithmic Trading - Building a Gold Bot on a Broker's REST API"
date: 2026-06-13
draft: false
tags: ["trading", "mt5", "python", "lightgbm", "quant", "backtesting"]
categories: ["Trading"]
description: "I spent seven and a half months and 392 commits building an algorithmic trading system for gold. It doesn't work. This is the first post in an autopsy of how I built it, what the prop-firm rules did to the design, and why a REST API in front of MetaTrader was the right call."
showToc: true
---

## The punchline first

I spent about seven and a half months — 392 commits — building an algorithmic trading system for XAUUSD (spot gold). The final verdict, written in my own project notes, is one word: **DEAD**. The live edge measured 41.2% win rate against a 44.4% breakeven. It loses money.

I'm writing the whole thing up anyway, in a series, because the interesting part of a trading project is never the strategy that worked. It's the parade of things that looked like they worked and didn't, and the specific reasons each one fell over. Most of those reasons are software-engineering reasons, not finance reasons — lookahead bias in a backtest, a train/serve skew in a model, a sequence-generation artifact that made the backtest 10 points luckier than reality. Those generalize.

This first post is the setup: what I built, why the architecture looks the way it does, and the constraint that shaped every later decision. The later posts are the failures, one per post.

## The constraint that drives everything: prop-firm rules

I was building against a **prop-firm funded account**: $100,000 of simulated capital, with two hard rules.

- **Daily drawdown limit: -5%.** Lose 5% in a day and the account is dead.
- **Total drawdown limit: -10%.** Equity can never fall below $90,000, ever.

If you've only ever optimized for return, this inverts your whole objective. The binding constraint is not "how much do I make" — it's "can I be *certain* I never breach a drawdown floor." A strategy that returns 9%/week but has a 1-in-50 chance of an 11% drawdown week is worthless here: that one week ends the account. A strategy that returns 3%/week and provably never draws down past 7% is the one you want.

This means **drawdown is the design target, not a side effect.** Almost every risk mechanism I built later — circuit breaker, weekly equity governor, rolling-peak governor, direction-aware position pools — exists to put a hard ceiling on drawdown, accepting a return haircut in exchange. The prop rules turn risk management from hygiene into the actual product.

## Why a REST API in front of MetaTrader

The broker runs **MetaTrader 5 (MT5)**. MT5's native automation story is MQL5 (a C-like language that runs inside the terminal) or the Python package, which only talks to a terminal running on the same Windows host. Both pin your strategy code to the broker's platform and OS.

I didn't want that. So the trading platform sits behind a small **HTTP REST API** on a LAN box, and everything I wrote talks to it over plain HTTP:

```
GET /copy_rates_from_pos/{symbol}?timeframe=5&start_pos=1&count=500
GET /copy_rates_range/{symbol}?timeframe=5&date_from=...&date_to=...
GET /positions_get?symbol=XAUUSD
GET /symbol_info_tick/XAUUSD
GET /account_info
GET /history_deals_get?position=<ticket>
```

(The host is a private box on my network; treat the address as `http://<broker-host>:8000`.)

Three things this bought me, all of which mattered later:

1. **The strategy runs anywhere.** Python on Linux, scheduled however I like, no Windows terminal in the loop.
2. **It's mockable.** A REST boundary is trivial to stub. Every backtest and reconciliation tool talks to the same `MT5API` client class, pointed at either the live box or recorded data.
3. **One source of market truth.** Live trading and backtesting both fetch bars through the exact same endpoint, so "the data the backtest saw" and "the data live saw" are the same bytes. When I later hunted the backtest-to-live gap, I could rule data mismatch out in one line: live OHLC matched fetched OHLC to 0.000.

The cost is that the API is **single-threaded**. You fetch sequentially, in chunks, with backoff — never fan out parallel requests at it. That's a real constraint when you're pulling years of history for 20 instruments, and it bit me during the later research phase.

## The trading loop

The live system is an **M5 bar-close loop**: gold, five-minute bars. On each closed bar it fetches a snapshot (balance, open positions, the latest OHLCV), computes signals, sizes positions, and sends orders. No intrabar decisions — everything keys off completed bars, which is the single most important rule for not fooling yourself (more on that in the lookahead post).

A few platform details that cost me real debugging time and are worth knowing if you ever touch MT5:

- **Broker time is UTC+3.** Every date the platform hands you or expects — bar timestamps, the bounds on a history query, the timestamp on a closed deal — is in broker wall-clock, not UTC. Query the last few hours of deals with a UTC `date_to` and you'll silently under-fetch by three hours and reconstruct the wrong account state. I learned this the way everyone does: a balance calculation that was wrong by exactly one offset.
- **Order comments truncate to 16 bytes.** I tag each order with the system that opened it, so I can recover the open book after a restart. MT5 silently truncates the comment field, which orphaned any system with a long name on restart. The fix was a canonical 16-byte id (`XAU-S{N}`) hard-capped at send, never relying on the platform to slice it for me.
- **Closed-trade PnL** is the sum of `profit + commission + swap` across the deals for a position, found by querying `history_deals_get?position=<ticket>`, not by filtering a date range. Date-range filtering misses deals that closed just outside your window.

None of this is glamorous. All of it is the difference between a bot that recovers cleanly from a restart and one that double-counts its own positions.

## The naive beginning

The first two months (October–December 2025) were the standard beginner arc, and the commit log is honest about it: a Donchian channel breakout, an EMA-crossover, an RSI/Bollinger mean-reversion scalper. I wired up trailing stops, break-even logic, slippage simulation, position recovery. Useful plumbing, no edge.

The very first real fix in the repo, dated 2025-10-29, is titled *"solve lookahead bias in backtesting."* I'd love to say that early scar inoculated me. It did not — six months later I produced a backtest claiming +9%/week that turned out to be reading the future through a timeframe filter, and had to throw out 18 sweeps. That's the next post.

## What the system actually became

By the end it was two subsystems sharing one repo:

- The **live M5 ML champion**: 25 signal functions (momentum and mean-reversion) feeding a LightGBM model that sizes each position by predicted win probability, with the governor stack and pool caps on top. This is the thing that went live and died.
- A **research subsystem**: a renko/tick engine and a pile of experiments for hunting edge in other instruments and data sources. All honest negatives.

The architecture held up fine. A clean REST boundary, one data path for live and backtest, bar-close discipline, atomic state files that survive restarts — none of that is what killed the project. What killed it was that the edge was real but **thin** — about two points of win rate over breakeven — and thin edges don't survive the gap between a backtest and a live account.

The rest of this series is about that gap, told one failure at a time:

- The +9%/week backtest that was reading the future.
- The honest backtest that still beat live by ten points, and the one-line cause.
- A model trained on one stop loss and traded on another.
- Every uncorrelated edge I went looking for, and why only one survived.
- The kill rule I wrote before I needed it, so I'd actually pull the trigger.
- The conclusion I'm acting on: closing the derivatives accounts and trading spot.

If you only read one, read the third. The same-bar re-entry bug is the most general lesson in the whole project, and almost nobody's backtest controls for it.
