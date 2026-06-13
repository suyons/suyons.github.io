---
title: "Algorithmic Trading - The +9%/Week Backtest That Was Reading the Future"
date: 2026-05-26
draft: false
tags: ["trading", "backtesting", "lookahead-bias", "quant", "python"]
categories: ["Trading"]
description: "Over one weekend I ran 18 parameter sweeps that pushed a gold strategy from +5%/week to +9%/week, each one a new record. Every single one was invalid. A timeframe direction filter was reading up to 235 minutes into the future. Here's exactly how the leak worked and how to catch it."
showToc: true
---

## A weekend of new records

There's a specific feeling when a backtest keeps getting better. You add a filter, the weekly return ticks up and the drawdown ticks down. You tighten a threshold, it improves again. By the eighteenth iteration you're not testing hypotheses anymore, you're chasing a number, and the number is telling you you're a genius.

My commit log from one Tuesday in May 2026 reads like a ladder:

```
sweep20: H1(N=1) filter breakthrough — WR=58.8%, MDD=3.37%
sweep21: H1+H4 dual filter WR=65.5%; +5%/wk milestone
sweep26: H1+M30 filter — ratio=3.08x, +7.11%/wk
sweep29: +8%/wk milestone — risk=0.045 achieves +8.04%/wk
sweep31: +9%/wk achieved — risk=0.090 +9.07%/wk, P>10%=0.00%
```

A 65.5% win rate. Nine percent a week with, supposedly, zero probability of breaching the drawdown limit. On a $100k account that's the kind of number that makes you start drafting the email to the prop firm.

It was all fake. Every sweep from 18 through 35 was invalid, and the reason is the most classic backtesting bug there is, dressed up well enough that it took me a day to see it.

## The seductive filter

The strategy traded gold on 5-minute bars. The "breakthrough" was a **higher-timeframe direction filter**: only take a long on the M5 chart if the higher timeframes — H1 (hourly), M30, H4 (four-hour) — also point up. Only short if they point down. Intuitively great. You're aligning your fast entries with the slow trend, and that *should* raise your win rate.

And it did, enormously. The problem is in the word "point up," and specifically in *when you're allowed to know it*.

Here's the leak in its simplest form. Say it's 10:05 and an M5 bar just closed. I want to know the direction of the current H1 bar — the one that runs 10:00–11:00. So I look at that H1 bar's open and close:

```python
# WRONG — this is the bug
h1_dir = np.sign(h1["close"] - h1["open"])
```

The H1 bar from 10:00–11:00 does not *close* until 11:00. Its `close` value, at 10:05, is the price an hour from now. By filtering my 10:05 trade on `h1_dir`, I'm only taking trades that agree with where price will be up to 55 minutes later. Stack H1, M30, and H4 together and the worst case is a filter peeking **up to 235 minutes into the future** before deciding whether to enter.

Of course the win rate was 65%. I was placing trades that I'd already confirmed were going to win. A coin that you flip and then are allowed to bet on after it lands comes up heads exactly as often as you like.

## Why it hid for a full day

What makes this bug nasty is that it doesn't look like cheating. The code is reading a `close` column that exists in the dataframe. There's no `shift(-1)`, no obvious peek at `bars[i+1]`. The lookahead is *semantic* — the H1 bar's row exists in your data the moment its timestamp does, but the `close` field inside it isn't knowable until the bar completes. Pandas will happily hand you a "current" H1 row whose close is the future.

It also passed every sniff test I had at the time. The equity curve was smooth. The improvement was monotonic and physically plausible (trend alignment is a real effect). Out-of-sample held up — because the leak is in every sample, in-sample and out, equally. A leak that's present in your test set too will sail through a train/test split looking pristine.

## The fix, and the number that exposed it

The fix is a one-liner, and it's the single most important line in any multi-timeframe backtest:

```python
# RIGHT — only use the LAST COMPLETED higher-timeframe bar
h1_dir = np.sign(h1["close"] - h1["open"]).shift(1)
```

`shift(1)` says: at any M5 bar, you may only use the higher-timeframe bar that has *already closed*. At 10:05 you're allowed to know the direction of the 09:00–10:00 H1 bar, which is finished and real. Not the one still forming.

When I applied that shift and reran, the +9%/week champion didn't get a little worse. The entire family of "breakthroughs" evaporated. The fixed timeframe direction filter, used honestly, *hurt* the strategy — win rate dropped to **42.5%**, below the breakeven I'd later pin at 44.4%. The filter had never been adding skill. It had been adding a peek at the answer.

I tossed sweeps 18 through 35 and rebuilt the baseline from scratch.

## The general defense: the lag-robustness check

Catching this once by reading code is luck. You want a mechanical test that flags it regardless of how cleverly it's hidden. The one I adopted, and now run on everything, is a **lag-robustness check**:

> Re-run the entire backtest with one extra bar of lag inserted everywhere a signal feeds a decision. If the results are roughly unchanged, the strategy isn't depending on information it shouldn't have. If a one-bar lag craters it, you're leaking.

Concretely: valid if the win rate moves by less than 3 percentage points under the extra lag. A real edge degrades gracefully when you make it act a little later. A lookahead leak collapses, because you've taken away the future it was feeding on. The +9%/week filter would have failed this instantly — that's the whole point of running it.

This is now a fixed convention in the project, written into the engine notes:

```
# sig[j-1] fires at bar j-1 close → entry at opens[j]   (NO lookahead)
# Bias check: rerun with 1-bar extra lag; valid if |ΔWR| < 3pp
```

Signal computed on the bar that closed at `j-1`, order filled at the open of bar `j`. Never the same bar the signal is computed on. Never a higher-timeframe value that hasn't closed.

## What it cost and what it taught

The direct cost was a weekend of sweeps in the bin and a deflating Tuesday. The lasting cost was learning to distrust my own excitement. The tell, in hindsight, was the monotonic improvement: real edges are lumpy and grudging, they give back as much as they give. A filter that improves *every* metric *every* iteration isn't a filter, it's a leak.

The thing I'd underline for anyone building a backtest: **lookahead bias does not look like a bug.** It looks like success. It produces clean code, smooth curves, and out-of-sample numbers that hold, because the leak contaminates every split equally. The only reliable defense is to assume your best result is cheating until a lag test says otherwise — and to be most suspicious exactly when the number is best.

The next post is about the opposite and more unsettling problem: a backtest that had *no* lookahead, that I'd hardened against exactly this kind of leak, and which still beat my live account by ten points of win rate. That gap turned out to have a single dominant cause, and it's one almost no backtest controls for.
