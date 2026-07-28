---
title: "Cloud Cost Optimization - Why 'Just Move It to Lambda' Wasn't the Answer"
date: 2026-07-28
draft: false
tags: ["cloud-costs", "aws-lambda", "serverless", "finops", "capacity-planning"]
categories: ["Infrastructure"]
description: "After terminating a batch of unused cloud resources, forecasting next month's bill led to the obvious follow-up question — would moving the Node.js services to Lambda be cheaper? Compute said yes by a wide margin. The bill said no."
showToc: true
---

After terminating a batch of unused cloud resources mid-month, the question was simple: what will next month's bill actually be? The follow-up question was the interesting one — the remaining servers run Node.js under PM2, so would converting them to serverless functions be cheaper? The compute math said yes, emphatically. The bill said no. The gap between those two answers is the whole story.

## Reading a partial-month billing export

The billing export was a mid-month snapshot — services terminated on the 27th, so every line covered only the first 26 days. Two details make or break a forecast built from a file like this:

- **Two different usage units coexist in one file.** Hourly-billed items showed 624 hours (26 days × 24h); monthly-billed items showed "26 days / 31 days" instead. You can't scale both with the same multiplier — hourly items scale to 744 hours (a full month), monthly items go to full price with no scaling at all. Treating them the same silently over- or under-counts a chunk of the bill.
- **The provider rounds down to the nearest unit per line item**, not per invoice. With roughly 25 line items that's a small but systematic difference, and reproducing it is what makes the estimate reconcile against next month's actual invoice instead of quietly drifting off by a few percent.

So the forecast has to be a per-line recomputation — unit price times full-month usage — not a blanket `× 31/26` multiply of the total:

```python
H = 744                              # full month, in hours
round_down = lambda x: int(x // 10 * 10)   # provider rounds down per line, to the nearest 10

round_down(237 * H)   # hourly VM  -> scale hours, then round
170_240                # monthly VM -> full monthly rate, no scaling
```

Only genuinely usage-based lines — outbound traffic, load-balancer ingress — get proportional scaling. Everything else is a fixed rate and lands within rounding error of the next invoice. Worth flagging separately in the writeup: only the usage-based lines carry real forecast variance.

The terminated resources turned out to be 34.8% of the prior bill, dominated by a single standby database instance whose license line alone was larger than its compute line.

## The serverless question: right math, wrong denominator

The Node.js services run under PM2 on always-on VMs. Serverless functions bill per request plus per GB-second, so for a bursty internal app the compute cost should collapse. Does it?

For a 4 vCPU / 16 GB VM, the AWS Lambda break-even looks like this:

| Function memory | Cost per busy hour | Break-even |
|---|---:|---|
| ~7,076 MB (≈4 vCPU) | ~580 KRW | 304 h/month (41% duty cycle) |
| ~1,769 MB (≈1 vCPU) | ~145 KRW | 1,215 h/month — exceeds the month |
| 512 MB (typical API) | ~42 KRW | 4,198 h/month — exceeds the month |

*(Lambda unit pricing changes over time and by region — treat the KRW figures above as a snapshot for this comparison, not a current quote.)*

Expressed as traffic instead of duty cycle: a 512 MB function averaging 200ms per request breaks even at roughly 66 million requests a month — about 25 requests per second, sustained, all month. An internal document-management system doesn't do that. At a realistic million requests a month, the Lambda compute bill would be a rounding error next to everything else.

So serverless wins the compute comparison by a wide margin. And it still wouldn't have helped, for three reasons.

**1. Compute was only 41% of the bill.** Databases, VPN, load balancers, network storage, snapshots, backups, and OS licenses were the other 59% — completely untouched by how the application code is packaged. Eliminating every VM caps the possible saving at 41%, and you can't actually reach that cap (see reason 2). Work out the ceiling before evaluating whether an optimization is worth building.

**2. Not all the "servers" were Node.js.** Of four VMs, one ran the Node app on Linux. Two ran Windows Server hosting a document-management application — paying a per-month OS license, doing file handling and document conversion that runs straight into Lambda's timeout and payload-size limits. The fourth ran a licensed document-editing product that can't be repackaged as a function at all. The addressable target was one VM out of four.

**3. Migrating across providers adds costs that didn't exist before.** The workload runs on a non-AWS cloud provider; Lambda is AWS. Putting a function inside a VPC so it can reach private resources requires a NAT gateway — a *fixed* monthly charge on the order of a third of the VM it replaces, payable even if the function never executes. And if the database stays where it is, every query now crosses the public internet between two clouds, which for a document-management system is a design failure before it's a performance problem. Moving the database too isn't a Lambda migration anymore; it's a cloud migration.

The net trade on offer: take on a fixed NAT charge, a VPN build-out, and cold-start/connection-pool problems, to eliminate a variable cost that was already a fraction of the total bill.

## What actually moved the number

The changes that came out of this were dull and effective, ordered by return on effort:

- **Committed-use discounts.** The discount column was empty across every line in the export. Machines that had been running continuously for six months to two years were all on full on-demand pricing. A one-year commitment is a console change with zero code risk, and it beats the entire serverless project on savings alone.
- **Attack the license, not the compute.** The database's SQL Server license line was larger than the database's compute line — roughly a quarter of the total bill in a single row. If the application isn't deeply bound to vendor-specific SQL features, an engine migration has an order-of-magnitude better return than repackaging the app tier.
- **Right-size before re-architecting.** One VM had been bumped from 2 to 4 vCPU weeks earlier. Checking whether that was actually necessary took five minutes and was worth half the monthly saving of the entire serverless proposal.
- **Sweep the orphans.** Six public IP addresses for four servers, and four old snapshots still billing monthly. Unattached resources bill exactly like attached ones.

## Outcome and takeaways

The forecast landed at roughly a 35% month-over-month reduction from the terminations alone, reconciling line by line against the source export. The serverless proposal was declined with numbers attached, not an opinion.

- **Compute the ceiling before designing the solution.** Work out the largest amount an optimization *could* save before evaluating how much it actually saves. If the addressable target is 41% of the bill and only a quarter of that is reachable, you know the answer before doing any architecture work.
- **Serverless pricing is a duty-cycle question.** Per-GB-second billing runs roughly 2–3× the price of always-on compute while it's actually running. That's a bargain at 5% utilization and a penalty at 60%. Convert the comparison to a break-even duty cycle or request rate — one number that's obviously true or false beats a spreadsheet.
- **Serverless has a fixed-cost floor, and it isn't zero.** "Pay only for what you use" stops being true the moment a function needs private network access. Price the NAT gateway and the egress before pricing the invocations.
- **The cheapest optimization is usually a billing setting, not an architecture.** Unused commitment discounts, oversized instances, and orphaned resources routinely outweigh anything achievable by rewriting code — with none of the engineering risk.
- **In licensed stacks, follow the license.** When a per-core database license costs more than the hardware it runs on, that line is the optimization target. Nothing done to the application tier competes with it.
