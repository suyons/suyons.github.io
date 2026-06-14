---
title: "WireGuard Troubleshooting - ping Worked, SSH Said 'closed', and the Server Never Changed"
date: 2026-06-14
draft: false
tags: ["wireguard", "vpn", "networking", "ssh", "routing", "macos"]
categories: ["Infrastructure"]
description: "I couldn't SSH into my home server over WireGuard from a café. ping worked, the port was 'closed', and nothing had changed on either machine. The culprit was the café Wi-Fi using the same 192.168.0.0/24 subnet as my home LAN, so my SYN went to a stranger's device instead of through the tunnel."
showToc: true
---

## "It worked yesterday and the server hasn't moved"

I'm at a café. I want to SSH into my Ubuntu box at home over WireGuard. The tunnel is up, `ping` to the server's LAN IP answers in single-digit milliseconds, and then:

```
$ tcping 192.168.0.11 22
192.168.0.11 port 22 closed.
```

Closed. Not timed out — closed. The server hadn't been touched since yesterday, when this exact command worked. `sshd` was listening on `0.0.0.0:22`, the firewall was open, SSH to `localhost` on the box itself worked, and the logs were silent. Both endpoints were static. Nothing on either machine had changed.

The thing that changed was the café. And the one word that told me so was **`closed`**.

## `closed` and `timed out` are not the same failure

This is the part worth keeping even if you never touch WireGuard. When a TCP connect fails, the failure mode is a diagnostic signal, and most people read right past it.

- **`closed`** means a TCP **RST** came back. Something received your SYN and *actively refused* it. The packet reached a live host; that host had nothing listening on the port (or didn't want you).
- **`timed out`** means your SYN went into a hole. Nothing answered at all — a firewall `DROP`, a routing black hole, reverse-path filtering, an MTU problem.

For a VPN, that distinction carves the suspect list in half. All the usual "WireGuard plumbing" failures — wrong `AllowedIPs`, MTU too high, rp_filter dropping asymmetric routes, the tunnel just being down — produce **timeouts**. They drop packets silently. They do not send you a RST.

So a RST plus a working `ping` rules out the entire plumbing category in one shot. My SYN was reaching *something*, and that something was answering "no." The tunnel wasn't broken. The question wasn't "why are my packets disappearing" — it was "who is sending these refusals, because it clearly isn't my sshd."

## The host answered ping but had no SSH

Here's the realization that cracked it: `192.168.0.0/24` is one of the most common subnets on Earth. Home routers ship with it. So do café and hotel access points. If the network I'm physically sitting on *right now* also hands out `192.168.0.x` addresses, then `192.168.0.11` is a device on *this* café LAN — not my home server through the tunnel.

I checked from the Mac:

```
$ ipconfig getifaddr en0
192.168.0.27                       # the café put me on 192.168.0.x

$ route -n get 192.168.0.11
  interface: en0                   # routed out the LOCAL Wi-Fi, not the tunnel

$ arp -n 192.168.0.11
? (192.168.0.11) at aa:bb:cc:11:22:33 on en0   # a real local device answered ARP
```

There it is. Some random device on the café network owned `192.168.0.11`. It answered ARP (so `route` resolved it locally), it answered `ping` (it's alive), and it had no SSH — so it returned a RST. `closed`. My traffic for "my server" never went near the tunnel.

## Why a directly-connected route always wins

When you join a Wi-Fi network, your OS installs a directly-connected route for that network's subnet:

```
192.168.0.0/24  ->  en0
```

My WireGuard config is a full tunnel — it wants to carry `0.0.0.0/0`, default route into `utun6`. So in principle the server's subnet should ride the tunnel. But routing doesn't pick "the route I meant"; it picks the **most specific prefix that matches** (longest-prefix match). A directly-connected `/24` is more specific than a tunneled `default`, so for any `192.168.0.x` destination the café LAN wins and the tunnel never sees the packet.

That's also why it "broke today." Yesterday's café was probably on `192.168.5.x` or `10.x` — no overlap, so my `192.168.0.0/24` only existed inside the tunnel and everything routed correctly. The variable was never my server. It was whether the local network collided with my home subnet, and today it did.

## A misconception I had to drop first

Before I understood the collision I wasted a few minutes on a wrong mental model: I assumed WireGuard somehow "proxies" my home `192.168.0.0/24` into its tunnel network, and that I could just SSH to the server at its tunnel address instead. That's wrong on two counts, and it's a common misread of how WireGuard works.

- **WireGuard routes packets. It does not NAT or rewrite addresses** (not by default). A LAN IP stays exactly that IP inside the tunnel; it doesn't get translated into the tunnel's subnet.
- The tunnel network — say `10.10.10.0/24` — only addresses the **peers themselves**. My Mac is `10.10.10.2`, the WireGuard endpoint is `10.10.10.1`. The home server's host number `.11` belongs only to the `192.168.0.0/24` world. There is no `10.10.10.11`. That address simply doesn't exist.

This matters for my topology specifically: the WireGuard endpoint is my **home router** (`192.168.0.1`), and the Ubuntu server is a *separate* box behind it at `192.168.0.11`. The server has no tunnel address at all. The only way to reach it is to get packets *destined for `192.168.0.11`* into the tunnel, to the router, which then forwards them onto the home LAN. If those packets get siphoned off to the local café LAN before they ever enter the tunnel, the server is unreachable no matter how healthy the VPN is.

## Three fixes, in order of permanence

### 1. On the spot: a /32 host route that out-specifics the café

Longest-prefix match is the bug, so it's also the fix. A `/32` host route is more specific than the café's `/24`, so it wins and forces that one address through the tunnel:

```bash
sudo route -n add -host 192.168.0.11 -interface utun6
route -n get 192.168.0.11      # now: interface: utun6
tcping 192.168.0.11 22         # open
```

Caveat: this route lives until WireGuard next reconnects and rewrites its routing table, at which point you re-add it. It only works because my tunnel is full (`AllowedIPs` already covers that destination on the server side). It's a patch, not a cure, but it gets you in *right now*.

### 2. Jump through the router

The router is reachable over the tunnel at its peer address, and from the router's own perspective `192.168.0.11` is just its LAN — no collision exists there, because the router isn't sitting on the café network. So bounce through it:

```bash
ssh -J user@10.10.10.1 user@192.168.0.11
```

This needs the router to run an SSH daemon (OpenWrt with dropbear, for instance). Nice property: it sidesteps the client-side routing problem entirely, because the collision only exists on my laptop.

### 3. The real fix: re-subnet the home LAN off the default

The durable answer is to stop using a subnet that every café and hotel on the planet also uses. Change the home router from `192.168.0.0/24` to something rare:

- `192.168.77.0/24`
- `10.42.0.0/24`

Then the server becomes, say, `192.168.77.11`, which won't collide with the `192.168.0.x` / `192.168.1.x` defaults you meet while traveling. This kills the problem permanently instead of patching it per-trip.

One thing you *can't* do: fix it by changing the café's subnet (not yours) or your laptop's address (still the same subnet number). It's the **subnet number itself** that has to differ between the network you're on and the network you're tunneling to. The only place you control that is your home router.

## What I'd tell myself before blaming the server

The transferable lesson isn't about WireGuard. It's that when both endpoints are static and a connection breaks, **the variable is the network between them** — and routing decisions are made by longest-prefix match, not by intent.

- Read `closed` vs `timed out` as real evidence. A RST means the host was reached and refused you (so suspect *which* host you actually hit); a timeout means packets were dropped (so suspect routing, firewall, MTU). They point at opposite halves of the problem.
- When a VPN destination misbehaves, check whether the local network's subnet overlaps the remote one. `route -n get <ip>` tells you which interface a destination will actually use — tunnel or local — before you waste time on the server.
- `192.168.0.x` and `192.168.1.x` are collision magnets the moment you leave home. Pick an uncommon home subnet up front and you never have this debugging session.
- WireGuard routes; it doesn't translate. LAN IPs keep their addresses inside the tunnel, and the tunnel subnet only names the peers — not the hosts behind them.

The server was fine the whole time. It was answering exactly the right packets — I just wasn't sending them to it.
