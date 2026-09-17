---
title: "Image Compression Design - Why 'Just Convert to JPEG' Doubled My Screenshot Sizes"
date: 2026-09-17
draft: false
tags: ["python", "pillow", "image-compression", "tooling"]
categories: ["Backend"]
description: "A one-line 'just convert everything to JPEG' script made a folder of UI screenshots bigger, not smaller — the fix was to stop assuming one format wins and let the two candidates race."
showToc: true
---

I needed to shrink a folder of UI screenshots going into a report — same resolution, same DPI, just fewer bytes. The obvious move is "convert everything to JPEG, done." I tried it on a batch of screenshots first. The files got *bigger*. At quality 95, a screenshot that started life as a compact PNG came back out roughly twice the size.

That's not a fluke. JPEG's compression model — discrete cosine transform blocks plus chroma subsampling — is built for photographic gradients. A UI screenshot is the opposite: flat color panels, a handful of distinct colors, hard-edged text and borders. That's exactly what PNG's run-length and prediction filters are good at, and exactly what forces JPEG to spend bits encoding blocky noise around every sharp edge it wasn't designed for.

## Stop picking a format, let two candidates race

The fix isn't "use PNG instead of JPEG for screenshots" — that just swaps one blanket assumption for another; a real photo dropped into the same folder would lose to a bloated PNG the same way a screenshot loses to JPEG. The fix is to stop guessing and encode both, then keep whichever comes out smaller:

```python
def encode(im, fmt, quality, dpi):
    buf = io.BytesIO()
    opts = {"dpi": dpi} if dpi else {}
    if fmt == "PNG":
        im = im.quantize(colors=256, method=Image.Quantize.MEDIANCUT)
        im.save(buf, "PNG", optimize=True, **opts)
    else:
        # subsampling=0 (4:4:4): keeps chroma subsampling from smearing text/thin lines.
        im.save(buf, "JPEG", quality=quality, subsampling=0, optimize=True, **opts)
    return buf.getvalue()

fmt, data = min(
    ((f, encode(rgb, f, quality, dpi)) for f in ("PNG", "JPEG")),
    key=lambda pair: len(pair[1]),
)
if len(data) >= src.stat().st_size:
    return src, None  # the original was already smaller than either candidate
```

Two details matter here beyond "try both." First, the PNG candidate is quantized to a 256-color palette before saving — a plain `optimize=True` PNG save barely helps on a 24-bit screenshot, because most of the size is the color depth, not the entropy coding. Second, the JPEG candidate forces `subsampling=0` (4:4:4, no chroma downsampling), because the default 4:2:0 subsampling is exactly what smears thin text and hairline borders — the one artifact that makes a lossy screenshot look obviously wrong. Neither tweak is free: full chroma sampling makes the JPEG candidate bigger than it needs to be for cases where it should win, but that's the right trade when the failure mode of the alternative is visibly broken text.

The last line matters too: if both candidates lose to the original, leave it alone. A script that "compresses" a file into something larger than it started isn't compressing anything.

## When smaller is also lossless

Below 256 colors, palette quantization isn't lossy at all — every pixel maps to its own palette entry, no rounding, no information lost. The script checks for that up front and tags the result accordingly:

```python
with Image.open(src) as im:
    dpi = im.info.get("dpi")
    rgb = flatten(im)
    lossless = rgb.getcolors(maxcolors=256) is not None
```

`Image.getcolors(maxcolors=256)` returns `None` the moment the image has more than 256 distinct colors — it's not counting them, just bailing out past the threshold, so this check is cheap even on large images. Most UI screenshots — flat backgrounds, a handful of accent colors, black text — land well under that limit, so most of what this script processes gets marked `[lossless]` in its output, not just "smaller."

## The DPI number that's never quite the number you set

The one property this script guarantees is preserved is resolution and DPI, not just pixel count — a screenshot embedded in a printed document at the wrong DPI displays at the wrong physical size even though every pixel is identical. PNG doesn't store DPI directly; it stores an integer pixel-per-meter density, and 96 DPI doesn't convert to a round number in that unit:

```
96 dpi × (100 / 2.54) = 3779.5275... px/m → rounds to 3780
3780 px/m × (2.54 / 100) = 96.012 dpi
```

Round-trip a 96 DPI image through PNG and you get 96.012 DPI back — not a bug, just the unit conversion's own rounding. The self-check pins this with an explicit tolerance instead of asserting exact equality, which would fail on every single run:

```python
def selfcheck():
    tmp = pathlib.Path(tempfile.gettempdir()) / "compress-image-selfcheck.png"
    ref = Image.new("RGB", (400, 300), "white")
    draw = ImageDraw.Draw(ref)
    for y in range(0, 300, 7):
        draw.line((0, y, 400, y), fill=(0, 64, 160))
    draw.rectangle((50, 50, 350, 250), outline="black", width=3)
    ref.save(tmp, dpi=(96, 96))

    dst, fmt, lossless = compress(tmp, QUALITY)
    assert lossless, "a 3-color image should be judged lossless"
    with Image.open(dst) as out:
        assert out.size == ref.size, f"resolution changed: {out.size}"
        dpi = out.info.get("dpi")
        assert dpi and max(abs(v - 96) for v in dpi) < 0.1, f"DPI changed: {dpi}"
        assert out.convert("RGB").tobytes() == ref.tobytes(), "pixels changed"
    dst.unlink()
    print(f"selfcheck OK (format chosen: {fmt})")
```

That last assertion — comparing raw pixel bytes after a full round trip through quantize-and-save — is the one that actually proves the "lossless" claim rather than just asserting the flag came back `True`. I ran this self-check against the current script (Pillow 12.3.0) before writing any of this up; it passes, choosing the palette PNG path for the synthetic test image.

## Where it stays simple on purpose

Animated GIFs get flattened to their first frame — `Image.open` only reads the current frame under this code path, and nothing here iterates the rest. That's a known ceiling: fine for the screenshot-and-diagram traffic this script actually sees, wrong for anyone pointing it at animated assets without checking first. If that ever matters, the fix is iterating `ImageSequence.Iterator` and re-encoding as an animated PNG or WebP, not patching around the current single-frame path.

## Takeaways

- **"Just use format X" is a bet on your image's statistics, not a fact about compression.** Screenshots and photos have opposite statistical structure; a script that picks a winner per-image instead of per-project handles both without a config flag.
- **A DPI round-trip test needs a tolerance, not an equality check.** The unit PNG actually stores in doesn't map back to whole DPI numbers — asserting exact equality would make the test flaky by construction, not by bug.
- **`getcolors(maxcolors=N)` returning `None` is a cheap over-threshold check, not a count.** Useful anywhere you need "does this exceed N" without paying for "what is the exact number."
