---
title: "Your Logged-Out 404 Leaks Your Route Map"
date: 2026-06-05
draft: false
tags: ["nextjs", "authentication", "authorization", "security", "middleware", "qa"]
categories: ["DevOps"]
description: "On a QA pass through a private, employees-only Next.js shop, a logged-out request to a nonexistent URL returned a 404 page instead of the login redirect. That looks correct. For a fully private app it's a route-enumeration leak, and it comes from allowlisting the wrong set of routes."
showToc: true
---

## The 404 that looked correct

I was running a manual QA pass on a Next.js (App Router) storefront that's gated behind login — an internal, employees-only mall, not a public shop. The checklist had the usual route-guard items:

> Logged out, visit `/cart` → redirect to `/login?from=/cart`. **Pass.**
> Logged out, visit `/mypage` → redirect to `/login?from=/mypage`. **Pass.**

Then a throwaway line near the bottom:

> Visit a URL that doesn't exist → show 404.

Logged out, I typed `/nonsense-xyz` into the address bar. I got the app's 404 page. Checklist says pass, so I marked it pass. Then I sat with it for a second and changed my mind.

For this app, the 404 is the bug.

## Three responses, and what they tell a stranger

Here's what a logged-out visitor gets back, depending on what they ask for:

- `/cart` — a real, protected route → **redirect to `/login`**
- `/terms` — a real, public route → **renders the page**
- `/nonsense-xyz` — doesn't exist → **404 page**

Three distinguishable responses. That third one is the problem, because the difference between "redirect to login" and "404" is a yes/no oracle for *does this route exist*. Without ever logging in, I can probe paths and sort them:

- Redirect to `/login` → a real protected route.
- 404 → not a route.

So `/admin` redirecting while `/admyn` 404s tells me `/admin` is real. `/mypage/orders` redirecting while `/mypage/ordres` 404s confirms the spelling. For an app whose entire premise is that you see nothing until you authenticate, handing out a route map to anyone with a browser undoes the point. The pages stay locked, but the *shape* of the app — what exists, how it's named, where the surface area is — leaks to an unauthenticated stranger.

This is the kind of thing that's invisible in normal use because no honest user ever types a wrong URL on purpose. It only shows up when you, or someone less friendly, start poking.

## Where the 404 comes from

The leak isn't in the 404 page. It's in the matcher. The middleware in this app (the same file I [wrote about before](/posts/nextjs-middleware-cookie-presence-auth/) for a different reason) guards an explicit list of protected prefixes:

```ts
// middleware.ts — allowlisting the PROTECTED routes
const PROTECTED = ["/cart", "/mypage", "/checkout"];

export function middleware(req: NextRequest) {
  const path = req.nextUrl.pathname;
  if (!PROTECTED.some((p) => path.startsWith(p))) return NextResponse.next();

  // ...session check, redirect to /login if missing...
}

export const config = {
  matcher: ["/cart/:path*", "/mypage/:path*", "/checkout/:path*"],
};
```

`matcher` decides which requests even invoke the middleware. A request to `/nonsense-xyz` matches none of those prefixes, so the middleware never runs. Next.js then tries to resolve the route, finds no `page.tsx` and no match, and renders `not-found` — the 404. Nothing checked whether the visitor was logged in, because as far as the routing layer was concerned, there was nothing here to protect.

That's the mechanism: **a route that doesn't exist can't be on your protected list, so it skips the gate by construction.** The gate only covers paths you remembered to enumerate, and you can't enumerate the infinite set of paths that aren't real.

## The actual mistake is default-allow

The matcher is a symptom. The real shape of the bug is the policy: this middleware **allowlists the private routes**, which means everything not on the list is implicitly public. That set — "everything not on the list" — includes more than you think:

- Routes that don't exist (the 404 case).
- Routes that exist but you forgot to add to `PROTECTED`.
- Routes a teammate adds next month without reading the middleware.

Every one of those is open by default. For a public site with a few gated corners — a blog with an admin panel — default-allow is the right model; most of the site *should* be reachable without login, and you guard the exceptions. But this app is the inverse. It's private end to end. The only things a logged-out visitor should ever see are the login page itself and maybe a terms page. For an app like that, allowlisting the private routes is backwards. You want to allowlist the *public* ones and deny everything else.

Default-deny flips which mistakes are safe. Forget to list a private route under default-allow and it's exposed. Forget to list a public route under default-deny and it merely redirects to login — annoying, visible immediately, harmless. The failure mode you want is the one that fails closed.

## The fix: allowlist public, deny the rest

Invert it. Run the middleware on essentially everything, keep a short list of routes that are allowed without a session, and send everyone else to `/login` — whether or not the route they asked for is real:

```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

// Everything here is reachable while logged out. Nothing else is.
const PUBLIC = ["/login", "/terms"];

export function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;

  const isPublic = PUBLIC.some(
    (p) => pathname === p || pathname.startsWith(p + "/"),
  );
  if (isPublic) return NextResponse.next();

  if (!hasValidSession(req)) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}

export const config = {
  // Run on every path except static assets and Next internals.
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\..*).*)"],
};
```

Two things changed. The matcher is now a negative lookahead that covers all application routes instead of three prefixes, so the middleware runs even for paths that don't resolve to a page. And the policy is `PUBLIC`-allowlist plus default-redirect, so an unknown path hits the gate, fails the session check, and redirects to `/login` — identical to how a real protected route behaves.

Now the oracle is gone. Logged out, `/cart`, `/mypage/orders`, and `/nonsense-xyz` all return the same thing: go to login. You can't tell a real route from a typo from a probe, because the gate answers before routing ever decides whether the page exists.

A small but real detail: I don't carry a `from=` param back to login for these. The original protected routes set `from` so an honest user lands where they meant to go after signing in. For an unknown path there's nowhere good to send them — round-tripping `/nonsense-xyz` through login just deposits them on a 404 *after* they authenticate. Plain `/login` is the honest answer.

## The 404 still matters — after login

Inverting the matcher doesn't make the 404 page pointless. It moves its audience. A *logged-in* user who fat-fingers a URL or follows a stale link should absolutely get a clean 404, and on this app that was its own checklist finding: Next.js was serving its bare default not-found page, which we replaced with a styled one that matches the app and offers a way back. Revealing "this route doesn't exist" to someone who's already authenticated leaks nothing — they're inside. The rule is just about *order*: authenticate first, resolve the route second. The 404 is for people who got past the gate.

There's a matching subtlety worth checking while you're in here. The same QA pass turned up a public route that rendered fine logged out — and then nobody could explain how a logged-out user was supposed to reach it in the first place. A page that's public but unreachable is either a missing link or a route that shouldn't be public at all. Default-deny surfaces those: the moment everything-not-listed redirects to login, every entry in your `PUBLIC` array has to justify itself.

## What I'd check in review

Log out. Type garbage into the address bar — `/asdf`, `/admin-x`, a real route name with one letter wrong.

- You get a **404 page** → your gate is an allowlist of protected routes, and the routing layer is answering "does this exist" to people who aren't logged in. For a private app, that's a route-enumeration leak.
- You get the **login page**, same as every real protected route → the gate runs before routing, and an outsider can't tell your real paths from noise.

Then read the matcher. If it lists the routes you want to *protect*, you've allowlisted the wrong set — the dangerous default is "everything I didn't think to name is open." Flip it: name what's public, deny the rest, and let the 404 be a thing only your authenticated users ever see.
