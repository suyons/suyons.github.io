---
title: "Cancel the Payment Before You Cancel the Order"
date: 2026-06-02
draft: false
tags: ["payments", "tosspayments", "distributed-systems", "prisma", "nextjs", "qa"]
categories: ["DevOps"]
description: "A QA checklist line — 'when the provider returns 4xx on cancel, the order must stay PAID' — encodes a real two-system consistency bug. If you flip the order status before the refund clears, a rejected cancellation leaves you with an order marked cancelled and money you never gave back."
showToc: true
---

## The checklist line that looks paranoid

I was writing out the QA checklist for an order-cancellation flow. Most of the cases are the obvious ones: the cancel button opens a modal, "leave" closes it without touching the order, a confirmed cancel flips the status and the refund goes out. Then one line that reads like overkill until you've been burned:

> When the payment provider returns a 4xx on the cancel call → show a failure message **and** verify the order is still `PAID` in the database.

That second clause is the whole point. It's not testing that cancellation works. It's testing that a *failed* cancellation leaves nothing half-done. And the only way that test can fail is if your code already moved the order to `CANCELLED` before it knew whether the provider agreed.

This is the bug I want to pull apart, because it's the kind that passes every happy-path test, never shows up in development, and surfaces in production as a customer who was charged for an order your system swears it refunded.

## Cancellation is a write to two systems

Placing an order is mostly one system's problem: your database. Cancelling one is two. You have to undo the charge at the payment provider (here, Toss Payments) *and* move your own order row from `PAID` to `CANCELLED`. Those are two separate networked writes to two systems you don't control together, and there is no transaction that spans both. You can't `BEGIN`, refund the card, update the row, and `COMMIT` as one unit. The provider's API has already done its thing by the time your `db.order.update` runs — or fails.

So the only lever you actually have is **ordering**: which write you attempt first, and what you do when the second one doesn't go the way you hoped.

The version almost everyone writes first updates the database first, because that's the system right in front of you:

```ts
// WRONG: flip the order, then ask the provider to catch up
await db.order.update({
  where: { id: orderId },
  data: { status: "CANCELLED" },
});

await tossCancel(order.paymentKey, "Customer requested cancellation");
```

Read it in the order it runs. You mark the order cancelled. *Then* you call the provider. If that call comes back `200`, you got lucky and the two systems agree. If it comes back `4xx`, you now have an order row that says `CANCELLED` and a charge that is still very much captured. The customer's money is gone, your fulfillment side thinks the order is dead, and the two facts will never reconcile unless a human notices.

A 4xx here is not exotic. The provider rejects cancellations for ordinary reasons: the payment was already cancelled (a double-click, a retry), the cancellable window has closed, the requested amount doesn't match, the payment isn't in a cancellable state. Toss surfaces these as documented error codes — `ALREADY_CANCELED_PAYMENT` and the like — with a 4xx status. Every one of those is a case where your optimistic `UPDATE` has already lied.

## Provider first, and gate the local write on its answer

Flip the order. Call the provider, and treat your own database update as something you've *earned the right to do* only after a success:

```ts
// call the provider first; only transition locally if it agrees
const res = await tossCancel(order.paymentKey, reason);

if (!res.ok) {
  // the provider refused — the order is still PAID, and that's correct.
  // surface the failure; do NOT touch the order row.
  throw new TRPCError({
    code: "BAD_REQUEST",
    message: "Cancellation was rejected by the payment provider.",
  });
}

await db.order.update({
  where: { id: orderId },
  data: { status: "CANCELLED" },
});
```

Now the 4xx path does nothing to your data. The order stays `PAID`, which is the truth — the money was never given back — and the user gets an error instead of a silent inconsistency. That is exactly what the checklist line asserts: not "cancel works," but "a rejected cancel is a no-op on our side."

The cancel call itself is a plain authenticated POST. Toss uses HTTP Basic auth with your secret key as the username and an empty password:

```ts
async function tossCancel(paymentKey: string, cancelReason: string) {
  const auth = Buffer.from(`${process.env.TOSS_SECRET_KEY}:`).toString("base64");
  return fetch(
    `https://api.tosspayments.com/v1/payments/${paymentKey}/cancel`,
    {
      method: "POST",
      headers: {
        Authorization: `Basic ${auth}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ cancelReason }),
    },
  );
}
```

Nothing clever. The discipline is entirely in *when* you let yourself write to your own database, not in the call.

## 4xx is the easy case. The ambiguous one is the timeout

Provider-first closes the failure mode the checklist tests, and it closes the most common one. But it would be dishonest to stop there, because it trades a frequent clean failure for a rare ambiguous one, and you should know exactly where the remaining gap is.

Walk the new ordering again. You call the provider. It actually cancels the payment — the refund is real — and then your `db.order.update` throws, or the process dies, or the network drops the response on the way back. Now the money *was* returned but your order still says `PAID`. The inconsistency just moved: instead of "cancelled but charged," you have "refunded but still marked paid." Rarer, because it needs a failure in the narrow window between the provider's success and your commit — but not impossible.

The distinction that matters: a **4xx is a definitive no.** The provider is telling you it did not cancel, so keeping the order `PAID` is correct and final. A **5xx or a timeout is a "don't know."** You cannot tell whether the cancellation went through, and so you must not assume either outcome — not "cancelled," and not "still paid."

Two things keep that window from becoming lost money:

1. **An idempotency key on the cancel request.** Send the same key on a retry and the provider collapses duplicate attempts into one, so a retry after an ambiguous failure can't double-refund. Toss keys cancellation idempotency off the `Idempotency-Key` header; set it to something stable per cancellation attempt, like the order ID plus a cancel sequence, not a fresh UUID each call.
2. **A reconciliation pass.** Periodically ask the provider for the true status of any order stuck mid-cancellation and converge your row to match. This is the backstop for every "don't know" the live request couldn't resolve.

If you want to make the in-between state explicit rather than implicit, give it a name — a `CANCELLING` status you set before the provider call and resolve to `CANCELLED` or back to `PAID` after. That turns "an order that's neither cleanly paid nor cleanly cancelled" from a thing you infer from logs into a row you can query and a reconciliation job can sweep.

For most shops, provider-first plus an idempotency key gets you the correctness that matters; the reconciliation job is the thing you add the first time a real order falls into the gap. What you should not do is the original ordering, because that version doesn't have a rare failure window — it's wrong on every single 4xx, which is to say routinely.

## The sticky-note version

The order of two writes you can't make atomic is a real design decision, not an implementation detail. For anything that touches money:

- **Call the external system first. Mutate your own row only after it says yes.** Your database is the one you can fix later; the charge is the one you can't quietly take back.
- **Treat 4xx and 5xx differently.** A 4xx is a definite no — stay put. A 5xx or timeout is unknown — don't guess; reconcile.
- **The test isn't "cancel works." It's "a cancel that fails changes nothing."** That line on the checklist is worth more than the five happy-path lines above it, because it's the only one that catches the inconsistency before a customer does.
