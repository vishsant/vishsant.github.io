+++
date = "2026-07-05"
draft = false
title = "The Hidden Cost of a \"Free\" CPU Core: Cache-Aware Scheduling in Linux"
+++

Every engineer learns the same rule of thumb about scheduling: keep the cores busy. If a thread is runnable and a CPU is sitting idle, put the thread there. Idle time is waste, and a good scheduler spreads work across every core you own so nothing sits on a queue while silicon goes unused. It is the first thing you picture when you hear "load balancing," and for decades the Linux scheduler has done roughly that, aggressively, by default.

The rule has a hidden assumption baked into the word "idle." It treats an unused core as interchangeable with any other, a blank worker ready to take your thread at no cost. But a core is not just an execution unit. It is an execution unit wrapped in its own private, fast memory, and that memory remembers whatever ran on it last. Move your thread to a different core and the execution unit comes free. The memory does not come with it.

## The Warm Core

Modern CPUs put a small amount of very fast cache right next to each core. On the AMD laptop I am writing this on, each physical core has its own 32 KB L1 data cache and its own 512 KB L2, and they are the reason your code runs at full speed: once a thread has touched its working set, those bytes live a few nanoseconds away instead of a few hundred. A thread that has been running on a core for a while has effectively built a nest there. Its hot data is sitting in that core's L1 and L2, already paid for.

Those caches belong to the core, wired into the silicon beside it. They are not part of the thread, and they do not travel with it. When the scheduler migrates your thread to a different core, the thread arrives with its registers and nothing else. The new core's L1 and L2 are full of some other thread's data, or nothing useful. Every line your thread touches is a miss, served from the shared last-level cache or from main memory, until it slowly rebuilds the nest it already had on the core it left.

That rebuild is the cost nobody puts on the scope. The destination core looked idle, so moving there looked free. What happened is that you threw away a warmed cache and agreed to pay, in cache misses, to warm a new one.

## The Penalty

The idea so far has been simple: a thread that stays on one core keeps its data warm in that core's caches. Move it somewhere else, and those caches are gone. That sounds plausible enough. The question is how expensive the rebuild is.

To answer that, we need a workload whose speed depends almost entirely on one thing: whether its data is already sitting in the cache. If the data is warm, it should run quickly. If the data has to be fetched back into the cache after every move, it should slow down. The difference between those two tells us the price of abandoning a warm core.

I wrote a tiny benchmark that does exactly one thing: it repeatedly walks through a fixed-size buffer and reads one byte from every cache line. The buffer size is the trick. At 256 KB, it is much larger than the core's 32 KB L1 cache but comfortably fits inside its 512 KB L2 cache.

```c
/* bigger than L1 (32 KB), fits in L2 (512 KB) */
#define WS (256 * 1024)

/* one timed pass: sweep the whole buffer, read-only, a few times over */
for (int r = 0; r < 8; r++)
    for (size_t i = 0; i < WS; i += 64)   /* one 64-byte cache line */
        sum += buf[i];
```

Once a core has run this loop, the entire buffer lives in its L2 cache. Run the loop again on the same core, and most reads come straight from that warm cache. Run it on a different core, and that new core starts with an empty L2. It has to fetch the entire working set back into its own cache before it can enjoy those fast accesses.

The loop never changes. The only thing I vary between runs is which CPU executes it, and I control that with one system call: *sched_setaffinity()*. It lets me tell Linux exactly which core the thread is allowed to run on.

```c
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(target_cpu, &set);
sched_setaffinity(0, sizeof(set), &set);
```

From there I tested three situations. In pinned, the thread always runs on core 0, so its cache stays warm. In migrate, the thread moves to a different idle core before every pass, an artificial worst case that starts every pass cold and that no real scheduler would ever inflict. In churn, I still call *sched_setaffinity()* before every pass but pin the thread to the core it is already on, so the kernel runs the same syscall while the thread never moves.

That third case matters. Someone could reasonably argue the slowdown comes from calling *sched_setaffinity()* over and over, not from moving between cores. Churn separates those two costs.

Start with the best case.

```console
$ ./idlecore pinned
pinned    20570 ns/iter
```

Each iteration finishes in about 20.6 microseconds, because the thread keeps finding its data exactly where it left it. Now force a migration before every pass.

```console
$ ./idlecore migrate
migrate   56266 ns/iter
```

Same instructions, same data, same machine. The only difference is that every iteration starts on a fresh idle core, and the runtime nearly triples to 56 microseconds. How much of that comes from the migration itself, and how much is just the repeated syscall? Churn answers it.

```console
$ ./idlecore churn
churn     22317 ns/iter
```

Calling *sched_setaffinity()* every iteration moves the runtime from 20.6 to 22.3 microseconds, under two microseconds of overhead. The remaining thirty-four appear only when the thread leaves its core. That is the real penalty, and since I forced the hop every pass, it is the cost of a single migration in isolation, the very cost Linux normally works to avoid.

Most of that penalty goes to rebuilding the core's warm state: refilling its L1 and L2 caches, repopulating its address-translation cache (the TLB), and paying the scheduler's own price for moving the thread between CPUs.

An idle core advertised itself as free. It was because the CPU came at no extra charge. The cache did not.

## The Concession

The scheduler was never blind to this cost at the level of a single core. Since the Completely Fair Scheduler arrived in 2007 it has protected the warm core in front of it, declining to migrate a task that just ran unless the imbalance justifies it. What it could not see was structure past that core: it treated every cache beyond the one you were on as a single undifferentiated pool. Linux 7.2 teaches it to see the pool's shape. A long-running effort led by Tim Chen and Chen Yu at Intel landed a feature called Cache Aware Scheduling, gated behind CONFIG_SCHED_CACHE in the fair scheduler. Its cover letter states the goal plainly: "to aggregate tasks sharing data to the same LLC cache domain, thereby reducing cache bouncing and cache misses."

The mechanism is a direct proof that idle is not free. Each process now gets a "preferred" last-level cache, the domain where its threads already have data warm. During load balancing the scheduler tries to pull that process's threads toward their preferred cache rather than fling them at whichever core happens to be free, as long as that cache is not overloaded. It even carries a new migration type inside the balancer for exactly this "move it toward its warm cache" decision.

---

This doesn't make "keep the cores busy" wrong. It makes it incomplete. A scheduler has two goods that pull against each other, and it cannot maximize both at once. Spreading threads across idle cores maximizes how much CPU you can bring to bear this instant. Packing threads near their warm caches maximizes how fast each thread runs.

Every migration is a bet about what matters more: the core that is free this instant, or the cache that is warm this instant. For most of Linux's history that bet undervalued every cache beyond the core right in front of the scheduler. Linux 7.2 is the release where the kernel started pricing both sides of it.
