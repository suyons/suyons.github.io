---
title: "Node.js Deployment Troubleshooting - Four Ways a Path Can Be Wrong on Windows"
date: 2026-07-31
draft: false
tags: ["node-js", "webpack", "windows", "git", "path-resolution"]
categories: ["DevOps"]
description: "Deploying a bundled Node.js backend to Windows turned up four separate bugs that all trace back to the same root: a path or a line ending that was almost right, and silent about it."
showToc: true
---

What started as "why does Git keep asking for my password?" turned into a full deployment of a Node.js backend on Windows, and along the way four separate bugs turned out to share one root: a path or a line ending that was *almost* right. None of them produced a useful error message. Each one only gave up its cause when I tested the component directly instead of trusting its configuration.

## A credential file that Git silently refused to read

The setup looked correct. A `~/.git-credentials` file existed, `credential.helper=store` was set globally, and the entry matched the host being cloned. Git prompted for a username anyway.

The instinct is to suspect the config, but the config checked out: an empty `credential.helper=` reset had cleared any helper inherited from the system config before `store` was set, and `HOME` pointed exactly where the file lived. So instead of re-reading config, I asked the helper directly:

```
$ printf 'protocol=https\nhost=example.com\n\n' | git credential-store get
(no output)
```

Nothing. The helper had the file but found no match. The reason was only visible in the raw bytes:

```
Before:  https://user:token@example.com<CR><LF>
After:   https://user:token@example.com<LF>
```

`git-credential-store` splits its file on line feeds only. On Windows, an editor that writes CRLF leaves a carriage return glued to the end of the last field, so the helper had faithfully stored a credential for a host literally named `example.com\r` — which never equals the host Git asks about. Converting the file to LF endings fixed it immediately.

The deeper lesson is about the failure mode, not the fix. **A credential helper that finds no match is not an error.** It exits 0 with no output, and Git just falls through to prompting — no warning, no log line, nothing to grep for. When a subsystem is configured correctly but behaves as if it isn't, invoke that subsystem directly (`git credential fill`, `git credential-store get`) instead of re-reading configuration that already looks fine. And skip hand-editing this file entirely going forward — `git credential approve` writes it correctly.

## Ignoring a directory while keeping its skeleton

The repo needed its upload directory ignored, but the directory *structure* had to survive a fresh checkout, because the app assumes those subdirectories already exist. The obvious one-liner doesn't work:

```gitignore
# Before — .gitkeep can never be re-included
storage/
```

Once a directory is excluded, Git stops descending into it, so any later negation for files inside it never gets evaluated. Three lines are required:

```gitignore
# After
storage/**
!storage/**/
!storage/**/.gitkeep
```

`storage/**` ignores the contents recursively, `!storage/**/` re-admits the *directories* so Git keeps walking, and the last line un-ignores the placeholder at any depth — including the top level, since `**/` also matches zero directories. Worth verifying with `git check-ignore -v` against paths that don't exist yet, like a hypothetical `storage/new/deep/file.bin` and its sibling `.gitkeep` — that proves the rule covers directories created later at runtime, not just the ones present today.

## The bundle that couldn't find its JARs

With the repo cloned, the deployment crash-looped 82 times in a few seconds:

```
java.lang.NoClassDefFoundError: com/grapecity/documents/excel/Workbook
```

The library files were sitting right there in the source tree. The catch is that this backend is bundled with webpack, and **webpack replaces every module's `__dirname` with the output directory**. A classpath built like this:

```js
java.classpath.push(path.resolve(__dirname, "./library.jar"));
```

resolves to `<source-dir>/library.jar` when run from source, but `<output-dir>/library.jar` inside the bundle — a location nothing had ever copied the files into. Adding a copy step to the build fixed it permanently:

```js
// After — webpack config
new CopyPlugin({
  patterns: [{ from: 'GcExcel/*.jar', to: '[name][ext]', context: __dirname }],
})
```

That snippet hides a second Windows trap. My first attempt passed an absolute path built with `path.resolve(...)`, and the build failed with `unable to locate '...\GcExcel\*.jar' glob`. The plugin only recognizes POSIX separators as glob syntax — a Windows path with backslashes is treated as a literal filename instead. A relative pattern with an explicit `context` sidesteps the issue entirely.

There's a debugging lesson buried in here too. While the process was crash-looping, the logs also filled with an unrelated-looking database error: `ORA-12154: TNS could not resolve the connect identifier`. It was tempting to treat that as a second bug. A direct probe (`select 1 from dual`, same credentials) succeeded on the first try, proving the database was fine and the errors were just collateral from the restart storm. **A crash loop manufactures secondary symptoms — verify each one independently before chasing it.**

## One module for storage paths

The `__dirname` discovery had a wider blast radius than the JARs. Roughly 74 call sites across nine files built file paths from `__dirname`, while the upload middleware used the current working directory instead. In the bundle those resolve to *different trees* — the uploader wrote to one location and every reader looked in another.

The fix was to funnel every path through a single module:

```js
// Before — every caller re-derived the root, and got a different answer in the bundle
const filePath = path.join(__dirname, '/storage/document/' + fileId);

// After — one module owns the layout
const filePath = storagePath('/storage/document/', fileId);
```

```js
// The module itself
module.exports = function storagePath(directory_name, file_id) {
  const target = file_id ? `${directory_name}${file_id}` : `${directory_name}`;
  if (process.env.STORAGE_TYPE == "NAS") return process.env.STORAGE_NAS_PATH + target;
  // Default is LOCAL: build from the working directory (the process root).
  return path.join(process.cwd(), target);
};
```

Two design details mattered more than they look. First, the directory argument carries its own leading and trailing slashes, so callers that append a filename to a *directory* result keep working unchanged — `path.join` preserves a trailing separator, and several call sites depend on that. Second, an unset `STORAGE_TYPE` defaults to local instead of throwing: a missing environment variable shouldn't take down an application whose behavior is unchanged from before the feature existed.

The refactor itself was scripted, not hand-edited, and that's where I got burned. The conversion script scanned for the closing parenthesis of each call, and its scanner mishandled `${...}` interpolation inside template literals — it bailed out mid-file on three files, *after* already rewriting their import statements. The result compiled but would have thrown `ReferenceError` at runtime on the first request, because the import no longer provided the symbol the remaining call sites still used.

What caught it wasn't review. It was two mechanical checks: grep for the *old* identifier expecting zero hits, and `node --check` on every touched file. The grep surfaced 39 stragglers; `node --check` confirmed nothing else was left syntactically broken. **After any scripted refactor, assert the absence of the thing you replaced.** Counting successful replacements tells you what worked. Only the absence check tells you what didn't.

## Takeaways

- **Silence is a failure mode.** The credential helper, the ignored directory, and the missing library each failed without saying anything useful. Test the component directly — don't infer behavior from configuration that looks correct.
- **Bundlers rewrite `__dirname`, and that turns "where am I?" into a trap.** Any file path built relative to a module is a latent bug once the code is bundled. Anchor runtime paths to the working directory, in exactly one module.
- **Windows tooling disagrees about separators and line endings, quietly.** A carriage return broke credential lookup; a backslash stopped a glob from matching. Both were invisible in normal output and obvious the moment I looked at raw bytes.
- **A scripted refactor needs a mechanical absence check, not just a success count.** Grepping for the identifier you removed catches the files your script silently gave up on.
