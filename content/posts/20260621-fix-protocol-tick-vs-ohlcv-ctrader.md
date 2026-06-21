---
title: "FIX Protocol - I Built a Backtester on a Stream With No History, Then Deleted It"
date: 2026-06-21
draft: false
tags: ["fix-protocol", "ctrader", "trading", "api-design", "python", "hft"]
categories: ["Backend"]
description: "FIX is a tick-and-execution protocol with no history endpoint. I spent a refactor building candle aggregation, a backtester, and a Yahoo Finance backfill on top of it — then realized I was fighting the protocol and ripped all of it out for a pure tick strategy. Plus the flat-start crossover bug that silenced every SELL, and why the real engineering in a FIX client is the plumbing, not the alpha."
showToc: true
---

## The protocol decides the strategy, and I learned it backwards

I took a two-year-old abandoned trading bot and tried to modernize it into a clean cTrader client. The plan was ordinary: connect over the broker's FIX API, pull some price history, run an RSI strategy across two timeframes, place orders. I got the connection working, built a candle aggregator, wrote a no-lookahead backtester, even wired in a Yahoo Finance fetcher to backfill real EUR/USD bars.

Then I deleted almost all of it.

Not because it was buggy — it worked. I deleted it because I'd built it on top of a protocol that doesn't do any of that, and every line of it was a workaround for that mismatch. The lesson cost me most of a session: **know what your transport is actually for before you design the strategy that rides on it.** FIX is not a data API with a streaming bolt-on. It is a stateful, ordered, tick-driven conversation. Quotes in, orders out. The moment I stopped fighting that, the right design was obvious and small.

## What FIX actually is

FIX (Financial Information eXchange) is a plain-text, tag-based messaging protocol from 1992 that banks, brokers, and trading firms use to stream quotes and fire orders machine-to-machine. If two financial systems talk to each other about quotes and orders, odds are they speak FIX.

The thing to internalize is that it is **not** request/response. There is no `GET /candles`. You open a long-lived TCP socket (TLS in production), log in, and then a continuous, ordered stream of messages flows both ways until someone logs out. Each side keeps sequence numbers and exchanges heartbeats; a gap in the sequence triggers a resend. That machinery is the whole point — it gives FIX guaranteed, ordered, gap-detected delivery, which matters when a dropped message is a real order for real money.

Every message is a flat list of `tag=value` pairs separated by the SOH control character (`0x01`, rendered as `|` in docs):

```
8=FIX.4.4|9=120|35=D|49=SENDER|56=TARGET|34=12|52=20260620-04:00:00|...|10=157|
```

Tag `35` is the message type, and the type tells you which of two layers you're in. Session messages keep the pipe alive: `A` logon, `0` heartbeat, `1` test request, `2` resend request, `5` logout. Application messages carry the trading content: `V` market data request, `W`/`X` snapshot and incremental tick refresh, `D` new order, `8` execution report. A good client keeps these cleanly separated — the strategy should never see a heartbeat.

Notice what's in that list and what isn't. There are messages for live quotes and messages for orders. There is no message for "give me last month's candles." That absence is the entire story.

## cTrader specifics: two sessions, and no history at all

cTrader (by Spotware) exposes FIX 4.4 over **two** separate sessions you log into with the same credentials:

| Session | Purpose               | TLS port | Plain port |
| ------- | --------------------- | -------- | ---------- |
| QUOTE   | market data (bid/ask) | 5211     | 5201       |
| TRADE   | orders, positions     | 5212     | 5202       |

It tags which session a message belongs to with `50=SenderSubID` / `57=TargetSubID` set to `QUOTE` or `TRADE`, uses `TargetCompID=cServer`, and authenticates with `553=Username` / `554=Password` (the FIX password you set under cTrader → Settings → FIX API, distinct from your account login).

What this FIX interface gives you: real-time top-of-book bid/ask, market depth, order entry and execution reports, position reports, and a security list mapping instrument IDs to names. What it does **not** give you: historical bars, candles, trendbars, or tick history. None. To get OHLCV on cTrader you have to use a completely separate protobuf-based Open API and its `ProtoOAGetTrendbarsReq`. The FIX side is execution and live quotes, full stop.

This is the fact I should have led my design with instead of discovering halfway through.

## Why a timeframe strategy fights the protocol

A FIX quote stream is a sequence of ticks: `(timestamp, bid, ask)`, pushed whenever the book moves. There is no "5-minute bar" anywhere in it. Two consequences fall straight out:

1. **You cannot backtest from FIX.** No history endpoint means no historical series means nothing to replay. You can record ticks live and replay those, but you cannot ask the broker for last month's data.
2. **A static-timeframe strategy is expensive and awkward.** Want a 4-hour RSI? You either seed it from external history (now you've bolted a second data source onto your "FIX bot") or you aggregate live ticks forward for weeks until you've accumulated enough 4h bars. Both are working against the grain.

That's exactly the corner I painted myself into. The candle aggregator, the backtester, the Yahoo Finance backfill — every one of those existed to manufacture OHLCV that FIX refuses to provide. It was a lot of correct code solving a problem I had created by picking the wrong strategy for the transport.

What FIX is genuinely good at is the opposite shape: low-latency reaction to the live tick stream plus immediate execution. Scalping, market-making, HFT-style strategies, and the institutional plumbing around them (copiers, smart order routers, OMS bridges). So the honest choices are a hybrid — Open API or a data vendor for bars, FIX only for low-latency quotes and execution — or you go pure-FIX and lean all the way into a tick strategy. I went pure-FIX.

## The tick strategy that fits the grain

The replacement consumes raw ticks and needs no history. Fed one `(bid, ask)` at a time, it returns `BUY`, `SELL`, or `HOLD`. It tracks a fast and a slow EMA of the mid-price — a zero-crossing of `fast − slow` is the directional signal — plus an EMA of the spread as a microstructure guard that drops any signal fired while the book is unusually wide. A candle strategy literally cannot see that spread; a tick strategy must. It warms up from the live stream itself, so there's nothing to backfill.

```python
def update(self, bid, ask):
    mid = (bid + ask) / 2
    spread = ask - bid
    if self._fast_ema is None:                  # seed on first tick
        self._fast_ema = self._slow_ema = mid
        self._spread_ema = spread
        self._ticks = 1
        return Signal.HOLD
    self._fast_ema   += self._fast_alpha   * (mid - self._fast_ema)
    self._slow_ema   += self._slow_alpha   * (mid - self._slow_ema)
    self._spread_ema += self._spread_alpha * (spread - self._spread_ema)
    self._ticks += 1

    diff = self._fast_ema - self._slow_ema
    previous, self._prev_diff = self._prev_diff, diff
    if self._ticks < self.slow_period or previous is None:
        return Signal.HOLD                       # warming up
    crossed_up   = previous <= 0 < diff
    crossed_down = previous >= 0 > diff
    if not (crossed_up or crossed_down):
        return Signal.HOLD
    if spread > self.spread_tolerance * self._spread_ema:
        return Signal.HOLD                       # book too wide
    return Signal.BUY if crossed_up else Signal.SELL
```

The live loop is then exactly as small as it should be — subscribe, tick, decide, execute, no candles in sight:

```python
strategy = TickMomentumStrategy()
for bid, ask in stream_ticks(client, "EURUSD"):    # polls the quote, de-dupes
    signal = strategy.update(bid, ask)
    if signal is not Signal.HOLD:
        execute(client, signal, "EURUSD", 0.01)    # routes to the FIX TRADE session
```

One honest ceiling: `stream_ticks` polls the subscribed quote and de-dupes, so under load it can miss ticks. For true HFT you'd hook the FIX market-data callback directly instead of polling. I marked that in the code rather than pretending it was production-grade.

## The flat-start bug that silenced every SELL

The crossover logic above is the second version. The first one looked reasonable and was quietly broken:

```python
is_bullish = fast > slow      # a boolean, and the bug
```

From a flat start the two EMAs are seeded equal, so `fast == slow`, so `is_bullish` is `False`. A downward move keeps it `False`. The code was watching for a `True → False` transition to fire a SELL — and from a cold start that transition never happens, because it began at `False`. BUY worked; SELL was structurally dead. You could stare at this and not see it.

The fix is to track the **sign of `fast − slow` crossing zero** instead of a boolean threshold, which handles the equal-start case correctly (`previous <= 0 < diff` and its mirror). What actually caught it was a unit test feeding a deterministic downward tick sequence and asserting a SELL came out. It failed loudly and immediately. This is the case for testing strategy logic with synthetic sequences even when you can't backtest: the protocol denied me historical replay, but a hand-built tick series in a unit test still pins the behavior.

## The real work was the plumbing

Here's the part that surprised me most. The "strategy" is almost incidental. The engineering that actually mattered in a FIX client was the session machinery:

- **The constructor could hang forever.** It blocked on a security-list event that never arrives if the market is closed or the host is unreachable — a naive client just waits, silently, indefinitely. The fix was a bounded `login_timeout` that raises `TimeoutError`, closes the sockets, and re-raises instead of swallowing. A stateful protocol means you own the failure modes of a conversation, not just the failures of a call.
- **Worker threads must be daemons.** The FIX session runs background threads for heartbeats and reads. Until I made them daemon threads, a `TimeoutError` would surface but the process wouldn't exit — it sat there held open by a thread waiting on a dead socket.
- **TLS is a toggle you have to get right.** A `use_ssl` flag selects ports 5211/5212 over 5201/5202 and wraps the socket with `ssl.create_default_context()`. I validated it with a real handshake to the broker and confirmed the negotiated cipher, because "it connected" and "it connected securely" are different claims.

None of that is glamorous and all of it is where a FIX integration lives or dies. Heartbeats, sequence numbers, reconnect, bounded waits — get those wrong and the cleverest signal in the world never places an order.

## Secret hygiene is a process, not a one-time check

One more lesson, learned the embarrassing way. Early in the refactor I audited the repo for secrets, found none committed, and moved on. Later I added some "example" connection values to the README to make setup clearer — and those examples were a real demo host and account string, copied from my actual config. I'd introduced a leak *after* declaring the repo clean.

Scrubbing the live file isn't enough once something's in history; it lives in every prior commit. The fix was `git filter-repo` to rewrite the entire history, a force-push, and a grep across all commits to confirm zero matches. The password itself never leaked — it only ever lived in a git-ignored `.env` — but the host and account were enough to make me rewrite history. The takeaway: re-audit right before publishing, not just at the start, because the dangerous additions are the helpful-looking ones you make later.

## What carries over

Strip away FIX and cTrader and a few things generalize to any integration:

- **Let the transport's nature pick the strategy.** Half my churn was building OHLCV machinery onto a protocol that has no concept of OHLCV. Find out what your API is *for* before you design on top of it; a feature it doesn't have is telling you something about the design it wants.
- **Test the logic you can't replay.** No history meant no backtest, but deterministic synthetic inputs in a unit test still caught a bug that was invisible to the eye. "I can't backtest" is not "I can't test."
- **The plumbing outweighs the cleverness.** In a stateful protocol, bounded waits, daemon threads, and TLS are the difference between a toy and a tool. Budget for them up front.
- **Audit for secrets before you publish, not just before you start.** The leak I shipped came after my clean audit, disguised as documentation.

The companion code, cleaned up and pure-FIX, is public at [github.com/suyons/trading-ctrader-fix-api-example](https://github.com/suyons/trading-ctrader-fix-api-example) — a fork of `ejtraderLabs/ejtraderCT`. The most valuable commit in it is the one that deletes the most.
