+++
date = "2026-02-15"
draft = false
title = "NUMA in Linux: When Memory Becomes a Distributed System"
+++

We think of memory as a single, flat space.

A program asked for an address. The system returned one. Every byte was equally reachable. Every access roughly equal in cost.

This mental model served us well for decades.

It matched the hardware. A processor connected to a single bank of RAM. Every byte took the same time to reach. We called this Uniform Memory Access, and we built our intuitions around it.

The address mattered. The location did not.

That assumption shaped everything: algorithms, data structures, performance models.

Then hardware changed.

And memory started lying to us.

## The Assumption We Stopped Questioning

Memory feels uniform because it used to be.

One processor. One memory controller. One bus.

Every byte traveled the same path.

Latency varied with load and caching, but not geography.

Then multi-socket systems appeared.

Each socket needed bandwidth. The single memory controller became a bottleneck.

So hardware designers split it.

Each processor got its own memory controller. Its own local RAM. Its own island of fast access.

Remote memory was still reachable.

But no longer equal.

Local access: ~80ns, Remote access: ~150–200ns

The ratio varies. The pattern does not.

Uniform Memory Access (UMA) became **Non-Uniform Memory Access (NUMA)**.

The interface did not change.

The physics did.

So Linux had to adapt.

## A Server With Multiple Memory Islands

A four-socket server is not one machine.

It is four tightly coupled machines.

Each socket and its RAM form a **node**.

```text
Node 0              Node 1
┌──────┐            ┌──────┐
│ CPU  │            │ CPU  │
│ 64GB │            │ 64GB │
└──┬───┘            └──┬───┘
   └────── interconnect ─┘
┌──────┐            ┌──────┐
│ CPU  │            │ CPU  │
│ 64GB │            │ 64GB │
└──────┘            └──────┘
Node 2              Node 3
```

Accessing local memory:

CPU → Local Memory Controller → RAM

Accessing remote memory:

CPU → Interconnect → Remote CPU → Remote Memory Controller → RAM

The second path:
- doubles latency
- reduces bandwidth
- consumes shared interconnect capacity

That **interconnect** is finite.

And shared.

At scale, your server behaves like a small distributed system with a constrained network between nodes.

Yet the kernel must maintain the abstraction. Programs written for uniform memory must still work. A pointer must still point to valid memory, regardless of where that memory lives.

This is the core tension in NUMA systems.

The **hardware broke uniformity. The kernel had to manage the pieces. But the interface promises simplicity.**

Something has to give.

## The Topology Linux Sees

Linux does not see “256 GB of RAM.”

It sees:
- Node 0
- Node 1
- Node 2
- Node 3

And a **distance matrix** describing how far each node sits from every other node.

You can inspect it:

```bash
cat /sys/devices/system/node/node*/distance
```

A simple four-socket system might show:

The distances are not in nanoseconds.

They are abstract.

A distance of 10 means local. A distance of 20 means one hop. A distance of 40 means two hops.

The numbers encode topology, not latency.

Every memory allocation decision consults this map.

The abstraction your process sees:

```text
[Virtual Address] → [Physical Memory]
```

The topology the kernel manages:

```text
[Virtual Address] → [Node 0 RAM] or [Node 1 RAM]
                       local          remote
                       ~80ns          ~180ns
``` the kernel manages:

When a process asks for memory, the kernel prefers the node where the process runs. If that node has free memory, the allocation stays local. If not, the kernel looks at the distance table and picks the nearest node with space.

But *preference is not guarantee*.

## Allocation and the First Lie

*malloc()* does not allocate physical memory.

It allocates *virtual* memory.

The physical page is assigned on first access.

This is the **first-touch policy**.

Whichever CPU touches the page first determines which NUMA node the page belongs to.

This works beautifully for single-threaded programs.

It breaks subtly for parallel ones.

Example:

```c
char *buf = malloc(SIZE);
#pragma omp parallel for
for (int i = 0; i < SIZE; i++)
    buf[i] = 0;
```

The loop iterations are divided among multiple threads (typically one per CPU core). Each thread initializes a different chunk of the buffer. Because of first-touch policy, the physical page for each chunk is allocated on the NUMA node where the thread that first touches it runs. Because of this buffer's pages are spread across all nodes.

Later, if one thread processes the whole buffer sequentially, half its accesses are remote.

This is a classic way to demonstrate how parallel initialization can unintentionally distribute memory, leading to remote accesses later if a single thread processes the whole buffer.

Nothing is wrong.

The pointer is valid.

The abstraction holds.

The locality is gone.

This is the first illusion NUMA introduces:

*Allocation does not determine placement. Access does.*

## Migration: When Compute Moves but Memory Doesn’t

The scheduler does not care about memory.

It cares about CPU load.

If one core is busy and another is idle, the scheduler moves a task.

```text
Time 0: Task on Node 0 → local memory
Time 1: Task migrates to Node 1 → Node 0 pages are remote
```

Now: The task might have been running on node zero for an hour. It might have allocated gigabytes of local memory. The scheduler does not check. It sees an imbalance, and it acts.

Now:
- CPU is on Node 1
- Pages are on Node 0
- Every access crosses the interconnect
- Latency doubles.
- Bandwidth drops.

The scheduler uses heuristics to reduce cross-node movement.

It groups CPUs by NUMA node. It penalizes migrations.

But fairness and load balancing sometimes win over locality.

Linux can migrate memory pages.

But migration is expensive:
- Copy page
- Update page tables
- Invalidate TLBs

It must be conservative.

**Too aggressive → thrashing. **

**Too slow → remote penalties persist.**

This is not a bug.

It is a tradeoff between stability and locality.

## Bandwidth: The Cost Nobody Measures

Latency is not the only cost.

Bandwidth matters too.

A single access to remote memory takes twice as long. But a flood of accesses can saturate the interconnect.

Each link has a maximum throughput. On modern systems, interconnects provide around 40 gigabytes per second per direction.

That sounds large.

A single core can generate 10–20 GB/s during streaming workloads. Four cores can saturate a link. Eight cores can oversubscribe it by a factor of two.

```text
Interconnect capacity: 40 GB/s
Demand: 4 × 15 GB/s = 60 GB/s
Delivered: 40 GB/s; latency: 3–5× higher
```

When saturation occurs:
- Latency spikes
- Throughput collapses
- Performance becomes noisy

Even if your process uses mostly local memory, others may not.

This is the hidden cost of NUMA.

NUMA introduces ***shared network contention*** inside a single machine.

There is no quality-of-service on the memory fabric.

First come, first served.

This is where the distributed analogy becomes literal.

Nodes compete for network bandwidth.

And the kernel cannot enforce fairness at that layer.

### When the Abstraction Leaks

Linux could force programs to manage NUMA explicitly.

It could require applications to specify node preferences. It could reject allocations that cross nodes. It could force programs to manage locality explicitly.

It does not.

Memory remains memory.

Pointers are pointers.

This is a deliberate choice. Linux prioritizes compatibility over performance.

Most programs do not need NUMA awareness.

But some do.

High-performance databases. In-memory analytics engines. Real-time systems.

For them, memory placement becomes architecture.

They can use `mbind()` to pin memory to specific nodes.

```bash
numactl --cpunodebind=0 --membind=0 ./application
```

They can use They can use *numa_alloc_onnode()* to allocate explicitly. They can use *migrate_pages()* to move memory manually. They can set CPU affinity to control where threads run.
- *numa_hit:* allocations satisfied from the preferred node.
- *numa_miss:* allocations that fell back to a remote node (first-touch betrayal).
- numa_foreign: allocations intended for this node but served elsewhere.
- *local_node:* memory allocated on the node where the process ran.
- *other_node:* memory allocated on a different node.

If *numa_miss* is high, your workload is leaking across nodes. If *other_node* dwarfs local_node, your process is likely running on the wrong socket.

NUMA exposes a fundamental truth.

*Hardware topology leaks through abstractions at scale.*

The larger the machine, the more the illusion of uniformity breaks.

## The Attack Surface Hidden in the Interconnect

NUMA has a security boundary problem - one that literally no tool will show you.

On a multi-tenant system (cloud instance, shared server), your process and a malicious process could run on different sockets. They are isolated by VMs, cgroups, namespaces.

But they share the interconnect.

A malicious process on Node 1 can flood the link to Node 0. It's not attacking your memory. It's attacking the path to your memory.

Result: Your latency spikes. Your throughput collapses. Your performance becomes unpredictable. And no security tool will flag it, because the attacker never touched your memory.

This is a *denial-of-service attack using NUMA topology*.

There is no fix. There is no cgroup for interconnect bandwidth. The hardware has no QoS.

The abstraction that made NUMA invisible also made this attack invisible.

High memory latency is not always a memory problem. Sometimes it is a network problem inside your machine, caused by someone else.

When you see *perf* report 200ns access times, don't just blame the memory controller. Open *numastat*. Look for cross-node traffic. Look for the process flooding the interconnect.

The tooling still calls it memory.

But the physics call it networking.

---

Memory in Linux is not a pool.

It is a set of geographically distributed regions connected by finite bandwidth links.
- Locality determines latency.
- Bandwidth determines scalability.
- Migration determines stability.

Linux maintains the illusion of uniform memory.

But underneath, it manages topology.

If performance degrades as you scale cores, if latency doubles unexpectedly, if throughput collapses under load, you may not have a compute problem.

You may have a topology problem.

Drop in comments if you've blamed 'Linux' for NUMA latency issues, just curious to know the real world pain Linux absorbed without saying a word out loud.
