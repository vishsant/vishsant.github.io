+++
date = "2026-04-05"
draft = false
title = "io_uring Explained: Linux Async I/O Without the Syscall Overhead"
+++

For most of my career, I assumed the I/O model I learned in university was the right one.

To read from a file, call ***read()***. To write, call ***write()***. To handle many connections, use ***select()*** or ***poll()*** or ***epoll()***. The pattern is taught as **settled engineering**: events arrive, you handle them, you call the appropriate syscall, you move on.

What the courses don't teach is what crossing the syscall boundary actually costs - and what happens to that cost when your server is doing it half a million times a second.

***io_uring***, introduced in Linux 5.1 in 2019, is the kernel's answer to that question. It is not a faster version of read(). It is a different model entirely - one that changes where the work happens, how the kernel and userspace communicate, and in its most aggressive configuration, whether a syscall needs to happen at all.

## The Tax

When userspace code calls a syscall, the CPU performs a privilege level transition. It moves from user space to kernel space - or in x86 terms, from privilege level 3 (ring 3) to privilege level 0 (ring 0).

The transition is not free. The CPU saves the current register state, switches the stack pointer to the kernel stack, loads the appropriate privilege level, validates the syscall number, dispatches to the handler, runs the handler, prepares the return value, and reverses the transition to hand control back to your code.

```text
User Space (ring 3)
    ↓ syscall
Kernel Space (ring 0)
    → validate
    → dispatch
    → execute I/O
    ↓ sysret
User Space (ring 3)
```

On a modern CPU, this round trip typically costs on the order of hundreds of nanoseconds, depending on hardware, cache state, and whether kernel page table isolation (KPTI) or similar mitigations are active. In isolation, that is nothing. At scale, it is everything.

A web server handling 100,000 requests per second, issuing 10 I/O operations per request, makes on the order of one million syscall boundary crossings per second - purely to submit I/O it already decided to do. (Not every syscall is an I/O submission; accept, epoll_wait, and others factor in too - but the order of magnitude holds.) The submissions are not the bottleneck. The boundary crossings are.

## The Half-Solution

**epoll** was designed to solve a related but different problem.

Before epoll, a server handling thousands of connections had to check each one to see if it was ready. select() and poll() degraded linearly - the cost grew with the number of file descriptors, whether or not they were active. epoll fixed this. It uses an **event-driven model**: you **register interest in a set of file descriptors, and the kernel notifies you only when one of them has data waiting**.

**For connection-heavy servers, epoll was transformative.** Nginx, Redis, and most modern event loops are built on it.

**But epoll does not reduce the syscall cost for individual operations**. It tells you that a socket is readable. You still call read() to read from it. One notification, one read(), one boundary crossing. At high request rates, the syscall overhead remains. You have traded a bad polling problem for a smaller but persistent per-operation tax.

The bottleneck moved from "*how do I know which FD is ready*" to "*how do I do the actual work without paying the boundary cost for each piece of it.*"

io_uring is the answer to the second question.

## The Ring

**io_uring** is built around** two ring buffers shared between userspace and kernel**: the **submission queue (SQ)** and the **completion queue (CQ)**.

When you call **io_uring_setup()**, the **kernel allocates both rings and maps them into your process's address space via mmap().** From that point, both you and the kernel can read and write the rings directly - no data is copied between them, because the same physical memory is visible to both.

```text
Userspace process

Submission Queue (SQ)
    → [SQE][SQE][SQE]
    → you write requests here

    ↓ shared memory (mmap)

Completion Queue (CQ)
    → [CQE][CQE][CQE]
    → you read results here

    ↓

Kernel
    → read SQEs
    → execute I/O
    → write CQEs
```

You describe I/O work by filling in a Submission Queue Entry (SQE) - a 64-byte structure that contains the operation type (read, write, accept, connect, fsync, and dozens more), the file descriptor, the buffer address, the length, and a user-supplied tag for correlation. You place the SQE into the ring and advance the tail pointer.

When the kernel processes the work, it writes a Completion Queue Entry (CQE) - a compact 16-byte structure containing the result code (a signed integer: negative values are -errno-style error codes) and your correlation tag - into the completion ring and advances its tail.

Your application reads the CQE, processes the result, and advances the CQ head to indicate it has consumed the entry.

The critical point: you write many SQEs and read many CQEs through the shared ring without calling the kernel for each one. When you are ready to submit a batch, one call to **io_uring_enter() delivers everything queued in the ring**.

This is **syscall amortization**. One crossing, many operations. The per-operation boundary tax drops toward zero as batches grow.

## The Disappearing Syscall

The ring model already reduces the syscall frequency substantially. But it does not eliminate syscalls - you still call io_uring_enter() to tell the kernel submissions are waiting.

**SQPOLL **removes even that.

When you create an io_uring instance with the **IORING_SETUP_SQPOLL** flag, the kernel spawns a dedicated thread - visible in ps as **io_uring-sq**. This thread runs in the kernel and continuously polls the submission ring for new entries. It does not wait to be notified. It watches.

Your application writes SQEs and advances the tail. The kernel thread, already running, notices the change and processes the entries without waiting for you to ask. The submission path becomes a shared-memory write - no privilege boundary, no context switch.

Note that this eliminates syscalls for submission. Waiting on completions may still require a syscall - for example, calling **io_uring_wait_cqe()** to block until results are ready. The elimination is in the hot submission path, which is typically where the latency matters most.

The tradeoff is CPU. The kernel thread occupies a core while it polls. Under high I/O loads, this is a favorable exchange: one core polling instead of half a million boundary crossings per second. Under light loads, it is wasteful - the thread burns cycles watching an empty ring. The kernel handles this by sleeping the polling thread after a configurable idle period (**sq_thread_idle**), setting a flag that tells your application to wake it with a single io_uring_enter() before submitting.

SQPOLL is not the default configuration. It is the configuration for applications where I/O submission latency matters enough to dedicate a core to eliminating it.

## What the Kernel Can Do for You

Beyond reads and writes, io_uring today supports the full I/O surface of Linux:
- **Network:** accept, connect, send, recv, sendmsg, recvmsg
- **Files:** open, close, statx, fsync, fdatasync, truncate
- **Splice and tee:** zero-copy data movement between file descriptors
- **Timeouts and cancellation:** timeout, cancel for managing in-flight operations
- **Poll:** poll_add, poll_remove - readiness-notification semantics inside io_uring (analogous to epoll, but wired into the ring rather than through epoll_ctl/epoll_wait)

This coverage means an application can express its entire I/O lifecycle - open a file, read it, send the data over a socket, close the file - as a sequence of SQEs, submitted in a single batch or chained so each operation triggers the next automatically using the **IOSQE_IO_LINK** flag (set in the SQE's flags byte). The kernel executes the chain without the application re-entering for each step.

In practice, most production applications use the **liburing** library rather than raw syscalls. liburing **provides C wrappers that handle the ring setup, index arithmetic, and atomic pointer management**, leaving the application code focused on describing operations rather than maintaining ring state.

The interface is explicit about submission and completion as separate, asynchronous events. There is no blocking in the hot path unless you ask for it.

The question for any high-throughput I/O application is no longer "*should I look at io_uring.*" It is "*at what request rate does the syscall boundary become the bottleneck, and how soon do I hit it.*"

The answer depends on your workload. But the engineers who know where the boundary is build systems that know how to stay on the right side of it.

The fastest path through the kernel is the one you take least often.

io_uring is how Linux learned to charge less tax - and in the right conditions, to stop charging entirely.
