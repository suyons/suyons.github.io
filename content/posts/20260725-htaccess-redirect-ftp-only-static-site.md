---
title: "Self-Hosting Troubleshooting - Redirecting a Domain You Can Only Reach by FTP"
date: 2026-07-25
draft: false
tags: ["apache", "htaccess", "dns", "self-hosting", "http-redirects"]
categories: ["Infrastructure"]
description: "No SSH, no Apache config, no front controller — just FTP access to a docroot of 120 loose HTML files. Why .htaccess was the only mechanism that could redirect every request to the new domain, and the three things it structurally can't fix."
showToc: true
---

A client's old site — a 2020-era pile of hand-written `.html` plus a little PHP — still answered at its old domain. The business had since moved to a new domain, and every request to the old one, any scheme, any host, any path, needed to 301 to the new site. The constraint was the whole problem: the hosting agency would give FTP access only. No SSH, no Apache config, and a manager who by his own admission couldn't configure httpd either.

## Why the usual answers didn't apply

The instinct is "add a redirect at the entry point." There is no entry point. The docroot was roughly 120 loose `.html` files at the top level (`clinic1_01.html`, `overview3.html`, and so on) plus a handful of PHP endpoints. No router every request flows through, so an application-level redirect would mean editing every file individually — and would still miss every image and stylesheet.

`auto_prepend_file`, the usual PHP-level trick for prepending a redirect to every request, was out for the same reason: it only fires for PHP requests. The overwhelming majority of URLs on this site were static `.html`, which PHP never touches. It would have redirected the couple of PHP endpoints and left the other hundred-plus pages serving the old site untouched.

That leaves the one mechanism that catches every request — including static assets — and is deployable with nothing but write access to the docroot: a per-directory `.htaccess`. This is precisely the situation Apache's override system exists for. No root, no conf file, no restart, just a file dropped into the web root.

## The file

```apache
RewriteEngine On
RewriteRule ^ https://www.example.com/ [R=301,L]

<IfModule !mod_rewrite.c>
  Redirect 301 / https://www.example.com/
</IfModule>
```

`RewriteRule ^ ...` matches every request regardless of path, so all ~120 pages and every asset collapse to a single permanent redirect to the new site's root. The `<IfModule>` block is a `mod_alias` fallback for the (unlikely) case `mod_rewrite` is disabled on the host. We confirmed there was no pre-existing `.htaccess` to merge with before writing this one — a naive drop-in that clobbers an existing file is its own outage.

Uploaded via FTP to the web root. Verified: the old URLs now 301 straight to the new site.

## The caveats that matter more than the file

Three things `.htaccess` structurally cannot fix, worth writing down because they resurface on every "just redirect it" request:

- **HTTPS depends on a certificate, not on this file.** A redirect can only be sent after the TLS handshake completes. The old site was served over plain `http://`, which strongly implies no certificate was installed for that domain. So an `https://` request to the old domain hits a browser cert error and never reaches the 301 — no amount of `.htaccess` fixes that, because the failure happens a layer below Apache's request handling entirely. Fixing it means installing a cert server-side, the one thing still gated behind the agency. We accepted the gap: virtually nobody types `https://` for an old bookmark.
- **Apex vs. `www` is a DNS question, not a docroot question.** The rule redirects any host that lands in this docroot, but whether the bare apex domain reaches this docroot at all depends on where its A record points. If it points elsewhere, this file never runs for it.
- **`AllowOverride None` would silently void the whole thing.** Rare on shared FTP hosting, but if the redirect had failed silently, this is the first suspect — and only the agency can toggle it. It worked, so we know override is on for this host.

## Takeaways

- **"Redirect every request" has a different right answer depending on what a request *is*.** With a front-controller app, `auto_prepend_file` or an entry-point edit is the clean fix. With a loose static-file site, only the web server layer sees every request, so `.htaccess` is the one tool that covers the assets too. Check the docroot's actual shape before reaching for a mechanism.
- **A redirect lives above TLS, so it can never rescue a missing certificate.** Any "redirect https" requirement quietly carries an "…and someone installed a cert" precondition. Name that up front instead of discovering it when a visitor reports a security warning.
- **If the domain is self-owned, registrar-level URL forwarding is the cleaner long-term move.** It redirects at the DNS layer, bypasses the server and the agency entirely, and typically provisions its own certificate for the forward — the one gap `.htaccess` can't close. Flagged for later; the `.htaccess` file was the zero-dependency fix that shipped today.
