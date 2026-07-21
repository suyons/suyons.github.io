---
title: "Linux Security - A Keyring Fix That Broke Unattended Git in Ten Minutes"
date: 2026-07-21
draft: false
tags: ["git", "credentials", "gnome-keyring", "systemd", "security", "linux"]
categories: ["Security"]
description: "Moving a headless server's plaintext git credentials into a keyring-backed helper worked perfectly from an interactive shell and broke every unattended systemd job that authenticates to git within ten minutes — because a keyring needs a login session, and system-scope units don't have one."
showToc: true
---

A routine security pass on a headless Ubuntu server flagged the long-standing plaintext `~/.git-credentials` file as a finding worth fixing. The fix looked straightforward: swap `credential.helper=store` for a keyring-backed helper, `git-credential-libsecret` on top of `gnome-keyring`. The build went smoothly, the config flip took one command, and ten minutes later a systemd timer that pulls a private notes repository every minute was failing every single cycle.

## Why the migration looked like a clean win

Generic security guidance here is unambiguous: don't store long-lived tokens in plaintext on disk if a keyring is available. `libsecret` and `gnome-keyring` were already present as library dependencies on the box, git's own source tree ships a `contrib/credential/libsecret` helper, and the plan was simple — compile it, point git at it, migrate the token, shred the old file. Nothing about that plan considered the machine's other jobs.

Building the helper needed a small dependency chain the server didn't have yet — `libsecret-1-dev`, `libglib2.0-dev`, `pkg-config`, `make` — plus the `gnome-keyring` package itself for the daemon binary. `libsecret-1-0` alone is just the client library; it doesn't get you a running daemon. Once everything was installed, the right integration point turned out to be the socket-activated systemd unit the package ships and enables on its own:

```
$ systemctl --user start gnome-keyring-daemon.socket
$ systemctl --user is-active gnome-keyring-daemon.socket
active
```

That's the correct mechanism — it starts the daemon on demand for every login session, no manual `eval $(gnome-keyring-daemon --start ...)` hack in `.bashrc` needed. (An earlier attempt at that hack got removed once the socket unit proved sufficient; running both would have produced duplicate daemons.)

## The break

`git config --global credential.helper` is global. It doesn't care which script or session calls it. The server also runs two systemd units that have nothing to do with any interactive login: a `note-pull.timer` firing every minute, and a `note-autopush.service` watching the filesystem for edits, both authenticating over plain HTTPS. Both depend on whatever `credential.helper` resolves to.

Minutes after the switch, `journalctl` showed this on every cycle:

```
note-pull.service: could not connect to Secret Service: Cannot autolaunch D-Bus without X11 $DISPLAY
note-pull.service: fatal: could not read Username for 'https://github.com': No such device or address
note-pull.service: note-pull: pull failed; aborted rebase, leaving working tree unchanged
```

The root cause is structural, not a misconfiguration to patch around. `note-pull.service` and `note-autopush.service` are **system**-scope units — `User=young`, but no `PAMName=`, no `--user` linkage, no `XDG_RUNTIME_DIR` or `DBUS_SESSION_BUS_ADDRESS`. They run whether or not anyone is logged in, and they have no path to the per-session D-Bus bus that `gnome-keyring-daemon` publishes its Secret Service on. A keyring that needs a session to unlock is, from a system-scope service's point of view, indistinguishable from a keyring that doesn't exist.

## The fix, and why it's not a half-measure

```bash
git config --global credential.helper store
```

One line, reverting to plaintext. The next `note-pull.timer` cycle pulled cleanly.

I'd actually worked out why this would happen a month earlier, in the abstract, while evaluating SSH keys as an alternative to the same plaintext file: any credential mechanism that requires a human or a session to unlock it — a passphrase-protected SSH key, a keyring, a `pass`/GPG store — fails the same way on a box that has to authenticate unattended and survive a reboot with nobody logged in. The only way out of that constraint is hardware-backed sealing, a TPM, and this server doesn't have one. That reasoning existed before the failure did; today just supplied the concrete instance, and re-deriving it empirically cost about ten minutes of broken automation that rereading my own notes first would have skipped.

The keyring tooling stays installed but unused — harmless, and available if some future job ever runs inside an actual login session where a keyring can unlock. The underlying "plaintext is bad" finding isn't wrong in general. It's wrong as a global default on a machine with unattended systemd automation, which is a fact about this host, not about credential stores in the abstract.

## Takeaways

- **A global git config change affects every consumer, including ones you forgot existed.** `credential.helper` isn't scoped to a repo or a session; systemd timers running in the background are consumers too.
- **"It builds and the manual test works" isn't "it's deployed correctly."** The keyring helper worked perfectly from an interactive shell. The failure only showed up in the unattended path, and that needed its own explicit check — `journalctl -u note-pull.service` — to surface at all.
- **A generic security recommendation needs a threat model, not just an implementation.** "Move off plaintext" is correct advice in general and wrong advice for this specific host without first asking what else depends on the credential path you're about to change.
