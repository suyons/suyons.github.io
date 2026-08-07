---
title: "GitLab CI/CD Troubleshooting - Two Servers, One Byte-Identical Bundle"
date: 2026-08-07
draft: false
tags: ["gitlab-ci", "windows-server", "pm2", "git", "load-balancing"]
categories: ["DevOps"]
description: "Deploying the same Node.js app to two servers behind a load balancer meant reconciling a force-pushed branch, catching a silent capability drop, and designing a rollback that has to undo the server that already succeeded."
showToc: true
---

One pipeline deploying one app to one machine is the easy version of this problem. The harder version showed up the same afternoon: the same app running on **two** servers behind a single load balancer, where "deployed" is no longer a property of a machine but of a pair. Along the way: a force-pushed branch to reconcile, three process-manager apps to fold into one, and a rollback that has to undo a deployment that *succeeded*.

## Reconciling a force-pushed branch without guessing

The first `fetch` reported `+ d9c4117...58c19e3 main -> origin/main (forced update)`. The local branch and its remote had diverged: 55 commits on one side, 58 on the other. The tempting moves are both wrong — a merge fabricates history that never existed, a hard reset risks discarding real work.

Neither the commit count nor the graph answers the actual question, which is whether any local work exists *only* locally. Comparing the two commit-message sets does:

```powershell
git log --format='%s' main --not origin/main | Sort-Object > loc.txt
git log --format='%s' origin/main --not main | Sort-Object > rem.txt
Compare-Object (Get-Content loc.txt) (Get-Content rem.txt)
```

Three lines came back, all on the remote side. The remote was a rebase of the same 55 commits plus 3 new ones — every local commit survived under a new hash. `git diff` between the two tips confirmed it independently: exactly the seven files those three commits touched, nothing else. Two independent checks agreeing is what turned `git branch -f` to the remote tip into a mechanical decision rather than a leap of faith.

## "Replace these three apps with this one" hid a capability drop

The instruction was to replace the existing test-environment apps with a newly built one. Enumerating what was actually running showed three processes, not one: a web app and **two** scheduler processes, distinguished only by an environment variable naming which queue each one polled.

The new build folds the schedulers into the web process, so collapsing three into one looked right — except the test environment's config named a single queue. Deploying as instructed would have made one app silently stop processing the other queue's jobs. Not a build failure, not a health-check failure — just work that quietly stops happening.

Flagging that before deleting anything mattered: the fix wasn't to keep two processes, it was to fix the code. A follow-up pull brought down a commit converting the variable to a comma-separated list read with a `whereIn`-style query. One app, both queues, no gap.

The generalizable part: **"replace N things with this one thing" is a request whose safety depends entirely on what those N things were doing.** The instruction describes the mechanism; it doesn't assert the capabilities are equivalent. Enumerate before you delete.

## Ordinals are ambiguous; hostnames are not

The two servers were described as "server #1" and "server #2" in conversation. The box in front of me had a hostname ending in `-02` while being called #1. Raising it produced a reply that could be read either way — which is precisely the problem, and a stronger signal than either individual reading.

The fix was to stop using ordinals as identifiers at all. Job names, runner tags, and stages all derive from hostnames instead, and each runner picks its own tag by looking itself up:

```powershell
$tagByHost = @{ 'app01' = 'deploy-app01'; 'app02' = 'deploy-app02' }
$Env:RUNNER_TAG_LIST = $tagByHost[$env:COMPUTERNAME]
```

Naming alone is a convention, though, and conventions don't fail loudly. So each job asserts it landed where it thinks it did:

```powershell
if ($env:COMPUTERNAME -ne $env:EXPECTED_HOST) {
  throw "wrong server. expected=$env:EXPECTED_HOST actual=$env:COMPUTERNAME"
}
```

Without that guard, swapping the two runners' tags produces a **green pipeline that deploys one machine twice and leaves the other on old code**. Behind a load balancer, the symptom is requests that behave differently depending on which backend answers — one of the more miserable things to debug from the outside.

## The runner registered with the wrong shell

The vendor's example registration script defaults to `RUNNER_SHELL = 'powershell'`. That means Windows PowerShell 5.1, not PowerShell 7. The deploy script parses the process list with `ConvertFrom-Json -AsHashtable`, a parameter that only exists in PowerShell 6+.

Running the same one-liner under both shells settled it in seconds:

```
Windows PowerShell 5.1 -> FAIL: A parameter cannot be found that matches parameter name 'AsHashtable'
pwsh 7.6.4             -> OK
```

Caught before the runner service was installed, so it cost one line in a config file instead of a failed first deploy. Vendor example files encode a generic default, not your requirements — and the mismatch here was invisible until exercised, because both shells answer to the name "PowerShell".

## No wrapper needed, but the logon account is load-bearing

The question raised was whether the runner needed a service wrapper. It doesn't — the runner binary ships its own Windows service manager. The part that actually deserved attention was the account it runs as:

```powershell
gitlab-runner install --user ".\Administrator" --password "<REDACTED>"
```

Omit that and the service runs as the system account, which breaks two things at once. The process manager's daemon is **per-user**, so the job would start a second daemon and a duplicate process fighting for the same port. And stored Git credentials live in a user profile, so the pull would fail to authenticate. Both failures look like unrelated mysteries; both have the same one-line cause.

## A timestamp that looked like a skipped build

After the first two-server deployment, the built bundle's modification time still read an hour old — the time of a manual build, not the pipeline's. That reads exactly like a skipped build step.

It wasn't. The commit had touched only the CI config, so the bundle's contents were byte-for-byte what was already on disk, and the bundler's `compareBeforeEmit` option (on by default) declines to rewrite a file whose contents haven't changed. Rather than assert that explanation, I rebuilt and watched the timestamp stay frozen — turning a plausible story into a demonstrated one.

Worth keeping: **modification time is not evidence that a build ran, and unchanged output is not evidence that it didn't.** The startup line in the application log, timestamped to the minute of the deploy, was the fact that actually settled it.

## Byte-identical: verifying versus guaranteeing

The hard requirement was that both servers always serve an identical bundle. The obvious implementation — let each server build, then compare hashes — only *detects* divergence after both machines have already built, and leaves you deciding what to do about a mismatch mid-deploy.

Building the guarantee in is strictly better than checking for it. The first server publishes its bundle as a pipeline artifact; the second still builds — which keeps its other build outputs fresh and proves the source compiles there — but then **overwrites its bundle with the artifact** before restarting anything:

```powershell
Copy-Item $shipped $bundlePath -Force
$finalHash = (Get-FileHash $bundlePath -Algorithm SHA256).Hash
if ($finalHash -ne $upstreamHash) { throw "hash mismatch after replace" }
```

A difference between the two local builds is still reported, as a warning — it's a real signal about build determinism — but it no longer decides whether the deployment is safe, because identity is established by construction. The step sits before the restart, so a failure here leaves that server on its previous bundle.

End-to-end this was confirmed from outside the pipeline: the artifact the second server received and the bundle it now serves hash identically, and the artifact's modification time was inherited from the first server's build.

## The rollback has to undo the server that succeeded

Rolling deployment means the first server is already live when the second one fails. A rollback that only touches the failing machine leaves the pair split — exactly the state the byte-identity requirement exists to prevent.

So rollback jobs run on **both** runners with `when: on_failure`, each deciding independently whether it has anything to undo. The discriminator is a small state file written before the build, carrying the pipeline ID:

| scenario                     | first server  | second server           |
| ----------------------------- | ------------- | ------------------------ |
| first fails                   | roll back     | no-op (never deployed)   |
| first succeeds, second fails  | **roll back** | roll back                |

The no-op path matters as much as the rollback path — a rollback job that restarts a server which was never touched is its own outage. Since this code only ever executes when something is already broken, it's the last place to find out it was wrong, so the gate logic got a standalone test across four states: no state file, a file from an earlier pipeline, a matching pipeline, and a first-ever deploy with no previous bundle to restore. Four for four.

One ordering change came out of the same reasoning. Persisting the process list to disk had been sitting immediately after the restart; it moved to after the health check. Saving a broken state doesn't just record it — it makes the broken app **come back after a reboot**.

## Outcome and takeaways

Two pipelines ran green end to end. Verified from outside: matching commit on the deployed checkout, a new process ID, HTTP 200 on the configured port, a fresh startup line in the application log, and the artifact-versus-served hash comparison above. The rollback path itself has not been exercised — nothing was deliberately broken on a live test server — and that's stated rather than glossed, because an untriggered safety net is an untested one.

- **Two independent checks beat one confident one.** The branch reconciliation was safe because a message-set comparison and a content diff agreed. Either alone would have been a judgment call.
- **Derive identifiers from something the machine can answer.** Ordinals live in people's heads and disagree between them; hostnames don't. And when a naming convention carries real consequences, assert it at runtime — conventions fail silently, assertions don't.
- **Prefer constructing an invariant over checking it.** "Both servers must serve identical bytes" became true by shipping one file to both, not by building twice and hoping to catch a difference.
- **Rollback is a property of the deployment, not of the failing node.** If success on one machine plus failure on another leaves an inconsistent pair, the successful machine has to roll back too.

A callback to earlier the same day: a deploy checkout doubling as the working copy meant a commit could be pushed *from* the machine being deployed to. That morning it made a fast-forward a no-op. This afternoon it made the "previous commit" recorded for rollback equal to the commit being deployed. The bundle snapshot — the thing actually being served — was still correct, so rollback works; only the source-restore leg degrades to a no-op. Same root cause, a different symptom, worth designing out rather than remembering.
