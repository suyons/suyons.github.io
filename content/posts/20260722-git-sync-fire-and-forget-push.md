---
title: "Self-Hosting - The Sync Script That Could Strand a Commit Forever"
date: 2026-07-22
draft: false
tags: ["git", "bash", "systemd", "automation", "self-hosting"]
categories: ["DevOps"]
description: "A one-way autopush script for a two-machine notes repo worked for months, until a rejected push exposed the gap between 'did this run just commit something' and 'do I actually have anything unpushed'."
showToc: true
---

Two machines, one private notes repo, and until this week, one-directional automation: edit a file, an autopush watcher fires `git add -A && git commit && git push`, done. It worked for months. It also had a hole nobody found until a push actually got rejected.

## The one-way version

`save.sh` (and its PowerShell twin `save.ps1`, since one of the two machines is Windows) did exactly one thing — commit whatever's dirty, then push:

```bash
if [[ -z "$(git status --porcelain)" ]]; then
  echo "no changes to save"
  exit 0
fi

ts="$(perl -MPOSIX -MTime::HiRes=time -e 'my $t=time; printf "%s.%03dZ", strftime("%Y-%m-%dT%H:%M:%S",gmtime(int($t))), int(($t-int($t))*1000)')"

git add -A
git commit -m "save: ${ts}"
git push
```

It never pulls. On a two-machine setup, the only thing keeping both sides converged was every push and pull independently succeeding, in order, forever.

## Where it actually breaks

The other machine runs a separate `note-pull` job on a systemd timer, pulling the same repo every minute — I've written before about the credential side of that setup. Its job, after fetching and rebasing, is to restore any local edits that were autostashed out of the rebase's way and recommit them. The old version of that step looked like this:

```bash
# Commit and push any restored local changes. (The autopush watcher would also
# catch these once the pop writes files, but committing here makes the pull
# self-contained.) The staged-empty guard skips no-op commits.
git add -A
if ! git diff --cached --quiet; then
    git commit -m "save: $(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")"
    git push origin HEAD
    echo "note-pull: committed and pushed restored local changes"
fi
```

Reasonable-looking: if there's something to commit, commit it, push it, log both. But look at the guard. The push only runs *inside* "did this invocation just make a commit" — not "do I currently hold anything unpushed."

That distinction is invisible in the common case. It only matters for one sequence: the autopush watcher's own `git push` gets rejected as non-fast-forward, because the other machine pushed to origin in the gap between autopush's commit and its push. Autopush is fire-and-forget — it doesn't retry. The next `note-pull` run fetches, rebases, and replays that stranded commit cleanly onto the new tip. But if *that particular* pull run had nothing stashed to restore, the "did I just commit" guard never fires, and the now-valid, still-unpushed commit just sits on that machine. It stays local until some unrelated file edit happens to trigger autopush again. If nothing does — the machine goes quiet for the weekend, say — it sits there indefinitely.

## The fix is a different question, not an extra check

```bash
# Commit any restored local changes. (The autopush watcher would also catch
# these once the pop writes files, but committing here makes the pull
# self-contained.) The staged-empty guard skips no-op commits.
git add -A
if ! git diff --cached --quiet; then
    git commit -m "save: $(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")"
    echo "note-pull: committed restored local changes"
fi

# Push whenever we hold commits origin doesn't. This covers the commit above AND
# the case autopush's fire-and-forget push can't: if a local commit's push was
# rejected as non-fast-forward, the rebase above replays it onto the new origin
# tip but leaves it unpushed — without this it would sit local until the next
# file change happened to trigger autopush again (indefinitely, if none did).
if [[ "$(git rev-list --count @{u}..HEAD 2>/dev/null)" -gt 0 ]]; then
    git push origin HEAD
    echo "note-pull: pushed local commits ahead of origin"
fi
```

The push already existed; the fix isn't adding one. It's replacing the condition under it. `git rev-list --count @{u}..HEAD` answers "are we ahead of upstream at all," regardless of *why* — a commit made thirty seconds ago in this run, or one rebased into place ten runs back that never found its own trigger to push.

## The rewrite

Once that shape of bug was visible on the pull side, `save.sh`'s one-way design had the identical blind spot, just with no pull step to ever notice it. The replacement, `sync.sh`, folds pull, commit, and push into one script, and every stage checks state instead of assuming it from what the previous stage did:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

git fetch --quiet

# 1. Pull if the remote has commits we don't. --autostash tucks any
#    uncommitted edits out of the rebase's way and restores them after.
if [[ "$(git rev-list --count HEAD..@{u})" -gt 0 ]]; then
  git pull --rebase --autostash
fi

# 2. Commit local changes if the working tree is dirty.
if [[ -n "$(git status --porcelain)" ]]; then
  ts="$(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")"
  git add -A
  git commit -m "save: ${ts}"
fi

# 3. Push if we have commits the remote doesn't.
if [[ "$(git rev-list --count @{u}..HEAD)" -gt 0 ]]; then
  git push
fi
```

Three conditionals, none of which needs to know whether either of the others ran this time: pull if upstream is ahead, commit if the tree is dirty, push if we're ahead of upstream. That's what makes it safe to invoke from a file-watcher, a timer, or by hand, in any order, without a stuck state that only clears on the next unrelated edit.

## What this doesn't fix

`--rebase --autostash` handles a clean linear rebase and nothing harder — a genuine merge conflict still stops the script and needs a human. Rare on two machines; the rewrite doesn't make it safer if a third one joins.

## Takeaways

- A "did I just do X" guard and a "do I currently have unpushed X" check look interchangeable and aren't. The gap between them only shows up when something upstream fails independently of the run you're looking at.
- Check state (`rev-list --count`, `git status --porcelain`) instead of inferring it from what the current invocation happened to do. The current run isn't the only thing that can leave work outstanding.
- A one-way autosave script on a multi-machine setup is a push with no corresponding pull. It works until two pushes collide, and "increasingly unlikely" isn't the same as "handled."
