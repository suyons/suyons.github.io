---
title: "An Order Total That Didn't Fit in an Integer"
date: 2026-05-31
draft: false
tags: ["prisma", "postgresql", "validation", "nextjs", "trpc", "qa"]
categories: ["DevOps"]
description: "On a QA pass I set an order quantity to 999,999 and checkout returned a 500. The cause was a money column typed as a 32-bit integer in Prisma, with no quantity cap to keep the total under its ceiling. Here's why Prisma Int bites for currency, and the two fixes you need."
showToc: true
---

## The 500 nobody designs for

I was doing a manual QA pass on a storefront checkout. Most of the list is the boring, necessary stuff: does the shipping fee show, does the total add up, does the Toss widget render. All green.

Then the line that actually earns its place on a checklist: **push the input past where a real user ever goes.** The quantity stepper had no upper bound, so I typed `999999` and hit pay.

The page showed a total in the trillions, sat for a beat, and the request came back `500`. The server log:

```
Invalid `prisma.order.create()` invocation:
Value out of range for the type: value "5389999946100" is out of range for type integer
```

That's not a validation message a user should ever provoke. It's the database refusing to write a number that doesn't fit the column. Two separate mistakes line up to produce it, and you want to fix both.

## Prisma `Int` is a 32-bit integer

Here's the column that blew up, more or less:

```prisma
model Order {
  id          String   @id @default(cuid())
  totalAmount Int
  // ...
}
```

`Int` in a Prisma schema maps to PostgreSQL `integer` — `int4`, signed 32-bit. Its ceiling is **2,147,483,647**. Just over two billion.

For a lot of apps that feels infinitely large, so nobody thinks about it. But this shop prices in Korean won, which has no minor unit — you store the whole amount as an integer, no cents. A mid-five-figure-won item is a five-digit number. Multiply by 999,999 and you sail past two billion into the trillions, which is exactly the `5389999946100` in the error. The column physically cannot hold it, Postgres raises `numeric_value_out_of_range` (SQLSTATE `22003`), and Prisma surfaces it as the invocation error above.

The mapping is the part people miss, so to be explicit about what Prisma gives you on Postgres:

| Prisma type | Postgres type | Max (signed)        |
|-------------|---------------|---------------------|
| `Int`       | `integer` / `int4` | 2,147,483,647   |
| `BigInt`    | `bigint` / `int8`  | 9,223,372,036,854,775,807 |
| `Decimal`   | `decimal` / `numeric` | arbitrary precision |

`Int` is the default reach for any whole number, and for counts, ages, quantities it's correct. For money it's a latent overflow waiting for a big enough order — or enough line items, or a currency with no minor unit, all of which shrink the headroom.

## Fix one: don't store money in `int4`

Switch the money columns to a type that can hold them. Two reasonable choices:

```prisma
model Order {
  id          String   @id @default(cuid())
  totalAmount BigInt   // int8: ~9.2 × 10^18, plenty for any real order
}
```

`BigInt` (int8) tops out around nine quintillion. No realistic order total reaches that. The cost is at the language boundary: Prisma maps `BigInt` to JavaScript's `bigint`, and `JSON.stringify` throws on a `bigint` —

```
TypeError: Do not know how to serialize a BigInt
```

— so anything that serializes an order (a tRPC response, a `res.json`) needs a serializer that handles it. tRPC with superjson does this for you; plain `JSON.stringify` does not, and you'll convert with `Number(order.totalAmount)` or `.toString()` at the edge. Know that going in.

If you'd rather not deal with `bigint` in JS at all, use `Decimal`:

```prisma
model Order {
  totalAmount Decimal @db.Decimal(15, 0)
}
```

`Decimal` is arbitrary precision and comes back as a `Prisma.Decimal`, not a float, so you also dodge the float-rounding problem if you ever do have a minor unit. It's the textbook answer for money. The tradeoff is ergonomics — every read is a `Decimal` object you call `.toString()` / `.toNumber()` on, and arithmetic is method calls, not `+`. For integer-won amounts I lean `BigInt`; for anything with fractional currency I'd take `Decimal` and never look back.

Whichever you pick, change it everywhere money lives — `unitPrice`, `shippingFee`, line subtotals — not just `Order.totalAmount`. A `BigInt` total computed from `Int` subtotals can still overflow on the intermediate sum before it ever reaches the wide column.

## Fix two: the quantity should never have reached the database

Widening the column makes the 500 go away, but it papers over the real defect: **a single order line accepted 999,999 units.** That's not a number any honest cart produces. It got to `prisma.order.create()` because nothing between the input box and the database said no.

Storage type is your last line of defense. The first line is input validation, and you want the rejection to happen at the boundary with a message a human understands, not as a `22003` from Postgres three layers down. With a tRPC + Zod mutation:

```ts
const addItemInput = z.object({
  productId: z.string(),
  quantity: z.number().int().min(1).max(99),
});
```

Now `999999` is rejected before any query runs, with a 400 and a field error you can render as "Maximum 99 per order" instead of a stack trace. Pick the cap from the business rule — per-item stock, a fraud threshold — but pick one. "Unbounded" is not a quantity.

And don't trust the number the client sends for the total. Recompute it server-side from the product's real price and the validated quantity, so a tampered payload can't dictate the amount charged:

```ts
const line = await db.product.findUniqueOrThrow({ where: { id: productId } });
const totalAmount = BigInt(line.price) * BigInt(quantity); // server is the source of truth
```

The client's idea of the total is a display convenience. The server's is the one that gets charged.

## Why both, and in that order

If you only widen the column, a `999999`-unit order succeeds — and now you've *persisted* a nonsense order that downstream code (inventory, payment capture, the warehouse) has to cope with. You traded a 500 for silent bad data, which is worse.

If you only cap the quantity, you've closed the path I happened to walk down, but the column is still a 32-bit ceiling. A high-value cart with many legitimate lines, or a currency change, can still overflow it without any single absurd quantity.

So: cap the input so garbage never enters, *and* size the column so a legitimately large total never overflows. The validation gives you a clean error at the edge; the column type gives you a floor under everything that slips past it. Defense in depth, where the two layers fail in different ways for different reasons.

The thirty-second version for a review checklist: every money column, check the Prisma type — `Int` is a bug in waiting. Every quantity or count from a client, check for a `.max()`. Then try to buy a million of something. The inputs nobody guards are the ones a QA pass exists to find.
