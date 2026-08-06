---
title: "C#/WPF Refactoring - A Green Build That Proved Almost Nothing"
date: 2026-08-06
draft: false
tags: ["csharp", "wpf", "mvvm", "refactoring", "dotnet"]
categories: ["Desktop Development"]
description: "Applying a C#/WPF style guide to a legacy .NET Framework desktop client turned up four defects that compiled cleanly and passed every automated check — and three of the checkers themselves turned out to be lying."
showToc: true
---

We took a laboratory-automation desktop client — a WPF application on .NET Framework 4.6.1, C# 7.3, roughly 4,500 lines of code-behind — and applied a C#/WPF style guide to it end to end. The interesting part wasn't the renaming. It was discovering, repeatedly, that "it compiles and it launches" is a much weaker signal than it feels like.

## The style guide doesn't fully fit the platform

The guide prescribed nullable reference types, a community MVVM toolkit with source generators, a dependency-injection container, and analyzer packages. On C# 7.3 with a `packages.config` project, all four are either impossible or require new dependencies we didn't want to pull in. Rather than silently skip them, we recorded them as explicit deviations in the project documentation and added an `.editorconfig` encoding everything that *was* achievable — with every rule set to `suggestion` severity, so style can never break the build.

The lesson generalizes: when a guide meets a legacy stack, the valuable output is a written record of *which* rules don't apply and *why*, not a half-applied guide that leaves the next person guessing which parts were skipped on purpose.

## Renaming: use the compiler's own model, not text

The codebase used Hungarian notation throughout (`sName`, `bFlag`, `iCount`). Purging it meant renaming hundreds of symbols. A find-and-replace would have been catastrophic in a specific way:

```csharp
// Before — a field and a constructor parameter share a name
private readonly Action execute;
public DelegateCommand(Action execute) { this.execute = execute; }
```

Textual replacement of `execute` renames both, producing a parameter named `_execute` — technically compiling, silently violating the convention it was meant to enforce. Driving the rename through the compiler's semantic model instead touches only the symbol you actually asked for:

```csharp
// After — field renamed, parameter untouched
private readonly Action _execute;
public DelegateCommand(Action execute) { _execute = execute; }
```

Three categories of name were deliberately left alone, because they're wire contracts rather than style: data-transfer object properties that a JSON serializer binds *by name* (with no explicit attributes anywhere, so a rename compiles clean and then silently deserializes to null against the live server), XAML binding paths, and configuration-file field names. Also spared: Win32 interop parameter names like `hWnd` and `lpDcb`, which look like Hungarian notation but are the documented names of the native API.

## The verification that mattered — and the checkers that lied

Formatting passes were gated on a script that parses the file before and after with the compiler front-end and compares the *token streams*: if only whitespace-and-comment trivia differs, behavior can't have changed. This closes a hole that `git diff -w` leaves open, since it happily ignores whitespace changes *inside string literals*.

The uncomfortable part: the first three versions of this checker all reported success while checking nothing.

- The syntax library failed to load, so both token streams came back empty — and empty equals empty.
- A helper used to classify tokens turned out to be an extension method, not callable from the scripting host, so the literal list came back empty too.
- The comparison used a case-insensitive equality operator, so `"hello"` and `"hellO"` compared equal.

Each version produced a confident "0 mismatches." The fix was to give every checker a negative control: feed it a change it *must* detect, and refuse to trust a pass until the failure case actually fails.

> A checker that cannot fail is not a checker. Prove it can fail before you believe a pass.

## The failure mode a green build cannot see

Moving files into a conventional folder layout produced a bug that compiled cleanly, passed both content checkers, and still broke the application. A window's XAML referenced an image with a *relative* URI:

```xml
<!-- Before — resolves relative to the XAML file's location -->
<Image Source="Resources/Logo.png" />
```

Once that file moved into a `Views/` folder, the URI resolved to `views/resources/logo.png` and threw at startup. Making it root-relative fixed it:

```xml
<!-- After -->
<Image Source="/Resources/Logo.png" />
```

What made this genuinely dangerous was the application's global exception handler. It marked every dispatcher exception as handled and logged it through a logger that isn't initialized until the main window loads. So a startup failure produced **a live process with no window and no log entry** — not a crash, not an error, just silence. We only found it by launching the app and enumerating its top-level windows programmatically.

## An abandoned refactor is worse than none

Partway through converting click handlers to commands, we found that someone had started this same migration before and stopped. Both view-models still contained command properties that were declared, constructed in the constructor, bound nowhere, and called never. One had an empty method body.

The trap was that the dead scaffold was **not equivalent to the live handler**:

```csharp
// Dead scaffold, never bound
BalanceCommandQueue.Put(10, Command.Zero);

// The click handler that actually ran
BalanceCommandQueue.Put(0, Command.Clear);
```

Different command, different delay. Anyone "finishing" the migration by wiring up the existing commands would have silently changed what the hardware button sends to a scale. We kept the live behavior and deleted the scaffolds. A second instance of the same pattern turned up nearby: a handler assigned `IsEnabled` directly on buttons whose `IsEnabled` was already bound to a view-model property, so a local value was silently overriding the binding.

## The payoff: rows that carry their own commands

The single most satisfying deletion came from binding a list row's button to a command on the row itself. The original walked the visual tree upward from the click's source to find the containing row:

```csharp
// Before — roughly 20 lines of tree walking to recover the row
var d = (DependencyObject)e.OriginalSource;
while (!(d is ListViewItem)) { d = VisualTreeHelper.GetParent(d); }
var info = (DetailRow)((ListViewItem)d).Content;
```

```xml
<!-- After — the row is already the data context -->
<Button Command="{Binding SelectCommand}" />
```

Where genuinely view-specific work remained — scrolling a row to center, refreshing an items control, building dynamic columns — the view-model raises an event and the view performs it, so no WPF control type leaks into a view-model.

## One more binding trap

Late in the session, a converted message box threw on *every* invocation:

> A TwoWay or OneWayToSource binding cannot work on the read-only property 'Message'

A `TextBox.Text` binding defaults to **TwoWay**, and the new view-model exposed `Message` as get-only. Both build configurations were clean; only opening the dialog revealed it. `Mode=OneWay` fixed it. Worth remembering: read-only view-model properties are fine behind a `TextBlock`, and a landmine behind a `TextBox`.

## Outcome and takeaways

Every click handler in the application is now a command binding, and code-behind is roughly half its original size. Business logic — server calls, login, configuration loading, log archiving, order queries — moved into view-models and services. A hardware-facing service that used to hold references to text blocks and write to them from a serial-port thread now emits semantic events and references no UI type at all.

The final blocker turned out not to be code. The weighing screen refused to open with "cannot identify the serial-port device." The application compiles a device driver *from a database column* at runtime, and for this equipment record that column was empty, so the class-name check failed. We copied a proven driver profile from an equivalent record, verified up front that it satisfied the reflection contract the compiler expects (correct namespace, class-name prefix, all twenty methods and ten properties present), and installed it into that one column. The scale then came up on its serial port and started streaming weights.

Three things worth carrying away:

- **Build-green is a floor, not a ceiling.** Every serious defect this session — a broken resource path, a swallowed startup exception, a TwoWay binding on a read-only property, a dead scaffold that didn't match live behavior — compiled perfectly.
- **Automate renames through the compiler, not through text.** And explicitly enumerate the names that are contracts rather than style, before a rename tool gets anywhere near them.
- **Distrust your own tooling until it demonstrates a failure.** Three separate verification scripts reported success while measuring nothing. A checker earns trust only after you've watched it catch something.
