---
title: "Oracle Troubleshooting - A Connect Timeout That Wasn't My Server's Problem"
date: 2026-07-15
draft: false
tags: ["oracle", "tns", "networking", "powershell", "troubleshooting"]
categories: ["Databases"]
description: "Diagnosing an Oracle ORA-12170 connect timeout from an app server with no visibility into the DB host, using nothing but PowerShell's built-in Test-NetConnection to prove the failure wasn't on my end."
showToc: true
---

## The error

An internal document-management app in a test environment started failing every database call with:

```
ORA-12170: TNS:Connect timeout occurred
```

No stack trace worth reading past that line — it's Oracle's client library giving up on the connection attempt, not the application throwing anything of its own.

## ORA-12170 is a network symptom wearing a database error code

The unhelpful part of ORA-12170 is that it's identical for at least four different root causes: a dead destination host, a stale IP after a server move, a firewall silently dropping the port, or a listener that isn't actually running. Search the error and a good chunk of the advice is to raise `SQLNET.INBOUND_CONNECT_TIMEOUT` — which only makes sense if the TCP handshake is completing and just running late. If the packet never reaches the listener in the first place, a longer timeout just makes you wait longer to fail.

I had no shell access to the database host and no application-side logs beyond that one error line. What I did have was an app server I could run diagnostics from, and a production database the same app server talks to successfully — a known-good path to compare the failing one against.

## Isolating the failure with what's already on the box

PowerShell's `Test-NetConnection` is built into Windows — no extra tooling to install, no firewall exception to request just to run a diagnostic. I ran it twice from the same app server: once against the production Oracle host that was working, once against the test-tier host that was timing out.

```powershell
PS> Test-NetConnection -ComputerName prod-db.example.com -Port 1521

ComputerName     : prod-db.example.com
RemoteAddress    : 192.0.2.10
RemotePort       : 1521
InterfaceAlias   : Ethernet 3
SourceAddress    : 192.0.2.50
TcpTestSucceeded : True
```

```powershell
PS> Test-NetConnection -ComputerName 198.51.100.20 -Port 1525

WARNING: TCP connect to (198.51.100.20 : 1525) failed
WARNING: Ping to 198.51.100.20 failed with status: TimedOut

ComputerName     : 198.51.100.20
RemoteAddress    : 198.51.100.20
RemotePort       : 1525
InterfaceAlias   : Ethernet 3
SourceAddress    : 192.0.2.50
PingSucceeded    : False
TcpTestSucceeded : False
```

Same source address in both. Different destination. One succeeds, one doesn't. That's the whole diagnosis, and it's a stronger claim than "prod works, test doesn't" — it rules out anything scoped to the app server itself: default gateway, DNS resolution, the app server's own outbound firewall rules. All of that is held constant between the two tests.

One detail that mattered: the test-tier listener was on port 1525, not the Oracle default of 1521. Assuming every target uses the default port would have made this a test of the wrong thing entirely — a "timeout" against a port nothing was ever listening on isn't evidence of anything.

## What a TCP test proves and what it doesn't

`Test-NetConnection` confirms or denies a completed TCP handshake — nothing about TNS itself. A failed result here is consistent with the destination host being down, its IP having changed without a corresponding update on the client side, or a firewall between the two hosts dropping that specific port. It would look identical in all three cases. That's a real limit of the tool, not a gap in the diagnosis: the point of running it wasn't to find the root cause on the DB side, it was to prove the root cause wasn't on mine, and hand off a narrowed problem instead of a vague one.

## Escalating with the isolation already done

The request to the team that owned the destination host asked for three specific checks, not "please look into it":

1. Whether the test-tier host is actually up, and whether its IP changed recently.
2. Whether the firewall between the app server and that host allows port 1525 specifically — not the Oracle default.
3. If the host is up, whether the TNS listener process is actually running — `lsnrctl status` on the database server itself.

Attaching the raw `Test-NetConnection` output next to the request meant whoever picked it up started from "port 1525 is unreachable from this app server, confirmed by TCP test, while port 1521 to a different host from the same source works fine" — not from "the app is broken, please investigate."

## Takeaways

- Don't reach for `SQLNET.INBOUND_CONNECT_TIMEOUT` before confirming a TCP handshake reaches the listener host at all — a longer timeout doesn't help a connection that was never going to complete.
- When you can't get visibility on the far side of a failure, isolate with a same-source comparison: one known-working destination, one failing destination, everything else held constant. What differs between the two results is the diagnosis.
- Don't assume a default port. Confirming the actual configured port before testing connectivity is what makes the comparison valid in the first place.
- A report with the exact command, source, destination, and result attached moves through another team's queue faster than a one-line description of the symptom.
