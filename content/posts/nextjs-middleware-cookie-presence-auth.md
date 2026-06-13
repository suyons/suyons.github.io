---
title: "Your Next.js Middleware Trusts the Cookie, Not the Session"
date: 2026-05-29
draft: false
tags: ["nextjs", "authentication", "security", "middleware", "trpc", "qa"]
categories: ["DevOps"]
description: "A QA pass on a Next.js App Router shop turned up a route guard that only checked whether a session cookie existed. Setting app-session=1234 walked straight past it — even though the data API correctly returned 401. Here's why that gap appears, why it's still a bug when the API holds, and the logout 'ghost session' that comes with it."
showToc: true
---

## The test that should have passed

I was doing a manual QA pass on a Next.js (App Router) storefront. One line on the checklist:

> Visit `/cart` while logged out → redirect to `/login?from=/cart`.

It passed. Then, out of habit, I opened devtools, set a cookie by hand —

```
app-session=1234
```

— reloaded `/cart`, and the app treated me as logged in. No redirect. Same for `/mypage`, `/mypage/orders`, `/mypage/wishlist`. The value `1234` is not a session. It's four characters I typed. The guard waved it through.

That's the whole bug, and it's worth pulling apart because the failure is subtle: the *real* data was never exposed, the API did its job, and the checklist item that was supposed to catch this still showed green.

## Presence is not validity

Route protection in Next.js usually lives in `middleware.ts`. The shape almost everyone writes first:

```ts
// middleware.ts — the bug
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

const PROTECTED = ["/cart", "/mypage", "/checkout"];

export function middleware(req: NextRequest) {
  const path = req.nextUrl.pathname;
  const needsAuth = PROTECTED.some((p) => path.startsWith(p));
  if (!needsAuth) return NextResponse.next();

  const session = req.cookies.get("app-session");
  if (!session) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("from", path);
    return NextResponse.redirect(url);
  }

  return NextResponse.next(); // cookie exists → you're in
}

export const config = { matcher: ["/cart/:path*", "/mypage/:path*", "/checkout/:path*"] };
```

`req.cookies.get("app-session")` returns a value if *any* cookie by that name is set. `1234` qualifies. The check answers "is there a cookie?" when the question is "is this a session I issued, and is it still good?"

The reason this slips through review is that it works for every honest user. A logged-in browser has a real cookie; a logged-out browser has none. The presence check and the validity check agree in both ordinary cases. They only diverge when someone sets a junk value on purpose — which no normal flow does, so no normal test exercises it. The checklist asked "logged out → redirect," and a truly cookieless browser *does* redirect. The item passed because it never tested the gap between *having a cookie* and *having a valid one*.

## Why it's still a bug even though the API held

Here's the part that made this interesting rather than just embarrassing. With `app-session=1234` set, the page shell rendered — but the data calls underneath it didn't lie:

```
GET /api/trpc/shop.cart.count,shop.wishlist.isWishlisted  →  401
```

The tRPC layer validated the session properly and refused. So the actual cart contents, the order history, the wishlist — none of it leaked. The real authorization boundary was the API, and it held.

So is it a real problem? Yes, for three reasons:

1. **The page renders an authenticated UI to an unauthenticated visitor.** Even if every data call 401s, the server-rendered HTML can ship layout, feature flags, route structure, and sometimes props that were computed before the API call fails. Anything baked into the initial render that *didn't* go through the protected API is exposed.
2. **It's a broken, confusing state.** A logged-out user staring at a "logged-in" page whose every panel is silently failing is a support ticket waiting to happen, and a tell that the trust model is inconsistent.
3. **It teaches you the middleware is decorative.** If the only thing actually guarding data is the API, then the middleware isn't a security control — it's a redirect convenience. That's fine, *as long as you know it* and don't add a feature later that trusts the middleware to have done the gatekeeping.

The lesson I'd put on a sticky note: **middleware auth is a UX optimization, not a security boundary.** The boundary is server-side, at the data layer, every request. Middleware just saves an honest user a wasted round trip to a page they can't use.

## Fixing both halves

There are two things to fix, and you want both.

**Make the middleware check validity, not presence.** Middleware runs on the edge, so you can't comfortably hit your database, but you *can* cryptographically verify a signed token. With a JWT session and [`jose`](https://github.com/panva/jose):

```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { jwtVerify } from "jose";

const secret = new TextEncoder().encode(process.env.SESSION_SECRET);
const PROTECTED = ["/cart", "/mypage", "/checkout"];

async function isValidSession(token: string | undefined): Promise<boolean> {
  if (!token) return false;
  try {
    await jwtVerify(token, secret); // checks signature + exp
    return true;
  } catch {
    return false; // tampered, expired, or garbage like "1234"
  }
}

export async function middleware(req: NextRequest) {
  const path = req.nextUrl.pathname;
  if (!PROTECTED.some((p) => path.startsWith(p))) return NextResponse.next();

  const valid = await isValidSession(req.cookies.get("app-session")?.value);
  if (!valid) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("from", path);
    return NextResponse.redirect(url);
  }
  return NextResponse.next();
}
```

Now `1234` fails `jwtVerify` and gets redirected. So does an expired token, and so does a token signed with the wrong key. If your sessions are opaque (a random ID looked up in a store, not a JWT), you can't verify them at the edge — verify a signed *wrapper* you control, or move the check to a server component / route handler that can hit the store. Either way, the rule is the same: never branch on existence alone.

**Keep the real check at the data layer.** The middleware fix stops the page from rendering, but don't let it become the thing you rely on. Every protected resolver / route handler should re-derive the session server-side and 401 on its own — which, in this app, it already did. Belt and suspenders, and the suspenders are the ones holding your pants up.

## The logout ghost session

The same cookie-shaped thinking produced a second symptom that looked scarier than it was. The flow:

1. Log in. Browse a product page.
2. Log out. You land on `/login`.
3. Press the browser **Back** button — and the authenticated product page is right there again, fully rendered.

That's not your session resurrecting. It's the [back/forward cache (bfcache)](https://web.dev/articles/bfcache): the browser froze the fully-rendered page in memory and restored the DOM snapshot on Back, without re-running your server logic or re-checking anything. The cookie was already cleared; the *pixels* were cached. The giveaway in this case: hitting refresh on that restored page kicked the URL to a malformed `/login?category=workpants` — the stale client state colliding with a real navigation.

It feels like a security hole ("I logged out and the protected page came back!"), and to a non-technical observer it is indistinguishable from one. The fix is to tell the browser not to serve authenticated pages from bfcache:

```ts
// on responses for authenticated pages
headers.set("Cache-Control", "no-store");
```

`no-store` makes the page ineligible for bfcache, so Back re-fetches and your (now fixed) middleware redirects to `/login`. If you'd rather keep bfcache for performance, the lighter-touch option is to detect the restore on the client and force a reload:

```ts
window.addEventListener("pageshow", (e) => {
  if (e.persisted) window.location.reload(); // page came from bfcache
});
```

I prefer `no-store` on anything behind auth. The reload trick works but flashes the protected content for a frame before bouncing, which defeats the point.

## What I'd actually check in review

If you take one thing from this: grep your middleware for the auth branch and read what it tests.

- `cookies.get(...)` with no verification of the value → presence-only bug. Every time.
- A protected page that renders while its own data calls 401 → your middleware and your API disagree about who's logged in. The API is right; fix the middleware, and don't promote the middleware to a security control.
- Authenticated pages without `Cache-Control: no-store` → Back button shows them after logout.

None of these are exotic. They're the default thing you write when "add auth" is a checklist item instead of a threat model, and they pass every test that only exercises the honest paths. The fake cookie is a thirty-second test. Add it to the list.
