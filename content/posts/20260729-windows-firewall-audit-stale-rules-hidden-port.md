---
title: "Windows Firewall Security - Auditing 51 Rules Turned Up Two Holes We Weren't Looking For"
date: 2026-07-29
draft: false
tags: ["windows-firewall", "network-security", "powershell", "mysql", "security-audit"]
categories: ["Security"]
description: "Cutting a Windows Server firewall from 51 hand-written inbound rules to 23 by auditing against live listening sockets instead of rule names — and the two security holes the process surfaced that nobody had gone looking for."
showToc: true
---

A long-lived application server had accumulated 51 hand-written inbound firewall rules over years of client projects. Nobody could say with confidence which ones still mattered. This is the story of cutting that down to 23 — and of the two security holes the audit turned up along the way, neither of which was the thing we set out to find.

## Measure against reality, not against names

The instinct with a rule list this messy is to read the names and delete whatever looks stale. That's how you take down production. A rule name records what someone *intended* years ago; it says nothing about whether the service behind it still runs.

The better signal is whether anything is actually bound to the port right now — a live cross-reference between every user-created rule and the current listening set:

```powershell
function Expand-Ports($s) {
  $r = @()
  foreach ($p in ($s -split ',')) {
    if     ($p -match '^(\d+)-(\d+)$') { $r += ([int]$Matches[1])..([int]$Matches[2]) }
    elseif ($p -match '^\d+$')         { $r += [int]$p }
  }
  ,$r
}

$tcp = @(Get-NetTCPConnection -State Listen | Select -Expand LocalPort | Sort -Unique)

# User-created rules are the ones with no Group - built-in Windows rules always have one
Get-NetFirewallRule -Direction Inbound |
  Where-Object { [string]::IsNullOrEmpty($_.Group) }
```

That `Group` check does most of the filtering work by itself. Every rule Windows ships with belongs to a group ("File and Printer Sharing", "Remote Desktop," and so on). Rules a human added through the GUI or `netsh` have an empty group. Filtering on it separates "stuff we added" from "stuff the OS needs" without maintaining a separate list of exclusions.

Classifying each rule as fully-used, partially-used, or entirely dead put 16 rules in the dead column — ports with nothing listening, including a scheduling service, several numbered placeholder rules, and two byte-identical duplicates.

An important caveat went into the report before anything was deleted: this is a point-in-time snapshot. A service that's stopped, scheduled, or started on demand looks identical to one that's gone forever. Several of the dead rules were named after specific past engagements — exactly the profile of something turned on only during active work with a given client. Stating that uncertainty, rather than burying it, is what let the operator make a real decision instead of rubber-stamping a list.

## A rename that silently opened a hole

Partway through, a rule vanished from the audit output between two runs of the script. It had been renamed by hand.

The rename itself was harmless. What came with it wasn't: the port range had also been edited, from `32038-32041` down to `32020-32040`. That widened the low end by eighteen ports and quietly **dropped port 32041 off the top** — and 32041 had a live service on it.

The only reason this surfaced is that the audit compared rules against listening ports rather than against the previous version of the rule list. A diff of rule names would have shown a rename and moved on. Comparing against reality showed a port with active traffic and no coverage.

The general lesson: when a firewall rule is expressed as a range, the endpoints are where bugs hide. Widening a range feels safe, and usually is — but a range edit is simultaneously a widening *and* a narrowing, and only one of those halves is intuitive.

## Consolidating overlaps into one authoritative range

With the dead rules gone, the remaining mess was overlap. Seven ports were covered by two rules each, and several rules straddled the boundary of the newly widened range — partly inside it, partly outside.

Straddling rules can't just be deleted. The audit split each one into the ports the target range already covered and the ports it didn't, then checked which of the leftovers were actually live:

```
Rule A   in-range: 32038, 32039   outside: 32041, 32050, 32051   <- 32041 is LISTENING
Rule B   in-range: 32033          outside: 32043, 32099          <- 32043 is LISTENING
Rule C   in-range: 32035-32037, 32040   outside: 40000, 32140    <- neither listening
```

The operator's call was to make one range authoritative for the whole band and delete everything inside it: widen it to `32020-32099` so it swallowed the stragglers, then remove the eight rules whose ports now fell entirely within it. One rule kept a trimmed existence because two of its ports sat outside the band entirely.

A pleasant side effect: seven ports with live services — discovered during the audit — had **no firewall rule at all** and had been silently blocked from outside. Widening the range fixed connectivity nobody had reported as broken.

The trade-off is worth stating plainly, because "one big range" isn't free. Eighty ports are now open where twenty are in use. Any future service that binds inside that band is externally reachable the moment it starts, with no firewall change and no review. That's a deliberate convenience-for-exposure trade, defensible on an internal server — but it should be a decision, not an accident.

## "Local subnet" is not a fixed CIDR

Tightening a database port's scope raised a question that sounds trivial and isn't: is the Windows Firewall GUI's **Local subnet** scope the same thing as typing `192.168.1.0/24`?

On a single-homed machine, effectively yes. On this one, no. `LocalSubnet` is a *dynamic keyword* — Windows resolves it at packet-evaluation time to the subnet of every currently active interface. This host had two interfaces up: a physical adapter on the office LAN and a WireGuard tunnel on a separate private range. `LocalSubnet` covered both. A static CIDR covered only one.

That difference had teeth. Scoping the database to the static LAN range would have cut off every remote user coming in over the VPN tunnel — a breakage that surfaces hours later as "the database is down," reported by people who weren't in the room to see the change.

```powershell
# Static: exactly one subnet, forever. Predictable, but silently stops
# matching if the network is ever renumbered.
Set-NetFirewallRule -DisplayName '<rule>' -RemoteAddress @('192.168.1.0/24','10.10.0.0/24')

# Dynamic: follows whatever subnets the host is currently on, including
# any interface that comes up later.
Set-NetFirewallRule -DisplayName '<rule>' -RemoteAddress LocalSubnet
```

Neither option is strictly correct. The static form states intent explicitly and won't drift; the dynamic form survives network renumbering and automatically covers new tunnels. The operator chose dynamic, which suits a host whose VPN topology changes more often than its rule set does. Either way, `LocalSubnet` expands on its own: bring up a spare network adapter with no DHCP server present, and it self-assigns a link-local address, quietly adding that entire range to the rule's scope.

## The side door nobody was watching

The last finding mattered most, and it came from a throwaway question about an unfamiliar port number.

Port 33060 was listening and looked database-related. It is: modern MySQL enables the X Plugin by default, and the *same* database process serves the classic protocol on 3306 and the X Protocol on 33060. Different port, different wire format, same data, same user accounts — and the X Protocol will happily execute arbitrary SQL.

| Port | Protocol | Scope | Live connections |
|---|---|---|---|
| 3306 | classic | local subnet | 15 |
| 33060 | X Protocol | **any** | **0** |

The port carrying all the real traffic had just been carefully restricted, while a second, entirely unused port reaching the identical database sat open to the world. The lock had been fitted to the front door with the side door left standing open.

The fix is to close it at the firewall — nothing connects to it, and the common database drivers speak the classic protocol only and will never touch it. Unloading the plugin outright would stop the port from binding at all, but that requires a database restart and a maintenance window, so firewall-level closure is the pragmatic move today.

The transferable lesson: **when you restrict a service, enumerate every port that service listens on.** Databases, message brokers, and application servers routinely expose administrative or alternate-protocol ports right next to their primary one. Hardening the port you were thinking about, while an equivalent path sits open beside it, produces the paperwork of security without the substance.

## Two PowerShell gotchas that both cost a failed command

Both share a root cause: cmdlets that take arrays don't accept the comma-joined strings they print back to you.

```powershell
# Before - fails with "The port is invalid"
Set-NetFirewallRule -DisplayName '<rule>' -LocalPort '40000,32140'

# After - the parameter wants an array, not a comma-joined string
Set-NetFirewallRule -DisplayName '<rule>' -LocalPort @('40000','32140')
```

And parsing a process manager's JSON output on Windows hits a case-collision the default converter refuses outright, because the payload contains environment variables differing only in case:

```powershell
# Before - "Cannot convert the JSON string because it contains keys with
# different casing" (an environment block holds both USERNAME and username)
$apps = pm2 jlist | ConvertFrom-Json

# After - hashtables are case-sensitive, so both keys coexist
$apps = pm2 jlist | ConvertFrom-Json -AsHashtable
```

That second one generalizes beyond this one command: any JSON originating from a Windows environment block is liable to trip the case-insensitive object converter, and `-AsHashtable` is the standard escape hatch.

## Outcome

The rule set went from 51 user-created inbound rules to 23 — 16 removed as genuinely dead, 3 as duplicates, 8 absorbed into a single authoritative range. Every deletion was verified afterward against the live listening set, confirming that no port with a running service lost coverage and that no port stayed covered by two rules. Two rules were modified rather than deleted, and the open X Protocol port was flagged for closure.

## Takeaways

- **Audit against runtime state, not against names.** Rule names describe intent from years ago; listening sockets describe now. Only one of them tells you what breaks if you delete something.
- **Say out loud what a snapshot cannot tell you.** "Nothing is listening" and "this is safe to delete" are different claims. Conflating them is how an on-demand service gets firewalled off until someone complains.
- **Back up before each stage, not once at the start.** Exporting the policy before every batch means each step gets its own restore point, and a mistake in the last stage doesn't cost the work of the first three.
- **Range endpoints are where errors hide.** Every range edit narrows something as well as widening something.
- **Check every port a service exposes.** Alternate-protocol and admin ports sit right next to the primary port, and hardening only the obvious one is theater.
