+++
date = "2026-03-22"
draft = false
title = "NULLFS - The Empty Filesystem That Reshapes Container Security in Linux"
+++

For over a decade, every Linux container started from a flawed foundation.

We just didn’t question it.

The root of every mount namespace in Linux was mutable. Every container inherited a filesystem root that could be written to. And every runtime built a careful, fragile sequence of operations to work around this fact.

The workaround became so old and so ubiquitous that it stopped looking like a workaround. It started looking like the way things are done.

A few weeks ago, I wrote about why Linux containers are not as isolated as most engineers believe. That piece explored the walls.

This one is about the foundation.

Because while engineers debated namespace boundaries and seccomp profiles, something more fundamental was wrong. Not visibly wrong. Not exploitably wrong in most configurations. Subtly wrong - that hides inside assumptions.

In February 2026, Christian Brauner merged a patch into Linux 7.0 that introduced a filesystem containing nothing - no files, no directories, no writable state. An immutable, empty root.

It is called **NULLFS.**

And it may be the most important container security improvement since user namespaces.

## The Root Nobody Questions

Every filesystem has a root - the anchor from which everything else descends. Without it, there is no hierarchy, no paths, and no structure.

When Linux boots, it needs this root immediately. Before disks are mounted and before user processes exist, the kernel must anchor the filesystem tree somewhere.

This anchor is **rootfs**.

Rootfs is an in-memory filesystem created during boot. Depending on configuration, it is either tmpfs or ramfs. It exists entirely in RAM, with no backing store. It is not meant to be permanent - it simply exists because the kernel needs a root before the real filesystem is ready.

In most systems, rootfs is quickly hidden. The initramfs loads, the real root filesystem mounts on top, and rootfs disappears beneath layers of overmounts. By the time the system is running, it is effectively invisible.

But it never goes away.

And it has a property that matters.

It is **mutable**.

## The Mutable Foundation

Rootfs is writable. That means the true root of the Linux filesystem hierarchy - the foundation beneath every mount, every namespace, every container - accepts writes.

In early Linux systems, this was not a concern. There were no containers, no mount namespaces, and the threat model was simple.

Containers changed everything.

When a container starts, the runtime creates a new mount namespace. This namespace gives the container its own view of the filesystem. But it does not start empty - it inherits a copy of the parent’s mount table.

Every mount the host has, the container initially has as well.

This includes rootfs.

That inheritance creates an immediate problem. The container now sees far more than it should, so the runtime must remove what is unnecessary.

The process looks simple on the surface: mount the container’s root, pivot into it, and recursively unmount everything else.

But there is a catch.

Rootfs cannot be unmounted. It is special. The kernel does not allow removing the root that anchors the entire hierarchy.

So even after cleanup, a mutable root still exists underneath everything - hidden, but present.

And the kernel must ensure the container cannot reach it.

## The Pivot

This cleanup sequence is what every container runtime performs. Docker, containerd, CRI-O, and runc all follow the same pattern: copy the parent mount table, pivot to the container root, and recursively remove what is not needed.

This is known as the **pivot dance**.

It works - but it comes with costs.

There is a performance cost. On systems with large mount tables - common in Kubernetes environments - copying and cleaning mounts takes measurable time. When many containers start simultaneously, they contend on shared kernel locks, turning this into a bottleneck.

There is a complexity cost. The sequence is fragile, with subtle interactions between mount propagation, shared subtrees, and locked mounts. These edge cases have led to real vulnerabilities.

One example is **CVE-2020-15257**, where a race condition in mount propagation allowed containers to access host mounts after the pivot dance. The cleanup sequence missed a mount that was still shared with the host.

There is also a security cost. The model is fundamentally **default-allow**. The container starts with everything and removes what it does not need. If the cleanup misses something - due to a race, propagation event, or unexpected mount - the container gains access to host resources.

For over a decade, this was the best available approach.

Then came a different question:

**What if the root was empty?**

## What NULLFS Actually Is

NULLFS is a filesystem that contains nothing.

And that changes everything.

It is a purpose-built pseudo-filesystem created by the kernel at boot. It has no writable state, no files, and no directories beyond its root. The traditional rootfs is mounted on top of it, but the true root of the hierarchy is now immutable and empty.

This change is small in implementation - just a thin layer beneath rootfs - but profound in effect.

Rootfs is no longer special. It becomes just another mount.

And that means **pivot_root()** finally works cleanly.

What once required complex workarounds now becomes simple. The old root can be pivoted away and detached without special handling. The initramfs boot sequence, which previously relied on fragile logic, reduces to a straightforward pivot and unmount.

A decade of accumulated complexity collapses into a predictable, minimal operation.

## OPEN_TREE_NAMESPACE — The Second Half

NULLFS fixes the foundation.

**OPEN_TREE_NAMESPACE** fixes how we build on it.

Traditionally, creating a mount namespace meant copying the entire parent mount table using **CLONE_NEWNS**. Every mount - relevant or not - was inherited, and then the runtime had to clean it up.

OPEN_TREE_NAMESPACE changes this model.

Instead of copying everything and removing what is unnecessary, the runtime specifies exactly which mount tree it wants. The kernel creates a new namespace containing only that tree, mounted on top of a nullfs root.

The result is a namespace that starts with exactly what the container needs - and nothing else.

The difference is both conceptual and practical.

The old model copies everything, then strips it down. Its performance scales with the size of the host’s mount table, and its security depends on correct cleanup.

The new model copies only what is required. Its performance scales with the container itself, and its security comes from starting empty.

Benchmarks show the impact clearly. Container creation throughput increases by roughly 40%, from around 73,000 containers to 109,000 in controlled tests.

But the deeper change is philosophical.

We move from:

**“Allow everything, then remove what you don’t want”**

to:

**“Allow nothing, then add what you need”**

## The Default-Deny Filesystem

Together, NULLFS and OPEN_TREE_NAMESPACE introduce something Linux has never had before: a mount namespace that starts genuinely empty.

Not empty after cleanup.

Empty by design.

This eliminates entire categories of problems.

The **MNT_LOCKED** mechanism becomes unnecessary for rootfs. Previously, it prevented containers from unmounting sensitive layers and exposing host data. With nullfs as the root, there is nothing beneath to expose.

Container escape techniques based on mount manipulation become harder. There are no inherited host mounts to reach, no shared structures to exploit.

Kernel threads can be isolated more cleanly, without inheriting filesystem views from init.

And the namespace semaphore bottleneck is reduced, because container creation no longer involves copying large mount tables.

This reflects a broader principle in system design.

The strongest systems do not rely on protections layered over unsafe states. They eliminate the unsafe state entirely.

A locked door can be picked.

A door that does not exist cannot be attacked.

NULLFS removes the door.

## What Changes Now

NULLFS is merged into Linux 7.0 and enabled by default. For most users, the change is invisible. Systems boot the same way, containers run the same workloads, and rootfs still exists - mounted on top.

But the foundation has changed.

Container runtimes can simplify their logic. The pivot dance can be removed. The **switch_root **workaround can finally be retired.

Kubernetes operators benefit from faster pod startup and reduced contention on heavily loaded nodes. Multi-tenant clusters gain a stronger isolation baseline by default.

Security engineers gain something more valuable: a system where an entire class of bugs no longer exists.

This is what makes NULLFS significant.

**It is not a patch for a vulnerability.**

**It is the removal of a category of problems.**

## What This Means for You

If you run containers in production, these changes matter in practical ways.

Container startup becomes faster, especially on systems with large mount tables. The runtime copies only what is needed, reducing both latency and contention.

Isolation becomes stronger by default. Instead of relying on cleanup, the system starts empty and builds upward. This eliminates many edge cases that previously led to vulnerabilities.

Runtime code becomes simpler. Fewer steps, fewer interactions, and fewer opportunities for subtle bugs.

To take advantage of this:
- Watch for container runtimes adopting OPEN_TREE_NAMESPACE
- Revisit assumptions about mount isolation and rootfs behavior
- Remove legacy workarounds tied to the old pivot dance model

In architecture, the strongest structures are not always the ones with the most material. They are the ones that distribute stress through design.

A Roman arch holds not because of mass, but because of shape.

NULLFS is an arch.

It replaces a complex, fragile workaround with a structural solution so minimal it barely exists. A filesystem with nothing in it. An immutable root that secures the system by offering nothing to attack.

The engineers who built container isolation worked within the constraints they had. They compensated for a mutable root because it could not be changed.

Now it has been.

The strongest foundation is not the one that resists everything.

It is the one with nothing left to attack.
