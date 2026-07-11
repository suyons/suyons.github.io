---
title: "WSL2 Troubleshooting - Byobu Sessions That Died the Instant You Detached"
date: 2026-07-11
draft: false
tags: ["wsl2", "byobu", "tmux", "powershell", "windows-11", "openssh", "self-hosting"]
categories: ["Infrastructure"]
description: "Getting byobu to run PowerShell 7 panes on Windows 11 was one config line. Making detach-and-reattach actually survive took chasing two separate ways WSL2 tears down process state out from under a long-lived tmux session."
showToc: true
---

## The setup

The goal looked small on paper: get [byobu](https://www.byobu.org/) — the terminal session manager built on top of `tmux` — running on a Windows 11 machine, with every pane inside it running PowerShell 7, not bash. Byobu is genuinely great for anyone who works over SSH: detach from a session, walk away, reattach later from anywhere, and every pane and window is exactly as you left it. The catch is that byobu is fundamentally a Unix tool. It's a shell-script wrapper around `tmux`, and `tmux` needs a POSIX environment to run at all. There's no native Windows build.

## Why WSL2, and not a lighter Unix-emulation layer

Before touching byobu, the simpler prerequisite was enabling the OpenSSH server. That turned out to be a non-event in the best way: Windows 11 ships OpenSSH Server as a built-in optional feature, already installed, already running as the `sshd` service, and already set to start on boot. The only thing to confirm was that Windows Firewall had an inbound rule open on port 22, scoped to the "Private" network profile — the safer default, since it won't accidentally expose SSH on a public or untrusted network.

With SSH access confirmed, the real question was how to run byobu at all. Three options were on the table:

1. **Windows Terminal panes** — fully native, zero installs, but no detach/reattach. That's the one feature byobu exists for, so this didn't solve the problem.
2. **MSYS2 + tmux** — a lighter-weight Unix-emulation layer than a full VM, installable via a package manager, capable of hosting `tmux` directly on Windows without WSL.
3. **WSL2**, with byobu installed inside a Linux distribution, configured so its panes launch `pwsh.exe` (PowerShell 7) instead of a Linux shell.

The interesting part of the decision wasn't technical, it was a matter of trust in the platform. MSYS2 is still, under the hood, a Unix-emulation layer bolted onto Windows — the same lineage as Cygwin. WSL2 is Microsoft's own officially maintained subsystem: a real Linux kernel in a lightweight VM, with first-class interop into the Windows side. Given a choice between an emulation layer and the thing Microsoft ships and supports, the latter won, even though it's the heavier option in terms of what's actually running underneath.

## The core trick: byobu's default-command

The machine already had WSL2 with an Ubuntu distribution installed, which made the install itself trivial:

```bash
apt-get install -y byobu
```

The part that makes this useful — rather than "byobu, but now you're stuck in bash" — is a single `tmux` configuration line. Byobu reads its own `tmux` config, and that config can override what command each new pane runs by default. Point it at the Windows-side PowerShell executable, reachable through WSL2's interop layer:

```
# ~/.byobu/.tmux.conf (inside the WSL2 Ubuntu distro)
set -g default-command "pwsh.exe -NoLogo"
```

That one line is the whole trick. WSL2's interop feature lets Windows executables be invoked directly from Linux as if they were native binaries, and `tmux` doesn't care what the "default command" actually is — it just runs it in every new pane. Point it at `pwsh.exe` and every split, every new window, every reattached session comes back as a full PowerShell 7 shell, with byobu's status bar and session persistence wrapped around it.

The last piece was making this invisible from the Windows side, with a one-line function in the PowerShell profile:

```powershell
function byobu { wsl -d Ubuntu -- byobu @args }
```

From that point on, typing `byobu` at a PowerShell prompt drops straight into a byobu session inside WSL2 — but every pane is PowerShell, not bash.

## First failure: the VM disappears 60 seconds after you disconnect

The initial smoke test passed: a detached session with a PowerShell pane, verified by capturing its output. Real-world use broke the core promise immediately. Detach with `F6`, work in other apps for a bit, run `byobu` again — the previous session was simply gone, replaced by a fresh empty one.

The first culprit is an easy-to-miss WSL2 platform default: `vmIdleTimeout`. WSL2 runs Linux inside a lightweight VM, and by default Windows tears that entire VM down about 60 seconds after the last `wsl.exe` client disconnects — regardless of whether Linux processes, like a detached `tmux` server holding your sessions, are still running inside. From Unix's point of view, a detached tmux server is a perfectly healthy long-lived daemon. From Windows' point of view, it's an idle VM eligible for shutdown.

The fix is one file, one line, on the Windows side:

```ini
# C:\Users\<you>\.wslconfig
[wsl2]
vmIdleTimeout=-1
```

After `wsl --shutdown` to apply it, a detached session survived well past the old 60-second window. Problem solved — or so it seemed.

## Second failure: interop panes die with the client that spawned them

The failure came back in a faster, sneakier form. Run a command in the pane, press `F6`, immediately run `byobu` again — and the session was gone, no timeout involved at all. The tmux *server* had survived; the *session* had died. That pointed at the pane process itself.

Reproducing the flow through a pseudo-terminal confirmed the mechanism: when byobu is launched interactively, the tmux server is started by the attached `wsl.exe` client, and every `pwsh.exe` pane it spawns is a Windows process hosted through that same client's interop channel. Ending the client — exactly what detaching does — makes Windows tear down its interop-hosted children. The PowerShell pane dies, the pane's death closes the window, and a single-window session dies with it. The "detached session" never outlives the keystroke that detached it.

The workaround: never let the interactive client be the one that starts the server. Create the session headless first — a short-lived background `wsl.exe` runs `new-session -d` and exits — then attach as a separate step. Panes spawned by a headless-created server are hosted by `wslhost.exe`, Windows' background interop host, which doesn't care when attach clients come and go.

```powershell
# Before — the pane's pwsh.exe dies the moment you detach
function byobu { wsl -d Ubuntu -- byobu @args }

# After — create the session headless, then attach; one session, F6-safe
function byobu {
    if ($args) { wsl -d Ubuntu -- byobu @args; return }
    wsl -d Ubuntu -- byobu has-session -t main 2>$null
    if ($LASTEXITCODE -ne 0) { wsl -d Ubuntu -- byobu new-session -d -s main }
    wsl -d Ubuntu -- byobu attach-session -t main
}
```

Under pseudo-terminal emulation this held up across multiple attach/detach cycles, scrollback intact, including a window created while attached.

## A false negative: stale state outlived the actual fixes

The ending was anticlimactic. After a full Windows reboot — with the headless-creation wrapper and `vmIdleTimeout=-1` already in place — the real-world symptom was simply gone: detach with `F6`, work elsewhere, run `byobu` again, and the session comes back intact. The fixes had been correct all along. Some piece of stale state from before them, most plausibly the WSL VM or its interop host still running under the old configuration, was what kept the interactive failure alive after the scripted reproductions had already gone green. `wsl --shutdown` was supposed to be the reset button; only the reboot actually drew a clean line.

## The last mile: making SSH land in PowerShell 7

With the local experience solid, the remote one was still broken in a mundane way. SSHing in dropped straight into `cmd.exe` — Windows OpenSSH's default shell — where `byobu` doesn't exist, because the wrapper lives in a PowerShell profile:

```
user@host C:\Users\user>byobu
'byobu' is not recognized as an internal or external command,
operable program or batch file.
```

The fix is a single registry value telling the OpenSSH server which shell to hand new sessions:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
    -Value "C:\Program Files\PowerShell\7\pwsh.exe" -PropertyType String -Force
```

No service restart needed — the value is read per connection. The next SSH login lands at a PowerShell 7 prompt, the profile loads, and `byobu` resolves to the wrapper function. Detach and reattach were then verified over SSH too, which was the riskiest remaining case: an SSH session stacks one more client lifetime on top of the interop chain (`sshd` → `pwsh` → `wsl.exe`), making it the most likely place for the pane-death problem to resurface. It didn't.

## Takeaways

- WSL2's process-lifetime model is genuinely different from Unix daemons: both the VM itself and interop-hosted Windows processes are tied to Windows-side client lifetimes in ways that violate assumptions tools like `tmux` are built on. `vmIdleTimeout` and headless session creation each fixed one layer of that.
- When a correct fix still appears to fail, suspect stale state before doubting the fix. The last "bug" here was residue from the pre-fix configuration that only a reboot cleared.
- The final integration step is often embarrassingly small. After two rounds of process-lifetime debugging, the SSH experience came down to one registry value.
