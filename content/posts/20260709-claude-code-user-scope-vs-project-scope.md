---
title: "Claude Code Configuration - The Rule That Belongs in Every Repo vs. the One That Belongs in Just One"
date: 2026-07-09
draft: false
tags: ["claude-code", "ai-agents", "developer-tooling", "git", "commit-conventions"]
categories: ["DevOps"]
description: "A user-scope CLAUDE.md silently applies to every repo an agent touches. Splitting rules into user-scope and project-scope stopped a team-specific commit convention from leaking into unrelated projects."
showToc: true
---

## The setup

Claude Code reads instructions from `CLAUDE.md` files at two levels: a user-scope file at `~/.claude/CLAUDE.md`, loaded into every session regardless of which repo you're in, and a project-scope file inside a given repo, loaded only there. For months I kept everything in the user-scope file — naming conventions, function-size limits, a preference for early returns, all of it. That was fine right up until one rule needed to be specific to one codebase, and I'd never separated the two.

## Gotcha: a user-scope rule doesn't ask which repo it's in

One client project has a small, mostly Korean-speaking team that reads commit history to understand what shipped and why — no PR descriptions, no changelog, the commit log is the record. For that repo specifically, I wanted a stricter convention than "write a good commit message": a Conventional Commits header for tooling and CI, then a body with two explicit bullets — root cause, then fix — naming the actual file, route, or column touched, so a bug reads like a diagnosis instead of a summary. Translated into English for illustration, the shape looks like this:

```
fix: password reset link still works after expiry

- The reset handler updated the password without checking the token's
  expires_at column, so an old link never actually stopped working.
- Added an expires_at < now() check to the reset-password route
  before it's allowed to update anything.
```

I dropped that convention straight into the user-scope `CLAUDE.md`, because that's where all my other rules already lived. It worked exactly as specified — on every repo. Including a personal open-source project with no team, no shared log-reading habit, and no reason to structure commits that way. The convention wasn't wrong, it just wasn't universal, and a user-scope file has no concept of "just for that one client." Every session, in every repo, inherits everything in it.

## The fix: sort rules by whether you'd want them in a repo you haven't told the agent about yet

The dividing line that actually holds up: if a rule should apply to a repo you haven't even created yet, it belongs in user-scope. If it only makes sense given who reads that specific repo's history, or what that specific codebase already does, it belongs in that repo's own `CLAUDE.md`, sitting next to the code it governs.

By that test:
- **User-scope** — naming conventions, function-size and argument-count limits, "no abstraction that wasn't explicitly requested," fail-fast error handling. These are opinions about code quality that don't change based on which repo I'm in.
- **Project-scope** — the bilingual commit convention above. It exists because of who reads that one team's git log, not because of any property of the code itself.

The commit convention moved to that repo's `CLAUDE.md`. The user-scope file went back to holding only rules I'd apply to a stranger's repo I was sending a first PR to.

## A rule that earned its place in user-scope: naming the shortcut you knowingly took

One rule that survived the cut, because it really is repo-agnostic: every intentional simplification gets flagged inline with a fixed marker, e.g. `# ponytail: single global lock, fine under 10 req/s — swap for per-key locking if that changes`. The point isn't the marker text — it's that it forces two things to be written down at the moment the shortcut is taken, not reconstructed later: what the shortcut's ceiling is, and what the upgrade path looks like when you hit it.

The value of this rule doesn't depend on the codebase. A global lock standing in for per-key locking, an O(n²) scan standing in for an index, a naive heuristic standing in for a real model — all the same shape: fine now, with a known and named breaking point. That's exactly the kind of rule that belongs in user-scope, because it's a habit about *how I write code*, not a fact about any one project.

## Takeaway

A user-scope `CLAUDE.md` is a global default with no awareness of context — anything you put there ships to every repo the agent ever touches, including ones you haven't started yet. Before adding a rule, ask whether it's a property of how you write code everywhere, or a property of one codebase's team, history, or constraints. The first belongs in user-scope. The second belongs in the repo it's actually about, or it'll eventually show up somewhere it doesn't fit — like a bilingual commit convention on a solo repo with no one else to read it.
