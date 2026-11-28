---
title: "SVG Troubleshooting - A Diagram That Followed the Wrong Dark Mode"
date: 2026-11-28
draft: false
tags: ["svg", "css", "draw-io", "dark-mode", "vscode", "documentation"]
categories: ["Web Development"]
description: "Making a draw.io entity diagram adapt to a dark editor theme worked in a headless browser test and then painted every box black in a light VS Code theme - because the SVG was reading the operating system's color preference, not the editor's."
showToc: true
---

An entity-relationship diagram had outgrown Mermaid, so I moved it to draw.io and embedded the export as an SVG in a Markdown doc. The follow-up request sounded small: white borders in a dark editor theme, black borders in a light one. My first answer made the light theme noticeably worse.

## The approach that looked right

A `.drawio.svg` file is both the rendered picture and the editable source, so it can be embedded with a plain image link - no build step. draw.io also has an "adaptive colors" export setting, and with it set to `auto`, the SVG comes out using CSS `light-dark()`:

```css
color-scheme: light dark;
stroke: light-dark(black, white);
```

I checked it by rendering the file in a headless browser forced into dark mode. Lines and text came out white. It looked done.

## What actually happened

In a light VS Code theme, every box rendered with a black background. The reason: an SVG loaded through a Markdown `![]()` image link is a separate document, and the editor's Markdown preview doesn't pass its own theme into that document. `light-dark()` resolved against the operating system's color-scheme setting instead - which happened to be set to dark, regardless of which editor theme was active. My headless test had only proven that the SVG reacts correctly to *a* color preference. It said nothing about which preference the actual host - the editor's preview pane - exposes to an embedded image.

That's the part I'd flagged as unconfirmed going in, and it turned out to be the one thing that mattered. A test that sets the input by hand doesn't show that the real host sets the same input.

## The fix: stop adapting, go back to fixed

Reverting to a fixed light palette took two separate changes, not one:

- Turning off draw.io's adaptive-colors setting wasn't enough on its own, because the command-line export still wrote `color-scheme: light dark` into the file regardless. That line needed patching directly in the exported SVG:

```css
/* Before: follows whatever color preference the embedding document exposes */
color-scheme: light dark;

/* After: always the same, regardless of host or OS setting */
color-scheme: light;
```

- Black diagram lines in the gaps between containers had been relying on a dark host background to read as readable contrast. Once the diagram stopped adapting, the gaps needed their own solid white background - not just white box fills - or the lines between containers disappeared on a dark page.

## Outcome and the actual takeaway

The diagram is now a fixed white card with black lines, confirmed readable on both light and dark host pages in a rendered check. The `color-scheme` patch has to be reapplied after every command-line export, since the exporter writes it fresh each time, and an editor extension that touches the file on save could plausibly undo it too - worth a comment in the project notes for whoever edits this file next.

1. An embedded SVG image does not automatically inherit the theme of the app displaying it. If the embedding app's theme and the operating system's theme can disagree, a fixed palette with its own explicit background is the safe default - "adaptive" only works when you control, or can verify, what it's actually adapting to.
2. The alternative - making the editor follow the OS theme instead of the file adapting to it (`window.autoDetectColorScheme` in VS Code) - only helps the subset of people who already want that behavior. It's a user-level choice, not something a shared file should assume.
3. Testing `light-dark()` by forcing a browser's color-scheme preference proves the CSS works. It doesn't prove which preference your real target - an editor preview, a static site renderer, a chat client - actually forwards to an embedded image.
