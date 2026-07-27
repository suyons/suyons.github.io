---
title: "Git History Cleanup - From Merge Spaghetti to a Clean Git-Flow"
date: 2026-07-25
draft: false
tags: ["git", "git-rebase", "version-control", "git-flow"]
categories: ["DevOps"]
description: "Linearizing a tangled backend repo's history, splitting it into git-flow branches, and the patch-id tricks that keep rebasing and branch pruning safe after a rewrite."
showToc: true
---

The `master` branch on one of our backend repos had turned into a knot of nested merge commits — dozens of feature branches folded in over months, merges inside merges, no way to read the graph without squinting. Cleaning it up meant four operations, each with its own way to go wrong: linearize a published branch, split it into `master`/`develop`, prune the branches that were actually done, and rebase one still-live feature branch across a whole-file reformat. The rule that held the whole session together: never take an irreversible step without a check proving it's safe first.

## Linearizing a branch other people already pulled

Almost all of the mess traced back to one thing — merge commits sitting directly on the mainline. The fix is a rebase that drops the merges and replays the real commits in a straight line. The catch: `master` was already pushed and shared, so rewriting it meant force-pushing over history other people had built on.

Two safeguards made that defensible. First, a backup tag pinned to the old tip before touching anything — an instant, complete undo path. Second, a tree-equality check after the rebase: diffing the new linear tip against the old messy tip has to come back empty, proving the rewrite changed the *shape* of history and not a single line of code.

```bash
# Before rewriting: pin an undo point
git tag backup/master-before-linearize master

# Replay everything since the divergence point as a straight line (merges dropped)
git rebase <base-commit>

# Prove no content changed — this diff MUST be empty
git diff <new-tip> backup/master-before-linearize
```

An empty diff there is the whole argument for force-pushing a rewritten shared branch with a straight face — you've mathematically shown the working tree didn't move.

The force-push itself hit a wall immediately: the remote rejected it because `master` was a protected branch. Branch protection is enforced server-side and doesn't care about your local permissions — someone with admin rights had to flip "allow force push" before it would go through, and we locked it back down right after. A clean local rebase says nothing about server-side policy.

## Splitting production from ongoing work

With `master` linear, the next step was a real git-flow layout: `master` tracks what's actually deployed, `develop` carries everything ahead of it. The production line ended at a specific known-deployed commit, so on paper the split was trivial — create `develop` at the current tip, move `master` back to the production commit.

The gotcha: linearizing had rewritten every commit hash, so the production commit the team remembered by its original SHA no longer existed under that name. Trusting a stale hash here would have been a mistake. Instead we matched the commit by its **patch-id** — a fingerprint of the actual diff content that survives a rebase — to find its new identity on the linear branch. Same change, new hash. Patch-id is how you follow a commit through a history rewrite.

## Pruning branches without trusting `--merged`

Dozens of merged feature branches were still cluttering the remote. The instinct is `git branch --merged` and delete everything it lists — except that command only counts a branch as merged if its tip is a literal ancestor of the target. After a rewrite, branches whose work got rebased into new hashes look *unmerged* even though their content landed just fine. The tool that actually answers the question is `git cherry`, which compares by patch-id instead of by ancestry:

```bash
# '-' = an equivalent change already exists on develop (safe to delete)
# '+' = genuinely unique work (deleting would lose it)
git cherry develop origin/feature/some-branch
```

This wasn't academic. A first attempt at a hand-rolled patch-id comparison had a silent bug: it fed `git log` output *without diffs* into `git patch-id`, which produces empty fingerprints and flags everything as unmerged, useless either way. Switching to `git cherry` fixed it and immediately caught two branches slated for deletion that in fact held commits that existed nowhere else. Surfacing that before deleting anything turned a near data-loss into a one-line decision.

One branch's story was different — its work had been migrated wholesale into a different repository. Before removing it we checked concretely: every endpoint and every key file from the branch confirmed present, some even extended, in the destination repo. Only then was it safe to drop from origin.

## Rebasing across a whole-file reformat

The hard part was folding one still-live feature branch back onto `develop`. It touched files `develop` had also touched, so conflicts were expected — the *shape* of one conflict was the actual lesson. The feature branch had run a formatter over a shared file, tabs to spaces, single quotes to double, across the whole thing. Meanwhile `develop` had landed a real security fix in that same file. Git can't align two versions that differ in almost every byte, so it flagged nearly the entire file as one giant conflict even though the semantic delta between the branches was a few lines.

The fix living only on `develop` was a wrapper that re-pins the authenticated user onto uploads, because the multipart parser was wiping a field the auth middleware had already set:

```javascript
// The fix that lived only on develop
function withCreatedWho(multipartUpload) {
  return function (req, res, next) {
    multipartUpload(req, res, function (err) {
      if (err) return next(err);
      if (req.method === "POST" && req.user && "created_who" in req.body) {
        req.body.created_who = req.user; // re-pin the logged-in user
      }
      next();
    });
  };
}
```

Applied at each of about a dozen upload call sites:

```javascript
// Before (feature branch — missing the fix)
return multipartUpload;

// After (resolved — feature branch's formatting + develop's fix)
return withCreatedWho(multipartUpload);
```

The decision that mattered was which side to build the resolved file on. The feature branch had eighteen commits, and later ones kept touching this same reformatted file — so resolving *onto the branch's formatting* and re-applying `develop`'s one security fix on top meant every later commit replayed cleanly. Resolving the other way would have re-triggered the identical whole-file conflict on every subsequent commit. When you rebase across a reformat, side with the new formatting and carry the semantic fixes forward — not the reverse.

A second conflict in the same rebase was a genuine design clash, not a formatting artifact. The feature branch re-activated an immediate "document read" flag write on a bare GET request; `develop` had deliberately replaced that with a stricter mechanism gated on a minimum viewing time, plus guards blocking submission until the read actually completed. Taking the feature branch's version would have quietly reopened a way to skip the viewing-time requirement. We kept `develop`'s stricter design, which made the feature branch's commit a no-op — and the rebase correctly dropped it as empty. An empty commit after a "keep theirs" resolution isn't a loss; it's the expected signal that the change was already superseded upstream.

## Outcome & takeaways

The repo went from a tangle of nested merges to two clean, linear branches — `master` at the production commit, `develop` carrying everything ongoing including the freshly rebased feature branch — every stale branch pruned and a backup tag left on the remote as a durable undo point. The merged result was then smoke-tested against the live development deployment: authentication succeeded, and every read-only endpoint across the areas the merge touched came back clean.

- **Back up, then prove equivalence.** A backup tag plus an empty `git diff` between old and new tips is what turns rewriting shared history from a gamble into a routine operation.
- **Follow commits by content, not by hash.** After any rebase, hashes are meaningless. `patch-id` and `git cherry` are how you tell "already merged" from "unique work" — get this wrong and you silently delete real code.
- **Branch protection is enforced server-side.** Your local permissions predict nothing about whether a protected branch will accept a force-push.
- **When rebasing across a reformat, side with the new formatting.** Resolve onto the reformatted version, re-apply the handful of real fixes on top, and downstream commits replay without re-fighting the same conflict.
- **State the actual scope of your verification.** The smoke test confirmed the running deployment behaved correctly, but nothing external proved that deployment was serving the exact commit we'd just pushed — worth saying plainly instead of rounding it up to "verified in production."
