---
title: "Search Ads Integration - The Easiest Keyword to Rank Was the Most Expensive to Bid On"
date: 2026-07-22
draft: false
tags: ["python", "api-integration", "seo", "ppc", "naver", "hmac"]
categories: ["Backend"]
description: "Building a budget-capped keyword scanner against Naver's Search Ad estimate API for a small clinic's paid-search account, and the moment organic-SEO intuition (low competition, low cost) turned out to be backwards for paid auctions (low competition, high cost)."
showToc: true
---

A small traditional-medicine clinic wanted a Naver Power Link (paid search) campaign, capped at roughly 10,000 KRW a day, 2,000 KRW max per click, with one goal: land in the mobile top 5, because position 6 and below falls to page 2 and might as well not exist. Organic keyword research had already turned up a promising angle — a diagnosis label specific to traditional East Asian medicine, with no real equivalent in mainstream biomedical terminology. Almost no one competes for it organically; the handful of clinics that use the term at all *are* the competition. Cheap to rank for, the SEO research concluded. I assumed that meant cheap to bid on too. It didn't.

## Two different APIs, two different questions

Naver's Search Ad platform has a `/keywordstool` endpoint for search volume and an entirely separate `/estimate/average-position-bid/keyword` endpoint for what it actually costs to hold a position. Volume research doesn't tell you price. I only found this out because the volume numbers alone weren't enough to plan a budget — I needed the bid ladder, not just the search count.

Querying that estimate endpoint for the head diagnosis term and its obvious variants:

| Keyword (translated) | Mobile rank-5 bid |
|---|---|
| diagnosis label (bare) | 5,550 KRW |
| diagnosis label (general form) | 5,640 KRW |
| diagnosis label + "clinic" | 6,290 KRW |
| general symptom + "clinic" | 5,850 KRW |
| regional generic "clinic" | 8,520 KRW |

Every one of them blows the 2,000 KRW ceiling before you even reach position 5. The organic-SEO read — few competitors, therefore cheap — was true for organic and false for paid. It's true for the *opposite* reason: because so few clinics use this diagnosis label, the ones that do bid hard for it, since there's no volume of competitors to spread the auction thin. Low organic competition and low paid-auction competition are not the same fact, and I'd been treating them as one.

Compare the long tail:

| Keyword (translated) | Mobile rank-5 bid |
|---|---|
| self-check for diagnosis label | 70 KRW |
| diagnosis label + symptoms | 1,530 KRW |
| generic symptom + "clinic" | 70 KRW |
| treatment-modality "efficacy" term | 70 KRW (position 1 costs ~800 KRW) |

The efficacy-related long-tail term was the standout: rank 5 at 70 KRW, and *position 1* only cost around 800 KRW — less than half the daily ceiling, for the top spot. Budget wasn't the constraint on the long tail at all. It was never going to be spent on the head term regardless.

## Why POST doesn't change the signature

The estimate endpoint is a POST, unlike the GET-based `keywordstool`. I expected that to mean a different signing rule. It doesn't — Naver signs the timestamp, method, and path, but never the query string or body, so the existing signing helper needed no changes:

```python
assert sign("sec", "1700000000000", "POST", ESTIMATE_PATH) == base64.b64encode(
    hmac.new(
        b"sec", f"1700000000000.POST.{ESTIMATE_PATH}".encode(), hashlib.sha256
    ).digest()
).decode()
```

The signed message is just `f"{timestamp}.{method}.{path}"`. A POST body full of keywords never enters the signature at all — worth knowing before you go looking for a body-hashing step that isn't there.

The request/response shape (excerpted — `credentials()`, `sign()`, and `parse_count()` are existing helpers elsewhere in the script):

```python
def request_bids(keywords, device, positions, opener=None):
    """Look up the average bid for each (keyword, position) pair.

    device: 'PC' | 'MOBILE'. Returns {keyword: {position: bid_krw}}.
    Power Link shows up to position 10 on PC, 5 on mobile, before falling
    to page 2 - positions only needs to cover 1-5.
    """
    api_key, secret_key, customer_id = credentials()
    timestamp = str(round(time.time() * 1000))
    items = [{"key": kw, "position": p} for kw in keywords for p in positions]
    body = json.dumps({"device": device, "items": items}).encode()
    request = urllib.request.Request(
        f"{API_HOST}{ESTIMATE_PATH}",
        data=body,
        method="POST",
        headers={
            "Content-Type": "application/json; charset=UTF-8",
            "X-Timestamp": timestamp,
            "X-API-KEY": api_key,
            "X-Customer": customer_id,
            "X-Signature": sign(secret_key, timestamp, "POST", ESTIMATE_PATH),
        },
    )
    with (opener or urllib.request.urlopen)(request, timeout=30) as response:
        payload = json.loads(response.read().decode())
    # The estimate array comes back in request order; some responses omit
    # the key, so fall back to the request item's own key.
    result = {}
    for item, est in zip(items, payload.get("estimate", [])):
        keyword = est.get("key", item["key"])
        position = est.get("position", item["position"])
        result.setdefault(keyword, {})[position] = parse_count(est.get("bid"))
    return result
```

One more constraint that only shows up empirically: the endpoint accepts at most 200 items per request — 300 returns a 400 — so real calls batch in groups of 100 keywords.

## Scanning 4,992 keywords without pricing all of them

Volume research had already produced a cache of 4,992 candidate keywords. The question became: which of those are affordable at all? Pricing every keyword's full 1-through-5 ladder would mean five bid lookups per keyword. Instead, the scan runs in two phases, using one invariant: rank 5 is the *cheapest* of the five positions Power Link shows on mobile, so if you can't afford rank 5, you can't afford 1 through 4 either.

```python
def budget_scan(ceiling=2000, min_mobile=30, batch=100):
    candidates = [
        k for k, r in src.items()
        if _int(r["monthly_mobile_search_count"]) >= min_mobile
        and ALLOW_RE.search(k) and not NOISE_RE.search(k)
    ]

    # Phase 1: cheap screen - one price point per keyword (rank 5 only).
    rank5 = {}
    for i in range(0, len(candidates), batch):
        chunk = candidates[i : i + batch]
        for keyword, ladder in request_bids(chunk, "MOBILE", (5,)).items():
            rank5[keyword] = ladder.get(5, 0)
        time.sleep(1.1)
    survivors = [k for k in candidates if 0 < rank5.get(k, 0) <= ceiling]

    # Phase 2: only survivors get the full 1-4 ladder filled in.
    ladders = {k: {5: rank5[k]} for k in survivors}
    kw_per_req = max(1, batch // 4)
    for i in range(0, len(survivors), kw_per_req):
        chunk = survivors[i : i + kw_per_req]
        for keyword, ladder in request_bids(chunk, "MOBILE", (1, 2, 3, 4)).items():
            ladders[keyword].update(ladder)
        time.sleep(1.1)
    return ladders
```

The whitelist (`ALLOW_RE`) and blacklist (`NOISE_RE`) filters in phase 1 do the real work. Sorting the raw 4,992 by click volume alone pulls in whatever the seed-expansion happened to surface — supplement brands, unrelated symptom terms, rival clinics' brand names, generic diet content — because the original volume research cast a wide net on purpose. A pure blocklist couldn't keep up with how broad that net was, so the filter is allow-list first (does this token belong to the clinic's actual specialty clusters), then subtract noise (product/commerce/off-specialty terms) from what's left.

One token was genuinely ambiguous: a diagnosis-adjacent word that's also, spelled identically, a common household noun unrelated to medicine. Allowing it as a bare token pulled in florist and home-decor keywords. Dropping it from the whitelist and relying on the compound form (the term plus "clinic") kept the medically relevant matches without the noise — a reminder that a whitelist built from single tokens breaks the moment one of those tokens has an unrelated everyday meaning.

The filter took the 4,992 down to 1,221 relevance candidates, and the rank-5 price screen took those down to 626 that clear the budget ceiling at all.

## Ranking what's left

626 affordable keywords is still too many, and not all of them match the clinic's actual specialty — plenty are affordable but off-theme. The final cut ranks by an ROI score: `intent_tier x sqrt(monthly_mobile_clicks) / log10(rank5_bid)`, where tier is a manual 2-5 scale (5 = core diagnosis and self-check terms, 4 = symptom/cause/treatment terms, 3 = treatment-modality terms, 2 = adjacent conditions with heavier mainstream-medicine competition). Top of the resulting list, translated:

| Rank | Keyword (translated) | Monthly mobile searches | Monthly clicks | Rank-5 bid | ROI |
|---|---|---|---|---|---|
| 1 | pit-of-stomach pain | 19,700 | 41.0 | 70 | 13.9 |
| 2 | reflux esophagitis symptoms | 35,800 | 488.2 | 1,820 | 13.6 |
| 3 | heartburn cause | 5,870 | 99.8 | 970 | 13.4 |
| 4 | gastritis symptoms | 21,200 | 109.6 | 1,360 | 13.4 |
| 5 | excess stomach acid symptoms | 2,550 | 62.2 | 640 | 11.2 |

The square root on clicks keeps a single high-volume term from dominating the ranking outright; the log on price penalizes anything that eats budget fast without proportionally more traffic.

## What this doesn't solve

The head diagnosis term — the single best intent match in the whole exercise — never makes the final list. At the 2,000 KRW ceiling it's simply not affordable, and no amount of relevance scoring changes that. Raising the ceiling for that one term specifically was left as a judgment call for the client, not something the tool should decide, since spending a large fraction of the daily budget on a single click has its own risk.

A hyper-local branded term (clinic name plus neighborhood) was deliberately left out of the list even though it's cheap on some rows and expensive on others — that kind of local-intent search is Naver Place's territory (the free local map/business listing), not Power Link's. Paying for a branded local search someone would have found for free isn't a keyword-selection problem the scoring formula should try to solve.

And realistically, the daily budget won't be fully spent most days — long-tail terms have low volume by definition. That's the intended outcome, not a shortfall: staying cheaply on page 1 beats occasionally overpaying for page 2 exposure on a term with more searches.

## Worth double-checking

The "POST signs the same as GET, no query, no body" behavior and the 200-item batch ceiling are both reverse-engineered from calling the live API, not from published documentation I can point to — if you're building against this endpoint yourself, verify current behavior rather than trusting this account to still hold.

## Takeaways

- Low organic-search competition and low paid-auction competition are separate facts. A rare term can be *more* expensive to bid on precisely because so few advertisers are fighting for it — each one bids harder, not softer.
- When an API call is expensive (rate-limited, batched, or just slow), find the cheapest single data point that can eliminate most candidates before paying for the full picture on all of them.
- A relevance whitelist built from single-word tokens breaks on the first homograph. Compound-token matching is the fix, not a bigger blocklist.
- Not every affordable, high-intent keyword belongs in a paid campaign — some intent is already served for free by a different product entirely, and paying for it anyway isn't a targeting error the scoring model should paper over.
