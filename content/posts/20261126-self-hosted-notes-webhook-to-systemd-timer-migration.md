---
title: "Self-Hosting - A Wildcard Cloudflare Access Policy Killed a GitHub Webhook, So a Systemd Timer Took Over"
date: 2026-11-26
draft: false
tags: ["git", "systemd", "cloudflare-access", "self-hosted", "linux"]
categories: ["Infrastructure"]
description: "The GitHub webhook that pulled a self-hosted notes vault couldn't get past a wildcard Cloudflare Zero Trust Access policy, so it was replaced with a per-minute systemd timer - and the stack dropped root along the way."
showToc: true
---

A while back I wrote up a self-hosted Obsidian-plus-Quartz stack: a watcher commits and pushes every save, and a GitHub webhook pulls changes made from any other machine. That webhook never worked in production, and I never came back to say so. The replacement - a per-minute polling timer - has been running quietly since, alongside two other changes that were worth writing down together: the stack dropped root, and the pull script had to learn to survive racing against the save watcher.

## Why the webhook had to go

The deploy target sits behind Cloudflare Zero Trust Access, and the access policy covering this host is a wildcard over the whole domain, not a per-path rule. The webhook approach needs exactly one path - `/hooks/note-pull` - to bypass Access so GitHub's POST can reach it unauthenticated (the HMAC signature on the payload is the real auth). With a wildcard policy already in place, carving out a clean exception for one path without reopening the rest of the domain wasn't something I could do cleanly on this setup. Rather than fight the access policy, I dropped the inbound path entirely.

That's the general lesson: a webhook requires *some* door open inward. If your perimeter tooling makes opening exactly one door and no others inconvenient, a polling timer sidesteps the whole problem - nothing needs to reach the server from outside.

## The replacement: a timer, not a daemon

```ini
# /etc/systemd/system/note-pull.timer
[Unit]
Description=Run note-pull at the top of every minute

[Timer]
OnCalendar=*-*-* *:*:00
AccuracySec=1s
Persistent=false

[Install]
WantedBy=timers.target
```

`AccuracySec=1s` is the line that actually matters here. The systemd default is `AccuracySec=1min`, which lets the scheduler randomize or batch the trigger anywhere inside that minute - fine for most timers, useless if you want "every minute, predictably." Pin it to `1s` or the "per-minute" timer quietly becomes a "within-the-minute, sometime" timer. `Persistent=false` is deliberate too: there's no point catching up on a missed per-minute run, the next one is seconds away anyway.

## The part that took actual thought: not stepping on the save watcher

The autopush watcher already running on this box commits and pushes on every file save, debounced over a 2-second quiet window. Add a second process that also commits and pushes on a timer, and you get two processes mutating the same working tree on overlapping schedules. The naive version of `note-pull.sh` was just `git pull origin HEAD`, and it broke the first time a save landed in the few-second window between a local commit and its push - `git pull` refused with "local changes would be overwritten."

The fix is a stash-pull-pop sandwich:

```bash
cd /home/<user>/note || exit 1
before="$(git rev-parse HEAD 2>/dev/null)"

stashed=0
if ! git diff --quiet || ! git diff --cached --quiet \
        || [[ -n "$(git ls-files --others --exclude-standard)" ]]; then
    if git stash push --include-untracked -m "note-pull autostash $(date -u +%FT%TZ)"; then
        stashed=1
    fi
fi

if ! git pull origin HEAD; then
    git rebase --abort 2>/dev/null
    [[ "$stashed" == 1 ]] && git stash pop 2>/dev/null
    exit 1
fi
after="$(git rev-parse HEAD 2>/dev/null)"

if [[ "$stashed" == 1 ]]; then
    if ! git stash pop; then
        # Conflicted pop: scrub back to clean HEAD rather than leave markers -
        # the autopush watcher would happily `git add -A` and commit them.
        # The stash itself survives a failed pop, for manual fixup later.
        git checkout -- . 2>/dev/null
        git reset --hard 2>/dev/null
        exit 1
    fi
fi

if [[ -n "$before" && "$before" != "$after" ]]; then
    sudo systemctl restart quartz.service
fi

git add -A
if ! git diff --cached --quiet; then
    git commit -m "save: $(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")"
fi

if [[ "$(git rev-list --count @{u}..HEAD 2>/dev/null)" -gt 0 ]]; then
    git push origin HEAD
fi
```

Any uncommitted work gets stashed before the pull, so the pull itself never sees local changes to object to. The pull runs with `pull.rebase=true` for a linear history; a rebase conflict or a stash-pop conflict both get the same treatment - abort/reset back to a clean tree rather than leave conflict markers sitting in the working copy, because the autopush watcher doesn't know what a conflict marker is and will cheerfully stage and commit one. The stash isn't dropped on a failed pop, so nothing is lost - it just waits in `git stash list` for a human.

The two processes do end up racing on the final commit-and-push step: popping the stash writes files, which the autopush watcher's `inotifywait` also sees, and both are capable of running `git commit`. They don't double-commit, because both paths gate on `git diff --cached --quiet` - whichever one runs second finds nothing staged and exits quietly. But it means a given `save:` commit's authorship, in the sense of "which systemd unit actually ran `git commit`", is not deterministic. That's fine for this use case (the commit message is just a timestamp, not something anyone greps for by process), but it's exactly the kind of nondeterminism that would bite if either script ever needed to log something unit-specific inside the commit.

One more deliberate asymmetry: Quartz only restarts when `git pull` actually advances `HEAD`. Quartz's own `--serve` mode watches the content directory and rebuilds on change, but git's bulk writes during a pull don't reliably emit the inotify events a live watcher depends on - so the fix isn't "watch harder," it's "restart the process so it re-globs from scratch," gated so a no-op pull causes zero downtime.

## Dropping root

The second change bundled into this migration: the whole stack - vault, Quartz, both git scripts - moved off root onto an unprivileged user. Two consequences fell out of that, and one of them was a surprise.

`root`'s home directory is mode `700`, so a service running as the unprivileged user can't reach root's Node.js install for anything - not even read access to the binary. The fix was mechanical: copy the Node install out to a shared, world-readable path and point every service's `PATH` at the copy instead of root's nvm directory.

The surprise showed up later, auditing the sudo grants `quartz-restart.service` and `note-pull.sh` rely on (both need to restart the Quartz unit, which requires root). The intended grant was narrow and purpose-built:

```
# /etc/sudoers.d/<user>-quartz
<user> ALL=(root) NOPASSWD: /usr/bin/systemctl restart quartz.service
```

Checking what was actually active turned up a second file, unrelated to this stack, already granting the same user unrestricted root:

```
# /etc/sudoers.d/<user>  (pre-existing, not part of the note stack)
<user> ALL=(ALL) NOPASSWD: ALL
```

So the account in question has always had full passwordless root through the first file - the narrow grant is redundant today. It's still worth keeping on disk: if the broad grant ever gets tightened or removed for unrelated reasons, the note stack's own scripts already call `sudo` expecting only the one narrow command, and they'd keep working without edits. But "redundant" was the wrong assumption to walk in with. If you're migrating a service off root and documenting the sudo rule you added for it, check `/etc/sudoers.d/` for what else is already there - your careful one-line grant may be moot next to something broader nobody mentioned.

## The git credential underneath both scripts

Both scripts authenticate the same way: a classic GitHub personal access token (`repo` + `workflow` scope) sitting in the global credential store (`git config --global credential.helper store`), not embedded in the remote URL. Classic over fine-grained was a deliberate choice, not a default - a fine-grained PAT is scoped to a single account and returns a 403 against repositories that live in an organization, and part of what this credential needs to reach is a client org's repos, not just personal ones. A classic token authenticates as the user across both the personal account and every org it belongs to, so one credential line in `~/.git-credentials` covers everything this host's git operations touch. The tradeoff is a broader-scoped token sitting in plaintext on disk - acceptable here only because the repos are private, the host sits behind Access, and the file is `chmod 600`.

## What's actually running now

- **Inbound path:** none. The webhook listener and its Access bypass rule are both gone.
- **Sync:** a per-minute `note-pull.timer` (`oneshot`, `AccuracySec=1s`) plus the existing save-triggered autopush watcher, with the stash-based script above mediating between them.
- **User model:** everything runs as an unprivileged account with a Node.js copy outside root's home and a sudo grant scoped to one `systemctl restart` command - which, on this host, happens to be redundant next to a broader pre-existing grant.
- **Auth:** one classic PAT, shared across every repo this host's git touches, rotated periodically by overwriting the credential-store entry - no per-repo URL edits needed when it rotates.

## Takeaways

1. A webhook needs an open door. If your access-control layer makes opening exactly one door and no others inconvenient, polling removes the need for a door at all - at the cost of up-to-a-minute latency, which for a personal notes vault is free.
2. Two independent processes that can both touch the same git working tree need an explicit arbitration rule, not an assumption that they won't collide. Here it's "both gate on `git diff --cached --quiet`, whichever runs second is a no-op" - simple, but it had to be designed, not discovered by accident.
3. On a conflict, prefer losing the attempt over leaving the working tree in a half-merged state, if anything else automatically acts on that tree. A stash that survives a failed pop is recoverable; a conflict marker auto-committed by an unrelated watcher is a bug report waiting to happen.
4. When you add a sudo rule for a service, go look at what else is already in `/etc/sudoers.d/` for that account. The rule you're adding may be the least interesting thing granting that user root.
