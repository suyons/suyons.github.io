---
title: "The Cart Shows 9999. The Server Stored 999."
date: 2026-06-08
draft: false
tags: ["react", "react-query", "optimistic-ui", "ecommerce", "validation", "qa"]
categories: ["DevOps"]
description: "A QA pass let me set a cart quantity to 9999. The API quietly clamped it to 999 — and the screen kept showing 9999, with a line total to match. When the server normalizes an input, an optimistic UI that doesn't reconcile is rendering a number the server already threw away."
showToc: true
---

## The quantity that two systems disagreed about

On a manual QA pass through a shopping cart, I changed an item's quantity to `9999`. Then `0`. Then `-1`. The stepper let me. No error, no clamp at the input — the number just went wherever I pushed it, and the line total and cart subtotal recalculated to match. `9999 × 15,000` is a big, satisfying number to see on a cart page.

The interesting part wasn't that the input accepted it. It's what the network tab showed. The request that went out carried `quantity: 999`, not `9999`. The server had a clamp. It took my absurd value, pinned it to the top of the allowed `1–999` range, stored `999`, and returned a perfectly sane cart.

And the screen still said `9999`.

So now there are two answers to "how many of this item are in the cart." The server says `999`. The page says `9999`, and the total under it — the number the customer reads as *what I'm about to pay* — is computed from a quantity that no longer exists anywhere but in the browser. The two will stay disagreed until something forces a refetch, at which point the number silently jumps and the user wonders if they misread it.

That gap is the bug I want to pull apart, because it's not "the input needs validation." The validation was *there*. It ran, on the server, correctly. The failure is that the client never listened to the answer.

## Optimistic UI quietly assumes the server won't change your value

Here's the shape of the stepper that produces this. It's the obvious one, and it's wrong in a way that passes every happy-path test:

```tsx
function QuantityStepper({ item }: { item: CartItem }) {
  const [qty, setQty] = useState(item.quantity);
  const updateItem = useUpdateCartItem();

  function change(next: number) {
    setQty(next);                                  // optimistic, unbounded
    updateItem.mutate({ itemId: item.id, quantity: next });
  }

  return (
    <div>
      <button onClick={() => change(qty - 1)}>−</button>
      <input
        type="number"
        value={qty}
        onChange={(e) => change(e.target.valueAsNumber)}
      />
      <button onClick={() => change(qty + 1)}>+</button>
      <span>{formatKRW(qty * item.unitPrice)}</span>  {/* line total from local qty */}
    </div>
  );
}
```

Read what `change` does. It sets local state to whatever `next` is — `9999`, `0`, `-1`, no bound — and fires the mutation. The line total renders off that local `qty`. The mutation's *response* is ignored. There's no `onSuccess` that reads what came back, because the mental model is "I already know the new quantity, I just typed it."

That model holds exactly as long as the server agrees with you. The moment the server transforms your input — clamps it, rounds it, normalizes it, snaps it to a stock limit — the optimistic value and the stored value fork, and nothing pulls them back together.

The server side is the responsible one here, which is what makes this sneaky:

```ts
// server: the clamp that's doing its job
const quantity = Math.min(999, Math.max(1, Math.trunc(input.quantity)));

await db.cartItem.update({
  where: { id: input.itemId },
  data: { quantity },
});

return getCart(userId); // returns the canonical cart, with quantity: 999
```

This code is correct. It refuses to store `9999`, refuses `0`, refuses `-1`, hands back a clean cart with the real numbers in it. It even returns the canonical cart so the client *could* reconcile. The client just throws that return value away.

## Why a wrong quantity is worse than a wrong number

If this were a quantity sitting in isolation, the cost would be cosmetic — a label that's briefly out of date. It isn't isolated. The quantity is an input to two other computations the cart shows, and both inherit the lie.

The line total is the obvious one: `qty × unitPrice`, rendered from the fake `9999`, so the customer sees a subtotal an order of magnitude off.

The subtler one is the free-shipping threshold. This cart shows a 3,500 KRW shipping fee under 30,000, and free shipping at or above it. That threshold gets evaluated against the subtotal — and if the subtotal is computed from a phantom quantity, the shipping line is wrong too. A cart that should show "3,500 shipping" can flip to "free shipping" on the strength of items the server never accepted. The customer reads a number, makes a decision based on it, and the decision is built on a value that exists only in their tab.

Then they hit checkout. Checkout recomputes from the server's real cart — it has to, that's where the money is — and the total changes. Best case, it's a confusing jump right before payment. Worse case, you've spent the entire cart page showing a price you were never going to charge, which is the kind of thing that turns into a support ticket that starts with "your site lied to me."

## The fix is to render the server's answer, not your guess

The instinct is to fix this at the input: add `min={1} max={999}`, clamp in `onChange`, disable the minus button at 1. Do that — it's a better interaction. But it is not the fix, and it's important to be clear about why. Input-side clamping makes the *common* case match, but it's the same rule implemented in two places, and the day the server's bound changes (a per-item stock limit, a promotional cap) the client's copy is stale and you're back to two systems disagreeing. The client guard is for UX. It is not the source of truth.

The actual fix is one sentence: **the displayed quantity must come from the same place the price is computed from — the server's response.** The mutation already returns the canonical cart. Use it.

```tsx
function QuantityStepper({ item }: { item: CartItem }) {
  const updateItem = useUpdateCartItem();

  function change(next: number) {
    // optimism is fine — but clamp it with the SAME rule the server uses,
    // so the optimistic value and the stored value can't fork.
    updateItem.mutate({ itemId: item.id, quantity: clampQty(next) });
  }

  // qty is read from the cart query cache, which the mutation refreshes
  // from the server response — not from a local useState the server can't see.
  return (
    <div>
      <button onClick={() => change(item.quantity - 1)} disabled={item.quantity <= 1}>−</button>
      <input
        type="number"
        min={1}
        max={999}
        value={item.quantity}
        onChange={(e) => change(e.target.valueAsNumber)}
      />
      <button onClick={() => change(item.quantity + 1)} disabled={item.quantity >= 999}>+</button>
      <span>{formatKRW(item.quantity * item.unitPrice)}</span>
    </div>
  );
}
```

The shared clamp is one function imported by both the client and the server, so there's exactly one definition of the bound:

```ts
export const QTY_MIN = 1;
export const QTY_MAX = 999;

export const clampQty = (n: number) =>
  Math.min(QTY_MAX, Math.max(QTY_MIN, Math.trunc(Number.isFinite(n) ? n : QTY_MIN)));
```

And the mutation reconciles the cache from what the server returned, so even if the optimistic value were somehow wrong, the truth wins within a round trip:

```ts
function useUpdateCartItem() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (vars: { itemId: string; quantity: number }) => api.cart.update(vars),
    onSuccess: (canonicalCart) => {
      qc.setQueryData(["cart"], canonicalCart); // render the server's numbers
    },
  });
}
```

If you want the snappy optimistic update with no flicker, do it in `onMutate` — but apply `clampQty` there too, and let `onSuccess` overwrite with the server's cart. The rule that keeps you honest: optimism is a *prediction* of the server's answer, and a prediction has to be reconciled against the real answer when it arrives. The bug at the top of this post is optimism that was never reconciled — a prediction the code treated as fact.

## One more decision the clamp was hiding

There's a product question buried under the `Math.max(1, …)`. What does stepping down to `0` mean? Clamping it to `1` says "you can never have zero of an item that's in your cart," which quietly means the minus button is *not* how you remove things — there must be a separate remove action, and the minus should stop at 1. The other reasonable design is that hitting 0 removes the line. Both are fine; what's not fine is a clamp that silently turns the user's `0` into `1` so the item they tried to remove is still there at quantity one. The clamp made the symptom (a stored `0`) disappear and left the intent (remove this) unhandled.

That's the theme, really. A server-side clamp is necessary and it was the correct call. But a clamp is the server *disagreeing* with the input, and any time the server disagrees, the client has two jobs: show what the server actually decided, and make sure the disagreement wasn't standing in for a feature you forgot to build.

The thirty-second version for a review: if a mutation can return a value different from what you sent — and a clamp, a normalize, or a rounding step means it can — then render the response, not the request. Type `9999` into your own cart, watch the network tab, and check that the number on screen matches the number that came back. If it doesn't, your cart is quoting a price you'll never charge.
