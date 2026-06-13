---
title: "Your Number Input Accepts Infinity"
date: 2026-06-03
draft: false
tags: ["validation", "zod", "react", "forms", "typescript", "qa"]
categories: ["DevOps"]
description: "On a QA pass through an admin product form, three numeric fields accepted values no product should have: a price of infinity, a 5000% discount, and a sale price above the list price. Two of those share one root cause, and the third is a gotcha in how <input type=number> and JSON treat large numbers."
showToc: true
---

## The form nobody guards

The storefront side of a shop gets all the validation attention. It's public, it's where the fraud is, it's where a hostile user pokes at every field. So that's where the Zod schemas pile up.

The admin product form gets none of it. Only internal staff touch it, the threat model assumes good actors, and "they know what a price is" stands in for validation. That assumption is how I ended up, on a manual QA pass, submitting a product priced at infinity.

Three fields on that one form accepted values no real product should have:

- A price field that took `1e999` and stored it as infinity.
- A discount-rate field that accepted `5000` — a 5000% discount.
- A sale price set *higher* than the list price it was discounting from.

Two of those are the same bug wearing different clothes. The first one is a genuinely sharp detail about how the platform handles numbers, and it's worth pulling apart on its own.

## `<input type="number">` is more permissive than it looks

You'd think a number input rejects anything that isn't a sane number. It doesn't reject nearly as much as you'd hope, because the spec for what counts as a "valid floating-point number" allows scientific notation. `1e3` is valid. So is `1e999`.

And `1e999` is larger than the biggest number JavaScript can represent (`Number.MAX_VALUE`, about `1.8e308`). When a value overflows that ceiling, JavaScript doesn't throw or clamp — it gives you `Infinity`:

```js
Number("1e999");            // Infinity
Number.isFinite(Number("1e999")); // false
```

So a user types `1e999` into your price field, the input accepts it as valid, `el.valueAsNumber` hands your code `Infinity`, and unless something downstream checks for it, that's your price now.

Here's the part that turns a weird value into a silent corruption. The form POSTs JSON to the API, and JSON has no representation for infinity. `JSON.stringify` doesn't error on it — it quietly substitutes `null`:

```js
JSON.stringify({ price: Infinity }); // '{"price":null}'
```

So the journey is: the user enters `1e999`, the browser calls it valid, your client reads `Infinity`, and the wire turns it into `null`. The server receives `{ "price": null }` and, if it's lucky, rejects it with a "price is required" error — which is exactly the confusing symptom I saw, an error message complaining about a *missing* price for a field I'd very much filled in. If it's unlucky, `null` sails into a nullable column and you've persisted a product with no price.

`parseFloat("1e999")` is `Infinity` too, so swapping parsers doesn't save you. The only thing that saves you is checking for finiteness explicitly.

## Fix one: clamp at the field and assert finiteness on the server

The HTML `min`/`max`/`step` attributes help the spinner and trigger native validity, but they don't stop a user from typing `1e999` and submitting — `:invalid` styling is not enforcement. You need a real check, and the cheapest correct one is in your schema.

With Zod, `z.number()` alone won't catch this, because `Infinity` *is* a number as far as JavaScript is concerned. You have to reject non-finite values and bound the range:

```ts
import { z } from "zod";

const price = z
  .number()
  .finite()              // rejects Infinity / -Infinity / NaN
  .int()                 // integer won, no minor unit
  .min(0)
  .max(9_900_000_000);   // 9.9B ceiling — pick from your real domain
```

`.finite()` is the line that matters here; it's the one most schemas leave off because nobody pictures infinity arriving through a form. The `.max()` is the business bound — set it to whatever the largest legitimate product could cost, not to the integer ceiling. (If you store money as a 32-bit int, the column has its own much lower ceiling, and an unbounded price overflows that long before it reaches infinity — a separate failure I've written about, but the lesson rhymes: size the bound to the domain, not the type.)

Run the same schema on the server, not just in the browser. The client check is for the user's benefit; the server check is the one that's actually load-bearing, because the client one can be bypassed by anyone willing to open dev tools or `curl` the endpoint directly.

## Fix two: the invariants live between fields, not in them

The discount and the sale-price problems look like two bugs. They're one: **a constraint that spans two fields can't be expressed by validating either field alone.**

A discount rate of `5000` is a perfectly valid number. `99` is too. What makes `5000` wrong is the implicit rule that a percentage lives in `0–100` — which is a bound, fine, expressible per-field. But "the sale price must not exceed the list price" is genuinely relational. There's no value of sale price that's wrong *in isolation*; `50,000` is fine under a list price of `60,000` and nonsense under a list price of `40,000`. No amount of per-field validation reaches it, because the field doesn't carry enough context to judge itself.

This is what `.refine` / `.superRefine` exists for — a check that runs after the individual fields parse and sees the whole object:

```ts
const productForm = z
  .object({
    listPrice: z.number().finite().int().min(0).max(9_900_000_000),
    salePrice: z.number().finite().int().min(0).max(9_900_000_000),
    discountRate: z.number().int().min(0).max(100),
  })
  .refine((p) => p.salePrice <= p.listPrice, {
    message: "Sale price cannot exceed the list price.",
    path: ["salePrice"], // attach the error to the field the user can fix
  });
```

The `discountRate` bound is a plain per-field `.min(0).max(100)` — a percentage is self-contained. The sale-vs-list rule has to be a `refine` over the object, and the `path` is what makes the error land on the `salePrice` input instead of floating at the top of the form as a vague "this is invalid."

If you wanted to be thorough you'd also assert the three agree — that `salePrice ≈ listPrice × (1 − discountRate/100)` within a rounding tolerance — so a hand-edited discount can't disagree with a hand-edited sale price. Whether that's worth it depends on whether the three are independently editable or one is derived. The cheap, high-value rule is just `salePrice <= listPrice`; it catches the fat-finger that inverts them.

## The actual lesson is about who you trust

None of these three is exotic. They're the kind of input a tired internal user produces by pasting into the wrong field or holding down a key, not an attacker crafting a payload. That's the trap in "only staff use this form, so it doesn't need validation." The validation isn't there to stop a malicious admin. It's there to stop a normal one from saving a product priced at infinity at 4:55 on a Friday, and to give them a sentence they can act on instead of a `null` that surfaces three systems away as a checkout that won't add up.

The thirty-second version for a review: every `z.number()` that comes from a form wants `.finite()` before you trust it — `Infinity` is a number and `1e999` is how it arrives. Every percentage wants a `0–100` bound. And every pair of fields with a relationship between them — sale below list, end after start, max above min — wants a `refine`, because the field can't validate what the field can't see. Then go to the admin panel, the one nobody guards, and type `1e999` into the most expensive thing on the page.
