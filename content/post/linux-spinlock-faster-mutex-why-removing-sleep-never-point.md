+++
date = "2026-08-02"
draft = false
title = "A Linux Spinlock Is Not a Faster Mutex: Why Removing Sleep Was Never the Point"
+++

Ask an engineer to define a spinlock and you will usually get a subtraction. A mutex tries to take the lock, and if it can't, it puts the thread to sleep until the lock is free. A spinlock is that, minus the sleeping. It waits in place instead. Same idea, less waiting.

The arithmetic is clean, it explains the performance folklore, but the underlying concept is wrong. Nothing was removed to make a spinlock. Something was added, and the thing that was added is the only reason the primitive exists.

## The Addition

Here is the generic SMP path, from include/linux/spinlock_api_smp.h:

Read the order, because the order is the argument. preempt_disable() runs first. Before the lock word is examined, before any spinning, before the third line does the actual acquire. With it, the kernel increments a counter that tells the scheduler this CPU is unavailable. Everything about the lock happens after that.

## A Lock Protects Data. Why Disable the Scheduler?

The obvious model of a spinlock is that it stops another CPU from entering the critical section. That is true, and it is not enough. Another CPU is not the only thing that can interrupt you. The scheduler can stop you, on the CPU you are already holding the lock from.

Let's take an example. Task A takes the lock on CPU 0 and starts modifying the shared structure. The scheduler preempts it mid-write. The data is still consistent and the lock is still held. A is simply not running.

On a multiprocessor machine that is a latency problem. Task B spins on a lock whose owner isn't running, and it waits for however long the scheduler takes to run A again, which is not a bounded quantity. Every cycle B burns is a cycle that could have gone to A.

On a uniprocessor machine the same picture is fatal. There is no CPU 1. If A is preempted and B is scheduled in its place, B spins on a lock only A can release, and A cannot run until B stops, which B never will. Deadlock, from a lock that was correctly implemented and correctly used.

Which is exactly why the uniprocessor build keeps preempt_disable() after throwing away everything else. include/linux/spinlock_api_up.h says it outright:

There is no real locking going on, and the lock still works, because on one CPU the preemption counter is the mutual exclusion. Strip a spinlock of everything that spins and preempt_disable() is what is left standing.

So the counter is not part of the locking algorithm. It establishes the conditions under which the locking algorithm is safe. The lock decides who is allowed in. preempt_disable() decides whether the one who got in can be stopped.

## Who Is Allowed to Interrupt You

Once the lock is an execution contract rather than an algorithm, the family of spinlock variants stops looking like API clutter. Each one closes a different door, because each execution context can interrupt a different set of others.

spin_lock() closes preemption: no other task takes this CPU from you. spin_lock_irq() closes hardware interrupts as well:

That variant exists because an interrupt handler runs on the CPU of the task it interrupted. If the handler takes a lock that task is already holding, it spins on a lock the task cannot release until the handler returns, and the handler cannot return until it gets the lock. Same CPU, same lock, no exit. spin_lock_bh() closes softirqs for the same reason one level down, where network and block-completion work runs asynchronously on your CPU and reaches your data.

These are not different ways of waiting. They are different ways of deciding who is allowed to interrupt the critical section.

So the subtraction has it backwards. You did not take the sleeping out of a mutex and get something smaller. You took a lock and bolted a preemption-off region onto it. The spinning is the visible part, the region is the product. And a region where the scheduler cannot run is not a smaller thing than a sleep. It is a larger claim on the machine, held on behalf of one thread, paid for by every other task that wanted that processor.

Sleeping under spinlock is therefore fatal rather than slow. mutex_lock() in kernel/locking/mutex.c opens with might_sleep(), and calling it with preemption off gets you reported from kernel/sched/core.c:

There is no recovery path because there is nothing to recover. The scheduler was told to stay out, and something just asked it to run.

## The Sleeping You Didn't Remove

A Linux mutex does not sleep on contention, at least not first. It asks whether the current owner is running on another CPU, and if it is, it spins and waits for the handoff. The loop lives in mutex_spin_on_owner():

Look at the exit condition rather than the loop. The mutex spins for exactly as long as spinning is a good idea, and the moment the owner stops running, or something more deserving wants this CPU, it stops and sleeps.

The above code becomes:

So the sleeping was never a fixed cost you could subtract. It was the mutex's fallback, reached only when spinning stops paying. Strip it out and you haven't removed an expense. You've removed the condition, the part that knows when to quit, and kept the spinning that was always there.

You cannot remove a cost from a system by deleting the code that pays it. The mutex sleeps because somebody has to wait, and taking the sleep out doesn't take the waiting out. It only changes who does the waiting, and where, and whether anyone can see it.

And now you learned the real difference between the mutex and a spinlock.
