---
title: "DNS Migration Troubleshooting - The CNAME to workers.dev That 522'd My Own Homepage"
date: 2026-07-17
draft: false
tags: ["cloudflare", "dns", "ssl", "workers", "incident-response"]
categories: ["Infrastructure"]
description: "An emergency migration of a company homepage from a shared host to Cloudflare after a bandwidth-exhaustion attack, and the chain of Cloudflare-specific gotchas — Universal SSL, a CNAME to workers.dev, and a stale record — that stood between 'change the nameservers' and actually working."
showToc: true
---

## The night the homepage had to move

Our company homepage went down because the shared host it ran on got hit with a bandwidth-exhaustion attack: something scraped it hard enough to blow through the plan's 5GB monthly bandwidth cap, and the provider shut the hosting off. Extending the contract to regain access didn't help — SSH still refused to connect. At that point the call was easy: stop paying for a setup this fragile, and move hosting, domain, and certificates to Cloudflare in one evening. "Change the nameservers and done" is not how it went.

### Getting back in first

SSH access came back eventually, but only after re-enabling the deprecated `ssh-rsa` host key algorithm on the client side — the old box's key exchange wouldn't negotiate anything else. Once in, we also noticed the live site had rotted a little on its own: a page that was supposed to render Korean text was showing broken hanja instead. Queued that as a content fix to ship alongside the migration.

### The verification window

Moving the domain to Cloudflare needs a domain-ownership verification code sent to the registrant contact — in our case, the one person who could receive it was driving. So the operation got pinned to a fixed time and confirmed in advance. The code landed on schedule, the registrar-side nameserver change went through immediately, and then Cloudflare's zone sat in "Pending" for about 40 minutes before flipping to "Active." The moment it did, DNS resolution through Cloudflare — including the `*.example.com` wildcard — worked.

## SSL: the registration step that doesn't exist

My mental model came from years of self-hosted nginx: a cert gets issued, then you register it somewhere — "install this cert" is always a separate step. That model is wrong for Cloudflare. Universal SSL is issued per zone, and the instant the zone goes Active, it's automatically applied to every proxied (orange-cloud) DNS record. There's no "upload your certificate" screen to go find, because that step doesn't exist by design. If you're looking for it, stop — you're pattern-matching against the wrong hosting model.

## Error 522: a CNAME to workers.dev is not the same as your own origin

With DNS live, the homepage still didn't load — and with a different error than the day before. The previous day it was a 1001 (DNS resolution failure). Now it was a 522 (connection timeout to the origin). `curl` against the site timed out flat. One browser in incognito mode looked fine — which turned out to be a cache hit, not a working site. That's the actual lesson here: "it loads in my browser" is not verification. `curl` is.

The root cause: the apex domain was pointed at the Cloudflare Workers deployment with a plain CNAME record, something like:

```
example.com.  CNAME  homepage.example-project.workers.dev.
```

Cloudflare treats a CNAME to a `.workers.dev` address as an *external* origin and routes the request back out over the public internet — which the `workers.dev` side isn't set up to serve for a bare CNAME lookup like that. Hence the timeout. The fix is the Workers/Pages **Custom Domain** feature, which binds a domain to a Worker internally, instead of round-tripping through DNS to an external-looking hostname.

Adding the custom domain failed on the first attempt too — a leftover `www` CNAME record conflicted with it. One of the stale CNAME entries had survived a deletion we thought had already happened. Once we tracked it down and removed it, the custom domain attached cleanly and the homepage came back.

## Post-migration blast-radius check

A nameserver move touches everything hanging off the domain, so every dependent service got checked individually rather than assumed fine:

- **Homepage** — up, plus the hanja-rendering fix shipped in the same pass.
- **Mail** — unaffected.
- **Office-document editor integration** — confirmed working across every client tenant we host it for but one, where the VPN password had happened to expire mid-operation. That one environment we took on faith rather than verifying live.
- **Dev/staging servers behind the same proxy** — reachable once the Custom Domain fix landed.
- **GitLab** — the old DNS setup relied on the wildcard record pointing at the office network. That got split out into its own explicit record pointing at the office's dynamic-DNS name. `nslookup` resolved it correctly well before any browser did — Gabia's propagation for record edits was noticeably faster than Cloudflare's, which was the opposite of what we expected walking in.

## Outcome

Hosting, domain, and certificate management for the homepage now live entirely on Cloudflare. Follow-ups queued for later: get the homepage source into a git repo (first checking whether one already exists somewhere), then wire up CI/CD so a push to the main branch auto-deploys to Cloudflare Pages — deferred behind a deadline that had nothing to do with this migration.

## Takeaways

- Cloudflare Universal SSL has no registration step. It auto-applies to proxied records the instant the zone goes Active — don't waste time hunting for an nginx-style "install cert" screen that isn't there.
- Never point a CNAME at a `.workers.dev` address. Use the Custom Domain binding, and expect stale CNAME records to block it until you find and delete them.
- When debugging "the site is down," the error *code* tells you which layer to look at. 1001 is DNS resolution; 522 is a timeout reaching the origin. Don't treat both as "it's still broken."
- Verify with `curl`, not a browser. A browser cache will happily show you a page that isn't actually being served anymore.
- DNS propagation speed is provider-specific. A clean `nslookup` result doesn't mean every client has caught up yet.
