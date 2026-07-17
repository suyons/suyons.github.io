---
title: "TLS Troubleshooting - A Certificate From an Appliance Nobody Told Me About"
date: 2026-07-17
draft: false
tags: ["tls", "ssl", "certificates", "certificate-trust", "windows"]
categories: ["Security"]
description: "A client's document system failed TLS validation from the outside while working fine from inside their network. The cert wasn't expired or misconfigured — it was a dynamically minted interception certificate from an SSL-inspection appliance, and the fix was to trust it, not to email the client about their expired certificate."
showToc: true
---

## The symptom chain

A partner's document-management system, reachable over HTTPS on a nonstandard port, started failing TLS validation from our side — browser flagged it "Not secure," and access degraded to plain HTTP. That in turn broke the embedded document editor: it started dying with nginx's `400 Bad Request — The plain HTTP request was sent to HTTPS port`. Two failures, one root cause. The editor's backend builds its editing-session URLs using the same protocol as the page that requested them, so once the outer page fell back to HTTP, it handed the editor an `http://` URL — which the HTTPS-only nginx listener in front of the editor rejected outright. Nothing wrong with the editor or the nginx config; that failure was purely downstream of the TLS problem.

First instinct: their certificate expired and nobody renewed it. Reading the actual certificate said otherwise.

## Reading the certificate

Pulled the chain from the browser's certificate viewer:

```
Subject:  CN = example.com
SAN:      DNS:example.com, DNS:*.example.com
Issuer:   CN = ePrism SSL, O = SOOSAN INT, C = KR
Validity: 2026-07-08 → 2027-01-22  (~6 months)
Key:      RSA 2048-bit
Sig alg:  SHA-256 with RSA
```

That's not an expired certificate. It's a *wrong* one — signed by an issuer that has nothing to do with any public CA, or with the client's own domain. Two details gave it away immediately: a generic wildcard SAN covering the whole domain rather than the specific host, and a validity window of about six months, which is short for anything except an automatically-renewed cert.

## What ePrism actually is

First hypothesis was a botched cert renewal on their end — some TLS-terminating box serving its own fallback certificate by mistake. Looking up the issuer killed that theory fast: ePrism is an **SSL-inspection appliance** — it sits inline on outbound/inbound traffic, terminates the real TLS connection, inspects the plaintext, and re-signs the response with its own internally-generated CA before handing it back to the client. The certificate's shape — wildcard SAN, short validity, non-public issuer — is exactly what a dynamically minted interception cert looks like. This wasn't broken. It was the appliance working as designed.

That also explains why nobody on their side had noticed: their own machines have the appliance's root CA pre-installed in the OS trust store, so internal users get a green padlock without ever seeing the substituted cert. External clients — anyone without that root CA already trusted, connecting from outside their network — are the only ones who fail validation. The certificate's issue date most likely marks when the inspection appliance's policy or configuration last changed to start covering this host.

## Picking the fix

Three options, in the order I considered them:

1. **Ask the client to add an SSL-inspection bypass for this one endpoint.** The "proper" fix on paper, but it means asking a partner to carve an exception into their security policy for one external vendor. Slow at best, refused outright at worst.
2. **Ask them to install a public CA certificate on the inspection device instead.** Same friction, and it works against the entire point of an inspection appliance — it exists specifically to re-sign with a CA the org controls.
3. **Trust the appliance's root CA on my own client, the same way their internal PCs already do.**

Went with option 3. Asking a partner's network to change its security posture to accommodate one external client is backwards when the client can just join the trust model the network already runs. My browser (Brave) has no per-site trust override — it uses the OS certificate store — so the actual fix was:

1. Open the certificate chain in the browser's certificate viewer.
2. Export the **issuer** certificate — "ePrism SSL," one level up from the leaf, not the `example.com` certificate itself.
3. Import it into the user-scope trusted root store: `certmgr.msc` → *Trusted Root Certification Authorities* → *Certificates* → *Import*, or from the command line:

```
certutil -addstore -user Root eprism-ca.crt
```

`-user` matters here — it scopes the trust to the current Windows user, not machine-wide, which limits the blast radius of trusting a third party's CA to the one account that actually needs it.

After restarting the browser, HTTPS validated cleanly, and the editor's 400 error fixed itself — with the outer page back on HTTPS, the editor's backend generated `https://` URLs again, which the nginx listener accepted without complaint.

## Outcome

The document system is back on HTTPS, the editor works, and the appliance's root CA is trusted in one user-scope Windows certificate store. It is not trusted anywhere else — notably not in the trust store of any AWS-hosted service that might need server-to-server HTTPS to this same endpoint later. If that need ever comes up, the same CA has to be added to that runtime's trust store explicitly (`NODE_EXTRA_CA_CERTS` for Node, the JVM `cacerts` keystore for Java), and whoever owns that will need to track the appliance's CA rotation going forward — trusting a third party's inspection CA means trusting all of its certificate decisions on that path, indefinitely, until someone revokes it.

## Takeaways

- An unfamiliar certificate issuer is a clue about who's sitting in the network path, not just a validation failure. `O = SOOSAN INT` pointed straight at an SSL-inspection appliance before I'd even looked anything up.
- A generic wildcard SAN plus an unusually short validity window is the signature of a dynamically minted interception certificate, not a misconfigured or expired one.
- If the cert is coming from an inspection appliance, "your certificate is broken" emails are aimed at the wrong problem. The client's own machines already trust it — the fix belongs on the outside client, not their server.
- Trace an error chain to its root before fixing anything downstream. The editor's HTTP/HTTPS mismatch looked like an app or nginx bug and was purely a symptom of the TLS fallback one layer up.
- Certificate trust lives in the OS store, not the browser. Export the issuer certificate, not the leaf, and import it at user scope to keep the blast radius of trusting a third party's CA as small as possible.
