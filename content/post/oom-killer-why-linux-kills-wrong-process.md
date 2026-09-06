+++
date = "2026-05-10"
draft = false
title = "The OOM Killer: Why Linux Kills the \"Wrong\" Process"
+++

You open the logs and realize Linux killed your database.

It’s frustrating.

Why postgres? Why not the leaking cron job? Why did the kernel kill the most important process on the box instead of the one that actually caused the crisis?

The answer is simple, if a bit cold:

**The OOM killer doesn’t care who started the fire. It only cares about putting it out as fast as possible.**

Most engineers think of the **OOM killer** as a detective that’s bad at its job. In reality, it’s an **emergency responder**. By the time it’s called in, the system has already tried reclaim, compaction, swap, compression, and throttling.

The kernel is no longer looking for fairness.

It’s looking for survival.

This article walks through that pipeline - from overcommitting memory to the final SIGKILL - and explains why Linux kills the process it was designed to protect last.

## The Kernel’s Big Lie

Memory management in Linux starts with a lie.

When a process calls malloc(), the kernel usually says “yes” long before it actually has the physical RAM to back it up.

This works because virtual memory decouples address space from physical hardware.

It’s also why **Copy-On-Write (CoW)** is so powerful.

Before CoW, fork() was expensive because the kernel had to duplicate every memory page immediately. BSD and Linux changed this by marking pages as shared and read-only after fork(). A physical copy is only created when either process writes to the page.

That single optimization transformed fork() from something expensive into one of the most efficient primitives in Unix systems. Shells, web servers, databases, and process managers all depend on this behavior today.

But the same optimism that makes CoW efficient also delays visibility into memory pressure. Allocations succeed before physical pressure becomes obvious. The real cost only appears later, when processes begin touching pages and the kernel has to find actual RAM to back them.

That delayed pressure drives everything that happens next.

## The Reclaimer

When a process touches a page and there’s no free RAM available, the kernel enters reclaim mode.

This logic lives primarily in **mm/vmscan.c**. Its job is straightforward in principle: **free enough pages for the allocation to succeed without killing anything**.

The kernel begins by scanning reclaimable memory for page cache entries that can be reloaded from disk, inactive anonymous pages, clean filesystem buffers. If pressure increases, the system begins swapping colder anonymous pages to disk. This is the moment systems start feeling sluggish under load. Latency spikes because memory access is now competing with disk I/O.

The kernel also attempts **memory compaction**, **rearranging fragmented pages into larger contiguous regions** for future allocations.

Most memory spikes end here.

Reclaim succeeds, the allocation proceeds, and the user never notices. But reclaim has limits. When the kernel repeatedly scans memory and “makes no progress,” the system enters a much more dangerous state.

## Compression and Back-Pressure

Before reaching for the power tool, modern Linux tries softer forms of survival. One important mechanism is **zswap**. Instead of immediately writing **cold pages** to slow disk swap, zswap **compresses them into a dense in-memory pool**. It cannot create infinite memory, but it can absorb temporary pressure spikes that would otherwise collapse into swap storms.

This is conceptually similar to memory compression systems used in macOS.

Modern Linux also introduces **back-pressure through cgroup v2**.

The **memory.high** control allows the kernel to** throttle allocations instead of immediately killing processes**. Once a cgroup crosses this threshold, allocations become slower and reclaim pressure increases aggressively within that group.

This gives applications a chance to shrink caches, scale down worker pools, reject requests gracefully. At the same time, **PSI (Pressure Stall Information)** measures how much time the system spends stalled on resource pressure.

This is effectively an early warning system.

The **some** metric measures** how often at least one process was stalled waiting for memory**, while **full** measures **moments where all runnable processes were stalled simultaneously** because memory pressure became severe enough to stop forward progress entirely.

The **avg10, avg60, and avg300** fields represent the **percentage of time the system spent stalled over the last 10 seconds, 60 seconds, and 5 minutes** respectively. In this example, some avg10=12.4 means workloads spent 12.4% of the last 10 seconds waiting on memory pressure. The more alarming metric is full avg10=0.2, which means that for 0.2% of that same window, the entire system effectively stopped making progress due to memory stalls.

As a rough operational rule, a low some value is normal under moderate load, but sustained values above 10 - 20% usually indicate reclaim pressure is becoming visible to applications. Once full rises above zero consistently, the system is entering dangerous territory. If pressure continues rising from there, the OOM killer is often not far behind.

A rising some value means workloads are slowing down under pressure. A non-zero full value means the system is approaching complete stall.

Modern Linux does not jump directly to SIGKILL. It attempts reclaim, compression, throttling, and pressure signaling first. But eventually, those mechanisms can fail. And when they do, the kernel enters the final stage.

## The OOM Path

The OOM killer lives in **mm/oom_kill.c**.

By the time it activates, the allocator is failing, reclaim has stalled, and the system is running out of forward progress. Depending on where the pressure originates, this may be a system-wide OOM event or a cgroup-local OOM.

The kernel now has to free memory immediately. It evaluates every process using **oom_badness()**. In simplified form:

The calculation primarily considers Resident Set Size (RSS), Swap usage, Page table memory, oom_score_adj etc. The **process with the highest score becomes the victim**.

This is the part most engineers misunderstand:

**The OOM killer is not looking for the process that caused the shortage. It is looking for the process whose end frees the most memory.**

That distinction explains almost every “wrong process” incident.

## Rational Logic, Irrational Outcome

Imagine a nightly batch job with a slow memory leak.

At 2:45 AM:
- The batch job consumes 2GB
- Postgres consumes 14GB
- The machine runs out of reclaimable memory

The OOM killer activates. The batch job caused the crisis. Postgres becomes the victim.

Why?

Because killing Postgres frees 14GB instantly. Killing the batch job only frees 2GB and may not stabilize the machine.

From the kernel’s perspective, this is a rational decision. From the engineer’s perspective, it feels catastrophic. The kernel does not understand your business priorities. It only understands reclaim efficiency.

## Teaching the Kernel Priorities

Linux gives engineers several mechanisms for influencing OOM behavior. The simplest is **oom_score_adj**. Every process exposes this value through:

You can adjust process priority directly:

Negative values protect a process. Positive values make it a preferred target. Containers improved this model dramatically. With cgroups, memory pressure becomes localized. If a container exceeds its memory budget, the OOM killer fires inside that cgroup instead of across the entire machine.

This changed production systems completely.

A leaking batch container no longer kills the host database. The blast radius becomes isolated. This is why Kubernetes memory limits matter so much. They are not just quotas. They are fault-containment boundaries.

## The OOM killer is not broken

It is doing exactly what Linux designed it to do. The misunderstanding comes from assuming the kernel shares human priorities. It doesn’t.

The kernel cannot distinguish between a database, a batch processor or a log shipper. It only sees memory pressure and reclaim efficiency.

The Linux memory subsystem is fundamentally a layered survival system with optimistic allocation, reclaim, compression, throttling, pressure signaling, final termination. The OOM killer is simply the last mechanism in that chain. And by the time it activates, every softer option has already failed.

Linux gives engineers the tools like oom_score_adj, cgroup memory limits, memory.high, PSI monitoring.

If those tools are never configured, the kernel falls back to its default logic:

Kill the process that frees the most memory. The OOM killer didn’t kill the wrong process. You just forgot to tell Linux which one mattered most.
