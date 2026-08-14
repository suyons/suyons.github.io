---
title: "Windows Git Troubleshooting - Two Bugs, One Password Prompt"
date: 2026-08-14
draft: false
tags: ["git", "windows", "credentials", "powershell", "debugging"]
categories: ["DevOps"]
description: "A git clone kept asking for a username and password despite a configured credential helper and a populated credential file. Two independent failures were stacked on top of each other, and fixing the first one only made the second harder to see."
showToc: true
---

A `git clone` over HTTPS kept asking for a username and password, even though a global credential helper was configured and a populated `.git-credentials` file was sitting right there in the user profile. What looked like a single misconfiguration turned out to be two independent failures stacked on top of each other — either one alone was enough to produce the prompt. It's a good case study in why "fix the obvious thing and retry" is a bad debugging loop.

## The symptom

The clone failed like this:

```
fatal: Unable to persist credentials with the 'wincredman' credential store.
See https://aka.ms/gcm/credstores for more information.
Username for 'https://git.example.com':
```

Meanwhile the configuration looked correct:

```
$ git config --global --list
credential.helper=store

$ ls ~/.git-credentials
-a---   2026-08-13   59   .git-credentials
```

A helper is set, the store file exists, and it isn't empty. So why the prompt?

## Bug 1: `credential.helper` is a list, not a value

The first mistake is conceptual, and it catches a lot of people. `git config --global --list` only shows the *global* scope. Asking Git where every setting actually comes from tells a different story:

```
$ git config --list --show-origin | grep credential
file:C:/Program Files/Git/etc/gitconfig   credential.helper=manager
file:C:/Users/<user>/.gitconfig           credential.helper=store
```

There were **two** helpers. `credential.helper` is a multi-valued configuration key: setting it at the global scope *appends to* whatever the system scope already defines, rather than replacing it. Git for Windows ships with `credential.helper=manager` (Git Credential Manager, GCM) baked into the system-level `gitconfig`, so the global `store` entry landed second in the list.

Git queries helpers in order and takes the first one that returns credentials. GCM ran first, tried to reach the Windows Credential Manager (`wincredman`), and failed — a typical failure mode for an account without an interactive desktop session, such as a service account or a detached Remote Desktop session. With the first helper erroring out, Git fell back to its own terminal prompt and never reached `store` at all.

The fix is an empty-string entry, which is Git's documented way to reset an inherited multi-valued list:

```sh
git config --global --unset-all credential.helper
git config --global --add credential.helper ""      # resets the inherited list
git config --global --add credential.helper store
```

That resolves to:

```
credential.helper=manager   (system)   ← still present, but now discarded
credential.helper=          (global)   ← reset marker
credential.helper=store     (global)
```

This is preferable to editing the system `gitconfig` under `Program Files`: it needs no administrator rights, and it survives a Git upgrade rewriting that file.

## Bug 2: a single carriage return

With the helper stack cleaned up, the `wincredman` error was gone — but the prompt was not. This is exactly the moment it's tempting to declare victory and tell the user to try again. Instead, the right move is a non-destructive end-to-end probe. Git exposes its credential machinery directly:

```sh
printf 'protocol=https\nhost=git.example.com\n\n' | git credential fill
```

Still prompted, which narrowed the problem to `store` itself returning nothing. Invoking the helper on its own confirmed it:

```sh
printf 'protocol=https\nhost=git.example.com\n\n' \
  | git credential-store --file=$HOME/.git-credentials get
# (no output — no match)
```

So the file existed, was 59 bytes, and contained an entry for the right host — yet the helper found no match. The next step was to stop trusting the rendered text and look at the raw bytes:

```
Length: 59
First 8 bytes: 68 74 74 70 73 3A 2F 2F      # "https://"
Last  4 bytes: 6B 72 0D 0A                  # "kr" CR LF
```

There it is. The line terminated with **CRLF**, not LF. Git's `credential-store` parser strips the line feed but leaves the carriage return attached, so the host it parsed and compared against was:

```
git.example.com\r
```

which will never equal `git.example.com`. Everything else — the scheme, the username, the password, the host spelling — was byte-for-byte correct. A single invisible `0x0D` was the whole bug.

The origin is mundane: the file had been written with PowerShell (`Set-Content`, `>`, `Out-File`), all of which emit CRLF by default on Windows. Git's own helper writes LF. Anything that hand-edits `.git-credentials` on Windows is a candidate for this failure.

The repair is a byte-level newline normalization. Because this file holds a live secret, the sensible sequence is *transform to a temporary file, verify the temporary file, and only then replace the original* — never overwrite a credential store and hope:

```powershell
$p = "$HOME\.git-credentials"
$b = [IO.File]::ReadAllBytes($p)
$out = [Collections.Generic.List[byte]]::new()
for ($i = 0; $i -lt $b.Length; $i++) {
  # drop CR only when it is part of a CRLF pair
  if ($b[$i] -eq 0x0D -and $i+1 -lt $b.Length -and $b[$i+1] -eq 0x0A) { continue }
  $out.Add($b[$i])
}
[IO.File]::WriteAllBytes("$p.new", $out.ToArray())
```

Verify against the copy before committing to it:

```sh
printf 'protocol=https\nhost=git.example.com\n\n' \
  | git credential-store --file=$HOME/.git-credentials.new get
username=server
password=<found>
```

Only after that does the replacement happen. The file went from 59 to 58 bytes — one byte, two hours of confusion.

## Verifying without side effects

The natural final check is to re-run the original `git clone`, but that writes a directory somewhere and forces a guess about where the repository should live. A cleaner end-to-end proof is `git ls-remote`, which performs the full authenticated round trip against the server and creates nothing on disk. Pairing it with `GIT_TERMINAL_PROMPT=0` makes the test honest — if credentials don't resolve automatically, the command fails instead of silently waiting on a prompt:

```sh
$ GIT_TERMINAL_PROMPT=0 git ls-remote https://git.example.com/group/project.git
f827777c...   HEAD
f827777c...   refs/heads/master
```

Refs returned, no prompt. That's a real authentication against a real server.

## Takeaways

- **`git config --global --list` hides the problem.** Use `git config --list --show-origin` whenever behavior disagrees with configuration. Scope matters, and several Git keys are multi-valued.
- **Multi-valued keys accumulate across scopes.** For `credential.helper`, `http.extraHeader`, and similar keys, setting a value at a narrower scope doesn't override a broader one — add an empty-string entry first to reset the list.
- **A fixed error message is not a fixed bug.** The `wincredman` error disappeared after the first fix, which was strong evidence something changed — and no evidence at all that the goal was met. Re-run the actual success condition, not the error.
- **When text looks right but a parser disagrees, dump the bytes.** Invisible characters — CR, a byte-order mark, a non-breaking space, a Unicode homoglyph in a hostname — are exactly the class of bug that survives visual inspection indefinitely.
- **On Windows, assume CRLF contamination in any config file a shell touched.** PowerShell writes CRLF by default. For files consumed by Unix-lineage tooling, write them with the tool itself (here, `git credential approve`) or force LF explicitly.
- **Never overwrite a secret store in place.** Transform to a temporary file, prove the new copy works, then swap. A failed in-place edit of a credential file means the credential is simply gone.
