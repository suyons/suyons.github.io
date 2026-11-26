---
title: "Claude Code Context Engineering - Splitting a 344-Line CLAUDE.md Into an Index and Four On-Demand Docs"
date: 2026-11-26
draft: false
tags: ["claude-code", "ai-agents", "developer-tooling", "context-engineering", "documentation"]
categories: ["DevOps"]
description: "A project-scope CLAUDE.md for a small self-hosted server had grown into a 344-line dump of host facts, credential rotation steps, and full systemd unit files, loaded into every session regardless of relevance - splitting it into an index plus topic docs fixed that, and forced a sharper secrets policy at the same time."
showToc: true
---

A project-scope `CLAUDE.md` is loaded into context at the start of every Claude Code session scoped to that repo - not fetched on demand, not summarized, the whole file, every time, whether the task at hand is "restart a timer" or "fix a typo in a note." The one for a small self-hosted server I run had grown to 344 lines: host facts, the full git credential rotation procedure, four systemd units with their complete unit files and scripts, and a paragraph-by-paragraph history of three abandoned approaches to syncing notes. None of that is wrong information. Almost none of it is relevant to a given task.

## The fix: an index, not a dump

The content didn't disappear - it moved into four topic files under `.claude/docs/`, and `CLAUDE.md` itself shrank to a lookup table:

| Before working on | Read |
|---|---|
| Host, users, sudo rules | `.claude/docs/server-host.md` |
| Git remote, authentication, token rotation | `.claude/docs/git-auth.md` |
| Autopush, pull, Quartz and restart units or scripts; install and verify commands | `.claude/docs/note-services.md` |
| Abandoned approaches and related notes | `.claude/docs/history.md` |

Four files, 343 lines total, replacing roughly 330 lines that used to sit inline in `CLAUDE.md`. The number that actually matters isn't the line count moved - it's that none of those 343 lines load unless the task touches that topic. A session that's just editing a note no longer pays the token cost, or the attention cost, of the git-credential rotation procedure sitting in context the whole time. Deciding to go read `git-auth.md` becomes a single judgment call driven by the index, instead of something the agent has to filter back out of a context it was never going to need.

This is the same principle as paging code into a function instead of inlining it everywhere - except the cost here isn't binary size, it's the model's attention budget on every single turn, paid whether or not the content is used.

## The split forced a sharper secrets rule

The old `CLAUDE.md` carried a reassurance: "the GitHub PAT is no longer embedded here - it lives in the credential store, so this file carries no live secret." True, but it's a statement about what *had* been done, not a rule about what's *allowed*. Moving the actual credential-rotation walkthrough into its own file (which still describes, correctly, how to overwrite a stored token - without the token itself) was the natural point to stop relying on "nothing happens to be here" and write the actual constraint down:

```diff
- This private vault may contain real values per the owner's explicit call.
+ This private vault may contain real IPs and domains per the owner's
+ explicit call, but credentials stay only in `.env` files.
```

```diff
- As of <date> the GitHub PAT is no longer embedded here - it lives in
- the credential store, so this file carries no live secret.
+ Never write credentials into this file, `.claude/docs/`, or notes.
```

The distinction matters because these files aren't just agent context - they're also markdown in a vault that gets rendered to a web viewer and pushed to a private GitHub repo. "No secret happens to be in here right now" survives exactly until someone (human or agent) pastes one in while debugging. "Credentials are only allowed to live in `.env`, full stop" is a rule that still holds after that. Splitting a file you're already restructuring is a good moment to upgrade an incidental fact into an actual boundary - you're rereading everything anyway.

## What the split doesn't fix

Every one of the four new files opens with the same line:

```markdown
<!-- Moved from CLAUDE.md on <date>. Not re-checked against the server. -->
```

That disclaimer is doing real work. Moving content into a tidier structure feels like it should come with a verification pass - four clean topic files, each with its own heading, reads as more authoritative than one long scroll, even when every fact inside is exactly as unverified as it was before the move. Flagging that explicitly stops the reorg itself from quietly becoming evidence that someone checked it. If a sudo rule or a systemd unit documented in one of these files has drifted from what's actually running on the server, splitting the file didn't catch that - it just made the drift easier to read.

## Takeaways

1. A project `CLAUDE.md` is loaded in full, every session, regardless of relevance. Treat length as a cost paid on every turn, not a one-time cost of writing it.
2. An index table ("before working on X, read Y") turns most of that content into something the agent fetches on demand instead of something it always carries - keep the file that's always loaded down to what's actually always relevant.
3. A reorg is a good time to turn an incidental fact ("nothing secret happens to live here") into an actual rule ("secrets are only allowed to live there") - you're already touching every line.
4. Label moved-but-unverified content as exactly that. A cleaner structure is not the same thing as a freshness check, and it's easy to let the former stand in for the latter if nothing says otherwise.
