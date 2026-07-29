---
title: "Windows Server Troubleshooting - A tmux Session That Rendered as a Black Screen, Every Time"
date: 2026-07-29
draft: false
tags: ["windows-server", "openssh", "conpty", "ssh", "terminal-multiplexer"]
categories: ["Infrastructure"]
description: "Installing a native Windows tmux clone for detach/reattach over SSH worked perfectly on the server side and rendered nothing on the client side, and the root cause turned out to be a five-year-old OpenSSH version gap."
showToc: true
---

A Windows Server 2019 box needed a byobu-style setup: multiple tabs, detach and reattach, survives an SSH disconnect. [psmux](https://github.com/psmux/psmux) fit the bill — a native Windows terminal multiplexer written in Rust, ConPTY-based, installable with `choco install psmux`, and it ships `tmux`/`pmux` shims on `PATH` so existing muscle memory works. Session persistence checked out: the psmux server runs outside any Job object, so OpenSSH's job-based process reaping on disconnect doesn't touch it, and a detached session kept running with no client attached, verified rather than assumed.

Then someone attached to it over SSH and got a black screen. Not garbled, not partial — nothing. Forever.

## Ruling things out in the wrong order

The obvious suspects went first, and both were wrong.

**The Chocolatey shim.** `choco` installs a ShimGen stub that re-execs the real binary; maybe the stub was swallowing output. Running the actual 6.8 MB binary directly, bypassing the shim entirely, reproduced the identical blank screen.

**A stale session or a botched install.** A reboot clears a lot of sins. This one reproduced byte-for-byte after a full restart.

Both dead ends shared a pattern worth naming: psmux got blamed twice, and psmux was fine both times. A tool that's new to the stack is the easiest thing to suspect and not always the actual cause.

## The evidence that pointed at input, not output

Before chasing the pty layer, it was worth confirming which half of the connection was actually broken. psmux logs client input to `~/.psmux/ssh_input.log`:

```
Windows build 17763
Console mode: orig=0x0098 requested=0x0298 actual=0x0298 VTI=YES
RESIZE 157x41
heartbeat: loops=120 records=97 chars=45 vk=4 mouse=0
```

Keystrokes were genuinely reaching the server. Pressing F2 (bound to "new window") created a real second window — confirmed server-side with `psmux ls` reporting two windows, even though the screen showed nothing. `psmux capture-pane -p` returned correct pane content on request. Input worked. Panes and shells worked. Only the rendered output, over this specific transport, was empty.

## The decisive test

If psmux itself isn't the variable, the transport is. A one-line PowerShell script sends the raw ANSI escape sequence that switches a terminal into its alternate screen buffer — no psmux, no third-party binary, just the same signal every full-screen terminal app relies on:

```powershell
$e=[char]27
Write-Host "$e[?1049h"; Write-Host "ALT SCREEN VISIBLE"; Read-Host "enter"; Write-Host "$e[?1049l"
```

Run locally on the console, it works as expected. Run over this SSH connection, it renders nothing. That single test isolates the failure to the SSH pty layer itself, with zero third-party code in the path.

## Root cause: OpenSSH predates ConPTY passthrough by two years

Windows Server 2019 ships OpenSSH 7.7p1 in-box:

```
C:\Windows\System32\OpenSSH\sshd.exe   FileVersion 7.7.2.2   "OpenSSH_7.7p1 for Windows"
```

Win32-OpenSSH only gained ConPTY passthrough in 7.9.0.0p1. Before that, the server fakes a pty by running `ssh-shellhost.exe`, which scrapes the Windows console screen buffer and forwards what it sees. That approach has no concept of the **alternate screen buffer** — the separate canvas every full-screen TUI (tmux, vim, htop, and psmux alike) draws into instead of the normal scrollback. Line-oriented output scrapes fine, which is why plain command output over this same connection looked completely normal. Alt-screen output doesn't exist in the model at all, so it renders as nothing.

Worth being precise about what wasn't the problem: build 17763 has ConPTY. The OS-level pty API is present and correct. It's specifically the *SSH server binary* that's too old to hand a client that API.

One more artifact confirms this is a parsing gap, not just a missing feature: psmux's terminal-color-detection query (an OSC 10/11/12 escape sequence) gets a reply that the old pty layer's input parser doesn't know how to consume, so the reply itself leaks into the pane as literal garbage text:

```
1;rgb:c5c5/0f0f/1f1f\12;rgb:3b3b/7878/ffff\
```

That's a parser choking on a modern escape sequence it was never built to understand — consistent with a pty implementation from before these sequences existed.

## The fix, and why it's not a one-liner

The fix is a version bump: replace the in-box capability with Win32-OpenSSH v9.8.3.0. The complication is that upgrading sshd over the same SSH connection you're using to run the upgrade is a good way to lock yourself out of a box with no other access path. The script that does this earns its length by treating that risk as the primary design constraint, not an afterthought:

```powershell
if ($env:SSH_CONNECTION -and -not $Force) {
    Write-Host "REFUSING TO RUN: this looks like an SSH session." -ForegroundColor Red
    Write-Host "This script stops sshd, which would kill this session mid-upgrade."
    Read-Host "Press Enter to exit"
    return
}
```

Everything else follows the same "assume this could fail badly, plan the recovery before touching anything" discipline:

- **Download and verify before any mutation.** The MSI is fetched first, checked against an expected byte size, hashed with SHA-256, and its Authenticode signature validated — all before the old capability is touched. If any of that fails, nothing on the box has changed yet.
- **Confirm an RDP fallback exists.** The script checks `TermService` is running before proceeding, since RDP is the only way in if SSH breaks mid-upgrade.
- **Back up what a fresh install would silently regenerate.** Host keys and `sshd_config` are copied out first. A new install that generates fresh host keys means every existing client sees a host-key-changed warning on the next connection — avoidable, but only if you back up the old keys before they're gone.
- **Preserve `DefaultShell`.** The registry value pointing sshd at PowerShell 7 instead of the default `cmd.exe` doesn't survive a naive reinstall unless it's explicitly recorded and restored.
- **A dedicated rollback script**, not just "reinstall the old MSI," with two documented rollback targets: a Chocolatey-packaged 8.0.0.1 build already sitting on disk (works with no internet), or reinstalling the exact pre-upgrade 7.7p1 capability from a Windows Update source (the pre-upgrade byte-for-byte state, but it may not have a source and fails with `0x800f0954` if it doesn't).

The upgrade script was written and syntax-checked but, at the time of writing, not yet run — it has to happen from an RDP session, deliberately outside the SSH connection it would otherwise kill.

## What's still unverified

Registering the F-key bindings (`F2` for new window, `F6` for detach, and so on) isn't the same as confirming a physical keypress fires them over a real terminal client — that check needs an actual TTY and hasn't happened yet. And whether the OSC color-query leak is a symptom of the old pty (most likely, given the timing) or a genuine psmux parsing bug that persists on a fixed connection is an open question for after the upgrade — if it does persist, that's worth filing upstream, since no matching issue exists yet.

## Takeaways

- **A black screen with working input is a rendering-layer bug, not a hang.** Confirm the server thinks it's doing the right thing (`capture-pane`, log the input stream) before assuming the whole session is broken.
- **Test the transport with zero third-party code in the path.** A single raw escape sequence, sent with nothing but PowerShell, isolated the fault to the SSH pty layer and ruled out the actual application in one step.
- **"The OS supports it" and "this version of this binary supports it" are different claims.** ConPTY existed on this build of Windows for two years before this particular SSH server could use it.
- **Suspecting the new tool twice doesn't make it guilty twice.** When the same component gets blamed repeatedly and clears every time, the prior is wrong, not the tool.
