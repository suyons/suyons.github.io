---
title: "Getting Started with Hugo"
date: 2026-05-10
draft: false
tags: ["hugo", "static-site", "tutorial"]
categories: ["Tools"]
description: "A quick walkthrough of setting up Hugo with PaperMod and deploying to GitHub Pages."
showToc: true
cover:
  alt: "Hugo logo"
  relative: false
---

## What is Hugo?

[Hugo](https://gohugo.io/) is a static site generator written in Go. It converts Markdown files into a complete website. The main draw: it's **fast** — we're talking milliseconds per page even for large sites.

## Prerequisites

- [Hugo extended](https://gohugo.io/installation/) (v0.120+)
- Git

```bash
# macOS
brew install hugo

# Verify
hugo version
```

## Create a new site

```bash
hugo new site my-blog
cd my-blog
git init
```

## Add the PaperMod theme

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Then set the theme in `hugo.toml`:

```toml
theme = "PaperMod"
```

## Create your first post

```bash
hugo new content posts/my-first-post.md
```

This creates a file with pre-filled front matter:

```yaml
---
title: "My First Post"
date: 2026-05-10
draft: true
---
```

Change `draft: false` when it's ready to publish.

## Run the dev server

```bash
hugo server --buildDrafts
```

Open `http://localhost:1313` to preview.

## Build for production

```bash
hugo --minify
```

The output lands in `./public/`. That's what you deploy.

## Deploy to GitHub Pages

With the GitHub Actions workflow included in this repo, every push to `main` automatically rebuilds and deploys the site. Just push your changes:

```bash
git add .
git commit -m "new post"
git push
```

Done — your site updates in ~1 minute.
