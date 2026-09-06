+++
date = "2026-03-29"
draft = false
title = "The Real Cost of Context Switching in Linux: Cache, TLB, and CPU Warmup Explained"
+++

There is a number that every systems programmer learns:

A context switch costs about 1-2 microseconds.

It appears in textbooks, in interview prep material, in conference talks. Run **lmbench** on a pinned, cache-friendly setup and the number confirms itself - published measurements commonly land around 1.2-1.5 microseconds. Fast. Cheap. Almost free.

But this number measures a specific thing: the **direct handoff cost**. **How long the kernel spends saving one process's state and restoring another's**. **It does not measure what happens to the CPU's performance after the switch completes.**

The difference between those two things can explain why a system shows moderate CPU utilization, no lock contention, no I/O wait - and still suffers higher latency than the numbers suggest it should.

This article traces the full cost of a context switch - the part everyone measures and the parts that are harder to see.

## The Registers

When the Linux kernel switches from one process to another, it calls **context_switch()** in **kernel/sched/core.c**. The first thing it does is save the current process's CPU state to its **task_struct.**

This includes the general-purpose registers, the stack pointer, the instruction pointer, and - depending on usage - the floating-point and SIMD (Single Instruction, Multiple Data - It’s a CPU feature that lets a processor perform the same operation on many data points at once.) state. The kernel then loads the next process's saved state from its task_struct, restoring the registers to where that process left off.

This is the part that lmbench measures. This is where the "1-2 microseconds" comes from. Save registers, restore registers, update kernel bookkeeping, return.

If this were the whole story, context switches would be nearly free.

But the registers are the cheapest thing the CPU may need to rebuild.

## The Translations

Every memory access your process makes goes through the** TLB - the Translation Lookaside Buffer**. It is a small, fast cache inside the CPU that stores recent virtual-to-physical address translations. Without it, every memory access would require a page table walk: four levels of indirection on x86-64, potentially four separate memory reads just to translate one address.

A TLB hit costs about 1-2 cycles. A TLB miss triggers a page table walk that can cost 10 to 100 cycles, depending on whether the page table entries are themselves cached in the data cache.

When the kernel switches to a new process, the address space changes. On older hardware, this meant flushing the entire TLB - every cached translation gone, every subsequent access paying the full miss penalty until the TLB warmed up again.

Modern x86 processors support **PCID - Process Context Identifiers**. Each TLB entry is tagged with a 12-bit identifier, allowing entries from different processes to coexist. This means the CPU does not have to perform a full TLB flush on every context switch. Entries from a previous process can survive and be reused if that process is scheduled again on the same core. PCID significantly reduces TLB pressure under normal scheduling patterns.

But PCID is not a complete solution. The TLB is finite. As the new process runs and generates new translations, older entries - including retained ones from other processes - are evicted through capacity pressure. And Linux still has to manage and sometimes recycle these identifiers, since the 12-bit space allows 4,096 PCIDs. The result is that context switches can invalidate or pressure TLB state, even if full flushes are no longer the norm.

The TLB is the first place where the effective cost of a context switch diverges from the direct cost.

## The Cache

The L1 data cache on a modern x86 core is typically 48KB. The L2 is 1-2MB. The L3, shared across cores, ranges from 16 to 64MB. Access latencies climb steeply as you move outward: an L1 hit costs roughly 4 cycles, an L2 hit costs about 12, an L3 hit about 40, and a miss to main memory costs 200 or more cycles.

When your process is running, the cache gradually fills with your working set - the data and instructions you are actively touching. Hot loops, frequently accessed structures, stack frames. Over time, this data migrates into the faster cache levels. This is what makes your process fast. Not just the clock speed - the cache hit rate.

A context switch disrupts this.

The incoming process has a different working set. Its code, its data, its stack - much of it is unlikely to be in cache. Early accesses after the switch tend to suffer more misses, hitting slower cache levels or main memory. The cache must be warmed for the new workload, line by line, one access at a time.

How much this costs depends heavily on the workload. If both processes have small, overlapping working sets, the disruption may be minimal. If the incoming process has a large working set that exceeds L1 or L2 capacity, the early misses can be substantial - potentially tens of microseconds of degraded throughput while the cache rebuilds. If the switch also involved migrating to a different core, the penalty is larger still, because even the L2 cache is cold.

This is not a fixed number. It is workload-dependent. But it is real, and it can dwarf the direct switch cost for memory-intensive applications.

## The Predictor

Modern CPUs do not execute instructions one at a time. They speculate. The **branch predictor** examines conditional branches - if statements, loop conditions, function returns - and guesses which path the code will take before the condition is even evaluated. When the guess is correct, the pipeline stays full and execution continues at full speed. When the guess is wrong, the CPU flushes the pipeline and restarts from the correct path. A misprediction costs roughly 15-20 cycles on modern hardware.

A well-trained branch predictor achieves 95-98% accuracy on typical code. The CPU builds this accuracy over time by observing your process's branch patterns and storing them in internal history tables - the **Branch History Buffer**, the **Branch Target Buffer**, the **Pattern History Table**.

After a context switch, this predictor state may be less useful for the incoming process. The history tables reflect the old process's patterns, not the new one's. Early execution may see elevated misprediction rates until the predictor relearns the current workload's behavior. Each misprediction flushes the pipeline - 15-20 cycles of wasted work on a deeply pipelined processor.

The predictor cost is not as dramatic as the cache penalty, but it compounds with it. Every misprediction stalls the pipeline at a moment when the cache is also producing more misses.

## The Prefetcher

The hardware prefetcher monitors memory access patterns and detects regular strides - sequential reads, array traversals, pointer chases. When it identifies a pattern, it fetches cache lines ahead of your code, so the data is already in cache by the time your process needs it.

A well-trained prefetcher can hide a significant fraction of memory latency for sequential and strided workloads. It is one of the reasons that iterating over an array feels fast - the hardware is reading ahead and absorbing the latency.

After a context switch, the incoming workload may not yet have established useful prefetch patterns. The prefetcher's internal state is microarchitectural and vendor-specific - it is not formally "reset" by the kernel, but the patterns it learned from the previous process are unlikely to help the new one. For workloads that depend heavily on sequential access - database scans, network packet processing, log parsing - the loss of useful prefetch locality can mean early accesses pay full memory latency that would otherwise be hidden.

## The Compound

Now consider the combined effect.

The kernel completes the context switch in 1-2 microseconds. The registers are saved and restored. The scheduler has done its job.

But the process that resumes may be running on a CPU where much of the microarchitectural state is less useful for its workload:

The key insight is that these costs do not appear in any standard tool as "context switch overhead." The degraded execution after a switch counts as normal CPU time. Your process is running. It is just running with a worse cache hit rate, more TLB misses, and more branch mispredictions than it would have if it had not been interrupted.

This is why latency-sensitive systems - trading platforms, game servers, real-time audio, network packet processors - go to lengths to minimize context switches and core migrations. Not because the switch itself is expensive. Because the effective cost extends well beyond the switch, in ways that depend on the workload and are invisible to standard tools.

## The Tools

Making the effective cost visible requires looking at hardware performance counters.

Voluntary switches happen when your process blocks - waiting for I/O, sleeping on a mutex. Involuntary switches happen when the scheduler preempts your process because its time slice expired or a higher-priority task arrived. Both carry the same potential for microarchitectural disruption. But involuntary switches are the ones you did not ask for, and they are often the ones worth investigating first.

If nonvoluntary_ctxt_switches is climbing and latency is unexplained, the scheduler may be imposing a cost that no dashboard will show as "context switch overhead." The cost shows up as slightly worse IPC, slightly more cache misses, slightly higher memory latency, never attributed to the switch that caused it.

---

A context switch itself may only cost a few microseconds.

But the effective cost can be much larger.

The direct cost is what the benchmarks measure. The effective cost is what your application experiences.

The next time someone tells you a context switch costs 1-2 microseconds, they are not wrong. They are measuring the kernel's work. Not the CPU's recovery.

We measure what is easy to measure.

Rarely what is expensive to endure.
