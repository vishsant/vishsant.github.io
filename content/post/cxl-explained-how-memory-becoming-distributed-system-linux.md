+++
date = "2026-04-12"
draft = false
title = "CXL Explained: How Memory Is Becoming a Distributed System in Linux"
+++

There is a mental model most engineers carry about server memory. RAM is local. It lives in DIMM slots on the motherboard. The CPU owns it. Every byte is roughly the same distance away.

NUMA broke that model partially. On multi-socket machines, memory became non-uniform - local access was fast, remote access crossed an interconnect and cost double the latency. But all the memory was still physically inside the same chassis. Still attached to a CPU. Still part of one machine.

**CXL** breaks the model completely.

With **Compute Express Link**, memory can sit on a separate device connected over PCIe. It can sit behind a switch shared by multiple servers. It can be pooled across an entire rack and dynamically allocated to whichever host needs it most.

And the CPU accesses it the same way it accesses local DRAM: load and store instructions. No special API. No driver protocol. No serialization layer. Just a pointer - to memory that might physically live in another chassis.

This is not a minor hardware upgrade. It is a fundamental change in what "memory" means to an operating system.

## The Problem CXL Solves

Modern servers face a tension that keeps getting worse.

CPUs keep adding cores. Workloads keep demanding more memory. AI inference needs hundreds of gigabytes of model weights resident in RAM. In-memory databases grow faster than DIMM slot counts. Analytics engines process datasets that exceed what any single machine can hold.

The traditional answer was: buy a bigger machine. More sockets, more DIMM slots, more RAM per slot. But this scaling path has physical limits. DDR5 DIMMs max out at 256GB. A dual-socket server has maybe 32 DIMM slots. You hit a ceiling around 4-8 TB. After that, you're stuck.

Meanwhile, most servers waste memory. Studies consistently show average memory utilization at 50-60% across data center fleets. One machine has 512GB allocated to a workload using 200GB. Another machine is OOM-killing processes because it's 30GB short. The memory exists in the fleet - just in the wrong place.

**CXL** solves both problems at once. It **lets you attach more memory to a server than its DIMM slots allow**. And it **lets you pool that memory across machines so utilization doesn't depend on** which **server** a **workload **landed on.

## What CXL Actually Is

CXL is an **open interconnect standard built on PCIe's physical layer**. It runs over the same electrical interface as PCIe - the same connectors, cables, and signal encoding. But it adds three protocols on top that PCIe doesn't have:

**CXL.io** is essentially PCIe. Device discovery, configuration, DMA, interrupts - the standard I/O protocol that every PCIe device already speaks. This is how the system finds and initializes CXL devices.

**CXL.cache** lets a device coherently access the host CPU's memory. A GPU or accelerator can read and write host RAM and stay in sync with the CPU's caches. No software coordination needed - the hardware handles coherency.

**CXL.mem** is the revolutionary one. It lets the host CPU access memory on a device using regular load/store instructions, with hardware-managed cache coherency. The CPU can read an address backed by CXL-attached DRAM as naturally as it reads local memory.

These **three protocols** are **multiplexed onto** the **same physical link** **using flow-control units** called **flits**. The device negotiates which protocols it supports during initialization. The result is a single cable that can carry I/O traffic, cache-coherent device access, and memory-semantic traffic simultaneously.

```text
 [ CPU Cores ]
                 │
      ┌──────────┴──────────┐
      │                     │
[ Memory Controller ]   [ CXL Root Complex ]
        │                       │
        ▼                       ▼
 [ Local DDR5 ]        [ PCIe / CXL Link ]
   (~80 ns)                    │
                               ▼
                        [ CXL Memory ]
                         (~170–250 ns)
```

The CPU issues a load instruction. If the address maps to local DDR5, the memory controller serves it at ~80ns. If the address maps to CXL-attached memory, the request travels over the PCIe/CXL link, reaches the memory expander's controller, accesses its DRAM, and returns - at roughly 170-250ns.

No page fault. No driver involved. No system call. The MMU translates the virtual address to a physical address. If that physical address lives in CXL memory, the hardware routes the request over the CXL link transparently.

## How the Kernel Sees CXL Memory

Linux exposes CXL memory as a **CPU-less NUMA node**.

```console
$ numactl --hardware

available: 3 nodes (0-2)
node 0 cpus: 0-15
node 0 size: 128000 MB
node 1 cpus: 16-31
node 1 size: 128000 MB
node 2 cpus:
node 2 size: 512000 MB
```

This is a design choice with deep implications. The kernel already knows how to manage non-uniform memory. It already has distance tables, allocation policies, page migration, and tiering heuristics. **By presenting CXL memory as "just another NUMA node," the entire existing memory management infrastructure works without modification**.

Node 2 has no CPUs. It's a 512GB CXL memory expander. Distance 40 means it's slower than both local (10) and cross-socket (20) access.

The kernel's allocator sees three tiers of memory speed. Every allocation decision - from buddy allocator to page reclaim - consults this topology.

But here's the subtle difference from traditional NUMA. In a multi-socket NUMA system, every node has both CPUs and memory. The kernel balances workloads across nodes that can run code. CXL nodes have memory but no CPUs. Nothing ever "runs" on node 2. It's pure capacity - the cold storage tier of the memory hierarchy.

This creates a new pattern: **the kernel doesn't schedule tasks to CXL nodes. It migrates pages there.**

## Memory Tiering: The Kernel Decides Where Your Pages Live

With CXL memory in the system, the kernel faces a continuous optimization problem: *which pages should live in fast local DRAM, and which should live in slower CXL memory?*

The answer depends on access frequency.

Hot pages - frequently accessed - should live in local DRAM for minimum latency. Cold pages - rarely touched - can live in CXL memory without meaningful performance impact. The challenge is that access patterns change over time.

Linux uses the **DAMON (Data Access MONitor) subsystem** to solve this. DAMON samples memory access patterns at low overhead (typically 1-5% CPU) and classifies pages into hot and cold regions. Its companion, **DAMOS (DAMON-based Operation Schemes)**, defines what to do with each classification.

The tiering policy works in two directions:

**Demotion: **When local DRAM is under pressure, DAMOS identifies cold pages on the fast node and migrates them to the CXL node. This is DAMOS_MIGRATE_COLD. The page table entry is updated. The TLB is flushed. The next access to that page follows the CXL path instead of the local memory controller.

**Promotion:** When DAMON detects that a page on the CXL node is being accessed frequently, DAMOS migrates it back to local DRAM. This is DAMOS_MIGRATE_HOT. The hot page returns to the fast tier where it belongs.

The result: local DRAM becomes a hardware-managed hot cache. CXL memory becomes the capacity tier. The kernel continuously rebalances based on observed behavior.

This is the same principle behind CPU caches, swap, and page cache - but operating at a new level of the hierarchy. The kernel is now a memory traffic controller managing multiple tiers of physically different memory.

## Memory Pooling: When Memory Leaves the Machine

Type 3 memory expanders are interesting. Memory pooling is transformative.

```text
 Multiple Hosts
  (0, 1, 2, 3 servers)
          │
          ▼
     [ CXL Switch ]
          │
   ┌──────┼──────┐
   ▼      ▼      ▼
[256GB] [256GB] [256GB]
  DRAM    DRAM    DRAM
          │
          ▼
   768 GB Shared Pool
 (allocated dynamically)
```

**CXL 2.0** introduced switching. A **CXL switch sits between multiple hosts and multiple memory** devices, **dynamically assigning memory regions to servers based on demand**.

This is not shared memory in the programming sense. Each host gets exclusive access to its assigned regions. There's no concurrent access to the same bytes from multiple hosts (unless explicitly configured in CXL 3.0's sharing mode). It's more like memory that can be dynamically provisioned - virtual DIMM slots that the fabric manager assigns on demand.

**CXL 3.0** goes further. It supports **multi-level switch topologies - leaf/spine fabrics** connecting hundreds of endpoints across racks. And it introduces** true memory sharing: multiple hosts accessing the same physical memory region with hardware coherency**. This opens the door to disaggregated shared-memory computing at rack scale.

## The Latency Question

CXL memory is slower than local DRAM. This is not a flaw - it's a physics constraint.

Local DDR5 latency: ~80ns. CXL memory on the same socket: ~170-250ns. That's roughly 2-3x slower.

For comparison, remote NUMA memory (cross-socket DDR5) is about ~150-200ns. CXL memory is in the same ballpark, sometimes slightly slower.

```text
Memory Access Latency Hierarchy

  L1 Cache      ~1ns
  L2 Cache      ~4ns
  L3 Cache      ~12ns
  Local DDR5    ~80ns
  Remote NUMA   ~150ns
  CXL Memory    ~200ns
  CXL Pooled    ~300ns+
  NVMe SSD      ~10,000ns
```

CXL sits between DRAM and storage - closer to remote NUMA than to disk. This positioning is deliberate. It's fast enough for memory semantics (load/store) but adds enough latency that intelligent tiering matters.

## CXL and Coherency: Why This Isn't Just PCIe

A natural question: *why not just use regular PCIe to attach memory?*

PCIe supports DMA. A device can read and write host memory. A host can MMIO into device memory. Why do we need a new protocol?

The answer is **cache coherency**.

When a CPU reads a memory address, the result lives in the CPU's cache hierarchy. If another entity (a device, another CPU) modifies that address, the cached copy becomes stale. On a multi-socket system, the CPUs coordinate through a coherency protocol - if CPU 1 modifies an address cached by CPU 0, CPU 0's cache line gets invalidated.

**PCIe has no such mechanism**. If you MMIO-read a PCIe device's memory, the CPU might cache the result. If the device changes the underlying data, the CPU's cache has stale data. Software must manually manage coherency - flushing caches, inserting memory barriers, coordinating access.

**CXL.mem solves this at the hardware level**. The CPU's cache agent tracks CXL memory the same way it tracks local DRAM. If the CPU caches a line from CXL memory, and that line needs to be invalidated (because another host modified it in a sharing scenario), the CXL protocol handles it. No software intervention.

This is what makes load/store semantics real. **Without coherency, you'd need a driver, a protocol, and explicit synchronization. With CXL's coherency, a pointer to CXL memory behaves like a pointer to local memory**. The hardware keeps everything consistent.

## The Philosophical Shift

There is a **pattern in systems engineering**. Every decade, **something we assumed was local becomes remote**.

In the 1990s, compute went remote. Processes stopped running on one machine and started running across clusters. We called it distributed computing and built RPC, message passing, and consensus protocols.

In the 2000s, storage went remote. Disks stopped living in the server chassis and moved to SANs, then network-attached storage, then object stores. We called it disaggregated storage and built protocols for access, caching, and consistency.

In the 2020s, memory is going remote.

CXL is doing to memory what Ethernet did to storage and what RPC did to compute. It's taking something that was physically bound to a machine and making it accessible over a fabric. And just like network storage needed new caching strategies and consistency models, network memory needs new tiering policies, coherency protocols, and allocation strategies.

The Linux kernel is already adapting. CXL memory appears as NUMA nodes. DAMON monitors access patterns. Page migration moves data between tiers. The memory allocator consults topology maps before every decision.

That's the deepest tradition in systems engineering: making the complicated invisible so that the simple remains true.
