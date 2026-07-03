---
title: "Node.js Configuration - The dotenv Override Flag That Fixed One Environment and Broke Another"
date: 2026-07-03
draft: false
tags: ["nodejs", "dotenv", "pm2", "environment-variables", "windows", "troubleshooting"]
categories: ["Backend"]
description: "Migrating a Node/Express backend from cmd-launcher env vars to per-profile .env files surfaced two opposite dotenv failures: one fixed by override: true, the other caused by it."
showToc: true
---

## The setup

An Express/Node backend was configured the old-school Windows way: a set of `backend*.cmd` launcher scripts that `set` a dozen environment variables before starting the app. That's manageable with one environment. It stops being manageable with four (local, dev, test, prod) sharing machines and secrets scattered across launcher scripts that nobody wants to touch.

The plan was to consolidate everything into per-profile `.env` files loaded by `dotenv`. It looked like an hour of work. It took the rest of the session, because `dotenv`'s defaults are exactly wrong for a process supervisor, and fixing that broke the fallback path.

The app runs on Node 16, which predates the built-in `--env-file` flag, so `dotenv` was the pragmatic choice. Design: pm2 only picks *which* profile to load; every actual value lives in `.env.<profile>`.

```js
// index.js — loaded before anything reads process.env
const appEnv = process.env.APP_ENV; // set by pm2 per app; unset for local
if (appEnv) {
  const fs = require('fs');
  const path = require('path');
  const envFile = [
    path.resolve(__dirname, `.env.${appEnv}`), // running from repo root
    path.resolve(__dirname, '..', `.env.${appEnv}`), // running from a build/ dir
  ].find(fs.existsSync);
  if (envFile) require('dotenv').config({ path: envFile, quiet: true, override: true });
}
```

Two things in that snippet — `override: true`, and the fact that loading is *guarded* by `if (appEnv)` instead of defaulting to a profile — are the whole story.

One correctness detail came up while porting values over. Under Windows `cmd`, `set JWT_KEY='changeme'` stores the quotes **as part of the value** — unlike a POSIX shell, which strips them. So a runtime secret that was meant to be `changeme` had actually been the literal five-character string `'changeme'`, quotes included, for who knows how long, and application code had grown a `.replace(/'/g, '')` in one spot to compensate. `dotenv` strips surrounding quotes by default, so a naive copy-paste would have silently changed the signing key and invalidated every live session. Instead the values were unified deliberately to the clean, unquoted form across all profiles, and the one-time session reset was accepted as the cost of removing the quote bug for good.

## Gotcha 1: dotenv does not override variables that already exist

The test instance came up bound to the **production** port and died with `Pipe 4000 is already in use`. The `.env.test` file clearly specified `4001`. The cause: `dotenv.config()` **does not overwrite an environment variable that's already set** in `process.env`. The pm2 daemon on that box had a stale `HTTP_PORT=4000` sitting in its inherited environment — a relic of how older apps on the same machine used to be launched — and every child process inherits the daemon's environment. The profile file was read fine; its port value just lost to the one already there.

```js
// Before — profile value silently loses to whatever the daemon inherited
require('dotenv').config({ path: envFile });

// After — the profile file is authoritative
require('dotenv').config({ path: envFile, override: true });
```

This generalizes past this one bug: when a long-lived process supervisor sits between you and your app, treat its environment as polluted. If the `.env` file is supposed to be the source of truth, say so explicitly with `override`.

## Gotcha 2: the same bundle now has to serve a .env world and a cmd world

Under deadline pressure, an operator wanted to fall back to the trusted `backend.test.cmd` launcher for one environment while the `.env` system stabilized. That exposed the opposite problem: the launcher `set`s its variables directly, but the built bundle *also* tries to load a `.env` file — and with `override: true` now in place, the profile file would clobber the values the launcher had just set. It was worse than that: an earlier version of the loader defaulted `APP_ENV` to `local` when unset, so a launcher that set nothing would silently pull in `.env.local` and stomp its own port.

The fix was to make `.env` loading strictly opt-in — load a profile **only** when `APP_ENV` is explicitly present. pm2 sets it per app. Local development sets `APP_ENV=local` via the nodemon config. A bare cmd launcher sets nothing, so it keeps whatever it injected untouched.

```js
// Before — always loads something, fights the launcher
const appEnv = process.env.APP_ENV || 'local';
// ...load .env.<appEnv> with override...

// After — no APP_ENV means "leave process.env alone"
const appEnv = process.env.APP_ENV;
if (appEnv) {
  /* load .env.<appEnv> with override */
}
```

`override: true` and "only load when explicitly told to" are not in tension — they're two halves of the same rule. Once you decide the profile file wins, you also have to decide precisely when it gets to compete at all. Skip the second half and `override` becomes a landmine for anyone still using the old launcher.

## Gotcha 3: a batch file with LF line endings self-destructs

Getting the cmd fallback path production-ready surfaced a third, unrelated failure. The rewritten `backend.test.cmd` threw surreal errors: `'109*' is not recognized as an internal or external command`, `'ABASE' is not recognized`, `'tp:' is not recognized`. Those fragments are *tails* of real values — a secret ending in a date and an asterisk, the word `DATABASE`, the string `http:` — each chopped at what looked like a random point.

The cause: the file had been saved with Unix (LF) line endings. `cmd.exe` reads batch files in fixed-size byte chunks and relies on carriage returns to delimit lines. With bare LFs, a line boundary can land mid-chunk instead of mid-newline, splitting a `set` statement's value at an arbitrary byte offset — the leftover fragment then gets interpreted as its own command. On a Korean-locale server there was a second layer to it: the file also needed CP949 encoding, or the non-ASCII values in it would corrupt parsing on top of the line-ending problem.

The fix was to write the file as **CRLF + CP949**, and verify it at the byte level — count CR-LF pairs, assert zero bare LFs, rather than trust that an editor "looks fine." Takeaway: a `.cmd`/`.bat` file is not just text. Its line endings are load-bearing, and any tool that emits LF by default — which is most cross-platform editors and scripted file writers — produces a file that opens cleanly and fails in a way that looks nothing like a line-ending bug.

## Other cleanups in the same pass

- **`NODE_OPTIONS=--max-http-header-size=32768`** was needed everywhere — role arrays inside JWTs pushed request headers past Node's 16 KB default, causing HTTP 431. Since it has to be set *before* Node starts, it can't live in `.env`. It's defined once as a shared object and spread into every pm2 app config plus the nodemon config, instead of copy-pasted per profile.
- **Node version pinning** moved from a hard-coded absolute interpreter path in the pm2 config — which only existed on one machine — to a `.node-version` file read by `fnm`. Caveat worth documenting for the next person: pm2 uses whatever Node its daemon started under, so a version bump requires restarting the daemon, not just the app.
- **pm2 topology**: test and prod run on the same host, so a single app definition switched by `--env` couldn't give them distinct names or scripts. The config was split into separate named apps per environment instead.

## Takeaways

Two lessons are worth carrying to any Node service behind a process supervisor:

1. `dotenv` does not override pre-existing environment variables by default, and a supervisor's inherited environment is a real source of values that look impossible to explain. If the `.env` file is meant to win, say so with `override: true` — but only after you've made loading conditional on something explicit (`APP_ENV` being set, here), or you'll break every code path that doesn't use `.env` yet.
2. Never let a Windows batch file get saved with LF line endings. Both failures in this migration presented as baffling symptoms — a wrong port, a "command not found" for a string fragment nobody typed — and both root causes were one line of configuration.
