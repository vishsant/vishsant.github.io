+++
date = "2026-06-07"
draft = false
title = "kill() Doesn't Kill: Pending Bits, the Delivery Boundary, and Linux's Unkillable Processes"
+++

Every engineer carries the same mental model of kill. You send the signal, the kernel walks over to the process, and terminates it. The kernel is the executioner; kill() is the order; the process is the victim. The man page even says "kill - send a signal to a process," and we read "send" as "do."

That model is wrong in almost every detail. The kernel never terminates anything. There is no code path in Linux where the kernel reaches into a running process and destroys it from the outside. What actually happens is stranger, slower, and - once you see it - explains a whole family of production mysteries, including the process that ignores *kill -9* for hours.

## The Bit

Here is what sending a signal actually does. Deep inside *kernel/signal.c*, after permission checks and bookkeeping, every signal you can send from userspace - *kill(), tgkill(), sigqueue()* - funnels into *__send_signal_locked(*), and the moment of "delivery" is this:

```c
/* kernel/signal.c — __send_signal_locked() */

sigaddset(&pending->signal, sig);
```

That is the kill. Mark signal sig as pending for this task/process. One bit, set in a bitmask hanging off the target's task_struct. SIGTERM is bit 14. SIGKILL is bit 8. Your process has a small integer field, and "killing" it means flipping one bit of that field to 1.

This single fact explains things you've probably noticed but never connected. Standard signals don't queue - sending SIGTERM five times is identical to sending it once, because a bit that is already 1 stays 1. And pending signals are visible: they're just data, so the kernel happily prints the bitmask in /proc/PID/status.

You can watch the note sit unread. Stop a process so it can't run, then "kill" it:

## The Note

So if sending a signal is just setting a bit, how does the target ever find out? It doesn't - not immediately. The kernel does two things after setting the bit, and both are notifications, not actions. It flags the task and wakes it up:

```c
/* kernel/signal.c — signal_wake_up_state() */

set_tsk_thread_flag(t, TIF_SIGPENDING);
/*
 * ...
 * By using wake_up_state, we ensure the process will wake up
 * and handle its death signal.
 */
if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
    kick_process(t);
```

The first line turns on TIF_SIGPENDING, a per-thread flag that means "There are signals waiting for you." Think of the pending signal bitmap as the mailbox and TIF_SIGPENDING as the mailbox light. The bitmap contains the actual messages; the flag is simply a fast way for the kernel to know that checking the mailbox is worthwhile.

The next step is getting the target's attention.

If the task is asleep in an interruptible state, wake_up_state() moves it back to TASK_RUNNING so it can eventually resume execution and discover the pending signal. But what if the target is not sleeping? Imagine a CPU-bound loop running on another core. There is nothing to wake. The task is already running. In that case wake_up_state() fails and the kernel falls back to kick_process(), which sends an inter-processor interrupt (IPI) to the CPU currently executing the task.

This is an important distinction. The IPI does not deliver the signal. The IPI merely forces the target CPU into the kernel so that the task will eventually pass through the place where signals are checked. Notification and delivery are separate events. The kernel's job ends after leaving the note and making sure the recipient has a chance to see it.

Read that kernel comment again, because it's the whole article in one line: the process will wake up and handle its death signal. The kernel does not handle the death. The victim does. The kernel's role ends at leaving the note and shaking the victim awake so it will eventually read it.

## The Boundary

Eventually is doing a lot of work in that sentence. The kernel checks signal_pending() in many places - but it acts on the bitmask at exactly one: the moment a task finishes running kernel code and is about to return to userspace. Every syscall return, every interrupt return, every trip back from the kernel passes through one gate in `kernel/entry/common.c`:

```c
/* kernel/entry/common.c — exit_to_user_mode_loop() */

if (ti_work & (_TIF_SIGPENDING | _TIF_NOTIFY_SIGNAL))
    arch_do_signal_or_restart(regs);
```

This is the delivery boundary, and it is the only place signals become real.

This design isn't laziness; it's the only safe option. A signal handler is user code. The kernel cannot execute user code from arbitrary kernel context - the process might be holding locks, mid-way through a filesystem operation, halfway through modifying its own page tables. The boundary is the one point where the task's state is clean, consistent, and provably safe to redirect.

## The Execution

So the process crosses the boundary, sees TIF_SIGPENDING raised, and calls get_signal() to dequeue what's waiting. For a handled signal, the kernel rewrites the process's own user stack so that it "returns" into its signal handler - the process runs its own handler, in its own context, on its own time. And for a fatal signal, `get_signal()` ends like this:

```c
/* kernel/signal.c — get_signal() */

fatal:
    ...
    /*
     * Death signals, no core dump.
     */
    do_group_exit(signr);
    /* NOTREACHED */
```

Notice what get_signal() does not do. It doesn't locate the target process, send an RPC, or schedule a kernel worker to terminate it. It simply calls do_group_exit(signr) with no task argument at all. That's because get_signal() is already executing in the context of the task that received the signal. When do_group_exit() runs, current is the victim. The process dequeues its own death sentence and executes it itself.

The kernel is not an executioner. It is a postal service with a very persuasive letterhead.

## "But Surely SIGKILL Is Different"

SIGKILL feels like it should be a different mechanism - the kernel's real weapon, the one that bypasses all of this. It isn't. It goes through the same funnel, the same bitmask, the same boundary. When a fatal signal will take down a whole process, complete_signal() walks every thread and does the only thing it can do - set bits and wake people up:

```c
/* kernel/signal.c — complete_signal() */

__for_each_thread(signal, t) {
    ...
    sigaddset(&t->pending.signal, SIGKILL);
    signal_wake_up(t, 1);
}
```

SIGKILL is special in policy, not mechanism. The kernel refuses to let you block it, ignore it, or install a handler for it - so when the bit is seen, death is the only possible outcome. But "when the bit is seen" is still the operative clause. SIGKILL doesn't kill the process. It makes refusal illegal. The victim still has to show up to its own execution.

## The Unkillable

Which brings us to the production mystery this model finally explains: the D-state process. The one stuck waiting on a dead NFS server, a dying disk, a wedged driver. The one where kill -9 does nothing, and then nothing, and then nothing again.

Look at the wake-up call SIGKILL sends. signal_wake_up(t, 1) wakes interruptible sleeps, stopped tasks, and tasks in TASK_WAKEKILL - sleeps that agreed to be woken by fatal signals. One state is conspicuously missing from that list: plain TASK_UNINTERRUPTIBLE — D state - which opted out of all wake-ups. It told the scheduler: do not wake me for anything until my I/O completes. The SIGKILL bit gets set. The wake-up bounces off. The task never runs, never reaches the boundary, never reads its mail. The note sits in /proc/PID/status, technically fatal, practically decorative, until the hardware responds - which, with a dead NFS server, can be never.

Your kill -9 didn't fail. It succeeded completely: the bit is set. There was simply never anything more it could do. The man pages told you SIGKILL "cannot be caught, blocked, or ignored" - and that's true at the API level. Nobody promised the process would ever be scheduled again.

The kernel community knew this was a real wound, and the fix is a small masterpiece of design honesty. In 2008, kernel 2.6.25 added a new sleep state:

```c
/* include/linux/sched.h */

#define TASK_KILLABLE    (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
```

`TASK_KILLABLE` is an uninterruptible sleep with one exception: fatal signals may wake it. Notice what the fix was not. It was not "make SIGKILL stronger" - there is no stronger; setting a bit is all the machinery there is. The fix was to teach sleeps to listen. Code that sleeps in TASK_KILLABLE (much of NFS was converted) can now be killed mid-wait. Code that still sleeps in plain TASK_UNINTERRUPTIBLE still can't. Eighteen years later, the conversion is still incomplete - which is why you still, occasionally, meet a process that cannot die.

## The Model

Here is the full path, end to end:

```text
/* sender's work is finished */
  kill(pid, sig)
        │
        ▼
/* one bit, set */
  sigaddset(&pending->signal, sig)
        │
        ▼
/* a note and a nudge */
  TIF_SIGPENDING + wake_up
        │
        ▼
/* the gap */
  ...victim gets scheduled...
        │
        ▼
/* the delivery boundary */
  exit_to_user_mode_loop()
        │
        ▼
/* read the note */
  get_signal()
        │
        ▼
/* execute the sentence */
  do_group_exit()
```

Every signal in Linux lives somewhere on this path, and every signal mystery is a question about the gap in the middle. Why didn't my SIGTERM arrive? The process hasn't crossed the boundary. Why do signal handlers run at weird times? Because "weird times" are syscall returns. Why can't I kill a D-state process? Because the gap is infinite.

Once you see it, the design principle underneath is bigger than signals: the Linux kernel almost never does things to processes. It arranges for processes to do things to themselves. Page faults, scheduling, signal delivery, even death - the kernel sets up the conditions and lets the task walk into them. Control without intervention.

kill() doesn't kill. It asks. SIGKILL just makes it illegal to say no, and somewhere on your systems right now, a D-state process is not saying no. It just never picked up its mail.
