---
title: "Self-Hosted Observability - Bolting Prometheus and Grafana onto a Single VM, No docker-compose"
date: 2026-08-08
draft: false
tags: ["prometheus", "grafana", "docker", "monitoring", "systemd"]
categories: ["Infrastructure"]
description: "Adding a Prometheus and Grafana stack to a single VM as individual systemd-managed containers turns up a host-networking trap that silently drops every exporter, a NAT hairpin that can't be probed the obvious way, and a case for hand-written dashboards over vendored ones."
showToc: true
---

We had two document services — an office-suite editor and a document-comparison backend — running as systemd-managed Docker containers on one virtual machine behind an nginx TLS edge, and no visibility into any of it. When something went wrong the only recourse was tailing logs. The goal was an APM dashboard on its own subdomain, locked to a single admin IP address, with one hard constraint from the outset: **no docker-compose**. The existing deployment models every container as its own systemd unit, and the monitoring stack had to look like it belonged.

## Requirements before code

The first move was not to write anything. "Add Prometheus and Grafana" hides a dozen decisions that are expensive to reverse once dashboards exist, so the session opened with structured questions: how literally to take "one more container" (Prometheus and Grafana ship as separate images); which exporters; retention; whether alerting was in scope; and whether dashboards should be code or clicked together in the UI.

That conversation produced the shape of the whole thing — six containers, ninety-day retention with a size cap, dashboards provisioned from the repository, and alerting configured but deliberately inert. It also surfaced a fact no amount of code-reading would have: an entire environment tree in the repo (a separate edge configuration for a second deployment) was dead and could be deleted.

## The decision that mattered: host networking, not published ports

The instinct when adding an exporter is to publish its port on loopback, mirroring how the application containers are already exposed:

```bash
# Before — the obvious approach, and it silently breaks everything
docker run --name node-exporter \
  --publish 127.0.0.1:9100:9100 \
  prom/node-exporter
```

This produces a Prometheus full of `connection refused`. A container reaches its host through the Docker bridge gateway, and **a port published on `127.0.0.1` is not reachable from that gateway address**. The scraper cannot see the exporters, and the uptime prober cannot reach the application containers on their own loopback ports either.

The two usual escapes are both worse. Publishing on `0.0.0.0` and leaning on a firewall exposes unauthenticated metrics to anything that can route to the box — and Docker's published ports bypass host firewall rules anyway. A user-defined bridge network fixes container-to-container traffic but still cannot see host-level `/proc`, nor the loopback-bound application containers.

```bash
# After — share the host network namespace, bind loopback explicitly
docker run --name node-exporter \
  --network host \
  --pid host \
  --mount type=bind,src=/,dst=/host,ro,bind-propagation=rslave \
  prom/node-exporter \
  --web.listen-address=127.0.0.1:9100 \
  --path.rootfs=/host
```

Every container in the stack joins the host network namespace, and every process is told explicitly to bind loopback. The consequence worth writing on the wall: with host networking there is no port mapping, so a missing or wrong listen-address flag silently binds all interfaces. **Those flags are security configuration, not tuning.** The deployment checklist now ends with a socket listing to prove nothing escaped.

## Probing public endpoints from a host that cannot reach itself

The cloud provider gives no NAT hairpin: the VM cannot connect to its own public IP address. A blackbox probe pointed at the real public URL would time out forever and page someone about a perfectly healthy service.

The fix is the prober's equivalent of `curl --resolve` — connect to loopback while overriding both the TLS server name and the HTTP `Host` header, so the request still traverses the genuine certificate, virtual host selection and proxy path:

```yaml
https_edit:
  prober: http
  http:
    valid_status_codes: [200]
    tls_config:
      server_name: edit.example.com # SNI: selects the cert, and validates against it
    headers:
      Host: edit.example.com # selects the nginx virtual host
```

A neat consequence: the admin dashboard's own virtual host allowlists a single client IP, so probing it from the VM returns `403`. Rather than fight that, the module treats **403 as the success condition**. A `200` would mean the allowlist had stopped filtering; a timeout would mean the service is down. The probe verifies the security control, not just liveness.

## Two bugs caught before they shipped, one after

Cross-checking the dashboards against the scrape configuration exposed a subtle self-inflicted wound. To halve the series count, the container-metrics job dropped rows with an empty `name` label:

```yaml
# Before — also deletes machine-level metrics, which carry no `name` label at all
- source_labels: [name]
  regex: "^$"
  action: drop
```

The machine-level metrics (core count, total memory) have no `name` label, so this deleted them too — and the dashboards divide by core count to express container CPU as a percentage of the host. Every CPU panel would have rendered as blank, with nothing in any log to explain why.

```yaml
# After — the metric-name guard is load-bearing
- source_labels: [__name__, name]
  regex: "container_.*;" # source labels join with ";", so this means: container metric, empty name
  action: drop
```

The one that escaped to the dashboard was more human. The project's context file described the machine by a hostname that was not, in fact, this machine — a leftover from an earlier setup box. That string got copied into a dashboard title and into Prometheus' external labels. The reviewer's reaction to seeing a strange machine name in the breadcrumb was entirely fair.

The fix was not just to correct the string but to remove the class of error: the dashboard is now titled generically, and a panel reads the machine name from the operating-system metric's `nodename` label, so the displayed value cannot disagree with reality. The stale line in the context file was corrected too, with a note explaining how it propagated — otherwise the next person repeats it.

Worth recording: while fixing this I initially claimed every stored sample had been stamped with the wrong label. That was wrong, and checking rather than assuming caught it. **External labels are applied only when data leaves Prometheus** — federation, remote-write, or the alert manager — and none of those were in use. Nothing had polluted the time-series database and there was no cleanup to do. An overstated impact assessment can cause as much wasted work as an understated one.

## Dashboards as code, and why the community ones were dropped

The plan called for vendoring well-known community dashboards. That got reversed during implementation for three concrete reasons: they carry datasource-input blocks that must be hand-rewritten for file-based provisioning; the most popular one is roughly 250 KB of JSON that nobody will ever meaningfully review in a diff; and they assume a fleet of machines, so much of their surface is node-selector variables that are dead weight on a single VM.

Three hand-written dashboards totalling about 35 KB replaced them. Every panel refers to something in this specific deployment, and several carry descriptions explaining what the panel means *here* — for example, that a raised plateau in edge connections is normal during collaborative editing because those are long-lived sockets, not a leak.

Two deliberate choices in there resist "fixing": the memory panel is not stacked, because the "used" series already excludes reclaimable cache and stacking it with cache would double-count and overstate pressure; and no panel uses two Y-axes, so measures with different units live in separate panels.

## Alerting that admits it is not watching

Five alert rules were written — disk filling, probe failing, container restart loop, certificate expiring, memory pressure — and all five ship **paused**, because no notification channel had been chosen yet.

This was a deliberate call over the alternative of shipping them active. A rule that evaluates and fires into a void is worse than no rule: the dashboard looks supervised while nobody is being told anything. The paused state is honest, visible in the UI, and turns into working alerting with one edit once a channel exists.

## Outcome and takeaways

The stack is deployed and verified: six units active and enabled, all ten scrape targets up, all six probes healthy, certificate issued, and — the check that mattered most — all seven listeners bound to loopback with nothing on a public interface, confirmed both by a socket listing and by failing to reach the scraper from the machine's own LAN address. The whole stack costs about 356 MiB of memory, comfortably under the 500–700 MiB estimated. Interestingly the container-metrics agent is the heaviest single piece at roughly three times the scraper's footprint — useful to know if memory ever gets tight.

Application-level metrics were scoped as a second phase and then explicitly deferred, because both halves require restarting a production service during an active rollout. Rather than lose the investigation, the findings were written down: which service is stopped, which configuration flag is false, and the non-obvious detail that because the container runs with `--rm`, manually starting that service would not survive a restart and needs the same auto-start treatment a sibling service already has.

Four lessons worth carrying:

- **Verify registry coordinates, don't infer them.** The container-metrics project had migrated registries; the universally-cited path is frozen several versions back and receives no new releases. Two projects in the same stack also disagree about whether image tags carry a `v` prefix. Ten minutes of checking beats a failed pull mid-deploy.
- **Validate statically before touching a live host.** Config parsers, unit-file verification, a web-server syntax check against a sandboxed prefix, and a structural pass over dashboard JSON for grid overlaps and duplicate identifiers — all of it ran before anything was installed, and it caught the metric-drop bug.
- **Never hardcode an identity you can read at runtime.** A hostname in a title is a lie waiting to happen.
- **Renaming a provisioned file is a two-step operation.** Changing a dashboard's filename means deleting the old copy from the host first; two provisioning files claiming the same unique identifier is a conflict, not a harmless duplicate.
