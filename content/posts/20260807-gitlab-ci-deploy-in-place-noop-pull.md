---
title: "Windows Server Troubleshooting - A Green Pipeline That Deployed Nothing"
date: 2026-08-07
draft: false
tags: ["gitlab-ci", "windows-server", "pm2", "git", "nodejs"]
categories: ["DevOps"]
description: "Automating a manual deploy-and-restart for a Node.js backend on a Windows build server turned up a pipeline that passed every step in 48 seconds while proving almost nothing — because its most important step had silently done no work at all."
showToc: true
---

A Node.js backend on a Windows build server was being deployed by hand every time: pull, test, build, restart. The goal was to automate that for the dev environment only, using a self-hosted GitLab Runner already installed on the same machine. Writing the pipeline was the easy part. Proving it actually worked was not — the naive first run silently skipped the single most important step.

## The unusual constraint: in-place deployment

Most CI pipelines clone the repository into a fresh workspace, build there, and ship an artifact somewhere. This setup does the opposite: the server's checkout **is** the deployment target. The running process serves a bundle built directly inside that directory, and the process manager launches it from there.

That makes the runner's default clone step actively harmful — it would build in a throwaway workspace while the live app keeps serving the old bundle. The fix is to tell the runner not to fetch anything and drive the checkout from the job script instead:

```yaml
variables:
  GIT_STRATEGY: none # the runner does not clone; we deploy in place
  DEPLOY_DIR: 'D:\PROJECT'
script:
  - Set-Location $env:DEPLOY_DIR
  - git switch $env:DEPLOY_BRANCH
  - git pull --ff-only origin $env:DEPLOY_BRANCH
```

`--ff-only` is deliberate. Without it, a divergent history on the deploy server produces a **merge commit on the production checkout** — a genuinely confusing state to debug later. With it, a history that can't fast-forward fails the pipeline loudly instead of quietly inventing a merge.

A consequence of `GIT_STRATEGY: none` is that splitting tests into their own stage would be wrong. Separate stages get separate — here, nonexistent — workspaces, so a test stage would examine the code as it was *before* the pull. Everything lives in one job, in order. The upside is a useful failure mode: if tests fail, the build never runs, so the previously built bundle keeps being served. Only the working tree is left updated.

## Pinning the toolchain, because service sessions have no shell hooks

The machine uses a Node version manager whose shell hook only applies to interactive sessions. A runner executing as a Windows service inherits none of it, so `node` resolves to whatever happens to be first on the system `PATH`. That matters because the project has native modules compiled against one specific Node ABI — building under a different major version produces a bundle that dies at startup.

Rather than trusting the environment, the job constructs its `PATH` explicitly and then asserts the result:

```yaml
- $env:Path = "$env:NODE_DIR;$env:PNPM_DIR;$env:Path"
- if ((node -v) -notlike 'v16.*') { throw "Not Node 16 -> $(node -v)" }
```

A small win came out of checking rather than assuming: the globally installed process manager lived inside that same Node installation directory, so one `PATH` entry supplied both `node` and `pm2`. A sibling project on the same box needed two separate entries because its build runtime and its process-manager runtime were different major versions — the two projects only looked alike from the outside.

There's a related subtlety in version logging. The package manager launcher on the box was newer than the version pinned in the project manifest, and it only self-switches to the pinned version **while inside the project directory**. Logging versions before changing directory would print a version that never touches the build. So the log line goes after the `Set-Location` — a small ordering fix that prevents a genuinely misleading build log.

## Restart: why `startOrRestart` beats `restart`

The obvious command is wrong in a subtle way.

Before (what you'd reach for):

```
pm2 restart MY_APP
```

After:

```
pm2 startOrRestart ecosystem.config.js --only MY_APP --update-env
```

`pm2 restart <name>` relaunches from the process entry the daemon already has saved — including the stored executable path and interpreter. If the ecosystem config changes which script or which interpreter to use, a plain `restart` cheerfully ignores it and relaunches the old configuration. `startOrRestart` re-reads the config file, so config changes actually take effect.

The health check also deserved more than a fixed sleep. This app initializes a bridge to a secondary runtime and registers scheduled jobs at startup, so a hardcoded wait can check too early and fail a deployment that in fact succeeded. Polling until healthy, with a deadline, is barely more code and strictly more correct:

```powershell
$deadline = (Get-Date).AddSeconds(60)
do {
  Start-Sleep -Seconds 3
  $app = @((pm2 jlist | Out-String | ConvertFrom-Json -AsHashtable) | Where-Object { $_.name -eq $env:PM2_APP })
  $pending = @($app | ForEach-Object { $_.pm2_env.status } | Where-Object { $_ -ne 'online' })
} while (($app.Count -eq 0 -or $pending.Count -gt 0) -and (Get-Date) -lt $deadline)
if ($app.Count -eq 0)     { throw "process not found" }
if ($pending.Count -gt 0) { throw "not online" }
```

Two non-obvious details are baked in. `-AsHashtable` is required because the process list embeds environment variables containing keys that differ only in letter case, and PowerShell's default JSON conversion rejects that outright. And `@(...)` wrapping matters because a single-instance app returns one object rather than an array, so `.Count` would otherwise be meaningless.

## The validation that almost didn't validate anything

The first pipeline run passed in 48 seconds. Every step green. And it proved less than it appeared to, because the pull step reported:

```
Already up to date.
```

The commit had been pushed *from the deploy checkout itself* — it's the only checkout on the box — so by the time the pipeline ran, the target directory was already at the right commit. The single most important step in the whole deployment, the fast-forward, had never actually moved anything. A green pipeline had certified a no-op.

Fixing this meant manufacturing the real condition: rewind the local checkout by one commit so the remote is genuinely ahead, then re-trigger the job. Retrying the existing job is the right move here, rather than creating a fresh pipeline through the API — the job is restricted to push events, and an API-created pipeline reports a different trigger source, so it would simply be skipped. A retry preserves the original triggering source. This time:

```
Your branch is behind 'origin/main' by 1 commit, and can be fast-forwarded.
Fast-forward
 1 file changed, 108 insertions(+)
```

That's the step doing real work. Worth noting: the rewind is safe precisely because build output is untracked. Rewinding the source doesn't disturb the bundle currently being served, and the pipeline restores the checkout as a side effect of its own pull.

## Outcome and takeaways

Both runs finished green, and the second one exercised the genuine fast-forward path. Beyond the pipeline's own status, the deployment was verified from outside: the application answered an HTTP request on its configured port, the listening socket belonged to the newly spawned process, and the application log recorded a fresh startup line for each run. A second concurrent push also entered a `waiting_for_resource` state before running, confirming two pushes can't build in the same directory at once.

- **A passing pipeline is not proof that a step ran.** Read the log for evidence of *work*, not just the absence of errors. "Already up to date" and "compiled successfully" carry very different weight, and a step that no-ops on the happy path can hide a real defect for months. If a step can no-op, deliberately construct the state where it can't.
- **Verify the environment instead of copying it from a neighbor.** Adapting a working pipeline from a sibling project was the right starting point, but half its complexity existed for constraints this project didn't share. Checking each assumption produced a simpler config, not just a correct one.
- **Check where a service actually runs before writing operational instructions.** The runner here was a Windows service, not a process under the process manager, despite a leftover config file in the runner directory strongly implying otherwise. Stale configuration files are confident liars; the service registry is the source of truth. And the process's logon account is what determines whether stored Git credentials and the process manager's daemon resolve at all — more on that in the next post.

One incidental find, unrelated to the pipeline itself: the application log showed a recurring scheduled-job error firing every five minutes, predating any of this work. Its unit test passes, because the test drives a fake database layer and can't catch a genuine query-binding mismatch. A reminder that a green test suite bounds what you've checked, not what works — I haven't root-caused this one yet.
