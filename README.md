<div align="center">

# suyons blog

**A personal technical blog — the things I've built, the problems I've solved, and the lessons learned along the way.**

[![Deploy](https://github.com/suyons/suyons.github.io/actions/workflows/deploy.yml/badge.svg)](https://github.com/suyons/suyons.github.io/actions/workflows/deploy.yml)
[![Built with Hugo](https://img.shields.io/badge/built%20with-Hugo-ff4088?logo=hugo&logoColor=white)](https://gohugo.io/)
[![Theme: PaperMod](https://img.shields.io/badge/theme-PaperMod-2d3748)](https://github.com/adityatelange/hugo-PaperMod)
[![Hosted on GitHub Pages](https://img.shields.io/badge/hosted%20on-GitHub%20Pages-222?logo=github)](https://pages.github.com/)

**🌐 [suyons.github.io](https://suyons.github.io)**

</div>

---

## About

I'm an infrastructure and platform engineer. This blog is a window into how I think and
work beyond what fits on a résumé — air-gapped financial networks, GMP-regulated
pharmaceutical platforms, self-hosted home infrastructure, and the day-to-day of building
and operating production systems.

Posts cover practical, hard-won topics: infrastructure, deployment, system design,
databases, and the kind of bugs you only understand after they've bitten you in production.

## Tech stack

| Layer        | Choice                                                                 |
| ------------ | ---------------------------------------------------------------------- |
| Generator    | [Hugo](https://gohugo.io/) (extended) `v0.161.1`                       |
| Theme        | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (submodule) |
| Comments     | [giscus](https://giscus.app/) — GitHub Discussions backed             |
| Hosting      | GitHub Pages                                                           |
| CI/CD        | GitHub Actions — build + deploy on push to `main`                     |

## Features

- 🎨 **Light / dark theme** with auto system detection — comments stay in sync.
- 🖼️ **Profile-headed home page** with a synced bio.
- 💬 **Comments & reactions** on every post via giscus, blended into the page background.
- 🔎 **Full-text search** and **tag** browsing.
- 📄 **Custom enriched 404** with quick links and recent posts.
- 📰 **RSS feed** and reading-time estimates.

## Project structure

```
.
├── content/
│   ├── posts/          # blog posts (one Markdown file per post)
│   ├── about.md
│   └── search.md
├── layouts/            # project-level overrides of the PaperMod theme
│   ├── list.html       # home pagination → /posts/ section
│   ├── 404.html        # enriched not-found page
│   └── _partials/
│       ├── home_info.html   # home header + profile image
│       └── comments.html    # giscus embed + theme sync
├── assets/css/extended/     # custom CSS layered on top of the theme
├── static/             # images and other static assets
├── themes/PaperMod/    # theme (git submodule)
├── hugo.toml           # site configuration
└── .github/workflows/deploy.yml
```

## Local development

> Requires [Hugo extended](https://gohugo.io/installation/) `v0.161.1+`.

```bash
# Clone with the theme submodule
git clone --recurse-submodules https://github.com/suyons/suyons.github.io.git
cd suyons.github.io

# Run the dev server with live reload (drafts included)
hugo server -D

# Build the production site into ./public
hugo --gc --minify
```

The site is then available at <http://localhost:1313/>.

## Writing a post

Create a Markdown file under `content/posts/` named `YYYYMMDD-slug.md`:

```markdown
---
title: "Your post title"
date: 2026-06-13
tags: ["infrastructure", "deployment"]
draft: false
---

Your content here.
```

## Deployment

Every push to `main` triggers the [`deploy.yml`](.github/workflows/deploy.yml) workflow,
which builds the site with Hugo and publishes it to GitHub Pages. Drafts are staged on the
`draft` branch and merged into `main` when ready to publish.

## Credits

Built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
theme, deployed on [GitHub Pages](https://pages.github.com/).
