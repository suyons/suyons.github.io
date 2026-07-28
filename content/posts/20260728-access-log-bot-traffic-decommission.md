---
title: "Access Log Security - Telling Bot Noise from Real Users Before You Delete a Server"
date: 2026-07-28
draft: false
tags: ["access-logs", "apache", "bot-detection", "web-scanners", "decommissioning"]
categories: ["Security"]
description: "A legacy document-editor host was flagged for decommissioning on the theory that nobody used it. A single day of Apache access logs had zero 2xx responses — but proving 'no real traffic' takes more than counting status codes."
showToc: true
---

A legacy document-editor host came up for decommissioning on a simple theory: if nobody's using it, delete it. The evidence on the table was one day of Apache `access_log` — 41 requests, every single one a 4xx. Zero successes. That looks like a clean case. It isn't, and the gap between "zero successful requests" and "nobody uses this server" is where the actual analysis has to happen.

## The headline number, and why it's not the finding

| Status code | Count | Meaning |
|---|---:|---|
| 403 Forbidden | 17 | Root `/` blocked — mostly scanner/bot probes |
| 404 Not Found | 18 | Probes for nonexistent paths (`.env`, `/manager/html`, etc.) |
| 400 Bad Request | 5 | Malformed/non-HTTP payloads |
| 405 Method Not Allowed | 1 | Disallowed HTTP method (a `CONNECT` proxy probe) |
| 2xx (success) | 0 | No successful requests of any kind |

Zero 2xx is consistent with "nobody uses this," but it isn't proof of it by itself. A real client hitting an auth wall or a broken session also produces nothing but 4xx. The status-code table tells you traffic failed. It doesn't tell you *why* it failed — for that you have to read what was actually being requested and by whom.

## Sorting the noise into categories

The useful move is going line by line and classifying every request, not trusting the aggregate. Three categories fell out immediately, and they explain all 41 requests without needing a fourth.

**Self-identifying scanners.** Several requests carry a user-agent that names exactly what they are:

```log
203.0.113.14 - - [27/Jul/2026:06:07:21 +0900] "GET / HTTP/1.1" 403 7620 "-" "python-requests/2.32.5"
203.0.113.16 - - [27/Jul/2026:06:55:12 +0900] "GET / HTTP/1.1" 403 7620 "-" "Mozilla/5.0 zgrab/0.x"
203.0.113.30 - - [27/Jul/2026:07:33:03 +0900] "GET / HTTP/1.0" 403 7620 "-" "Hello from Palo Alto Networks, find out more about our scans in https://docs-cortex.paloaltonetworks.com/r/1/Cortex-Xpanse/Scanning-activity"
203.0.113.40 - - [27/Jul/2026:09:08:13 +0900] "GET /.env HTTP/1.1" 404 196 "-" "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
```

`zgrab` is a mass internet-scanning tool in the Censys/Shodan mold. Palo Alto's Cortex Xpanse literally puts a human-readable explanation of the scan in its own user-agent string. Even Googlebot spends a handful of requests probing for `.env` and Docker config files alongside its normal crawling — search engines run opportunistic security sweeps too. None of this is an attack. It's the background radiation every public IP on the internet receives continuously, and it accounts for roughly half the log.

**Spoofed or reused user-agent strings.** A block of requests claims to be an iPhone on Safari 13.2.3 — a five-year-old point release, arriving from several unrelated IP addresses within the same day. Real mobile traffic doesn't cluster like that; a stale, fixed UA string reused across many source IPs is a signature of scanning frameworks that rotate identity poorly, not real users on old phones.

**Targeted path probing from a single source.** This is the category that actually deserves attention, because it's the closest thing to a real client in the log:

```log
198.51.100.20 - - [27/Jul/2026:06:29:04 +0900] "GET /SDK/webLanguage HTTP/1.1" 404 196 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/90.0.4430.85 Safari/537.36 Edg/90.0.818.46"
198.51.100.20 - - [27/Jul/2026:09:10:58 +0900] "GET /SDK/webLanguage HTTP/1.1" 404 196 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/90.0.4430.85 Safari/537.36 Edg/90.0.818.46"
198.51.100.20 - - [27/Jul/2026:12:04:36 +0900] "GET /SDK/webLanguage HTTP/1.1" 404 196 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/90.0.4430.85 Safari/537.36 Edg/90.0.818.46"
```

Same IP, same specific application path, three times across the day, hours apart, with a normal-looking desktop browser UA and a normal Chrome/Edge version. That pattern — one source repeatedly checking for one known API endpoint — is what an external prober checking for a known integration path looks like, not what a search-engine crawler or a mass scanner does. It's also not what a real editor session looks like: no cookies, no referer, no follow-up requests to any session-style path. It's worth a separate note in the writeup rather than being folded into "generic scanner noise," because if `/SDK/webLanguage` is a real integration endpoint for another internal system, that's a stakeholder to check with before deleting anything — not just a bot to ignore.

The rest of the log is malformed, non-HTTP traffic hitting the HTTP port outright — raw TLS `ClientHello` bytes, stray control characters, no user-agent at all — which is what you'd expect from scanners that speak the wrong protocol at a port and move on.

## What the log can't tell you

The traffic sample is consistent with "no real user session happened today": no 2xx responses, no authenticated flows, no referer chains, none of the static-asset request patterns (CSS, JS, fonts) a browser-rendered editor session would generate. That supports the decommissioning theory for the sampled day. It does not confirm zero usage for the month, and treating a one-day sample as sufficient is exactly how you delete something someone still depends on. Before signing off, the checklist has to cover what a single day of one log file structurally cannot show:

- **Full-month coverage.** Check rotated/archived logs, not just today, to confirm the pattern holds across the billing period being used to justify the decision.
- **Log rotation gaps.** Verify `logrotate` hasn't silently dropped a window where a real request came through.
- **Non-HTTP access paths.** If the editor supports desktop-app or API-based access, there may be a separate service port or WebSocket endpoint that logs elsewhere and wouldn't appear in this Apache log at all.
- **Internal/private network access.** Confirm no clients reach the server over an internal network or VPN that bypasses the public-facing vhost this log captures.
- **Application-level logs.** Some editors log session or license-check activity independently of HTTP page loads — check those, not just the web server.
- **Stakeholder confirmation.** Confirm with whoever owns or requested the server that it's genuinely deprecated, not just quiet on the day someone happened to look.

## Takeaways

- **Zero 2xx responses is a symptom, not a verdict.** It's consistent with "no real usage," but a broken auth flow produces the identical signature. Read what's being requested before concluding why nothing succeeded.
- **Self-identifying scanners are the easy majority.** `zgrab`, Palo Alto Cortex Xpanse, and even Googlebot's opportunistic `.env` probes name themselves. Filtering these out first shrinks the log to what actually needs judgment.
- **The same path from the same IP, hours apart, is the pattern worth chasing.** It's the one signature in a scanner-dominated log that looks like a system that expects this server to answer — worth a name and a stakeholder check, not a line in the noise total.
- **A one-day log is a data point, not a decommissioning decision.** Pair it with a checklist that covers what the log format structurally can't show — other ports, other log files, other months — before anything gets deleted.
