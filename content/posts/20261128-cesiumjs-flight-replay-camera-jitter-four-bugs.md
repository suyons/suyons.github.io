---
title: "CesiumJS Troubleshooting - Four Bugs Behind One Shaky Aircraft"
date: 2026-11-28
draft: false
tags: ["cesiumjs", "webgl", "3d-visualization", "real-time-sync", "performance"]
categories: ["Web Development"]
description: "Scaling a CesiumJS flight-replay viewer from 4 to 8 simultaneous aircraft surfaced four unrelated bugs behind the same symptom - a tracked aircraft that wouldn't hold still - from clock-sync drift to a one-frame race with the library's own render loop."
showToc: true
---

A web app replays recorded flight data in 3D on a CesiumJS globe, with multiple operators watching the same session. Scaling it from 4 to 8 aircraft and adding shared sessions surfaced visual instability in four different places, each with a different cause. None of them were the one I suspected going in.

## Bug 1: re-anchoring the clock on every poll

Playback time is computed as `anchor + (now − anchoredAt) × rate`. The first version polled the host's state and derived a fresh anchor on every response, using the server-reported age of that state:

```ts
// Before: shifted by the server-side age of the state, measured from the moment of arrival
anchoredAt = Date.now() - (session.serverNow - session.updatedAt);
```

That avoids comparing two machines' clocks directly, but it ignores how long the response spent in transit. That delay isn't constant, so each poll nudged the anchor by a slightly different amount, and the aircraft visibly drifted with it.

The fix: every screen computes from one shared estimate of server time, and the host's anchor moment is used as-is instead of re-derived per message.

```ts
// After: use the host's anchor moment as it is
anchoredAt = session.updatedAt;
```

The server-time estimate itself is a small NTP-style sync: each ping gives `offset = serverTime − (sentAt + receivedAt) / 2`, and among recent samples you keep the one with the shortest round trip, since it leaves the least room for one-way delay to be lopsided. Corrections apply gradually - about 1 ms per second - and only jump on the first sample or when off by more than a second. At 64× playback, a 1 ms clock step is 64 ms of simulated flight, so a sudden correction would be visible as a hop.

## Bug 2: paying full entity-rebuild cost every frame

Each aircraft was getting a brand-new position and orientation property assigned every frame:

```ts
// Before (every frame)
record.entity.position = new Cesium.ConstantPositionProperty(position);
```

Cesium treats a property assignment like that as an entity change and rebuilds its visualizers behind the scenes - not a cheap operation to repeat 60 times a second across 8 entities. The fix creates the property once and feeds it current values through a callback instead:

```ts
// After: created once
position: new Cesium.CallbackPositionProperty(() => current.position, false),

// After: every frame, just update the plain value it reads
record.current.position = position;
```

A second, related cost: wingtip trails added a point every frame. A 25-second trail held around 1,500 points, and 8 aircraft meant 16 trails rebuilt that often. The fix throttles trail points to one per 100 ms (250 per trail) while still letting the newest point track the wingtip smoothly between samples:

```ts
if (count >= 2 && now - trail.times[count - 2]! < TRAIL_POINT_INTERVAL_MS) {
  Cesium.Cartesian3.clone(position, trail.positions[count - 1]);
  trail.times[count - 1] = now;
  return;
}
```

Point: in a retained-mode scene graph, replacing an object can cost far more than mutating the value it already returns. If you're creating a new property or geometry every frame, that's the first thing to check before reaching for any other optimization.

## Bug 3: our handler ran after the library's own

With bug 1 fixed, the tracked aircraft was still shaking slightly around the screen center. Reading what Cesium actually does per frame explained why: the viewer subscribes its own tick handlers - for updating entities and the tracking camera - in its own constructors, and subscriptions run in registration order. Ours was added after, so it ran after.

Our handler moved the aircraft and only then advanced the clock. So on a given frame, the camera (Cesium's handler, running later that same tick on the *previous* clock value) was still catching up, while the model had already moved to the new position. The gap is roughly speed × frame duration, and frame duration is never quite constant, so the gap kept changing size - which reads as shaking, not drift. At 64× playback, estimated around 100 meters.

The fix makes every part of a frame use the same time: set the clock first, move the model, then let the library's own handlers run against consistent state.

```ts
const removeTick = viewer.clock.onTick.addEventListener(() => {
  viewer.clock.currentTime = nextTime();
  entityManager.tick();
});

// Move the library's own handlers behind ours
for (const [prototype, scope] of owners) {
  if (viewer.clock.onTick.removeEventListener(prototype._onTick, scope))
    viewer.clock.onTick.addEventListener(prototype._onTick, scope);
  else console.warn("Could not reorder the Cesium tick handlers");
}
return removeTick;
```

`owners` pairs Cesium's `CesiumWidget` and `Viewer` classes with their live instances, so their private `_onTick` handlers can be unregistered and re-added after ours. `_onTick` is a private name - if a library upgrade renames it, `removeEventListener` returns `false`, the code logs a warning instead of silently doing nothing, and the one-frame mismatch comes back. Depending on an unexported name is a real trade-off; making the failure loud is how you keep it from becoming a mystery regression two versions later.

## Bug 4: a second controller fighting the camera

Picking an aircraft manually tracked it correctly. Leaving the choice to the automatic default made the camera wander. The same wandering showed up after returning to a tab that had sat in the background.

The cause: with no aircraft chosen, the screen first kicks off an animated `camera.flyTo()` to a home view (up to 3 seconds), and automatic tracking assigns the first aircraft to `viewer.trackedEntity` right after. In the Cesium version in use (1.147), setting `trackedEntity` does not cancel a running `flyTo`. Both then move the camera every frame, in different directions, until the animation finishes. Picking an aircraft by hand skips the home flight entirely, which is why it never showed the bug.

The background-tab case has the same root cause for a different reason: Cesium starts the flight animation on the first frame it actually renders, not when `flyTo()` is called. A tab that hasn't drawn a frame yet still has the entire animation ahead of it once you switch back.

```ts
const entity = entityManager.getEntity(id);
if (!entity) return;
// Setting trackedEntity does not stop a running camera.flyTo
viewer.camera.cancelFlight();
viewer.trackedEntity = entity;
```

One rule this generalizes to: before handing a camera (or any shared resource with its own animation state) to a different controller, cancel whatever animation might still be running against it. "Setting the new owner" and "the old motion has stopped" are two separate facts, and a library's API won't always imply the second from the first.

## What's confirmed and what isn't

Type checks, lint, the test suite, a clock-sync verification script, and a socket smoke test against a development server all passed for every fix above. None of that confirms what it looks like on screen - I don't have browser automation in this workflow, so the actual visual smoothness of bugs 1 through 4 is still unverified by eye. One unrelated gotcha from the same session worth keeping in mind: after fixing an unrelated checkbox-limit bug, a retest kept showing the old behavior because the server was serving a static build made before the edit - type checks and test scripts don't touch what the browser actually loads, so a stale build can look exactly like a reverted fix.

## Takeaways

1. The same visible symptom - a camera or model that won't hold still - can have unrelated causes stacked on top of each other. Fixing clock sync didn't rule out a frame-order bug, and fixing that didn't rule out an animation-state collision.
2. In a retained-mode 3D library, assigning a new object every frame is not the same cost as updating the value an existing object returns.
3. Event-subscription order is part of a render loop's behavior. If a library subscribes to its own clock before you do, your handler runs after its.
4. Before handing a shared resource like a camera to a new controller, cancel whatever animation state the previous controller left running.
5. A passing type check and a passing test script say nothing about what's actually being served to the browser.
