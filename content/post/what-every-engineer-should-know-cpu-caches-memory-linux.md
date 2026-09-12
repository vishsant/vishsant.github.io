+++
date = "2026-03-08"
draft = false
title = "What Every Engineer Should Know About CPU Caches, Memory Ordering, and the Linux Memory Model"
+++

A certain class of bugs has nothing to do with your code at all.

They live in the hardware, in a layer most engineers never study because it is supposed to be invisible. It is the CPU cache, and at the level of multiple cores working in parallel, it has a secret life your code cannot see.

The goal of this article is not to teach you cache architecture.

That knowledge fades.

The goal is to give you a **mental model that stays** - an understanding of why a CPU, designed for performance, occasionally tells your program something that is no longer true.

By the end, you will not read concurrent code the same way.

And that change in how you read code will make you a better engineer.

## The Copy That Lies

In the early 2000s, engineers at Sun Microsystems spent weeks chasing a strange bug in a trading system.

The program was simple.

One thread produced prices. Another thread consumed them.

The producer updated a shared integer. The consumer read it.

No locks were used because the engineers believed something reasonable:

**A single integer write is atomic.**

And they were correct.

On a single-core machine, the program worked perfectly.

But on a dual-processor system, the consumer occasionally read stale prices.

Not always. Not reliably.

Sometimes the value it read was milliseconds old.

The program didn't crash. Nothing obviously broke.

It simply produced the wrong answer.

The bug was not in the code.

The **bug was in an assumption about memory.**

## The Model Most Programmers Have

Most programmers imagine memory as a single place.

```text
A CPU reads address 0x7fff1230.
That address contains a number.
Another CPU writes a new number there.
The next reader sees the new value.
```

A CPU reads address 0x7fff1230.

That address contains a number.

Another CPU writes a new number there.

The next reader sees the new value.

Simple. Logical. Intuitive.

This model works so well for single-threaded programs that we rarely question it.

But on modern processors, this model is wrong.

Memory is not one place.

## The Hierarchy Beneath Your Code

A modern CPU almost never reads directly from RAM.

Instead, it reads from **cache**.

A typical system looks roughly like this:

| Level | Latency |
|---|---:|
| L1 Cache | ~1–4 cycles |
| L2 Cache | ~10–20 cycles |
| L3 Cache | ~30–80 cycles |
| RAM | ~200+ cycles |

At 3 GHz:
- L1 access ≈ **1 nanosecond**
- RAM access ≈ **100 nanoseconds**

That difference is enormous.

If every load went to RAM, modern CPUs would spend most of their time waiting.

Caches exist to prevent that.

## The Consequence of Speed

Caches make processors fast by keeping **local copies of data**.

When a CPU loads a value from memory, it does not fetch a single byte.

```bash
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

It fetches a **cache line**, typically **64 bytes**.

That entire chunk is stored in the CPU’s cache.

Now imagine a multicore system.

Each core has its own cache.

And each cache can hold its **own copy of** the **same cache line**.

Two cores can now hold **two versions of the same data**.

## The Moment Truth Splits

Imagine this situation.

Core 0 and Core 1 both load a cache line containing a variable x.

Both cores now have identical copies.

Then Core 0 writes:

```c
x = 1;
```

Core 1 still has its copy.

Its copy still says:

Two cores.

Two truths.

Both internally consistent.

One of them is now wrong.

## RAM vs Belief

Think of RAM as **the record of the world**.

Caches are **beliefs about the world** held by individual cores.

And beliefs can become outdated.

If nothing coordinated those beliefs, multicore computers would collapse into chaos.

Different cores would compute using different realities.

Something must constantly reconcile these differences.

That system is called **cache coherence**.

## The Cache Coherence Protocol

Most processors implement a protocol called **MESI**.

Each cache line exists in one of four states.

| State | Meaning |
|---|---|
| Modified | This core has the only changed copy |
| Exclusive | This core has the only copy, matching RAM |
| Shared | Multiple cores have identical copies |
| Invalid | This copy is stale |

These states allow the processor to track **who owns the truth**.

## When a Core Wants to Write

Suppose a cache line is shared.

Multiple cores hold identical copies.

Now one core wants to modify it.

It cannot simply write.

If it did, other cores would hold incorrect copies.

So the core sends a message across the system:

“I am taking ownership of this line.”

All other cores invalidate their copies.

Only then can the writer proceed.

The writer transitions to the **Modified** state.

Now it holds the only correct version.

## False Sharing: A Silent Performance Killer

Cache lines are 64 bytes.

That means unrelated variables can share the same line.

Example:

```c
struct {
    int x;
    int y;
};
```

If x and y are used by different threads on different cores, something strange happens.

Every write invalidates the entire line.

The **cache line** **bounces between cores**.

Even though the variables are logically independent.

This phenomenon is called **false sharing**.

Nothing is wrong with the data.

But performance collapses.

Because the hardware is faithfully enforcing coherence.

## The Waiting Room for Writes

Now consider what happens when a core wants to write but cannot yet obtain ownership of the cache line.

Waiting hundreds of cycles would stall the processor.

So CPUs use something called a **store buffer**.

Instead of waiting, the write enters the buffer.

The CPU continues executing instructions.

Later, when the cache line becomes available, the write is committed.

This optimization dramatically improves performance.

But it introduces a surprising consequence.

## Two Cores, Two Realities

Imagine the following timeline.

```text
Time 0: Core 0 writes x = 1; the store enters its buffer.
        Core 0 reads x again and sees 1.
Time 1: Core 1 reads x and may still see 0.
Time 2: The store buffer drains; everyone sees 1.
```

Between Time 0 and Time 2:

Two cores read the same address.

And observe different values.

Both observations are valid.

Both processors behaved correctly.

The illusion comes from **timing**.

## The CPU Reads Ahead

Modern CPUs also execute instructions **out of order**.

If the processor sees a load instruction whose dependencies appear satisfied, it may execute it early.

This is called **speculative execution**.

It improves performance dramatically.

But it also means loads may happen before you think they do.

## The Classic Concurrency Trap

Consider this simple producer - consumer pattern:

```c
// Producer
data = 42;
ready = 1;

// Consumer
while (ready == 0) {}
use(data);
```

Consumer:

The programmer’s intent is obvious.

First write the data.

Then signal readiness.

But the hardware does not understand intent.

It understands instructions.

The CPU may reorder loads internally.

The consumer may observe ready == 1 but still read stale data.

This kind of bug is rare.

Which makes it dangerous.

Because it survives testing.

## The Memory Model

Every CPU architecture defines a **memory model**.

The memory model describes what kinds of reordering are allowed.

For example:

| Architecture | Memory model |
|---|---|
| x86 | Total Store Order (relatively strong) |
| ARM | Weak / relaxed |
| POWER | Very weak |

Stronger models are easier to reason about.

Weaker models give hardware more freedom to optimize.

## The Tool That Restores Order

To control reordering, programmers use **memory barriers**.

A memory barrier tells the CPU:

**All memory operations before this point must complete before any after it begin.**

Example in C11:

```c
atomic_thread_fence(memory_order_seq_cst);
```

Barriers do not change logic.

They restrict the processor’s freedom to reorder operations.

They restore predictability.

But they also cost performance.

Which is why systems like the Linux kernel place them with surgical precision.

## Reading Concurrent Code Differently

Once you understand this, concurrency looks different.

When you read shared memory code, you begin asking:
- Which core owns this cache line?
- When does the write become visible?
- Could a store buffer delay it?
- Could a load execute earlier than intended?

These questions are not theoretical.

They are the foundation of correct concurrent systems.

## The Mental Model

Think of RAM as **the official record of reality**.

Each CPU core maintains its own **temporary beliefs** about that reality.

Those beliefs are constantly reconciled through coherence protocols.

But reconciliation takes time.

During that time, different cores may hold **different versions of the truth**.

And that small window - between what used to be true and what is true now - is where the hardest concurrency bugs live.

The lesson is simple:

**never rely on timing, and never rely on intuition about memory.**

The job of you as a systems engineer is not to eliminate that window.

The hardware cannot afford that.

The job is to **know exactly when the truth must be enforced and to demand it with synchronization and memory barriers.**
