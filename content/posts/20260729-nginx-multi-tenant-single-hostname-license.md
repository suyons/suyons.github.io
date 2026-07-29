---
title: "Nginx Reverse Proxy - Eight Tenants, One Licensed Hostname, Zero Subdomains"
date: 2026-07-29
draft: false
tags: ["nginx", "reverse-proxy", "tls", "multi-tenancy", "node-js"]
categories: ["Infrastructure"]
description: "Moving eight internal apps from plaintext router port-forwards to TLS behind nginx, under a licensing constraint that ruled out per-tenant subdomains — and the failure-mode differences that decided every design choice along the way."
showToc: true
---

Eight internal document- and lab-management applications were reachable straight from the internet as plaintext HTTP, via router port-forwards that bypassed the reverse proxy entirely — and with it, TLS and any office-only access control. The job was to put every one of them behind nginx with real certificates. The complication: none of them was allowed its own hostname.

## The constraint that shaped the whole design

The embedded office-suite editor these apps depend on (OnlyOffice Document Server) is licensed to one exact hostname, with no subdomains permitted. That rules out the obvious `tenant-a.example.com`, `tenant-b.example.com` layout. Multi-tenancy has to be expressed as **one router-forwarded port per tenant, on a single licensed domain**.

So instead of the conventional "one config file per virtual host, all on 443," the result is a single file holding one `server` block per tenant, each terminating TLS on its own high port, all sharing a `server_name` and a certificate. Unusual, but it falls directly out of the licensing constraint rather than being a stylistic preference.

## Sharing a port with your own backend, and why that stopped

The first design kept each app on the same port number nginx served, splitting the two by address: nginx binds the LAN address, the app binds loopback.

```nginx
# Before
server {
    listen 192.168.1.50:32040 ssl;   # deliberately NOT a wildcard:
    ...                              # 0.0.0.0 would collide with the app
    proxy_pass http://127.0.0.1:32040;
}
```

That needs an app-side change too, since the stock build hardcodes its bind address:

```js
// Before (shipped build)
server.listen(port, "0.0.0.0");

// After (patched): host becomes configurable
server.listen(port, config.host);   // config.host = process.env.HTTP_HOST || "0.0.0.0"
```

It worked. Both sockets coexisted, and start order didn't matter. But it has a nasty property: that patch lives in a **build artifact**. The next deploy overwrites it, the app grabs the wildcard address on the shared port, collides with nginx, and the site goes down — with a symptom (socket collision) that looks nothing like "somebody deployed the app."

```nginx
# After
server {
    listen 32040 ssl;
    listen [::]:32040 ssl;
    proxy_pass http://127.0.0.1:31040;   # public 32xxx -> backend 31xxx
}
```

With separate ports, that same lost patch degrades to "the backend is also reachable on the LAN" — which a firewall rule already blocks — instead of taking the whole service down. The convention became public `32xxx` in front, backend `31xxx` behind, with a matching inbound-block firewall rule per backend port.

The general principle: **when choosing between two designs, compare their failure modes, not just their happy paths.** One degrades gracefully. The other collides.

## A supporting argument that had to be retracted

While arguing for that change, one of the arguments used was that the address-specific `listen` directive permanently broke `nginx -t`, which had started failing with a bind error while the service was running. That claim was wrong.

A full service restart cleared the error entirely. The actual rule is narrower: **a reload that changes a port's listen set leaves that port's socket un-rebindable until the service restarts.** Ports whose listen configuration didn't change keep testing fine, which is why long-standing ports never showed the problem.

The conclusion survived — the redeploy landmine described above was reason enough on its own — but the bad supporting argument still had to be withdrawn and the documentation corrected, since it had already been written down as fact. An incorrect argument is worth retracting even when it happens to point at the right answer.

The practical fallout is a rule worth encoding elsewhere: in that specific window, treat the test as passing when `syntax is ok` appears and every reported error is a `bind()` failure. That matters because an unattended upgrade hook had been gating its service restart on this test's exit code, and would otherwise have silently skipped restarting forever.

## Two small proxy details with outsized consequences

**The Host header.** These backends construct editor callback URLs as `` `${req.protocol}://${req.get("host")}` ``, so the forwarded header has to carry the port:

```nginx
# Before - $host strips the port, handing the editor a portless URL
proxy_set_header Host $host;

# After - preserves "host:32040"
proxy_set_header Host $http_host;
```

**Plaintext arriving on a TLS port.** By default nginx answers that with a bare 400. It exposes an internal status code for exactly this case:

```nginx
error_page 497 =308 https://apps.example.com:32040$request_uri;
```

The status choice matters more than it looks. A **301 makes most HTTP clients re-issue a POST as a bodyless GET** — precisely the failure that had silently broken document-save callbacks on this stack once before. 308 preserves method and body. This was verified by sending a POST through the redirect and confirming, in the access log, that the second hop was still a POST rather than a GET:

```
"POST /probe HTTP/1.1" 308   <- plaintext hop, redirected
"POST /probe HTTP/1.1" 404   <- followed over TLS, still POST
```

The hostname in that redirect is hardcoded rather than pulled from `$host`, so a forged Host header can't turn it into an open redirect.

## Three different ways an application names itself

Moving to HTTPS breaks any app that tells a third-party service where to find it. Across eight apps there were three distinct patterns, and they needed different fixes:

| Pattern | Fix |
|---|---|
| Dedicated callback-host environment variable | Point it at `https://…:port` |
| Generic application-URL variable | Same |
| Derives from the request (`req.protocol`) | **Needs a code change** — trust the proxy so the framework reads the forwarded-protocol header |

Only the third case requires touching source. No proxy configuration substitutes for it: without explicitly trusting the proxy, the framework reports `http` no matter what headers arrive, and emits URLs the TLS-only port can't serve. Identifying which bucket each app fell into — by reading how it actually built its URLs, not by assuming — determined whether a given app's migration was config-only or blocked on a release.

## A long uptime proves nothing about a restart

Five of the eight applications carry a native module built for an old Node.js runtime. The process manager's default interpreter setting resolves from `PATH` **at spawn time**, so a process started months ago can be running one Node version while a restart today lands on a different one, installed later. Several of these had been up for weeks and looked perfectly healthy — and would have crash-looped on the next reboot.

```
# Before
pm2 start bundle.js --name "APP"

# After
set NODE_14=%APPDATA%\fnm\node-versions\v14.21.3\installation\node.exe
pm2 start bundle.js --name "APP" --interpreter "%NODE_14%"
```

Discovering this by restarting is expensive — a crash loop and an outage. The cheap, non-destructive check is to hash the compiled native module against a copy whose runtime is already known; identical builds are byte-identical.

And then the heuristic failed exactly where it looked safest. After four apps in a row needed the older runtime, the pattern felt settled. The fifth app crash-looped with the same symptom, and the obvious move was to apply the same pin. Its module hash didn't match the others, though, and its version file asked for a runtime three major versions *newer*, not older — the opposite fix. Applying the established pattern blindly would have kept it down. That app was offline for about two minutes because the restart discarded the interpreter its process-manager entry had been holding.

The lesson isn't "check more carefully." It's that **a heuristic that has worked four times is at its most dangerous on the fifth**, and a cheap evidence check — a file hash, a version file — should override the pattern every time, not just the times it's convenient.

## Two-sided limits, and configuration that drifts

Users had started hitting "request header too large." Two independent layers cap header size here — the proxy and the application runtime — and the lower one always wins. Raising just one does nothing: proxy too low returns 400, backend too low returns 431.

The proxy-side setting had been declared per virtual host and had quietly drifted apart over time: 64 KiB in three blocks, 32 KiB in two, and entirely absent in two more, leaving those on a small default.

```nginx
# Before - per vhost, drifts apart over time
server {
    ...
    large_client_header_buffers 8 32k;
}

# After - declared once, inherited everywhere
http {
    large_client_header_buffers 8 64k;
    ...
}
```

Duplication that must stay in sync but has no mechanism keeping it in sync will drift. Hoisting it to one place removes the failure mode instead of fixing one instance of it. These buffers allocate on demand, so a generous size costs nothing on ordinary requests.

Testing with a 60 KiB header showed the split clearly: after fixing the proxy side alone, four of eight tenants still failed — now with 431, from the applications themselves. Only after raising both sides did all eight pass.

## Verifying a deny rule actually denies

An internal index page needed to be office-only, returning 403 to anyone else. Nginx access rules have a sharp edge worth knowing here: **declaring `allow`/`deny` in a block replaces the inherited list entirely**, rather than adding to it — so any narrower block has to restate everything it still wants to permit.

Rather than trust the config on paper, the rule was tested by temporarily removing the local subnet from the allow list — making requests from inside the office count as an outsider — confirming 403 on both the index and a sub-path (which also proves the inner `location` inherits the outer rule), then restoring the original list and re-confirming normal access. Checking the file afterward for a leftover test value is part of the exercise: a temporary edit that survives the test is worse than never testing at all.

## Reading a failure by its shape

One connectivity problem was diagnosed almost entirely from *which* kind of failure appeared. A request from inside the network to the public address failed, and the question was whether the firewall, the proxy, or the router was responsible.

The distinguishing detail: **"connection refused" is a TCP reset; a firewall block is a silent drop that shows up as a timeout.** A reset means something received the packet and actively rejected it. Since neighboring ports on the same router hairpinned correctly, the router was reachable and working — it simply had no forwarding rule for this particular port. No firewall rule and no proxy setting could have produced that specific error shape.

The corollary, learned the same day: a request that originates *inside* the network can never prove an external block is absent, because it always matches the local-subnet allow rule regardless of what an outside client would see. Reporting "I can't verify this from in here" was more useful than a green checkmark that meant nothing.

## Outcome

Eight applications now terminate TLS behind the proxy with valid certificates, each on its own forwarded port under a single licensed domain, with plaintext requests redirecting method-safely, office-only access control enforced and tested, and per-backend firewall rules keeping the old plaintext ports off the network. Every step was verified over the real public path rather than against loopback, and 64 KiB request headers now pass end to end on all eight.

Left open, and stated as such rather than papered over: two applications still emit `http://` URLs to the editor service until a source change ships — the 308 redirect keeps them working meanwhile, but that's a mitigation, not a fix — and roughly a dozen untouched background processes remain in the same unverified runtime state that five of these eight turned out to be in.

## Takeaways

- **Compare failure modes, not happy paths.** Two designs that both work in the demo can differ enormously in what happens when one assumption breaks later.
- **Retract a bad argument even when the conclusion holds.** A wrong reason that survives in documentation becomes someone else's wrong decision down the line.
- **A long-running process proves nothing about whether it can restart cleanly.** Uptime hides latent startup breakage — find it deliberately, with a check that doesn't require an actual restart.
- **A pattern that's held four times is most dangerous on the fifth.** Cheap evidence — a hash, a version file — should outrank an established heuristic every time, not just when convenient.
- **When a limit exists on both sides of a hop, raising one side changes nothing.** Find both, and check which status code you're actually getting back.
- **Test that a restriction actually restricts.** Put yourself on the wrong side of the rule and confirm it rejects you — then check your test edit didn't survive.
