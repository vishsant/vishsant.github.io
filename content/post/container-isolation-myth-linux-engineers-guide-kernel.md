+++
date = "2026-02-22"
draft = false
title = "The Container Isolation Myth in Linux: An Engineer's Guide to the Kernel Boundary"
+++

You deploy containers in production.

You trust them to isolate workloads.

You assume:

“Containerized” means “separated.”

That assumption is only half true.

A **container** gives you **discipline**. **Not isolation**.

And confusing those two is the most dangerous container misconception in production systems.

---

## What You Actually Got

When you start a container, Linux gives you two major mechanisms:

### 1. Namespaces → Selective Blindness

- Separate PID tree
- Separate mount view
- Separate network interfaces
- Separate IPC
- Separate UTS (UNIX Time-sharing System)

Each process believes it is alone.

But namespaces control **what you see**, not **what exists**.

### 2. Cgroups → Resource Governance

- CPU limits
- Memory limits
- I/O throttling

Cgroups answer one question:

“How much can you consume?”

They do not answer:

“What do you share?”

---

## What You Did NOT Get

You did not get:
- A separate kernel
- A hardware boundary
- Isolation from kernel vulnerabilities
- A separate syscall surface

Every container on your host runs on:

One kernel. One scheduler. One memory allocator. One attack surface.

---

## The Kernel Truth

Every process in Linux has a `task_struct` (see `include/linux/sched.h`).

Inside it:

```c
struct task_struct {
    struct nsproxy *nsproxy;
};
```

`nsproxy` does the trick.

It’s a struct containing pointers:

```c
struct nsproxy {
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net *net_ns;
};
```

Think of a namespace object as a **table of mappings**.

For example:
- The ***pid_namespace*** maps **virtual PIDs** (the PID you see inside the container) **to** the **global kernel PID**.
- The ***net namespace*** holds the **routing table, iptables rules, and socket** information.

When you start a container, the container runtime creates a new set of namespace objects (unless you tell it to reuse existing ones). Then it forks a process and sets its *nsproxy* to point to those newly created objects.

But all of them live:

In the same kernel memory. On the same heap. Under the same MMU.

```text
Host kernel memory
  task 1000 → nsproxy A → pid/net/uts namespaces A
  task 2000 → nsproxy B → pid/net/uts namespaces B

Container A: PID 1, 10.0.1.2
Container B: PID 1, 10.0.2.2

Host view: both tasks and both network paths
```

The hardware has no concept of namespaces.

Only pointer dereferences do.

The kernel always sees the truth.

**Namespaces only control the view**.

Same kernel. Different lenses.

---

## VM vs Container: The Structural Difference

Let’s remove the marketing layer.

**VM:**

Hardware → Hypervisor → Guest Kernel → Your Code

**Container:**

Hardware → Host Kernel → Namespaces → Your Code

A VM breakout requires:

Guest exploit → Hypervisor exploit

A container breakout requires:

A kernel bug.

That’s it.

Containers are fast because there is no second kernel.

But that speed comes from sharing the foundation.

---

## CVEs Don’t Care About Your Namespace

CVE means "Common Vulnerabilities and Exposures".

It's a standardized identifier for publicly known security vulnerabilities.

Think of it as a social security number for bugs.

If there’s a vulnerability in:
- BPF
- io_uring
- Memory management
- Page tables
- Filesystem drivers

Every container is exposed simultaneously.

*CVE-2022-0492* proved this clearly.

A container with *cgroup* access could write to *release_agent*, and the host kernel executed it.

Why?

Because even when namespaced, the control path still reached the host’s *cgroup* driver.

The view was separated.

The authority was not.

---

## Cgroups Are Not Partitions

*cgroups* limit consumption.

They do not partition physics.

Two containers within memory limits can still:
- Contend for cache lines
- Fight over memory bus bandwidth
- Influence speculative execution behavior
- Trigger NUMA imbalances

You can govern resources.

You cannot separate silicon.

---

## The Network Myth

Your container has:
- Its own IP
- Its own routing table
- Its own interfaces

But traffic between containers on the same host?

It flows through the host kernel.

Run *tcpdump* on the host. You’ll see everything.

“Network isolation” means:

Your neighbor cannot address your interface directly.

It does not mean:

The host cannot observe or manipulate traffic.

Again:

**View ≠ Boundary**.

---

## Where the Seams Appear

Containers don’t fail randomly.

They leak at seams.

Common ones:
- /proc misconfiguration
- /sys exposure
- Privileged capabilities (especially SYS_ADMIN)
- Writable host mounts
- Shared device files
- Overly broad seccomp profiles

Every convenience you add is a potential bridge.

**Functionality always competes with isolation.**

---

## The Accurate Mental Model

Stop thinking:

Container = Lightweight VM

Start thinking:

**Container = Tenant**

Tenant:
- Your own apartment (process tree)
- Your own lock (namespace view)
- Shared building (kernel)
- Shared foundation (vulnerabilities)
- Shared plumbing (syscall table)

If the foundation cracks, all tenants feel it.

Kernel CVEs are foundation cracks.

---

## How to Think About Containers in Production

1. Never run privileged containers. *SYS_ADMIN* is not a capability. It’s making you more vulnerable.
2. Use strict *seccomp* profiles. Every blocked syscall is one less door.
3. Treat writable host volumes as host access. A mount is a bridge.
4. Patch the host aggressively. Kernel CVEs are container CVEs.
5. Understand your runtime. Docker, containerd, CRI-O - the security model is documented. Read it.

Containers are powerful.

But they are not walls.

They are disciplined processes sharing one kernel.

---

## The One Line to Remember

A container is not a machine.

It is a process with boundaries drawn in software.

And the kernel always sees through them.

---

What was the longest container misconception you held?

For me:

Believing container root ≠ host root.

That illusion lasted until I mounted a host directory into a container for “convenience.”

The moment I edited a file inside the container and saw it change on the host, I realized:

Different namespace. Same filesystem.

What was yours?

Would love to hear your story.

If you enjoyed this, I write about systems engineering, Linux internals, and the evolving relationship between software and hardware. Follow for more deep dives on operating system architecture.
