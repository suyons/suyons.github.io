---
title: "Windows Server Troubleshooting - 97% Memory Usage Was Three Different Problems Wearing One Number"
date: 2026-07-28
draft: false
tags: ["windows-server", "oracle-database", "jvm", "memory-management", "performance-troubleshooting"]
categories: ["Infrastructure"]
description: "A production Windows Server box running four database and runtime engines had crept to 97% memory usage; separating working set from committed bytes per process ruled out the JVM and pointed at one Oracle parameter that pins the SGA in RAM."
showToc: true
---

A production Windows Server box running four database and runtime engines at once had crept up to 97% memory usage. Diagnosing it meant separating two numbers that get conflated constantly — what a process actually holds in RAM versus what it has merely reserved — and the two processes with the biggest footprints turned out to be lying about their memory in opposite directions.

## Getting an htop-equivalent on Windows

The first problem was tooling. There's no default `htop` on Windows, and the usual suggestions — `btop4win`, the cross-platform `bottom`, Sysinternals' `pslist -s` — are fine but none of them are built natively for the Win32 API the way `htop` is for `/proc`. **NTop** is, and it installs cleanly from Chocolatey:

```powershell
choco install NTop.Portable -y
```

Chocolatey drops the shim on PATH, so `ntop` works from any shell afterward. You get colored CPU/memory bars, a sortable process list, and `htop`-style keybindings (`F9`/`k` to kill, `F6` to sort, `/` to filter) — close enough to stop reaching for Task Manager.

## Reading the bars correctly: CPU wasn't the problem

NTop's header looked like this:

```
CPU[|||  15.8%]  Mem[|||||||||||| 97.4%]  Pge[|||||||||| 88.8%]
```

The instinct is to hunt for a CPU villain. CPU was idle at 15.8%. The actual pain was memory (97.4%) and the page file (88.8%) — two different symptoms that get collapsed into "the server is slow" if you don't separate them. Per process, that means pulling two distinct numbers: **working set** (what's resident in physical RAM right now) and **private/committed bytes** (what the process has reserved, some of which may already be paged out):

```powershell
Get-Process | Sort-Object WorkingSet64 -Descending |
  Select-Object -First 12 Name, Id,
    @{N='WorkingSet(MB)';E={[math]::Round($_.WorkingSet64/1MB,1)}},
    @{N='PrivateMem(MB)';E={[math]::Round($_.PrivateMemorySize64/1MB,1)}} |
  Format-Table -AutoSize
```

Two processes stood out, with opposite profiles:

- **Oracle** — roughly 5.1 GB resident, roughly 5.2 GB committed. A large but *honest* footprint: what it holds is close to what it's reserved.
- **A Java-based PDF rendering service** — only 1.8 GB resident, but **8.7 GB committed**. Roughly 7 GB had been pushed out to the page file, which is what was inflating the page-file bar specifically.

The system-wide numbers confirmed genuine overcommit rather than a merely busy machine: 16 GB physical RAM with 0.3 GB free, and 32 GB committed against a 35.9 GB commit limit (RAM plus page file).

## Does a reboot "flush" the page file?

A reboot recreates the page file fresh and restarts every process small, so the bars look healthy — briefly. Nothing is actually fixed; the same workload re-inflates within minutes, because the page file is a symptom, not the disease. Windows has no runtime "flush the page file" command. The only adjacent setting, `ClearPageFileAtShutdown`, is a security wipe that slows shutdown and does nothing for a running system. If you need relief without a reboot, restarting the single worst offender reclaims most of the memory with far less disruption than restarting the box.

## Checking the Java heap before blaming it

Before touching anything, read the process's actual command line rather than guessing:

```powershell
(Get-CimInstance Win32_Process -Filter "ProcessId = <pid>").CommandLine
```

The heap flags were already sane: `-Xms2048m -Xmx4096m` with G1GC. This matters because `-Xmx` only bounds the **heap**. This was a PDF rendering engine, and the roughly 4.7 GB gap between its 4 GB max heap and its 8.7 GB commit was off-heap native memory — direct byte buffers, native image buffers, metaspace — none of which `-Xmx` touches. Lowering `-Xmx` would not have reclaimed that gap and risked OOM-killing the JVM under load for no benefit. The correct lever for native growth is `-XX:MaxDirectMemorySize`, not `-Xmx`. So the JVM was left alone, and the bigger, safer win was Oracle.

## The core fix: shrinking Oracle's SGA, with a large-pages gotcha

Oracle on Windows runs everything inside a single `oracle.exe` process, so that 5 GB working set *is* the SGA. Connecting with OS authentication and reading the memory parameters told the real story:

```
sga_target      = 4896M   (~4.78 GB)   <- this is the 5 GB
sga_max_size    = 4896M
DEFAULT buffer cache = 3872M           <- dominant consumer
use_large_pages = TRUE                 <- the important gotcha
```

`use_large_pages = TRUE` changes the whole approach. With large pages enabled on Windows, Oracle allocates the entire `sga_max_size` up front at startup and locks it into physical RAM — it can never be paged out. That means the usual trick of lowering `sga_target` at runtime frees nothing at all, because the memory was never eligible for release in the first place. Returning RAM to the OS requires lowering `sga_max_size` and restarting the instance. There's no dynamic path around it.

Before:

```sql
sga_max_size = 4896M
sga_target   = 4896M
```

After, applied to the SPFILE and followed by a clean bounce:

```sql
ALTER SYSTEM SET sga_max_size=3G SCOPE=SPFILE;
ALTER SYSTEM SET sga_target=3G   SCOPE=SPFILE;
SHUTDOWN IMMEDIATE;
STARTUP;
```

Because it's an SPFILE change, it survives future reboots. One thing worth flagging so it doesn't look like a failed restart: on Windows the instance keeps the same OS process ID after `SHUTDOWN`/`STARTUP` — it re-initializes inside the existing service process instead of spawning a new one.

## Outcome

The restart landed cleanly and the numbers moved exactly as expected:

| Metric | Before | After |
|---|---|---|
| Oracle working set | 5147 MB | 3321 MB |
| Free RAM | 0.3 GB | 2.5 GB |
| Committed | 32 GB | 29.7 GB |

RAM pressure went from "0.3 GB free, about to fail allocations" to a comfortable 2.5 GB free. The page-file dimension is still tight — commit is still around 88% — driven mostly by the PDF engine and the remaining database services, so the next lever there is shutting down whichever engine isn't actually serving traffic, not another round of SGA tuning.

## Bonus: speccing a RAM upgrade from the command line

Since the plan is to physically add RAM, it's worth profiling the hardware from PowerShell before ordering anything:

```powershell
Get-CimInstance Win32_PhysicalMemory      # per-module vendor, part no., type, speed
Get-CimInstance Win32_PhysicalMemoryArray # total slots + max capacity
```

This turned up a single Samsung 16 GB DDR4-2666 non-ECC UDIMM (part `M378A2K43CB1-CTD`, CL19 at 1.2V) in a board with 4 slots, 3 free, 64 GB max. Two things worth knowing:

- **CAS latency isn't exposed by WMI.** Windows simply doesn't report CL. You infer it from the part number — the `-CTD` suffix maps to DDR4-2666, JEDEC CL19 — or read the SPD directly with a tool like CPU-Z.
- **A blank motherboard identity is common on white-box builds.** Every board/system DMI field reported `Default string`, meaning the integrator never programmed SMBIOS. When that happens, the CPU is the reliable source of truth: an Intel Xeon E-2234 (LGA1151, C242/C246 chipset) pins the platform to DDR4-2666 UDIMM, dual-channel, with optional ECC support.

The practical takeaway: the machine was running single-channel on one stick, so adding matched 16 GB modules both increases capacity and unlocks dual-channel bandwidth. Three more identical sticks fill all four slots for 64 GB. Since the CPU supports ECC UDIMM, a full ECC swap is an option worth considering for a database server — but only if you go all-ECC rather than mixing ECC and non-ECC modules.

## Takeaways

- **Read the right bar.** High "memory pressure" is often a commit/page-file story, not CPU. Separate working set from committed bytes per process before accusing anyone.
- **`-Xmx` is not the whole JVM.** Native/off-heap memory — common in image, PDF, and ML workloads — lives outside the heap. Bound it with `-XX:MaxDirectMemorySize`, and don't starve a heap that's actually in use.
- **Large pages pin the SGA.** If `use_large_pages = TRUE`, only `sga_max_size` plus a restart reclaims Oracle memory. Runtime `sga_target` changes do nothing.
- **A reboot buys minutes, not a fix.** If the workload wants 32 GB on a 16 GB box, it re-inflates. Right-size the consumers or add RAM.
