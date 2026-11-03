---
title: "ONLYOFFICE Troubleshooting - The 4 KB Header That Broke PDF Generation, and the Redirect That Undid the Fix"
date: 2026-10-20
draft: false
tags: ["onlyoffice", "docbuilder", "jwt", "nginx", "nodejs"]
categories: ["Backend"]
description: "A document-merge endpoint started failing with bare 400s once revision history pushed a doubly-nested JWT past nginx's header limit. The redesign fixed it, then exposed a second bug: a TLS-terminating proxy's redirect was silently stripping the Authorization header."
showToc: true
---

A document management backend generates merged PDFs — cover page, document body, revision history — by sending a script to ONLYOFFICE DocBuilder, the document-generation engine inside ONLYOFFICE Document Server. One morning, one specific document started failing with nothing more than `Request failed with status code 400`. Chasing that down led to a redesign of how the data gets sent, a return to an async job flow that had been quietly reverted weeks earlier, and a second bug that only showed up once traffic went through the real proxy.

## The symptom: a 400 with no DocBuilder error code

DocBuilder normally reports problems as numeric codes in a 200 response (`-3` for a conversion error, `-8` for an invalid token, and so on). A bare HTTP 400 meant the request never reached DocBuilder at all. The fastest way to find out why was to rebuild the exact request outside the server, one page at a time, and record the size of each part:

| page | fill data | script URL | `Authorization` header | result |
|---|---|---|---|---|
| cover | 509 B | 1.5 KB | 2.2 KB | 200 |
| body | none | 0.7 KB | 1.1 KB | 200 |
| revision history | 4 KB | 6.4 KB | **8.7 KB** | `400 Request Header Or Cookie Too Large` |

The answer came from the nginx reverse proxy in front of Document Server, which rejects request headers over 8 KB by default.

## Why a header held 8.7 KB

DocBuilder's HTTP API takes a URL pointing to a script, not the script itself. To avoid storing anything server-side, the backend put the template's fill values — the revision history table rows — into a signed JWT and attached it to the script URL as a query parameter. Document Server also requires every request to be signed with a shared secret, so the backend signed the whole request body, script URL included, and sent that JWT again in an `Authorization: Bearer` header.

That put the data two JWTs deep, base64-encoded twice:

```
Authorization: Bearer <JWT{ payload: { url: ".../script?token=<JWT{ sourceUrl, fill }>" } }>
```

Base64 adds about a third to the size, and the second layer encodes the first. 4 KB of table rows became an 8.7 KB header. Revision history only grows, so this document was simply the first one to cross the limit — nothing had changed except its size.

## Getting the data out of the header

Two experiments against the live server settled what Document Server actually reads:

- A request with *only* `{ token }` in the body, and no header at all, succeeds.
- A request with a deliberately wrong plain `url` field, but the correct `url` inside the token, still converted the right file.

Document Server reads only the signed token; plain body fields are ignored. That rules out "just send plain JSON" for a direct call — with JWT enabled, the data has to live in a token. What it doesn't need is a *second* copy in the header. DocBuilder also exposes an `argument` request parameter that scripts can read as a global `Argument` object, so the fill data doesn't need to be inlined into the script URL either.

The team already ran a small relay service — the "bridge" — next to Document Server. Moving the signing there gives the backend a clean contract: send plain JSON, and prove where it came from with a small token that holds only a hash of the body.

```js
// Before: the whole body is signed into a JWT and sent twice, once in the header
const token = jwt.sign({ payload: builderBody }, SECRET, { algorithm: "HS256" });
await axios.post(`${DOCSERVER}/docbuilder`, { ...builderBody, token },
  { headers: { Authorization: `Bearer ${token}` } });
```

```js
// After: plain JSON body, header token holds only the body's SHA-256 (~229 bytes)
const raw = JSON.stringify(builderBody); // { url, key, argument: { fills: [...] } }
const token = signSourceToken({ sha256: crypto.createHash("sha256").update(raw).digest("hex") });
await axios.post(`${BRIDGE}/docbuilder`, raw,
  { headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` } });
```

On the bridge side, the check is a hash comparison, and the token for Document Server only ever goes in the body:

```js
const claims = verify(bearerToken);
if (claims.sha256 !== crypto.createHash("sha256").update(raw).digest("hex"))
  throw new Error("token does not match body");
// ...
body: JSON.stringify({ token: sign({ payload: body }) }), // no Authorization header
```

The generated script now reads its fill values from `Argument` instead of having them baked in as a literal:

```js
// Before: fill values embedded as a JSON literal in the script
`${DOCX_FILL_SCRIPT}(${JSON.stringify(fill)});`
// After: fill values arrive per request; index i = the i-th file in the script
`if (typeof Argument !== "undefined" && Argument && Argument.fills && Argument.fills[${i}]) ${DOCX_FILL_SCRIPT}(Argument.fills[${i}]);`
```

Confirming that `Argument` actually worked took a deliberate test, not just a green response: the cover and revision PDFs came out about 5 KB smaller while the deployed script still ignored `Argument`, and returned to the exact original byte sizes once it read it.

Two smaller surprises along the way:

- **Deploy order matters when the contract changes.** The first check against the "updated" bridge got 400s that only the old code produces — the container hadn't actually picked up the new file. A one-line test (a tampered body expecting 401, a correctly signed one expecting 200) told the two versions apart before touching the backend deploy.
- **Internal URLs leak out.** In sync mode the bridge had been returning Document Server's own download URLs, which pointed at the container-internal `127.0.0.1:10000`. The fix keeps only the path and query of the result URL and swaps in the public host.

## Restoring the async path

With conversions working again, the next complaint was latency: "start PDF merge" took about six seconds to respond, because the route converted three pages one after another, merged them, wrote the file, and finalized the revision before replying. An async version had been written weeks earlier and then reverted. The old error logs explained why — the exact same `400 Request Header Or Cookie Too Large`. The async design had been fine; it had just hit the same header bug everything else had.

So it went back in, on top of the new contract. The route inserts a schedule row, hands the bridge one multi-file job with a `callbackUrl`, and returns immediately. The bridge polls Document Server and posts the resulting PDFs back as multipart form data; a callback route merges them and finalizes the revision. Response time dropped to about 140 ms.

## The bug that only appeared through nginx

The first documents processed asynchronously never finished. Their schedule rows stayed put, so the "latest revision" view kept showing an old version. The backend log showed every callback arriving and getting rejected with `401 jwt must be provided` — the `Authorization` header was simply missing.

The cause was a redirect. The backend built the callback URL from its own request, and since nginx forwards to it over plain HTTP, that URL started with `http://`. nginx answered with a `308` redirect to `https://`. Node's `fetch` follows redirects, but because the scheme changed it treats the target as a different origin and **drops the `Authorization` header**. A 308 preserves method and body, so the request still looked complete on the wire; only the credential was gone.

The end-to-end test hadn't caught this because its callback pointed straight at localhost, bypassing nginx entirely. The fix is one line — nginx was already sending `X-Forwarded-Proto`, it just wasn't being used:

```js
// Before
return `${req.protocol}://${req.get("host")}`;
// After: nginx terminates TLS; an http:// URL would 308-redirect and lose Authorization
return `${req.get("x-forwarded-proto") || req.protocol}://${req.get("host")}`;
```

The nginx access log confirmed it: before the fix, every callback showed a `308` then a `401`; after it, a single `200`. Cleanup meant deleting the stuck schedule rows and rerunning the merges — all three finished within 8 seconds, and the latest-revision view moved to the right version.

## Outcome and takeaways

PDF merges for documents of any size go through again. Trigger endpoints answer in about 140 ms instead of 6 seconds, and the sync path (used for printed reports) and the async path (used for document merges) now share one small "post to bridge" helper. What got verified, not just asserted: an offline test of the generated script including per-file fill selection, byte-for-byte PDF size comparisons before and after the `Argument` change, tampered-body rejection at the bridge, and real callbacks visible in the nginx access log.

- **Don't put data in headers.** Headers have small, proxy-specific limits — 8 KB is a common nginx default — and errors at that layer arrive as bare 400s that never reach your app's own error handling.
- **Signing tokens inflate their contents.** A JWT inside a URL inside another JWT encodes the same bytes twice, plus base64 overhead each time. To authenticate a large body, sign a hash of it instead of the body itself.
- **Find out what the server actually reads, don't assume from the docs.** One controlled experiment — wrong plain field, right token field — settled in minutes a question the API docs only hinted at.
- **Redirects strip credentials.** `fetch` drops `Authorization` on a cross-origin redirect, and a scheme change from `http` to `https` counts as cross-origin even on the same host. Behind a TLS-terminating proxy, build external URLs from `X-Forwarded-Proto`, not `req.protocol`.
- **Test through the real network path.** A localhost callback test proved the logic correct and still missed the bug that mattered. One request through the actual proxy would have caught it before it reached production.
- **Check why something was reverted before restoring it.** The logs showed the async path had been rolled back for the very bug just fixed — which turned "should we bring this back" into an easy call instead of a judgment one.
