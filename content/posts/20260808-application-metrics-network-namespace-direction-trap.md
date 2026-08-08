---
title: "Self-Hosted Observability - Application Metrics and the Trap That Only Bites in One Direction"
date: 2026-08-08
draft: false
tags: ["prometheus", "statsd", "jmx", "monitoring", "networking"]
categories: ["Infrastructure"]
description: "Wiring application-level metrics into an existing Prometheus stack meant discovering that a hard-won host-networking rule reverses direction for bridge-network clients, that StatsD over UDP fails silently, and that live connection counts need the kernel's socket table, not the access log."
showToc: true
---

A monitoring stack that only knows whether a service is *reachable* tells you nothing about whether it is *working*. Having previously stood up infrastructure monitoring for two document services on a single virtual machine, the remaining gap was application-level telemetry — and it had been deliberately deferred, because both halves required restarting a production service. With a maintenance window approved, this session closed that gap and then answered a follow-up question that turned out to need a completely different data source: *which customer organisations are connected right now?*

## Reconnaissance beats recollection

Notes from the earlier session recorded what phase 2 "would take". Re-verifying those notes against the live host before writing any code turned out to be the single highest-value step, because **three of the recorded conclusions were wrong**.

The document editor ships a bundled metrics daemon that was assumed to be the collection path. Reading its actual configuration showed it is a stock StatsD instance wired to the **console backend on a ten-minute flush** — it prints aggregates into a log file and forwards nothing. It was never a metrics pipeline; it was a logging one. The right move was to bypass it entirely and point the application's StatsD client straight at our own exporter.

The notes also predicted a manual configuration-file edit plus a manual service start that would need re-applying on every container recreate. Reading the image's entrypoint showed a single function that rewrites the config *and* flips the supervisor autostart flag, all driven by environment variables. So the correct integration was four environment variables on the systemd unit and nothing else. Better still, it revealed why the hand-edit approach would have quietly failed: that config file is regenerated on every start and is **not** on a persistent volume, so an in-place edit is destroyed by the next container recreate.

The lesson is not "the notes were sloppy." The notes were a reasonable inference from the outside. The lesson is that **inference about someone else's software is a hypothesis, and hypotheses are cheap to test on a live host.**

## The trap that only bites in one direction

The earlier session had established a hard-won rule: monitoring containers must join the host network namespace, because a port published on `127.0.0.1` is unreachable from the container bridge gateway. That rule is correct. It is also, applied naively here, exactly backwards.

The new collector's client is the document editor, which runs on the **bridge** network. For that container, `localhost` is the *container's* own loopback, and a collector bound to the host's `127.0.0.1` is invisible to it — the same wall, approached from the other side.

```bash
# Before — consistent with every other exporter in the stack, and receives nothing at all
--statsd.listen-udp=127.0.0.1:8125
--web.listen-address=127.0.0.1:9102
```

```bash
# After — UDP intake on the bridge gateway; the metrics endpoint stays on loopback
--statsd.listen-udp=172.17.0.1:8125    # reachable from bridge containers, not routable from the LAN
--statsd.listen-tcp=                   # empty: OFF. Otherwise a TCP listener opens on ALL interfaces
--web.listen-address=127.0.0.1:9102
```

Two things about that snippet deserve emphasis. The empty `listen-tcp` is **security configuration, not tidiness** — omit it and an unauthenticated listener appears on every interface. And this is now the stack's single deliberate non-loopback bind, which means the deployment checklist had to grow an explicit exception rather than keep asserting "everything is on loopback."

There is a nastier property here. StatsD over UDP is fire-and-forget: **a wrong destination address raises no error anywhere.** The exporter stays healthy, the scrape target stays green, every health check passes, and no metrics ever arrive. The only honest verification is the receiver's own packet counter, so that check went into the runbook as a first-class step rather than a footnote.

## Instrumenting the JVM without breaking the application

The second half attached a JMX-to-Prometheus agent to a Java service. Two decisions are worth recording.

The agent's recent releases are published on the project's release page **only** — the usual central Java package repository stops several versions back, so the habitual dependency URL returns a 404. This is the second registry surprise in this stack (an earlier one involved an image published exclusively under a variant tag). The generalisable rule: **verify that a coordinate exists before pinning it, rather than deriving it from a version number.** Ten minutes of checking beats a failed build midway through a maintenance window. Because the agent jar is injected into the JVM in the most privileged position available, its checksum is pinned and verified at build time — an unverified download there is a supply-chain hole, not a convenience.

The second decision was where to put the agent's configuration file. Mounting it from the host would allow editing rules without rebuilding the image, which is genuinely attractive. It was rejected:

```dockerfile
# After — configuration baked into the image, deliberately NOT a host mount
COPY jmx-exporter.yml /opt/jmx/jmx-exporter.yml
ENV CATALINA_OPTS="-Dapp.config=/etc/app/config.properties \
-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=9404:/opt/jmx/jmx-exporter.yml"
```

If that file were a host mount, a missing or unreadable file would make the `-javaagent` flag fail and **the application server would not boot**. A metrics configuration must never become an availability risk for the thing it observes. The cost — an image rebuild to change scrape rules — is the right trade.

A practical wrinkle: the deployment host has no build toolchain. Rather than install one, the existing application archive was extracted from the running image and only the thin top layer rebuilt. That is arguably *better* for a monitoring-only change, because the application bytes are provably identical and any behavioural change is attributable to the agent alone.

## Letting measurement overturn a design decision

The metric mapping shipped with a defensive time-to-live, on the theory that application-chosen metric names might embed identifiers and explode series cardinality. Then a real document conversion was driven through the API to see what actually arrives. Every observed name was flat and bounded — no identifiers anywhere.

```yaml
# Before — defensive, and actively harmful once the data was known
ttl: 30m
```

```yaml
# After — no expiry; the guard moved to the scrape config, where it fails loudly
sample_limit: 2000
```

On a low-traffic service a thirty-minute expiry deletes the cumulative event count *between* uses, so "how many documents were converted today" would quietly undercount — a silent wrong answer, the worst kind. Moving the guard to a scrape-level sample limit keeps the protection but changes its failure mode from silent corruption to a visibly failed target.

Equally important is what was *not* done. Several metric families remain unobserved because they only appear during interactive editing, and their names end in a separator implying a runtime-appended suffix. Whether that suffix is bounded (a file type) or unbounded (a document identifier) cannot be determined without real traffic — so no label-extraction rule was written for them, and the file records the open question plus the command to resolve it. **Guessing there would have been indistinguishable from knowing, right up until the database fell over.**

## A different question needs a different source

The closing request was: *how many users from each customer organisation are connected right now?* The instinct is to parse the web server's access log and group by source address. That instinct is wrong, and understanding why is the most transferable idea in this session.

The access log records a line when a request **ends**. Collaborative editing holds a long-lived connection open for the entire session — so a connection that is active *right now* has produced no log line at all. Log-based counting is structurally blind to exactly the connections the question is about. The kernel's socket table is the only place an in-progress connection exists, so the collector reads that instead and publishes gauges through the metrics agent's file-based collector.

Three implementation details are load-bearing rather than incidental:

- **Unmapped sources collapse into a single bucket.** The public port is scanned continuously by internet background noise; emitting a series per unknown address would be unbounded cardinality. A separate counter of *distinct* unmapped addresses provides the signal that a legitimate office address is missing from the table.
- **Known-but-idle groups emit an explicit zero.** A series that simply vanishes is indistinguishable from a broken collector on a graph.
- **The output file is published atomically** via a rename. The collector parses each file whole, so a half-written file fails *every* file-based metric, not just this one.

And the honest caveat, which went into the dashboard description rather than being quietly omitted: TLS virtual hosts share a single port and are separated by a handshake extension that lives *above* the transport layer. So this counts edge connections, not connections to one specific service. For customer organisations the difference is negligible; for our own address it is not, because the monitoring UI is served from the same port. A metric with a documented limitation is far more useful than one that overstates its own precision.

## Outcome and takeaways

Both halves are deployed and verified: every scrape target healthy, all probes passing, five dashboards provisioned from version control, every metrics endpoint on loopback with the one documented exception, and all units enabled to survive a reboot. Application metrics were confirmed with a **real conversion** rather than a green target, and the connection classifier was validated end-to-end — including holding open connections from an unmapped source to prove the aggregation bucket works and then decays back to zero. Application-layer instrumentation cost about 29 MiB of memory in total.

Four lessons worth carrying:

- **Verify inherited notes before building on them.** Three recorded conclusions about third-party software were wrong in ways that would have produced a working-looking but broken integration.
- **A rule about network namespaces has a direction.** "Bind loopback" and "join the host namespace" are the right answers only for consumers that live in the host namespace; invert the roles and the same reasoning demands the opposite bind.
- **Prefer loud failure to silent correctness loss.** A limit that takes a target down beats a time-to-live that quietly deletes counts; a packet counter beats a health check that is green regardless.
- **Pick the data source from the question's shape.** "How many *right now*" is a concurrency question, and logs — which are written at completion — cannot answer it in principle, no matter how carefully they are parsed.
