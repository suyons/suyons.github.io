---
title: "Running Two Mail Providers on One Domain with Subdomain MX Records in Route 53"
date: 2024-11-14
draft: false
tags: ["aws", "route53", "dns", "googleworkspace", "email"]
categories: ["Infrastructure"]
description: "How to add a second mail provider to a domain that already uses Google Workspace by pointing a subdomain (mail.mydomain.com) to NAVER WORKS — without touching the existing root-domain MX records — all configured through AWS Route 53."
showToc: true
---

## The Situation

The primary company domain `mydomain.com` was already configured for Google Workspace. Its Route 53 hosted zone contained the standard Google MX records:

```
mydomain.com.  MX  1   aspmx.l.google.com.
mydomain.com.  MX  5   alt1.aspmx.l.google.com.
mydomain.com.  MX  5   alt2.aspmx.l.google.com.
mydomain.com.  MX  10  alt3.aspmx.l.google.com.
mydomain.com.  MX  10  alt4.aspmx.l.google.com.
```

A business requirement came up to also onboard NAVER WORKS as a groupware platform. NAVER WORKS is a Korean all-in-one collaboration suite (mail, calendar, chat, drive) and requires a verified domain to issue `@<your-domain>` addresses to its members.

The constraint: the Google Workspace mail for `@mydomain.com` must keep working. NAVER WORKS users would use `@mail.mydomain.com` addresses instead.

---

## Why This Works: MX Records Are Per Hostname

DNS MX records answer the question "which mail server should receive email for this hostname?" The hostname is the key. `mydomain.com` and `mail.mydomain.com` are two distinct hostnames, so they can each carry their own independent set of MX records pointing to completely different mail providers.

```
mydomain.com       MX → Google Workspace servers   (@mydomain.com mail)
mail.mydomain.com  MX → NAVER WORKS servers         (@mail.mydomain.com mail)
```

The two record sets have zero interaction. Changing or adding MX records for `mail.mydomain.com` does not affect delivery to `@mydomain.com` addresses in any way.

---

## Step 1: Register the Subdomain in the NAVER WORKS Admin Console

Before touching Route 53, register `mail.mydomain.com` as a domain in the NAVER WORKS admin console:

1. Log in to the NAVER WORKS admin console.
2. Navigate to **Domain** settings and add `mail.mydomain.com` as a new domain.
3. NAVER WORKS will display a **TXT record value** for domain ownership verification. Copy this value — it looks something like:

   ```
   naver-site-verification=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

4. NAVER WORKS will also display its **MX record values**. Note them for Step 3:

   | Priority | Mail server |
   |----------|------------|
   | 10 | `kr1-aspmx1.worksmobile.com.` |
   | 20 | `kr1-aspmx2.worksmobile.com.` |

Do not proceed with verification in the NAVER WORKS console yet — the DNS records need to be in place first.

---

## Step 2: Add the TXT Verification Record in Route 53

1. Open the AWS Route 53 console and navigate to the hosted zone for `mydomain.com`.
2. Click **Create record**.
3. Fill in the form:
   - **Record name**: `mail` (Route 53 will append `.mydomain.com` automatically)
   - **Record type**: `TXT`
   - **Value**: paste the verification string from NAVER WORKS, wrapped in quotes — `"naver-site-verification=xxxxxxxx..."`
   - **TTL**: `300` (5 minutes is fine for initial setup; increase to `3600` after verification)
4. Save the record.

After DNS propagates (typically within a few minutes, up to 48 hours globally), return to the NAVER WORKS admin console and trigger the domain verification. Once verified, NAVER WORKS confirms ownership and the domain becomes active.

---

## Step 3: Add the MX Records for `mail.mydomain.com` in Route 53

1. In the same hosted zone, click **Create record** again.
2. Fill in the form:
   - **Record name**: `mail`
   - **Record type**: `MX`
   - **Value** (enter each on a separate line):
     ```
     10 kr1-aspmx1.worksmobile.com.
     20 kr1-aspmx2.worksmobile.com.
     ```
   - **TTL**: `300`
3. Save the record.

The hosted zone now looks like this for the relevant records:

```
mydomain.com.       MX  1   aspmx.l.google.com.          ← Google Workspace (unchanged)
mydomain.com.       MX  5   alt1.aspmx.l.google.com.     ← Google Workspace (unchanged)
...
mail.mydomain.com.  TXT     "naver-site-verification=..."  ← NAVER WORKS ownership proof
mail.mydomain.com.  MX  10  kr1-aspmx1.worksmobile.com.  ← NAVER WORKS mail
mail.mydomain.com.  MX  20  kr1-aspmx2.worksmobile.com.  ← NAVER WORKS mail
```

---

## Step 4: Verify Mail Delivery

Send a test email **to** an `@mail.mydomain.com` address from an external mailbox. If the message arrives in the NAVER WORKS inbox, the setup is correct.

To confirm the DNS resolution is working before sending a test, look up the MX records from the command line:

```bash
# macOS / Linux
dig MX mail.mydomain.com

# Windows
nslookup -type=MX mail.mydomain.com
```

Expected output:

```
mail.mydomain.com.  MX  10  kr1-aspmx1.worksmobile.com.
mail.mydomain.com.  MX  20  kr1-aspmx2.worksmobile.com.
```

Also verify the root domain is still intact:

```bash
dig MX mydomain.com
```

It should still return only the Google Workspace MX records — the NAVER WORKS records should not appear here.

---

## Common Mistakes

**Adding MX records at the root instead of the subdomain.** If you accidentally create the NAVER WORKS MX records with an empty record name (which means `mydomain.com`), you will break Google Workspace delivery. MTA servers pick the lowest-priority MX record, and if Google's records now share the record set with NAVER WORKS records, email routing becomes unpredictable. Always double-check that the record name is `mail`, not blank.

**Forgetting the trailing dot on mail server names.** Route 53 adds the trailing dot automatically for MX values, but if you paste values from documentation that includes it (e.g., `kr1-aspmx1.worksmobile.com.`) it does not double-add it. If you paste values without the dot, Route 53 treats the value as relative to the hosted zone, which produces an incorrect FQDN. Use the full hostname with the trailing dot to be unambiguous.

**Verifying ownership before the TXT record propagates.** If you click Verify in the NAVER WORKS console immediately after saving the Route 53 record, the verification will fail because DNS has not propagated yet. Wait a few minutes and check propagation with `dig TXT mail.mydomain.com` before retrying.

---

## Summary

MX records are scoped to a specific hostname, so two mail providers can coexist under the same registered domain without interfering with each other. Routing Google Workspace to `mydomain.com` and NAVER WORKS to `mail.mydomain.com` required three Route 53 records — one TXT for ownership verification and two MX for mail routing — none of which touch the existing Google Workspace configuration. The full setup takes under 15 minutes; most of the wait is DNS propagation.
