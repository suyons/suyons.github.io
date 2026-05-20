---
title: "Solving Hairpin NAT on a Cloud VM: The Windows Loopback Adapter Trick"
date: 2026-05-13
draft: false
tags: ["networking", "windows-server", "nat", "cloud", "onlyoffice", "loopback"]
categories: ["Infrastructure"]
description: "Cloud VMs can't reach their own public IP by default — a restriction called hairpin NAT. This post explains why it happens and walks through the Windows Loopback Adapter workaround, including the 'Weak Host' model fix that Windows requires."
showToc: true
---

## The Problem

While running a demo environment where OnlyOffice Docs and the host application were deployed on the **same cloud VM**, the editor refused to load any document. The browser spinner ran indefinitely even after confirming the OnlyOffice service itself was healthy.

The root cause: the host application passes a `callbackUrl` to OnlyOffice that points to the application's own endpoint. OnlyOffice's DocService fetches this URL to retrieve the document file. In the demo setup, both services were on the same VM — so OnlyOffice was trying to `GET` a URL whose hostname resolved to the VM's **public IP**.

Testing this directly:

```
tcping <public-ip> 32090

Probing <public-ip>:32090/tcp - No response - time=2007ms
Probing <public-ip>:32090/tcp - No response - time=2015ms
...
0 successful, 4 failed. (100.00% fail)
```

The VM could not reach its own public IP. This is the **hairpin NAT** (also called loopback NAT) problem.

---

## Why Cloud Providers Disable Hairpin NAT by Default

### 1. The Architectural "U-Turn"

Cloud networking is built on Software-Defined Networking. In a standard VPC, a VM's public IP is not assigned to the VM's network interface — it exists on a distributed gateway that performs 1:1 NAT. When a packet originates from inside the network but is addressed to an external public IP that maps back to inside the same network, the SDN router sees a U-turn packet.

Most cloud routers treat this as inefficient or anomalous and drop it.

### 2. Security: IP Spoofing Risk

Allowing a packet from an internal interface to carry an external public IP source address can bypass firewall rules that expect external traffic to come from external interfaces. Disabling hairpin enforces strict source routing — traffic appearing to come from the outside must actually originate from the outside.

### 3. Cost and Efficiency

Many cloud providers bill for egress. Routing traffic out to the public IP and back in could accidentally trigger egress charges for traffic that never left the datacenter. Even where it doesn't, bouncing traffic through the VPC edge and back adds latency and wastes resources.

### 4. NAT is Distributed

On a home router, NAT lives in one physical box and can easily handle U-turns. In a cloud VPC, the public IP is managed by a distributed gateway fleet. Coordinating hairpin behavior across that fleet is complex and adds maintenance surface for a niche use case.

---

## Solutions

For a production deployment, the clean answer is simple: **put OnlyOffice and the application on separate VMs** and use private IPs for internal communication. Hairpin NAT never comes up.

For a test/demo environment where co-location is the point, there are three options:

| Option | How | Suitable for |
|--------|-----|--------------|
| **Loopback Alias** | Assign the public IP to a local loopback adapter | Test environments, single VM |
| **Environment variable** | Use different base URLs for internal vs external | Code you control |
| **Private IP** | Replace public IP in callback URLs with the private IP | When the app allows it |

I chose the **Loopback Alias** because the demo environment has both services co-located by design, and I did not want to change the application's URL-generation logic.

---

## The Windows Loopback Adapter Fix

### Step 1: Install the Microsoft KM-TEST Loopback Adapter

This virtual network adapter lets you assign arbitrary IP addresses to the local machine without affecting real network interfaces.

1. Open **Device Manager** (right-click Start → Device Manager).
2. Select the computer name at the top → **Action** menu → **Add legacy hardware**.
3. Click **Next** → **Install the hardware that I manually select from a list (Advanced)**.
4. Select **Network adapters** → **Next**.
5. Manufacturer: **Microsoft** → Model: **Microsoft KM-TEST Loopback Adapter**.
6. Click **Next** through completion.

### Step 2: Assign the Public IP to the Adapter

1. Open **Network Connections** (`Win+R` → `ncpa.cpl`).
2. Find the new adapter (usually named "Ethernet 2" or "Local Area Connection 2"). Rename it to `Loopback` to avoid confusion later.
3. Right-click → **Properties** → **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Enter your VM's public IPv4 address.
5. Set Subnet Mask to `255.255.255.255`.
6. **Leave the Default Gateway blank** — this is critical. Adding a gateway here will break your actual internet routing.

### Step 3: Fix the Windows "Strong Host Model"

This step trips up everyone who does this the first time. Windows uses the **Strong Host Model** by default: an interface only processes packets that are explicitly addressed to it. Because the loopback adapter and the main NIC are two separate logical interfaces, Windows will not let them exchange traffic as expected.

You need to enable **Weak Host Send and Receive** on both interfaces:

```powershell
# List adapter names
Get-NetAdapter

# Enable weak host model on both interfaces
# Replace "Loopback" and "Ethernet" with your actual adapter names
Set-NetIPInterface -InterfaceAlias "Loopback"  -WeakHostSend Enabled -WeakHostReceive Enabled
Set-NetIPInterface -InterfaceAlias "Ethernet"  -WeakHostSend Enabled -WeakHostReceive Enabled
```

With this in place, when your application sends a request to the VM's public IP, the OS sees that the public IP belongs to the loopback adapter, routes the packet internally, and never sends it out to the VPC router.

### Verification

```
tcping <public-ip> 32090

Probing <public-ip>:32090/tcp - Port is open - time=0.5ms
```

The OnlyOffice editor then opened and loaded documents correctly.

---

## When to Use This (and When Not To)

**Use it:** temporary demo or test environments where two services that would normally talk over the network are co-located on one VM for cost reasons.

**Don't use it in production:** the proper architecture is separate hosts with private IP communication. The loopback alias approach works but is a workaround — it adds a hidden dependency on the VM's specific IP assignment, and it breaks if the public IP changes (elastic IPs reassigned, VM reprovisioned, etc.).

If you're embedding OnlyOffice in a production application, always configure the `callbackUrl` to use the application's private IP or internal DNS name so the OnlyOffice DocService communicates over the VPC fabric rather than through the public NAT path.

---

## Summary

Cloud VMs cannot reach their own public IP because hairpin NAT is disabled by default in SDN-based VPCs. On Windows, the fix is installing the Microsoft KM-TEST Loopback Adapter, assigning the public IP to it, and enabling the Weak Host model via `Set-NetIPInterface`. The OS then handles the loopback routing without touching the VPC router. Useful for test setups; not a substitute for proper service separation in production.
