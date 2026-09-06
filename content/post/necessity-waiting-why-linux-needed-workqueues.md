+++
date = "2026-01-18"
draft = false
title = "The Necessity of Waiting : Why Linux Needed Concurrency-Managed Workqueues"
+++

A user clicks a button.

That click becomes an electrical signal, a hardware interrupt, a fleeting bit of code in the deepest layer of your computer: the kernel.

Some responses are immediate. Others must wait - for a disk to spin, a network packet to arrive, a memory operation to complete.

For a user application, waiting is simple.

It asks the operating system to pause it, and the kernel schedules another task. But the **kernel itself** has no higher authority to handle its own pause. It** cannot block.** It must remain perpetually responsive. This absolute constraint - the need to wait without ever stopping - has shaped the architecture of modern computing.

This is the story of how Linux, after years of imperfect solutions, finally mastered the art of deferred work. It’s a story that begins with clever patches and ends with a fundamental rethinking of resource management.

In the kernel, work that cannot wait is handled in “interrupt context.”

This code path runs immediately, but under draconian rules: it cannot allocate memory, cannot acquire most locks, and absolutely cannot sleep. Its purpose is to acknowledge the hardware and schedule real work for later.

The **first attempt** at this “later” was the **bottom half**.

The interrupt handler would do the bare minimum and flag a bottom half to finish the job. The **flaw** was fatal: **bottom halves ran with interrupts disabled.** A long-running bottom half would make the system unresponsive, defeating its very purpose.

The kernel needed a safe place where code could sleep, lock, and think.

It needed a way to say, “*Do this later - without blocking everything else*.”

## The Awkward Compromise

**Tasklets** emerged **as** an **improvement** over bottom halves.

They **ran in softirq context**, which **allowed more concurrency**. They offered better concurrency and a simplifying guarantee: the same tasklet would never run on two CPUs simultaneously. They became the default tool for deferred work.

Yet they inherited a critical restriction: **tasklets could not sleep.**

This limitation forced a lie into system design.

A driver would queue a tasklet, only to discover inside it the need for an operation that might block. The tasklet would then have to queue another unit of work to a real thread.

What resulted was an architectural Rube Goldberg machine.

A single logical operation stretched across three contexts:

The interrupt, the tasklet, and a thread.

The tasklet was a testament to the gap between what developers needed (*“run this later, somewhere safe”*) and what the kernel offered.

The solution was adding complexity, not reducing it.

But what developers actually wanted was simple:

*“Run this later, in a context where sleeping is allowed.”*

The kernel needed something that felt like a tasklet but behaved like a thread.

## The First Unified Answer

By 2002, the pressure for a clean solution was immense.

Kernel developer **Ingo Molnar** provided it with a patch that introduced workqueues.

The API was beautifully simple.

A developer could wrap a function in a ***work_struct*** and queue it.

The kernel would execute it later, in a dedicated thread context where sleeping was allowed.

Workqueues offered two models: a single-threaded queue for ordered execution, or a per-CPU, multi-threaded queue where each CPU got a dedicated worker thread for that queue.

For its time, it was a victory.

It provided a unified, safe mechanism.

Drivers and subsystems could stop inventing their own deferral schemes. For years, it was the right tool.

But within its elegant design lay two seeds of future crisis.

## The Cracks in the Foundation

The **problems** emerged **with scale**.

By the late 2000s, servers shifted from 2 or 4 cores to 16, 32, and beyond. The 2002 workqueue design assumed this scale would never arrive.

Its flaws were structural:
1. **Wasteful Duplication:** Every subsystem (networking, storage, etc.) that created its own multi-threaded workqueue spawned a dedicated worker thread on every CPU. On a 32-core system with 10 such workqueues, that meant 320 kernel threads at boot—most perpetually idle, consuming memory and exhausting system resource limits.
2. **Artificial Bottlenecks:** The “one worker per CPU, per queue” model created hidden serialization. If a work item on CPU 4 blocked waiting for I/O, every other item in that queue on CPU 4 stalled behind it, even if the CPU itself was idle. Concurrency was a static promise, not a dynamic resource.

Developers faced a brutal trade-off: accept inefficient serialization, or spawn more wasteful queues.

Some subsystems, needing true performance, abandoned workqueues and built their own thread pools. The fragmentation was returning.

The 2002 solution was buckling under the weight of its own success.

## The Breakthrough: Concurrency as a Service

The necessary revolution came in 2010.

The system was re-architected from the ground up into the **Concurrency Managed Workqueue (CMWQ)**.

The core insight was a shift in ownership.

In the old model, a workqueue owned its threads. In the new model, workqueues own only a queue and a set of attributes (like priority or CPU affinity). The execution threads are moved into global, shared worker pools - a normal and a high-priority pool for each CPU.

When you queue work, it is placed into a shared pool.

Any idle worker from that pool can execute it. The unit of management shifts from the workqueue to the entire system. The thousand idle threads vanish, replaced by a just-enough fleet shared by all.

But this posed a brilliant new question: *how many workers are “just enough”?*

### The Rhythm of Need

The answer is concurrency management, the heartbeat of the modern workqueue.

Each pool doesn’t just hold workers; it listens to them.

It tracks which workers are running. When a worker executes a piece of code that blocks - on a lock, on I/O, on memory - the pool notices. If work is still pending, it immediately wakes or creates another worker. The goal is to keep the CPU busy, to never let a sleeping worker stall progress.

When the work drains, workers become idle. The pool lets them linger briefly, a cache for anticipated demand. If the quiet persists, they are terminated. The system breathes in and out, expanding concurrency under load and contracting to a minimal footprint at rest.

The developer who calls *schedule_work()* is abstracted from this symphony. They declare the what.

The kernel manages the how many and with what.

### When Nothing Becomes Something

The journey from bottom halves to managed workqueues reveals a deeper principle.

In a naive view, idleness is waste. The kernel’s evolution teaches that managed, efficient waiting is the foundation of scale.

The shared pool makes idleness cheap.

A parked thread costs almost nothing. By making idleness inexpensive, the system can afford ready capacity, eliminating the costly latency of creating threads for every work burst.

More profoundly, the concurrency manager makes waiting intelligent.

It uses the act of waiting (a worker blocking on I/O) as a signal to create more concurrency. It distinguishes this from a worker burning CPU, which suggests limiting parallel work to avoid overload. Waiting is no longer a static state; it is a dynamic signal in a feedback loop.

This pattern transcends kernels.

It is the same architecture in a web server’s thread pool, a database’s connection pool, or a cloud provider’s autoscaler.

The principles are identical: separate declaration from execution, share resources to amortize cost, and use system feedback to regulate flow.

---

Linux did not invent the thread pool.

It invented a specific, nuanced implementation that could survive the relentless, unforgiving context of a kernel - where there is no higher authority to catch a fall, and sleeping at the wrong moment means the death of the system.

The 2002 workqueues solved the problem of providing a safe place to wait. The 2010 workqueues solved the problem of providing a safe place to wait efficiently, dynamically, and at any scale.

Good system design is not the elimination of complexity.

It is the careful, deliberate placement of complexity into a managed layer, so that the interfaces above can be simple, robust, and focused entirely on their purpose.

The kernel cannot block.

So it learned to wait.

And in doing so, it learned how to run the world at full scale.
