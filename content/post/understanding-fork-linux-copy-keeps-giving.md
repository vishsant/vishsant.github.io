+++
date = "2026-03-04"
draft = false
title = "Understanding fork() in Linux: The Copy That Keeps on Giving"
+++

In 1971, a system call appeared in Unix Version 1. It was radical. It made process creation so simple that a child could understand it: call the function, get a copy of yourself, and let each copy decide what to do next.

That function was fork().

It worked so well that we never replaced it. We extended it, worked around it, added new system calls alongside it. But the function itself, the idea itself, the behavior itself, lived on. Today, every Python web worker, every multiprocessing daemon, every shell script that spawns a subprocess depends on the same mental model that ran on a PDP-7 with a single CPU and a single thread.

The world changed. fork() did not.

This is not an article about avoiding fork(). It is about understanding what fork() actually does when you call it in 2026, on a machine with 64 cores, multiple threads, and a heap full of memory that the kernel manages with mathematical care. The behavior is specific, the surprises are predictable, and the engineers who know them build more reliable systems than the engineers who don't.

Once you understand this, you will still use fork(). But you will use it with a clearer picture of what you are inheriting.

## The Original Insight

Ken Thompson first implemented fork() on the PDP-7, before Unix was formally released. When Unix Version 1 arrived in 1971, the system call was already there. Thompson needed a way to create new processes. The options were limited by what the hardware could do and what the team had time to build.

The simplest possible solution was also the most powerful one. Instead of designing a mechanism to construct a new process from scratch, he allowed a process to duplicate itself. The parent became the template. The child was the copy. After the copy existed, either one could choose a new program to run.

This decision shaped the next five decades of system design.

The convenience was real. Creating a process required no specification. No configuration. No description of what the child should look like. The child looked like the parent, because it was the parent, at the moment the call was made.

The developer only had to answer one question: ***what should this copy do differently?***

The Unix shell works this way. When you type a command, the shell calls *fork()*. The child process, still running shell code, immediately calls *exec()* to replace itself with your program. The parent waits. The child runs. This two-step pattern, **fork-then-exec**, became the foundation of process management across virtually every Unix-derived system ever built.

For thirty years, this model fit reality well enough. Systems ran one process at a time or a few dozen. Memory was small. Programs were focused. The state a child inherited from its parent was manageable and mostly expected.

Then systems grew. Programs grew. Memory allocations grew. Thread counts grew. The state that fork() copies became more complex, more surprising, and more dangerous. But the function call stayed the same.

One line. One result. Two processes.

The simplicity of the interface hid the growing complexity of what it actually did.

## What the Child Actually Receives

The documentation says that fork() creates a copy of the calling process. That description is correct and misleading at once.

Think of a house. A complete copy of a house would include the furniture, the food in the refrigerator, the photographs on the wall, and every key on the hook by the door. A process has the equivalent of all of these things. And when fork() runs, the child gets a copy of each one.

The child receives the virtual address space of the parent. Every allocated region of memory. The stack. The heap. The global variables. The read-only data segment holding string constants and compiled program instructions. All of it appears, from the child's perspective, as its own memory.

The child receives copies of all open file descriptors. Every file the parent had open, the child has open too. The socket the parent was listening on. The log file the parent was writing to. The database connection the parent established before fork. All of these appear in the child's file descriptor table.

The child receives the signal mask. Any signals the parent had blocked at the moment of the call remain blocked in the child.

The child receives the environment variables. The current working directory. The umask. The resource limits. The set of mounted filesystems visible through the namespace. The effective user ID and group ID. The nice value.

The **child does not receive the parent's threads**. Only the calling thread survives in the child. The other threads, if any existed, are gone. Their locks, their state, their half-completed work are all gone too. The child wakes up as a single-threaded process holding the memory of a multithreaded one.

This detail causes more production bugs than almost any other behavior in the function.

## Copy-on-Write: The Lie That Saves Memory

A 1970s machine had limited RAM. Copying the entire address space of a process on every fork() would have been prohibitively slow and wasteful. The designers understood this. They did not copy everything. They pretended to.

Copy-on-write is the mechanism that makes fork() fast and the reason it is misunderstood.

When the child process comes into existence, the kernel does not duplicate the parent's physical memory pages. Instead, it marks every writable page in both processes as read-only and shared. Pages that were already read-only, such as the program's code segment, remain shared without any change. The parent and child point to the same physical memory. Both see the same bytes. But the kernel watches.

The moment either process writes to a page, the kernel intercepts the write. It copies that page, gives the writing process its own private copy, marks it writable, and allows the write to proceed. The other process continues with its unchanged copy.

This is the lie that saves memory. The child appears to have a full copy of the parent's address space. It costs almost nothing to create. The cost only lands when a write happens.

The problem is that "when a write happens" is not always predictable. A large heap that is read-only after fork costs nothing. A large heap that the child writes to immediately costs megabytes of actual memory copied page by page, creating latency spikes that are hard to diagnose because they appear after the fork, not during it.

This is why forking a process after loading a large cache is dangerous. The cache looks free. The kernel tracks it as shared. But every cache update in the child triggers a copy, and those copies accumulate. Engineers have discovered this running Redis, Python multiprocessing workers, and any system that forks after warming a large data structure.

The performance assumption that **fork() is cheap is true in exactly one case: when the child never writes to inherited memory. In every other case, the cost is deferred, not eliminated.**

## File Descriptors and the Shared Cursor

Open a file. Seek to position 1000. Fork.

The parent and child now both have a file descriptor pointing to that file. The descriptor in the child is not a new handle to the file. It is a reference to the same underlying kernel object, called a file description, that the parent holds.

That file description contains the current file offset. When the parent reads 100 bytes, the offset moves to 1100. When the child reads 100 bytes from its own descriptor, the offset also starts at 1100, because both descriptors share the same position.

This matters for log files. When a web server forks workers and each worker writes to the same log file descriptor inherited from the parent, writes from different processes can interleave at the byte level. A line from one worker can appear in the middle of a line from another. The file descriptor is shared. The kernel treats writes from both processes as coming from a single stream with a single position counter.

The correct approach in most server architectures is to close inherited file descriptors in the child, then reopen them. Or to set the **close-on-exec** flag before forking, so the descriptors close automatically when the child calls exec.

But this requires deliberate action. The default behavior is inheritance. And inheritance, for file descriptors, means shared mutable state between processes that are supposed to be independent.

Network sockets behave the same way. A listening socket inherited by a child can accept connections. Two children accepting connections from the same listening socket leads to surprising behavior. Both see the connection request. Only one should handle it. Which one? The answer depends on timing and kernel scheduling, not on the programmer's intent.

The file descriptor table looks like a detail. In practice, it is one of the most consequential things that fork transfers.

## Signals in Two Bodies

Signals are a form of inter-process communication. They are notifications delivered by the kernel to a process, telling it that something happened: a child died, a pipe broke, a user typed control-C, or a timer expired. Processes can block signals, ignore them, or install handlers that run when they arrive.

When the parent forks, the child inherits the entire signal disposition table. Every signal the parent ignores, the child ignores. Every signal the parent has a custom handler for, the child has that custom handler too.

This inheritance is often correct. A daemon that ignores SIGHUP because it manages its own reload cycle wants its worker children to also ignore SIGHUP. But the inheritance is not always correct.

A parent that chose to ignore SIGCHLD because it does not care about child exit notifications might fork a child that then forks its own grandchildren. The grandchildren exit. Their parent, the first child, also has SIGCHLD ignored. The grandchildren become zombies. Nobody reaps them. The process table slowly fills.

The signal mask, which controls which signals are blocked rather than ignored, carries the same inheritance risk. A parent that blocks SIGPIPE during a critical section might still be in that blocked state when it forks. The child begins with SIGPIPE blocked. If the child writes to a broken socket without realizing SIGPIPE is masked, no signal arrives. The write returns an error that the child must check deliberately, rather than triggering a handler the developer expected.

The reset is not automatic. The developer must know to do it.

Signal handlers are more subtle yet. After fork, the child inherits the parent's signal handler function pointers. If those handlers use global variables, or call functions that use global state, and if the parent was in the middle of modifying that state when the signal arrives in the child, the behavior is undefined.

Signals were designed for a world where processes were simple and largely single-threaded. That world changed when programs gained threads. fork() made that earlier world feel portable into the present. It is not.

## The Multithreaded Trap

A server starts. It initializes a thread pool with sixteen workers. Each worker holds a mutex protecting its local queue. One thread is in the middle of taking the mutex when an unrelated part of the application calls fork().

The child process wakes up with one thread. The thread that called fork(), running alone in a world shaped by sixteen threads that no longer exist.

The mutex that thread seven was holding when the fork happened? Still locked in the child. Nobody holds it. Nobody will release it. The first thread in the child that tries to acquire it will wait forever.

POSIX is clear about this. After fork() in a multithreaded program, only async-signal-safe functions may be called in the child before exec(). The list of async-signal-safe functions is specific and short. It includes write(), _exit(), and a handful of others. It does not include malloc(). It does not include printf(). It does not include anything that acquires a lock internally.

This trap is not theoretical. Any server that calls fork() to create workers after creating a thread pool is vulnerable. Any logging library that uses an internal mutex is unsafe to call in the child. Any memory allocator, which all modern allocators are, is technically unsafe between fork() and exec() in a multithreaded program.

The safe pattern is strict: **call fork() before any threads are created, or call exec() immediately after fork() before touching anything that might be locked.** Any code between fork() and exec() in a multithreaded program is walking across frozen water.

## The Workaround

The engineers who designed POSIX knew about the multithreaded fork problem. They added a function called** *pthread_atfork()***.

The function registers **three handlers**: one that runs before the fork, one that runs in the parent after the fork, and one that runs in the child after the fork. The intent was to let library authors acquire their locks before the fork and release them after, preventing the child from inheriting a locked mutex whose owner has evaporated.

It acknowledges the problem without solving it. For a single library with two or three locks, pthread_atfork() works. For a large application with dozens of libraries, each potentially registering its own handlers, each handler running in sequence, each one representing a lock that must be correctly handled, the complexity grows without bound.

The handlers run in a specific order. Prepare handlers run in reverse registration order. Parent and child handlers run in registration order. Getting this right across library boundaries, where you do not control the order of registration, requires a level of global coordination that most applications cannot guarantee.

Python's multiprocessing module confronted this problem. Python's memory allocator uses locks. The GIL itself is a form of lock. Using fork in a Python program with threads triggers all of the hazards described here. The recommendation in recent Python documentation is to use the "spawn" start method instead of "fork" when creating multiprocessing workers. The "spawn" method starts a fresh Python interpreter rather than forking the existing one. It is slower to start and uses more memory. It is safe.

That trade-off, safety in exchange for efficiency, is what the engineering community reached after decades of debugging fork-related failures. pthread_atfork() is **not the solution**. It is the **formal acknowledgment that a real solution does not exist** within the existing model.

## Why Go Said No

In 2009, the Go programming language made a design decision that many engineers initially found surprising. The language had excellent concurrency support. Goroutines were cheap, the scheduler was sophisticated, and the runtime was designed for servers. But **Go's runtime does not support using fork()** and then continuing to run Go code. The syscall package does expose fork() and exec() for low-level use, but the runtime itself is not designed to be safe after a fork().

The reason was stated in the documentation and elaborated by the Go team over the years: the Go runtime uses multiple operating system threads. Goroutine scheduling, garbage collection, and internal bookkeeping all depend on threads running concurrently. The behavior of fork() in a multithreaded program, particularly with respect to mutexes and runtime state, made fork() unsafe to call from Go code that intends to keep running.

The team went further. The Go runtime is not async-signal-safe. Even calling exec() immediately after fork() in Go is technically unsafe, because the runtime may have acquired locks that the child now holds without ownership. The Go team provided os/exec as the correct way to create subprocesses. Under the hood, os/exec uses a carefully managed sequence written in assembly and restricted C that avoids touching Go runtime state between fork and exec.

Go did not reject fork() because it is bad engineering. It rejected the idea that fork() could be safely used within its runtime model. fork() was designed for a world where a process and a thread were the same thing. A process had exactly one execution context. Fork copied that context. Simple.

Go runs many execution contexts inside a single process. Copying a process in that model copies a snapshot of something that was never designed to be snapshotted. The runtime's internal consistency depends on all of those contexts being live and coordinated. A fork removes them all except one, leaving the child holding state that was only valid when all of them existed.

The pattern is consistent across languages. **The closer a runtime is to bare Unix process semantics, the more naturally fork() fits. The more infrastructure a runtime adds above the process, the more fork() fights that infrastructure.**

We built modern runtimes on top of Unix. We built Unix on a 1970s idea. And every runtime eventually discovers the seam.

## The Shape of the Inheritance

An estate lawyer once said, the hardest part of settling an inheritance is not the assets. The assets are listed. They have value. They can be divided. The hardest part is the liabilities. The debts that nobody remembered. The recurring obligations nobody disclosed. The responsibilities that transferred silently because nobody thought to ask.

fork() creates an inheritance. The child receives everything the parent owned without negotiation. Memory, file descriptors, signal handlers, mutexes, library state, thread-local data that belongs to threads that no longer exist. Some of this inheritance is useful. Some of it is a debt the child does not know it owes.

The framework for thinking about this is specific:

**What fork() copies and why it matters:**

**First: **virtual memory. Visible, manageable, copy-on-write deferred. Dangerous when the child writes to large inherited heaps.

**Second:** file descriptors. Shared, not duplicated. A shared file offset and a shared socket create contention between parent and child.

**Third:** signal dispositions. Inherited handlers and masks. Dangerous when handlers use global state or when the mask is unexpectedly set.

**What surprises engineers most:**

**Fourth:** open directory streams. The child inherits the parent's open directory handles. The underlying file descriptor references the same kernel object, but the userspace buffer that holds the directory stream state - including the read position - is subject to copy-on-write. On Linux, glibc places the DIR struct in heap memory, so each process gets its own copy after fork. The kernel file offset is shared, but the buffered position diverges independently. POSIX only guarantees that positioning may be shared; portability assumptions here are unsafe.

**Fifth: **memory-mapped files. The mapping is inherited. If the mapping was created with MAP_SHARED, writes from either process are visible to both and to the underlying file. The shared mutable state is real.

**Sixth:** thread-local storage from dead threads. The storage exists in virtual memory. The child inherits those pages. The values are stale. Any code that reads them after fork receives data from a thread that no longer exists.

The function has one line. The inheritance has fifty years of depth.

Understanding the shape of what transfers does not make fork wrong. It makes the engineer precise. Precise engineers close inherited file descriptors. They reset signal masks. They do not call allocators in the child of a multithreaded parent. They choose spawn over fork when the runtime demands it. They treat the gap between fork and exec as a constraint, not a playground.

We built modern servers on a 1970s idea. The idea was good. The world grew around it in ways its designers could not predict. The function did not change, because changing it would break everything that depended on its behavior.

So the behavior remains. The inheritance transfers. And the engineers who know what they are receiving build systems that last, while the engineers who trust the metaphor discover the liabilities at 3 in the morning, in a production incident, when a mutex nobody expected is held by a thread that no longer exists.

That knowledge is the only thing fork() does not copy.
