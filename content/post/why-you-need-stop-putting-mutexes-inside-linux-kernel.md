+++
date = "2025-11-23"
draft = false
title = "Why You Need to Stop Putting Mutexes inside Linux Kernel Interrupt Handlers?"
+++

So you are thinking:

*“Let’s grab a mutex in an interrupt handler. What could go wrong?”*

Everything.

Mutexes stop race conditions.

Interrupt handlers need synchronization.

So it feels logical to use a mutex.

But this is one of the fastest ways to create an unstoppable deadlock.

Let’s break it down.

You have a device interrupting the CPU — hardware says:

*“Hey! I need attention. Now!”*

Your interrupt handler fires. It needs to update some shared data.

But — that same data can also be modified by normal kernel threads.

So, logically… you lock it.

Except…

Interrupt handlers **can’t sleep**.

Mutexes **can** sleep.

And that’s where everything starts going wrong.

Because an interrupt handler always preempts whatever was running. Imagine the interrupted thread is the one holding the mutex.

Now your interrupt handler tries to acquire that same mutex.

It waits.

And waits. And waits…

But it will wait forever.

Because the mutex holder can’t run… since the interrupt handler is in the way.

This is a perfect deadlock — inside the heart of the kernel.

Your CPU?

Permanently stuck spinning.

So developers realized something:

*“Okay… we need locking in interrupts. But we can’t sleep. So… what if instead of mutexes, we used something that doesn’t sleep?”*

And that became the spinlock.

Spinlocks are just like your stubborn little sister.

They say:

*“I will keep spinning — burning CPU cycles — until I get what I want.”*

But there’s one more hidden trap…

What if the thread holding the spinlock gets interrupted and the interrupt tries to take the same lock?

Deadlock again.

So Linux takes a critical additional step:

**Spinlocks used in interrupt handlers automatically disable interrupts on that CPU.**

Why?

Because if interrupts fire while holding a spinlock, they might need that same spinlock. Better to stop interrupts temporarily than to freeze the system forever.

So the correct rule becomes:

**Never use a mutex in an interrupt handler.**

**Always use a spinlock that disables interrupts on that CPU.**

But there is another problem here.

By disabling interrupts while holding the spinlock… You’re not just preventing interrupt deadlocks…

You’re also temporarily removing the kernel’s ability to schedule anything else.

That’s why real interrupt handlers should do as little as humanly possible — ideally just enough to trigger a soft interrupt or kernel thread that can sleep normally and use mutexes safely.

Hard interrupts: short, fast, spinlocks only.

Soft interrupts: longer work, allowed to sleep, mutexes welcome.

This separation is one of the fundamental design choices that keeps modern kernels fast and reliable.

So now… next time you’re digging into kernel code and think:

*"Meh, I’ll just throw a mutex in here…"*

Stop.

Ask yourself:

Is this interrupt context?

Can it sleep?

What happens if this gets interrupted?

Because now you know the nightmare scenario hidden behind one innocent-looking lock.

So…
