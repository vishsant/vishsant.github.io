+++
date = "2026-05-24"
draft = false
title = "How Linux Threads Actually Work: clone(), Thread Groups, and Shared Memory"
+++

There is a syscall every C programmer believes exists. It is the syscall that creates a thread.

It does not exist.

Linux has no *sys_thread_create*. It never had one. The **kernel does not distinguish, at any fundamental level, between a thread and a process.**

This is not a quirk. It is the design.

## The Lie at the Syscall Boundary

Here are two real syscalls, side by side, from ***kernel/sys.c*** in the current kernel tree:

```c
SYSCALL_DEFINE0(getpid)
{
    return task_tgid_vnr(current);
}

/* Thread ID - the internal kernel "pid" */
SYSCALL_DEFINE0(gettid)
{
    return task_pid_vnr(current);
}
```

Look closely. ***getpid()*** does not return the PID. It **returns* task_tgid_vnr* - the thread group ID**. The thing you have called "the process ID" your entire career is, inside the kernel, the TGID. The actual per-task identifier - the one the **kernel** calls **pid** - **is** what **gettid()** returns, and **what userspace calls the "thread ID."**

The kernel even leaves a comment admitting it: "Thread ID - the internal kernel 'pid'."

Hold onto that inversion. It is the thread you can pull to unravel the whole thing.

## The Syscall That Carries Every Flavor of Concurrent Execution

***clone()*** is one of the strangest syscalls in Linux because it does not have a fixed identity. It is the syscall that creates a child task whose relationship to the parent is configurable.

Call **clone() with zero sharing flags** and you get something behaviorally identical to fork(): a **fully independent process** with its own virtual address space, its own file descriptor table, its own signal handlers, its own everything. **Call it with** CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD plus the four **flags** NPTL layers on top for POSIX plumbing, and** you get a POSIX thread**.

Everything in between is also legal. You can share the address space but not the file descriptors. You can share the file descriptors but not the signal handlers. You can share everything except the filesystem root. The kernel will let you. **The pthread library happens to set the maximum-sharing combination because that is what POSIX requires of a thread**. The kernel does not care. It will create any task you describe.

This is the part most engineers never see. The thread-vs-process duality is a userspace fiction. It exists because POSIX exists.

## What Each CLONE Flag Actually Does

The CLONE flags are not preferences. Each one is a pointer-share. Take **CLONE_VM**. When you set it, the new task does not get a copy of the parent's memory descriptor (***mm_struct***). It gets a **pointer** to the same one. Here is the exact branch in `copy_mm()`:

```c
/* kernel/fork.c — copy_mm() */

if (clone_flags & CLONE_VM) {
    mmget(oldmm);
    mm = oldmm;
} else {
    mm = dup_mm(tsk, current->mm);
}
```

That is the entire difference between a thread and a process in one if. **With CLONE_VM**, the **kernel bumps** a **reference count** (***mmget***) and **points the child at the same *mm_struct***. **Without it**, the **kernel calls *dup_mm*** and **builds a fresh address space**. When the parent allocates memory, a CLONE_VM child sees it. When the child writes, the parent sees it.

The other flags follow the same pattern. Each one collapses to a reference-count bump instead of a copy.

**CLONE_FILES** makes the new task **share the parent's file descriptor table** (*atomic_inc(&oldf->count)*). If the parent has fd 7 pointing to a socket, the child has fd 7 pointing to the same socket. Either side can close(7) and it closes for both.

**CLONE_FS** **shares the filesystem context **(*fs->users++*): the current working directory, the umask, the root directory. If one thread calls *chdir()*, every thread's working directory changes too.

**CLONE_SIGHAND** **shares the table of signal handlers **(*refcount_inc(&current->sighand->count)*). If one task installs a handler for SIGUSR1, every task in the group will invoke it. This is what makes signal delivery to "a process" behave like signal delivery to a group of threads.

And then there is the flag that decides everything.

## The Single if That Creates a Thread

**CLONE_THREAD** is the most important and the most subtle. It is the **flag that decides whether the new task is a thread or a process**, and in the kernel, that decision is literally one branch in `copy_process()`:

```c
/* kernel/fork.c — copy_process() */

p->pid = pid_nr(pid);
if (clone_flags & CLONE_THREAD) {
    p->group_leader = current->group_leader;
    p->tgid = current->tgid;
} else {
    p->group_leader = p;
    p->tgid = p->pid;
}
```

That is the whole thing. That if is the entire conceptual boundary between "thread" and "process" in Linux.

**With CLONE_THREAD set, the new task inherits the parent's tgid and points its group_leader at the parent's leader. It joins an existing thread group.** getpid(), which returns the TGID, now returns the same value as the parent. From userspace, they are "the same process."

**Without CLONE_THREAD, the new task sets *p->tgid = p->pid* and becomes its own group_leader. It is the founding member of a new thread group of one**. getpid() returns something new. From userspace, it is a new process.

A process, in Linux, is a group of tasks that share a TGID. A thread is just a task inside that group. The kernel never had to invent threads as a separate concept. It invented thread group identity: two ***pid_t*** fields and a*** group_leader*** pointer. "Thread" became the name we give to a task that joined an existing group.

## "But Isn't There a thread_struct?"

If you go looking through the headers, you will find ***struct thread_struct*** and ***struct thread_info***, and it is fair to ask whether they contradict the claim that Linux has no thread object. They don't. And understanding why sharpens the whole model.

**struct thread_struct is embedded inside task_struct. It holds architecture-specific CPU state**: register save areas, FPU and SIMD context, segment bases. struct thread_info is also embedded (or lives at the base of the kernel stack), holding low-level per-task flags. **Neither is a schedulable entity**. Neither is allocated on its own. Neither is what the scheduler runs. They are storage that lives inside the one structure the kernel actually schedules: task_struct.

So the precise statement is this. **Linux has no separate thread object that the scheduler treats as a distinct kind of thing. It has task_struct, and task_struct happens to contain a sub-struct named thread for the CPU's bookkeeping.** The name is a historical artifact, not a second abstraction.

## Why This Design Won

Before **NPTL (Native POSIX Thread Library**), Linux had a threading library called **LinuxThreads**, which also used clone() but did not properly leverage the kernel's thread group support. It worked, but it broke POSIX in subtle ways. Signals went to the wrong thread. getpid() returned different values from different threads. wait() did not behave correctly across threads. Threads showed up in ps as separate processes because, from the kernel's view, they were.

The fix did not come from the kernel. It came from userspace finally catching up to what the kernel had already shipped. CLONE_THREAD and the TGID concept had been in the kernel since Linux 2.4 in January 2001. LinuxThreads simply never used them. NPTL, the Native POSIX Threading Library, was first released in 2002 and integrated into glibc 2.3 the following year. It was the first threading library to actually call clone() with CLONE_THREAD set and let the kernel do the grouping. The kernel never gained threads. It had already gained the ability to group tasks under a shared identity. The library just had to use it.

This is the part of the design that is genuinely beautiful. **Linux** did not add threads in 1996 and it did not add them in 2003. It **generalized the process abstraction until threads fell out as a special case of it.**

**Windows took the other path**. The Windows kernel has a **Thread object and a Process object**, each with their own structure, their own scheduler integration, their own lifecycle. The kernel knows the difference. It has to, because the abstractions were built separately. The trade-off is that Windows can never have a "container" the way Linux has one, because Linux's containers are just tasks with different CLONE flags. They are not a new kind of thing. They are the same thing dialed differently.

## The Mental Model: The Sharing Spectrum

```text
Linux Task Sharing Spectrum
        (configured via clone() flags)

Container Task: clone(CLONE_NEW*)
        ↓ baseline isolation
Regular Process: fork() / clone() with minimal sharing
        ↓ increasing sharing
POSIX Thread: clone(CLONE_VM | CLONE_FILES | CLONE_FS |
                    CLONE_SIGHAND | CLONE_THREAD)
```

Once you internalize this, the whole landscape simplifies. Forget the words "thread" and "process." There is only one primitive in Linux for creating a concurrent task, and it is clone().

**A container is a task that shares less. A thread is a task that shares more. A "process" is just the conventional midpoint**. The kernel has no opinion about which is which. It only has opinions about which bits you set.

Once you accept the spectrum, several Linux behaviors that previously seemed arbitrary become inevitable.

***pthread_kill*** exists because, at the kernel level, you are sending a signal to a specific task. If you want to send a signal to a "process," meaning all threads, you use kill(), which targets the thread group (**PIDTYPE_TGID** in ***kernel/signal.c***), and the kernel walks the group to find an eligible thread to deliver to.

***/proc/PID/task/*** exists because the kernel never collapsed the threads into a single object. Each task is still a real, addressable entity in ***/proc***.

That is the model the kernel has had since 1996. The rest of us are still catching up.
