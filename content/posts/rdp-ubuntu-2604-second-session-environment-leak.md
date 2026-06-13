---
title: "Linux RDP Troubleshooting - Your RDP Session Inherited the Desktop You're Already Logged Into"
date: 2026-06-11
draft: false
tags: ["linux", "ubuntu", "xrdp", "wayland", "gnome", "remote-desktop"]
categories: ["Infrastructure"]
description: "Enabling xrdp on Ubuntu 26.04 to remote in from a Mac turned into a half-day tour of the Linux desktop stack. Almost every failure traced back to one thing: xrdp starts a second session for a user who's already logged in locally, and that session leaks the local one's Wayland display and D-Bus bus."
showToc: true
---

## "Just enable RDP" is a lie on Ubuntu 26.04

The wish was mundane. Leave my laptop on the desk, lid closed, plugged in, and remote into it from a MacBook over RDP. Enable xrdp, connect, done.

It took an evening. The connection failed, then it succeeded into a black screen, then it succeeded into a desktop where the notification daemon wouldn't start. Each fix felt unrelated to the last. They weren't. Nearly every failure came from the same root cause, and once I saw it the rest of the evening was cleanup.

The root cause: **xrdp logs you in as a user who is already logged in locally, and the new session inherits the local session's environment** — its Wayland display, its D-Bus session bus, its `systemd --user` instance. Most of what looks like an xrdp bug is really two sessions for one user stepping on each other.

Two facts about Ubuntu 26.04 set the whole thing up, and you can't fix them, only route around them:

1. **GNOME ships Wayland-only.** `/usr/share/xsessions/` is empty. There is no X11 GNOME session anymore.
2. **xrdp serves an X11 display.** xorgxrdp speaks X11 and nothing else.

So "RDP into my GNOME desktop" is dead on arrival. GNOME can't run on the display xrdp provides. That single incompatibility decides the whole architecture, and everything below follows from refusing to accept it quickly enough.

## Round 1: xrdp connects, then immediately dies

First connect from the Mac errored out instantly. The xrdp log was a pile of red herrings:

```
[ERROR] Xorg server closed connection
[ERROR] xrdp_mm_chansrv_connect: error in trans_connect chan
[ERROR] SSL_shutdown: Failure in SSL library (protocol error?)
```

The SSL and chansrv lines are downstream noise — the connection collapsed and these are the death rattle. The real cause was in `~/.xsession-errors`:

```
** ERROR **: A graphical session is already running!
```

GNOME allows one session per user. I was logged in locally, so xrdp's attempt to start a second GNOME session aborted (SIGABRT), which tore down Xorg, which produced all the SSL noise. And even if it hadn't aborted, fact #2 above means GNOME-over-xrdp could never have worked: it needs Wayland, xrdp gives it X11.

The fix is to stop trying to run GNOME over RDP and give the RDP session a desktop that actually speaks X11. XFCE does.

```sh
sudo apt install xfce4 xfce4-goodies dbus-x11
echo "startxfce4" > ~/.xsession
```

## Round 2: the black screen

XFCE started and rendered nothing. Black screen, no window manager (`xfwm4` never came up), and the log was full of:

```
xfdesktop: Your compositor must support the zwlr_layer_shell_v1 protocol
xfce4-panel: Wayland detected without layer-shell support
Name 'org.gtk.Settings' lost on session bus
```

Read those carefully and the root cause from the intro is staring back at you. My XFCE session was a *second session for the same user*, so it inherited the local GNOME session's `WAYLAND_DISPLAY` and **shared its D-Bus session bus**. XFCE's GTK apps saw a Wayland display in the environment and tried to draw on GNOME's compositor — the one attached to the physical screen, not the X11 display xrdp set up — while colliding with the local session over D-Bus service names like `org.gtk.Settings`.

The session wasn't broken. It was leaking. The fix is to slam the door on the inherited environment: force the X11 backend and give the RDP session its *own* private D-Bus bus.

```sh
# ~/.xsession
unset WAYLAND_DISPLAY
export GDK_BACKEND=x11 QT_QPA_PLATFORM=xcb XDG_SESSION_TYPE=x11
exec dbus-run-session -- startxfce4
```

`dbus-run-session` is the important part. It spawns a brand-new D-Bus session bus and runs XFCE under it, so the RDP desktop stops sharing service names with the desktop on the physical screen. `unset WAYLAND_DISPLAY` plus the backend pins make sure GTK and Qt apps draw on X11 instead of hunting for GNOME's compositor. That brought up a real XFCE desktop.

## Round 3: the leak has a long tail

A private D-Bus bus fixes the session, but not every daemon lives inside it. A notification popup kept failing:

```
Notification daemon failed — Wayland layer-shell not supported
```

`xfce4-notifyd` is D-Bus-activated through the **shared `systemd --user` instance**, and that instance still carried the local session's `DISPLAY=:11`. There's only one `systemd --user` per user — it doesn't get a private copy per session — so anything it activates inherits the wrong display. The fix is a small X11-forcing wrapper plus a user-level service override so the daemon spawns with the right environment instead of whatever `systemd --user` happened to hold.

Same root cause, one layer further out. If you only remember one thing about multi-session Linux: a second login as the same user shares more state than you think — the session bus, the user systemd instance, environment variables — and each shared thing is its own potential leak.

Reconnects were the other papercut. Drop the RDP connection and you'd land on a stale or broken session. `/etc/xrdp/sesman.ini` has a grace policy for exactly this:

```ini
KillDisconnected=true
DisconnectedTimeLimit=600
```

Brief network drops resume the existing desktop; anything longer than ten minutes gets killed so the next connect is a clean session.

## The GNOME detour, and why it's a dead end from a Mac

XFCE worked, but I wanted GNOME, so I tried `gnome-remote-desktop` (GRD), GNOME's native RDP server — it serves Wayland directly, sidestepping the X11 problem entirely. **Desktop Sharing** needs an active local session, and I was logged out, so I used **Remote Login**: the headless, GDM-handover mode built for exactly this.

First failure was `error 0x207`, with `SEC_E_NO_CREDENTIALS` in the log — I hadn't set the gateway credential. `sudo grdctl --system rdp set-credentials` fixed that and NLA passed. Then three seconds of black screen and `0x207` again, with a different cause:

```
ntlm_read_AuthenticateMessage: Message Integrity Check (MIC) verification failed!
AcceptSecurityContext status SEC_E_MESSAGE_ALTERED [0x8009030f]
```

This one is not my misconfiguration. GNOME Remote Login uses RDP *server redirection* to hand the connection from GDM to the user session, and Microsoft's macOS "Windows App" client mishandles the redirected NTLM credentials, breaking the message integrity check. Windows `mstsc` and FreeRDP complete this handshake. The macOS client does not.

I tried FreeRDP on the Mac to dodge it — note that `xfreerdp` is the X11 build and wants XQuartz; the native macOS one is `sdl-freerdp3` — but at that point I was several yaks deep for no real gain. I reverted to xrdp + XFCE, which the macOS client connects to with plain RDP, no redirect, no MIC handshake to fail.

```sh
sudo grdctl --system rdp disable && sudo systemctl disable --now gnome-remote-desktop
sudo systemctl enable --now xrdp xrdp-sesman
```

If you're on a Mac, don't spend the evening I did: GNOME Remote Login's redirect handshake is incompatible with Apple's RDP client today. Use xrdp + an X11 desktop, or connect from Windows `mstsc`.

## The lid, because it's a headless host now

A laptop that's a remote host should not suspend when you close the lid on AC power, but should still sleep on battery. The catch: the *active session* governs the lid, and a fallback is needed for the login screen with no session. Both layers:

```sh
# GNOME owns the lid while a session is active
gsettings set org.gnome.settings-daemon.plugins.power lid-close-ac-action nothing
gsettings set org.gnome.settings-daemon.plugins.power lid-close-battery-action suspend
```

```ini
# /etc/systemd/logind.conf.d/10-lid.conf — fallback when no session owns it
HandleLidSwitchExternalPower=ignore
HandleLidSwitch=suspend
```

Set only the gsettings half and the lid still suspends from the login screen, before you've logged in over RDP — which is precisely when you need it not to.

## What I'd tell myself at the start

The transferable lesson here isn't about xrdp specifically. It's that **a second login session for a user who's already logged in locally inherits the first session's environment**, and on a modern Wayland desktop that inheritance is poison for an X11 remote session. Trace the black screen, the wrong compositor, and the dead notification daemon back far enough and they're all the same bug: leaked `WAYLAND_DISPLAY`, a shared D-Bus session bus, a shared `systemd --user` instance carrying the local `DISPLAY`.

So the recipe that actually works on Ubuntu 26.04, from a Mac:

- Don't try to run GNOME over xrdp. GNOME is Wayland-only; xrdp is X11. Install XFCE for the RDP session.
- Wall off the inherited environment in `~/.xsession`: `unset WAYLAND_DISPLAY`, force `GDK_BACKEND=x11`, and wrap the desktop in `dbus-run-session` so it gets a private bus.
- Expect a long tail — daemons activated through `systemd --user` still inherit stale state, and need their own environment fix.
- Skip GNOME Remote Login if your client is a Mac; its redirect/NTLM-MIC handshake doesn't survive Apple's RDP client.
- Tune the lid at both the GNOME and logind layers, or it suspends out from under you at the login screen.

It works now, it survives reconnects, and it took understanding one root cause instead of patching five symptoms.
