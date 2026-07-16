---
title: "Windows Power Management Troubleshooting - A Laptop That Kept Sleeping No Matter How Many Timeouts I Killed"
date: 2026-07-16
draft: false
tags: ["windows", "windows-11", "power-management", "modern-standby", "powercfg", "lenovo", "troubleshooting"]
categories: ["Infrastructure"]
description: "Getting a laptop to stay awake 24/7 on AC power without changing its battery behavior turned into five layers of diagnosis: a hidden powercfg setting, a vendor daemon silently overriding the OS, an undocumented unattended-sleep timeout, and finally Modern Standby itself refusing to honor any of it."
showToc: true
---

## The goal

Keep a laptop awake 24/7 while it's plugged into wall power, lid closed or not, but let it behave normally — sleep on lid close, sleep after 15 minutes idle — the moment it's running on battery. That sounds like two checkboxes in the Windows power plan. It took five separate fixes, because on a modern Windows laptop "prevent sleep" isn't one setting, it's a stack, and each layer can silently undo the one above it.

## Layer one: the standard fix, split by power source

The textbook answer is Control Panel's power options: set the lid-close action to "Do nothing," set the sleep timeout to "Never." I did the equivalent through `powercfg` instead of the GUI, because it's scriptable and because I wanted AC and battery configured independently rather than trusting a GUI checkbox to apply per-source correctly:

```powershell
# AC only: never sleep on lid close, never sleep from idle
PS> powercfg /setacvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
PS> powercfg /change standby-timeout-ac 0
PS> powercfg /setactive SCHEME_CURRENT

# Battery: leave the defaults alone (sleep on lid close, sleep after 15 min idle)
```

`LIDACTION` value `0` is "do nothing," `1` is "sleep," `2` is "hibernate." That's the entire textbook fix, correctly split by power source. The laptop kept sleeping on AC anyway.

## Layer two: a setting that refuses to change and doesn't say why

Querying the lid-close action for the active scheme came back with nothing:

```powershell
PS> powercfg /query SCHEME_CURRENT SUB_BUTTONS LIDACTION
(no output)
```

No error, no value, no listing — as if the setting didn't exist. That silence is itself the diagnosis: the setting is present but flagged hidden, something OEM images do to push users toward the vendor's own power-management app instead of the OS controls. There's a dedicated flag for unhiding it:

```powershell
PS> powercfg /attributes SUB_BUTTONS LIDACTION -ATTRIB_HIDE
PS> powercfg /query SCHEME_CURRENT SUB_BUTTONS LIDACTION
Power Setting GUID: 5ca83367-6e45-459f-a27b-476b1d01c936  (Lid close action)
  GUID Alias: LIDACTION
  Current AC Power Setting Index: 0x00000000
  Current DC Power Setting Index: 0x00000001
```

Once unhidden, the same `/setacvalueindex` command from layer one actually took effect. The lesson: if a power setting doesn't error but also doesn't visibly change anything, check whether Windows is even exposing it before assuming you mistyped the GUID or scoped it wrong.

## Layer three: a vendor daemon sitting underneath the OS

Setting still didn't hold. The machine was running Lenovo Vantage in the background, with its own service. Vendor system-management suites like this frequently ship their own power and thermal policies that sit below the Windows power plan and can silently override it — the OS setting reads back as correct, but the hardware still does what the vendor daemon wants.

I stopped and disabled the service first, non-destructively, to confirm it was actually the blocker. Once confirmed, the whole application and its service came out at the user's request. One thing I deliberately left alone: a separate companion driver that only handles the Fn-key hotkeys (brightness, volume, airplane mode). It has nothing to do with power management, and pulling it would have broken keyboard shortcuts for zero benefit.

## Layer four: the timeout nobody shows you in Settings

Vendor software gone, lid setting holding — and the machine still slept from pure inactivity. Windows' own diagnostics gave the real answer. `powercfg /lastwake` reports what woke the machine last, and the System event log (source: Power-Troubleshooter) records a plain-text reason for every sleep transition. The reason for the last sleep event was `Application API`, not a lid or idle event. Something in the background was explicitly asking the OS to sleep.

That points at a setting most people never encounter: the unattended sleep timeout. It's distinct from the normal idle timeout — it controls how long the system waits after an unattended wake (background maintenance, Windows Update, etc.) before going back to sleep. It was set to two minutes on AC. Practically: Windows Update's background worker (visibly running in Task Manager at the time) woke the machine, found no user present, and the unattended timeout put it back to sleep two minutes later — completely independent of the "main" sleep timeout I'd already set to never.

It's in the same `SUB_SLEEP` subgroup as the standard idle timeout, also hidden by default:

```powershell
PS> powercfg /q SCHEME_CURRENT SUB_SLEEP
# find "Unattended Sleep Timeout" in the output and note its GUID —
# it's assigned per Windows build, don't hardcode one from a blog post

PS> powercfg /attributes SUB_SLEEP <unattended-sleep-timeout-guid> -ATTRIB_HIDE
PS> powercfg /setacvalueindex SCHEME_CURRENT SUB_SLEEP <unattended-sleep-timeout-guid> 0
PS> powercfg /setactive SCHEME_CURRENT
```

Set to never on AC, left untouched on battery, same split-by-source discipline as layer one.

## Layer five: Modern Standby doesn't play by these rules at all

Every visible and hidden timeout disabled, vendor software gone — and the machine still slipped into sleep. `powercfg /a` lists which low-power states the hardware actually supports, and it explained everything:

```
The following sleep states are available on this system:
    Standby (S0 Low Power Idle) Network Connected

The following sleep states are not available on this system:
    Standby (S3)
        The system firmware does not support this standby state.
    Hibernate
        Hibernation has not been enabled.
```

This laptop uses Modern Standby (S0 low-power idle), not the classic S3 suspend-to-RAM state. Modern Standby is designed to let a laptop keep doing background work while "asleep" — mail sync, notifications, Windows Update — and it is not fully governed by the classic sleep-timeout settings at all. It behaves more like a screen-off state than the sleep those timeouts were built to prevent, so disabling every timeout still leaves the machine periodically dropping into S0 idle on its own schedule.

The only fix at that point was disabling Modern Standby entirely, via a registry override:

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Power
    PlatformAoAcOverride    REG_DWORD    0
```

This forces the platform to fall back to classic ACPI sleep/hibernate, which does respond to the standard timeouts. Whether that fallback is actually S3 or hibernate-only depends on what sleep states the firmware implements underneath Modern Standby — worth checking `powercfg /a` again after the reboot this requires, since not every OEM board has a real S3 path to fall back to.

It's also a machine-wide switch — it can't be scoped to "AC power only," because it changes what "sleep" means at the platform level for both power sources. To compensate, I changed the battery lid-close action from sleep to hibernate, since a conventional sleep state might no longer be available for the battery profile to fall back on once Modern Standby is off:

```powershell
PS> powercfg /setdcvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 2
PS> powercfg /setactive SCHEME_CURRENT
```

## Outcome

Final shape: on AC, the lid does nothing, no timeout of any kind triggers sleep, and Modern Standby is off so no background process can silently re-sleep the machine. On battery, the lid triggers hibernate and the normal idle timeout applies, so unplugged battery life isn't sacrificed. The Modern Standby change only takes effect after a reboot and needs re-verifying with `powercfg /a` afterward — don't declare victory before that.

## Takeaways

- "Prevent sleep" on a modern Windows laptop is a stack, not a setting: the visible idle timeout, a per-scheme setting that can be flagged hidden, vendor power-management software that overrides the OS plan outright, an unattended-sleep timeout almost nobody knows exists, and on newer hardware, a whole different low-power architecture (Modern Standby) that ignores the old rules.
- When a power setting silently does nothing, check whether it's hidden before assuming you configured the wrong scope.
- `powercfg /lastwake` and the Power-Troubleshooter event log tell you *why* the machine last slept — that's faster than re-toggling the same checkbox and hoping.
- `powercfg /a` is the fastest way to find out whether you're fighting classic S3 sleep or Modern Standby, and they are not the same problem.
- A machine-wide override (disabling Modern Standby) can force you to compensate on the profile you didn't touch — here, giving battery mode hibernate instead of sleep, since sleep's availability changed for both power sources at once.
