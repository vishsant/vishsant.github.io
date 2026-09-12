+++
date = "2026-03-11"
draft = false
title = "Linux Interrupt Coalescing Explained: Why Your NIC Delays Packets on Purpose"
+++

If you run:

```bash
ethtool -c eth0
```

You’ll see parameters like:

```text
rx-usecs
rx-frames
tx-usecs
tx-frames
```

Most of us glance at them once and move on.

But these settings control one of the most important performance tradeoffs in modern networking.

They determine **how long your network card is allowed to hide packets from the CPU**.

That behavior is called **interrupt coalescing**.

And understanding it requires understanding something uncomfortable:

**A packet arriving is not free.**

## When the Interrupt Costs More Than the Packet

Early Ethernet networks operated at speeds where packet arrival rates were modest.

But when **Gigabit Ethernet** became common in the late 1990s, packet rates exploded.

A 1 Gbps link carrying minimum-sized Ethernet frames (64 bytes) produces ~1.48 million packets per second.

In a traditional interrupt-driven network stack, each packet generates:

```text
1 packet → 1 interrupt
```

That means: ~1.48 million interrupts per second

Every interrupt forces the CPU to:
1. Pause its current execution
2. Save registers and program state
3. Jump to the interrupt handler
4. Acknowledge the interrupt
5. Schedule packet processing
6. Restore execution

Even on modern processors, this sequence costs roughly 1000–3000 CPU cycles.

At a million interrupts per second, that overhead alone can consume **a large fraction of a CPU core**.

Not doing application work.

Not processing requests.

Just responding to the network card.

The system becomes **trapped in a loop of reacting**.

## What Actually Happens When a Packet Arrives

A modern NIC does not push packets directly into the CPU.

Instead, it uses **DMA (Direct Memory Access)** to copy incoming packets into a memory structure shared with the kernel.

This structure is typically a **descriptor ring buffer**.

```text
NIC  → writes packet into DMA ring
CPU  → reads packet from DMA ring
```

The NIC owns the producer side of the ring.

The kernel owns the consumer side.

When the NIC places a packet into the ring, it must notify the CPU that work is available.

This notification is the interrupt.

The interrupt handler itself does almost nothing.

Its job is simply to acknowledge the interrupt and schedule packet processing in the **softirq** context.

The actual packet processing path then:

```text
DMA ring → driver → network stack → socket buffers → application
```

Each interrupt therefore represents **a crossing between hardware and software execution contexts**.

And **crossings are expensive.**

## The First Attempt: Interrupt Throttling

The obvious solution seemed simple:

**If interrupts are too frequent, limit them.**

This led to **interrupt throttling**, where hardware or drivers restrict the maximum interrupt rate.

For example:

```text
max interrupts: 10,000/sec
```

But throttling introduced a new problem.

Packets arriving between interrupts accumulate in the ring buffer.

Instead of being processed immediately, they wait until the timer fires.

At high packet rates this creates bursts:

```text
Packets arrive continuously
CPU processes them in bursts
```

Latency becomes unpredictable.

A packet may sit in the ring for **tens or hundreds of microseconds** before processing begins.

The system had traded **CPU overload** for **latency spikes**.

Neither outcome was ideal.

## NAPI: A Hybrid Model

Linux addressed this with the introduction of **NAPI (New API)**.

NAPI combines two approaches:
- interrupt-driven networking
- polling

Instead of processing only one packet per interrupt, the kernel does this:
1. NIC raises an interrupt.
2. Kernel disables further interrupts from that NIC.
3. Kernel begins **polling the DMA ring.**
4. Packets are processed in a loop.
5. Once the ring is empty, interrupts are re-enabled.

This changes the cost model dramatically.

Instead of:

```text
1 packet → 1 interrupt
```

We get:

```text
1 interrupt → many packets
```

At high packet rates, the kernel stays in the polling loop, dramatically reducing interrupt frequency.

But this solution still assumes something:

That multiple packets are already present in the ring when the interrupt occurs.

At moderate traffic levels, that assumption often fails.

Packets arrive one at a time.

Each still triggers an interrupt.

To reduce interrupt rates further, the batching must move into the hardware.

## Interrupt Coalescing: Batching in the NIC

Interrupt coalescing teaches the NIC to **delay interrupts intentionally**.

Instead of firing an interrupt immediately, the NIC waits until one of two conditions is met:
- a certain number of packets arrive
- a timer expires

Example configuration:

```bash
# Interrupt when 50 microseconds pass or 8 packets arrive
rx-usecs 50
rx-frames 8
```

The NIC effectively hides the first packet for 50 microseconds.

That delay is intentional.

## Why This Works

The key insight is that interrupts have a fixed cost.

Processing one packet or ten packets inside the interrupt processing path costs nearly the same overhead.

Batching therefore improves efficiency dramatically.

CPU work drops by a factor of three.

Cache locality improves.

The kernel processes packets in tighter loops.

But this benefit comes at a cost.

## The Latency - Throughput Tradeoff

The first packet in a coalescing window waits.

Its latency increases by up to the coalescing timer.

For many systems, this delay is invisible.

For others, it is catastrophic.

Consider two workloads.

**High-Frequency Trading:**

Goal: minimize latency

Typical configuration:

```text
rx-usecs: 0
rx-frames: 1
```

Every packet triggers an interrupt immediately.

CPU overhead increases.

Latency is minimized.

**Streaming or CDN Workloads:**

Goal: maximize throughput

Typical configuration:

```text
rx-usecs: 100
rx-frames: 64
```

Packets arrive in large batches.

Interrupt rates drop dramatically.

CPU efficiency improves.

The extra latency is irrelevant to video playback.

Most systems fall somewhere in between.

Web servers, databases, and application backends typically use moderate coalescing settings.

Default distributions are tuned for balanced workloads, not extremes.

## Where the Same Tradeoff Appears Elsewhere

Once you recognize the pattern, it appears everywhere in systems design.

**Nagle's Algorithm**

Small TCP writes are delayed until enough data accumulates to form a full segment.

Purpose: reduce packet overhead.

Cost: increased latency.

**Disk Write Caching**

Operating systems buffer writes and flush them in batches.

Purpose: reduce disk seek overhead.

Cost: potential data loss without fsync.

**GPU Command Buffers**

Graphics drivers accumulate draw calls before submitting them to the GPU.

Purpose: reduce CPU–GPU synchronization overhead.

Cost: delayed execution.

Each case reflects the same principle.

Crossing boundaries between subsystems has a cost.

Batching amortizes that cost across multiple operations.

Latency increases.

Efficiency improves.

The correct balance depends entirely on the workload.

## Measuring Interrupt Behavior

Many systems are never profiled at the interrupt level.

Yet interrupt behavior directly affects network performance.

You can observe interrupt counts in real time:

```bash
watch -n1 'grep eth0 /proc/interrupts'
```

Coalescing parameters can be inspected with:

```bash
ethtool -c eth0
```

And tuned with:

```bash
ethtool -C eth0 rx-usecs 50 rx-frames 8
```

But tuning should always be guided by measurement.

Changes affect both CPU utilization and tail latency.

Optimizing one without observing the other risks moving the problem elsewhere.

---

Interrupt coalescing is not merely a network tuning parameter.

It illustrates a fundamental systems principle.

When communication between components has a fixed overhead, immediate action is often inefficient.

Batching improves efficiency by amortizing that overhead across multiple operations.

The cost is delay.

The benefit is throughput.

Modern systems constantly negotiate this tradeoff.

Sometimes the fastest system is the one that reacts immediately.

Sometimes the fastest system is the one that waits.

Interrupt coalescing is simply the network card making that decision on behalf of the CPU.
