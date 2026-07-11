---
title: "Self-Hosting - A Software License That Didn't Care How Good the Reverse Proxy Was"
date: 2026-07-11
draft: false
tags: ["nginx", "reverse-proxy", "software-licensing", "self-hosting", "git"]
categories: ["Infrastructure"]
description: "A working nginx reverse-proxy setup for an embedded commercial document editor had to be torn down anyway, because the editor's license was bound to one exact domain — not the subdomain the proxy lived on."
showToc: true
---

## The setup that had to come undone

A recently-added reverse-proxy configuration had to be unwound after it turned out to conflict with a third-party software license — a useful case study in why it pays to check licensing constraints before building infrastructure around a subdomain.

The system in question is an internal document management application with an embedded browser-based document editor, the kind of component several products embed for in-browser document viewing and editing. To make the app reachable over HTTPS with proper access control, it had been placed behind an nginx reverse proxy on a subdomain, with the editor's own backend services also proxied through virtual paths under that same subdomain, so all traffic could be served from one hostname.

## The subtlety that made it work at all

That setup worked, but it depended on getting one subtlety right: the embedded editor generates absolute URLs in its own responses — for asset downloads, WebSocket connections, and save callbacks — based on the hostname it's told about via forwarded headers. Getting that right took several rounds of fixes:

- A buffer size increase to handle oversized login response headers.
- A header rewrite so the editor's generated URLs routed back through the proxy instead of leaking the internal document-server hostname.
- A plain-HTTP exception carved out for the save-callback path specifically, because the callback client didn't handle the app's blanket HTTP→HTTPS redirect gracefully — a POST request hitting a 301 redirect gets silently converted into a bodyless GET on redirect-follow, which broke document saving in a way that was easy to misdiagnose as a totally unrelated failure.

## The constraint no proxy config could fix

None of that mattered once the actual constraint became clear: the embedded editor's commercial license is bound to one exact domain name, with no subdomains permitted. The proxy setup had been built on `docs.example.com` — a subdomain of the licensed domain, not the domain itself. No amount of proxy configuration fixes a license mismatch; the fix has to happen at the domain level.

## The fix: off the proxy, onto the licensed domain directly

The resolution was to stop routing this application through the reverse proxy entirely. It's now exposed directly on the licensed apex domain, using a router-level port forward straight to the backend server, on plain HTTP rather than through the proxy's TLS termination. That means giving up the proxy's access-control list and TLS termination for this one app — but it's the only way to satisfy a license that's strict about the exact hostname.

For future cases where multiple tenants need to share one licensed domain, the tenant-partitioning idea on the table is port-based separation: one port per tenant on the same licensed domain, forwarded on the router, rather than one subdomain per tenant.

## Reverting a multi-step config change without losing the parts worth keeping

A worthwhile technique from this cleanup: since every step of the original setup had been committed separately — buffer fix, header-rewrite fix, callback-redirect fix, cleanup — reverting wasn't a matter of hand-editing the config back into a plausible-looking state. It meant diffing against the commit right before the feature was first introduced and restoring that exact known-good version, then deleting the vhost file outright once the subdomain was retired for good.

That avoids the classic revert mistake of leaving a fix bundled in with feature code that shouldn't be un-fixed. In this case, the login response header buffer-size increase was a real bug fix unrelated to the license issue, and it was deliberately kept rather than reverted along with everything else.

## Outcome

The reverse-proxy vhost for this application is gone; nginx now only fronts the services it's meant to; the application is reachable directly on its licensed domain via a router port-forward. The nginx config passed validation and reloaded cleanly at every step, and the change history remains fully recoverable, because the old configuration was never force-deleted from version control — just removed from the current working state. Anyone who needs to see exactly how the old header-rewrite or callback workaround was written can still check it out from an earlier commit.

## Takeaways

- When embedding third-party licensed software behind infrastructure you control, verify the license's domain-binding rules *before* investing time in reverse-proxy plumbing for a subdomain. A perfectly working proxy configuration is worthless if the software refuses to function under the hostname the license doesn't cover.
- Unwinding a multi-step infrastructure change later means carefully separating "fixes worth keeping" from "infrastructure that has to go," not just reverting to the last known state wholesale. Committing each step separately is what makes that separation possible after the fact.
