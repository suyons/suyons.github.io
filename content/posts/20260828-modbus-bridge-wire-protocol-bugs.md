---
title: "Modbus Integration Troubleshooting - What a Modbus Wire Will and Won't Tell You"
date: 2026-08-28
draft: false
tags: ["modbus", "typescript", "nodejs", "express", "protocol-design", "testing"]
categories: ["Backend"]
description: "Four separate bugs in an HTTP-to-Modbus TCP bridge, all traceable to the same root mistake: inventing structure the wire protocol never actually provided."
showToc: true
---

I spent a session hardening an HTTP-to-Modbus TCP bridge — a small Express service that relays values from battery management system (BMS) controllers to an internal web app. What began as "point it at a local test server" turned into four separate bugs, each traceable to the same root mistake: inventing structure the wire protocol never provided.

## The simulator that lied

To test without hardware, I wrote a fake Modbus server. Temperature values were meant to be 32-bit floats spanning two 16-bit registers. The client read back `-nan` and `9.18e+37`.

The bug was in how the fake device answered. Modbus libraries expose a per-register hook, and I had used it directly:

```ts
// Before — called once per 16-bit word
const temperatureVector = {
  getHoldingRegister: () => randomTemperature(),
};
```

Each *half* of a float got an independent random draw. Reassembled, two unrelated 16-bit halves are just noise. The fix generates the whole float first, then slices out whichever words were requested:

```ts
// After — generate whole float slots, then slice
function readTemperatureWords(offset: number, length: number): number[] {
  const firstSlot = Math.floor(offset / FLOAT_WORDS);
  const lastSlot = Math.floor((offset + length - 1) / FLOAT_WORDS);
  const buffer = Buffer.alloc((lastSlot - firstSlot + 1) * BYTES_PER_FLOAT);
  for (let slot = firstSlot; slot <= lastSlot; slot++) {
    buffer.writeFloatBE(randomTemperature(), (slot - firstSlot) * BYTES_PER_FLOAT);
  }
  const start = (offset - firstSlot * FLOAT_WORDS) * 2;
  return Array.from({ length }, (_, i) => buffer.readUInt16BE(start + i * 2));
}
```

A test double has to be *coherent*, not merely *random*. Anything spanning multiple units must be generated whole and then split, never assembled from independent draws.

## One word, two words

The bridge itself had the mirror-image bug. It read a single register per address and returned it as an unsigned integer, so `-12.3` arrived as the float's high word `0xC144` and was reported as `-1606.0`. Fixing it took three coordinated changes: decode two words as a float, thread a per-endpoint register width through the read path, and repair an off-by-one in the block-chunking logic.

```ts
// Before — one unsigned word per address
const decodeValue = (raw: number) => raw;

// After — two words, big-endian IEEE-754
function decodeValue(words: number[]) {
  const buffer = Buffer.alloc(VALUE_WORDS * 2);
  buffer.writeUInt16BE(words[0], 0);
  buffer.writeUInt16BE(words[1], 2);
  return Number(buffer.readFloatBE(0).toPrecision(FLOAT32_DIGITS));
}
```

The `toPrecision(7)` deserves a note. Widening a 32-bit float to a JavaScript double exposes the representation error — `-12.3` serializes as `-12.300000190734863`. Seven significant digits is what a 32-bit float actually carries, so trimming there removes the artifact without inventing precision. It's a principled cutoff, not cosmetic rounding.

The chunking bug was subtler. The code packed addresses into read requests while the span stayed under the protocol's 125-register ceiling, but measured that span from the *first* address only — forgetting the last address needs its own width too. Blocks ending near the limit read one word short.

## The sentinel that collided

Originally the bridge filled unreadable addresses with `-1`. That seemed harmless until we learned the BMS uses `-1` as *its own* error flag. Two very different facts — "the device reported an error" and "the bridge could not read at all" — became the same value.

```ts
// Before — synthesize a value the device never sent
return words.map((w) => (w ? toSigned(w[0]) : FAILED));

// After — absence is not a number
return words.map((w) => (w ? toSigned(w[0]) : null));
```

The rule that emerged: a bridge translates, it does not invent. Every number in a response is now a decode of bytes the device actually sent. `null` is not a substituted value; it is the absence of one. A single response can carry all three states and keep them distinct:

```json
[{"address":40001,"value":-95.3},{"address":40003,"value":-1},{"address":40005,"value":null}]
```

A related trap surfaced while testing: `JSON.stringify` has no literal for `NaN` or `Infinity` and silently converts both to `null`. Any design that gives `null` a specific meaning inherits that collision for free. We accepted it here — those patterns only arise from corrupt reads — but it's worth knowing before assigning `null` a meaning of its own.

## Failure has a granularity

The most useful correction of the session was conceptual. A Modbus read is a *block* operation: "give me N words starting at address X." Failure therefore happens per block, never per address. An exception reply is a five-byte frame carrying a function code and an error code — no register data whatsoever. When a read fails there is nothing to translate for *any* address in that block.

That reframing explains everything else. Per-address failure values were always a fiction. It also made the fake device's failure injection obvious: roll the dice once per request, not once per register.

```ts
function failSometimes(name: string, addr: number, length: number) {
  if (Math.random() >= REQUEST_FAILURE_RATE) return;
  throw { modbusErrorCode: 0x04, msg: "Slave device failure (simulated)" };
}
```

Getting this right required adding a bulk-read hook to a fake server that previously had only the per-register one — otherwise the library would have called it 125 times per request and scattered failures across individual addresses, exactly the behavior we were trying to disprove.

## Measure, don't reason, about request counts

Midway through, a reasonable-sounding concern was raised: surely the bridge issues one Modbus request per address? Rather than argue from the code, I instrumented a fake server to log every request it received. The measurements settled it immediately:

| HTTP request | Modbus requests |
| --- | --- |
| 100 alarm addresses, one server | 1 (`qty=100`) |
| 100 float values (200 words) | 2 (`qty=124`, `qty=76`) |
| Two addresses 500 apart | 2 (`qty=2`, `qty=2`) |
| 126 addresses | 2 (`qty=125`, `qty=1`) |

The batching was already correct. Equally useful, the exercise disproved the proposed alternative: a single request for 1000 registers is not legal Modbus, because the reply's byte-count field is one byte wide. That ceiling is why the second row splits.

An instrumented probe answers this class of question in minutes and leaves evidence behind. Reading the code and asserting a conclusion produces neither.

## Deleting a mode nobody used

The service had shipped with two operating modes: one returning random values without opening a socket, one talking to real hardware. In practice only the real path was ever used. Removing the branch deleted a type, a request-local mode flag, two random generators, a parameter threaded through every service function, and an entire test file — roughly 130 lines net.

The interesting consequence was that the suite could no longer run without a reachable Modbus server. That's a real cost, worth naming rather than papering over: the fake device is now a *prerequisite* for testing, not a convenience.

## Outcome and takeaways

Everything landed and was verified against a live fake device rather than asserted: temperature values in range at one decimal place, alarm codes confined to their valid set, failure injection measured at 5.6% of blocks and error flags at 5.0% of words, and the documented error paths returning their specified status codes.

One consequence needed handling. Injecting 5% block failures made the end-to-end suite flaky — measured at one failure in twenty runs, because a failed alarm block turns the whole response into a gateway error. Wrapping only the happy-path reads in a bounded retry fixed it, and twenty-five consecutive runs passed afterward. The retry keeps its diagnostic value, since a bridge that genuinely reads nothing fails every attempt.

Three things worth carrying forward:

- **Don't invent values at a boundary.** A sentinel that overlaps real data destroys information. If a value is absent, say absent.
- **Match your abstraction's granularity to the protocol's.** Modbus fails per block; a per-address failure model was always going to produce impossible states.
- **Instrument rather than argue.** Two disagreements about behavior were settled in minutes by a logging test double, and the evidence outlived the conversation.

## What this doesn't fix

Still open, and documented rather than quietly fixed: even-numbered addresses on the two-word endpoint silently return a value assembled from two different readings' halves, and each request still opens and tears down its own TCP connection.
