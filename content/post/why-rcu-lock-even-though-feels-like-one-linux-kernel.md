+++
date = "2025-12-21"
draft = false
title = "Why RCU Is Not a Lock, Even Though It Feels Like One in Linux Kernel"
+++

You are in a large library.

People are reading quietly at tables.

Every now and then, a librarian needs to replace an old reference book with a new edition.

The traditional way is to ask everyone to stop reading, clear the table, swap the book, and then let reading resume. It works, but it put everything to a halt.

There is another way.

The librarian could quietly place the new book on a new shelf, update the catalog to point to the new location, and only later, when every reader has finished with the old copy, remove it.

Reading never stops.

This is the essence of **RCU - Read, Copy, Update **in the Linux kernel.

On the surface, it seems to solve the same problem as a lock: coordinating access to shared data. It feels like a lock. You “protect” data with it. You talk about “critical sections.”

But calling RCU a lock is like calling a river a road because both can get you somewhere.

It misses the fundamental nature of the thing.

### A back story of RCU

In the early days of computing, synchronization was often a story of mutual exclusion.

If two threads needed to touch the same piece of memory, one would acquire a lock and the other would wait. This model is intuitive. It mirrors how we handle scarce physical resources: one person uses the bathroom, others queue.

Locks became the default tool for shared data.

But locks have a cost.

The act of acquiring and releasing them requires atomic instructions, memory barriers, and cache-line bouncing across CPUs. In read-heavy workloads -situations where data is read thousands of times for every time it’s written - this cost becomes a tax on the common case. You are slowing down every reader to protect against a rare writer.

The Linux kernel team faced this in the development cycle.

They needed a way to protect data structures like the directory cache, where lookups (reads) vastly outnumbered updates (writes). The goal was not just to make things faster, but to make waiting disappear for readers altogether.

RCU was the answer.

It changed the question from *“How do we make readers wait safely?*” to ***“****What if readers didn’t have to wait at all?****”***

Think of the library problem again.

With RCU, the librarian makes a copy of the changed section, updates the master index, and lets the old version live until the last reader is done. No interruption.

This is the **first clue** that RCU is not a lock: **it does not enforce mutual exclusion between readers and updaters**.

Look at the simplest RCU-protected data structure: a global pointer.

```c
struct foo { int a; char b; long c; };
DEFINE_SPINLOCK(foo_mutex);
struct foo __rcu *gbl_foo;

int foo_get_a(void)
{
    int retval;
    rcu_read_lock();
    retval = rcu_dereference(gbl_foo)->a;
    rcu_read_unlock();
    return retval;
}
```

A reader accesses it like this:

Notice what’s missing: no lock acquisition, no atomic operations in the read path. On most architectures, *rcu_read_lock()* and *rcu_read_unlock()* compile to nothing. They’re markers, not barriers.

An updater works differently:

```c
void foo_update_a(int new_a)
{
    struct foo *new_fp, *old_fp;
    new_fp = kmalloc(sizeof(*new_fp), GFP_KERNEL);
    spin_lock(&foo_mutex);
    old_fp = rcu_dereference_protected(gbl_foo,
                                       lockdep_is_held(&foo_mutex));
    *new_fp = *old_fp;
    new_fp->a = new_a;
    rcu_assign_pointer(gbl_foo, new_fp);
    spin_unlock(&foo_mutex);
    synchronize_rcu();
    kfree(old_fp);
}
```

The updater makes a copy, modifies it, swings the pointer, then waits before freeing the old copy. Readers during the transition see either the old or new version - never a torn, inconsistent state.

The initial reaction to this idea is often disbelief. It feels unsafe. It feels like a clever hack. But at its heart, it’s a different **philosophy**: ***instead of blocking concurrency, manage it through versioning and patience.***

### The Weight of Waiting

When you hold a lock, you are holding a shared resource. You are also holding up everyone else who needs it.

This creates latency, contention, and a vulnerability to failures what if the thread holding the lock crashes or sleeps?

In lock-based design, readers and writers are adversaries.

They compete for the same token.

RCU removes this competition by removing the shared token.

There is no lock to hold.

For readers, entering an RCU-protected region is as simple as marking a point in time *rcu_read_lock() *and exiting it *rcu_read_unlock()*.

The real work, the “waiting,” is shifted.

It moves from the many (readers) to the few (updaters). After an updater removes a piece of data, it does not reclaim memory immediately. It calls *synchronize_rcu()* and waits - not for a lock, but for a “grace period” to pass.

A grace period is defined as the time during which all pre-existing RCU readers must have completed. In practice, this means every CPU in the system has undergone a context switch. The kernel implements this efficiently through per-CPU counters and batching.

Consider the alternative: *call_rcu()*. This asynchronous variant registers a callback to be invoked after the grace period:

```c
call_rcu(&old_fp->rcu, foo_reclaim);

void foo_reclaim(struct rcu_head *rp)
{
    struct foo *fp = container_of(rp, struct foo, rcu);
    kfree(fp);
}
```

Now the updater returns immediately. The reclamation happens later, in a soft interrupt context. This is crucial for real-time systems or network stacks where blocking is unacceptable.

The waiting is no longer a busy, contentious spin. It is a deferred acknowledgment that time will solve the problem. The kernel can batch these waits, spreading the cost across many updates.

### Separation of Concerns in Time

Good software design often involves separation of concerns. UI code is separated from backend logic. Network layers are separated from application layers. RCU introduces a separation of concerns in time.

An update is split into two distinct phases: **removal** and **reclamation**. Removal is instantaneous and safe to do concurrently with readers. Reclamation happens later, after a grace period. This temporal separation is the core architectural insight of RCU.

In the Linux kernel documentation, the example is clear.

To update a field in a global structure:
1. Allocate a new copy.
2. Copy the old data into it.
3. Modify the needed field.
4. Atomically swap the global pointer to point to the new copy (using *rcu_assign_pointer()*).
5. Wait for a grace period (*synchronize_rcu()*).
6. Free the old copy.

Steps 1-4 are removal. Steps 5-6 are reclamation.

Between step 4 and step 6, readers may still be accessing the old copy. And that’s fine.

Look at the code flow visually:

```c
new = kmalloc(...);
*new = *old;
new->field = updated_value;
rcu_assign_pointer(global_ptr, new);
synchronize_rcu();
kfree(old);
```

This is unlike any lock.

A lock guards a critical section of *code*. RCU governs the lifecycle of *data*. It ensures that data cannot vanish while it is being observed, not by blocking observers, but by keeping the data alive until observation is complete.

The separation becomes even clearer with list operations. Consider deleting an element from an RCU-protected list:

```c
spin_lock(&list_lock);
list_del_rcu(&node->list);
spin_unlock(&list_lock);
synchronize_rcu();
kfree(node);
```

The *list_del_rcu()* merely unlinks the node from the list. The node itself persists in memory until the grace period elapses. Readers traversing the list may or may not see it, depending on timing, but they will never see corrupted list pointers.

This temporal decoupling enables remarkable performance in read-mostly scenarios, but it also demands a different mental model. **The programmer must think in terms of versions and lifetimes, not critical sections and hold times.** You stop asking “Is the lock held?” and start asking “Can this object still be alive?”

It’s a shift from thinking about code paths to thinking about data existence across time.

### The Trust in Hardware

RCU’s design leans heavily on guarantees provided by modern CPU architectures. It trusts that a write to a single, aligned pointer is atomic. It trusts that a reader will see either the old value or the new value of that pointer, never a torn, partially updated value.

This trust allows the atomic “swap” that makes the copy-update pattern safe. The *rcu_assign_pointer()* and *rcu_dereference()* macros are not locks - they are wrappers that insert the necessary memory barriers for a given architecture, ensuring visibility and ordering.

Consider a common pattern:

The *rcu_assign_pointer()* ensures that if a reader sees the new pointer, they also see all the initialization that happened before it. The *rcu_dereference()* ensures the compiler doesn’t reorder or cache the pointer value incorrectly.

This **reliance on hardware primitives is another point of departure from locks. **A lock is a software construct built from hardware atomic operations. RCU is a software protocol that uses hardware atomicity as a building block for something higher-level.

### The Philosophy of Deferred Work

At the heart of RCU is a philosophy that is broadly useful in system design: defer work until it can be done cheaply and safely.

In RCU, the expensive part, freeing memory, running destructors is deferred until no reader can possibly be affected. This is not just an optimization. It is a principled stance that clean-up is not urgent. It can wait.

The kernel offers *call_rcu()* for this: instead of blocking in *synchronize_rcu()*, you register a callback to be invoked after the grace period. The updater moves on immediately. Reclamation happens in the background, in a soft interrupt context or a worker thread.

For the common case of just freeing memory, there’s an even simpler primitive:

```c
kfree_rcu(old_fp, rcu);
```

This single line registers the object for automatic freeing after the grace period, eliminating the need for a custom callback function. There’s also:

Which might block under memory pressure but otherwise behaves similarly.

This pattern appears elsewhere in successful systems. Garbage collection defers memory reclamation. Event-driven architectures defer response handling. Log-structured file systems defer writes. It’s the idea that latency can often be turned into throughput if you’re willing to postpone the inevitable.

### Toy Implementations: Seeing Through the Illusion

Sometimes the best way to understand a complex system is to see a simplified version. The Linux kernel documentation includes two “toy” RCU implementations that strip away the performance optimizations to reveal the core ideas.

The first toy implementation uses a global reader-writer lock:

```c
static DEFINE_RWLOCK(rcu_gp_mutex);
void rcu_read_lock(void) { read_lock(&rcu_gp_mutex); }
void rcu_read_unlock(void) { read_unlock(&rcu_gp_mutex); }
void synchronize_rcu(void)
{
    write_lock(&rcu_gp_mutex);
    write_unlock(&rcu_gp_mutex);
}
```

This looks suspiciously like a lock, because it is one!

It demonstrates the minimal semantics: readers hold a read lock, *synchronize_rcu()* waits for all readers by acquiring a write lock. But this implementation has terrible performance and can deadlock in real kernels.

The second toy implementation is more illuminating:

```c
void rcu_read_lock(void) { }
void rcu_read_unlock(void) { }
void synchronize_rcu(void)
{
    int cpu;
    for_each_possible_cpu(cpu)
        run_on(cpu);
}
```

Here, the read-side primitives do absolutely nothing. *synchronize_rcu()* ensures every CPU has scheduled at least once. Since RCU read-side critical sections cannot block, a context switch guarantees they’ve completed.

This implementation reveals the fundamental insight: **RCU doesn’t synchronize by locking; it synchronizes by tracking progress**. The kernel’s production RCU uses per-CPU counters, quiescent state detection, and grace period state machines, but the principle remains.

These toy implementations make clear why RCU isn’t a lock:

1. In the first toy, the lock is just an implementation detail to explain semantics
2. In the second toy, there’s no lock at all
3. In the real implementation, there are no locks in the read path

The takeaway: **RCU is defined by its semantics** (wait-free reads, deferred reclamation), not by any particular implementation. **A lock is defined by its implementation** (mutual exclusion via atomic operations).

This distinction matters when you encounter RCU variants like SRCU (Sleepable RCU), which allows blocking in read-side critical sections, or RCU Tasks, which works at task granularity rather than CPU granularity. They all preserve the core semantics while adapting to different constraints.

### Choosing Your Synchronization Primitive

The Linux kernel offers multiple synchronization primitives.

How do you choose?

The RCU documentation provides a decision tree, but the philosophy behind it matters more.

Here’s a simplified model:
- Is your data read-mostly? (RCU shines here)
- Can you tolerate readers seeing slightly stale data? (RCU requires this)
- Is memory overhead from copies acceptable? (RCU needs extra copies)
- Can you handle the complexity of grace periods? (RCU has learning curve)

If all of these are fine, go with RCU.

The choice isn’t about which is “better” but which matches your constraints.

RCU trades write-side complexity for read-side performance.

It exchanges immediate reclamation for scalability.

Locks serialize access.

RCU serializes reclamation.

So, RCU isn’t a “better lock.”

It’s a different tool for a different problem.

They solve related but distinct problems.
