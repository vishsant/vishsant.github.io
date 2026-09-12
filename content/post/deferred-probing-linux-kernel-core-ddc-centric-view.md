+++
date = "2026-01-21"
draft = false
title = "Deferred Probing in the Linux Kernel Core: A dd.c – Centric View"
+++

Every time you plug in a USB drive, connect a wireless mouse, or boot your laptop, the Linux kernel faces thousands of split-second decisions. Each device that appears must find its driver. Each driver must decide whether it can handle that device. And in the space between those lives a surprisingly complex decision tree that determines whether your hardware works at all.

Most of the time, this process is invisible. The system boots. The devices work. We move on with our day.

This article examines a single file in the Linux kernel: ***drivers/base/dd.c***.

Despite its unassuming name "*device driver binding"* it contains some of the most consequential logic in the driver model. Not the logic for how drivers operate, but for what happens when they cannot.

When a driver attempts to bind to a device, three outcomes are possible:
- The binding succeeds.
- The binding is deferred.
- The binding fails permanently.

The first outcome is routine. The third is final.

The second is where all the complexity lives.

This is not a line-by-line walkthrough. It is an exploration of the decision-making logic behind deferred probing: how the kernel reasons about time, dependency, and uncertainty when its view of the system is only partially formed.

### The Unordered Boot

A computer does not boot in an orderly sequence.

The CPU starts, memory initializes, and the kernel loads. Devices then appear in arbitrary order. A USB controller may register before the keyboard attached to it. An I²C temperature sensor may appear before the bus it depends on.

The kernel has no master schedule. It cannot stop the boot process and wait for all hardware to settle. Drivers probe devices as soon as they can, which often means they probe too early.

When a device fails to probe because a required resource is missing, it is not discarded. Instead, its identity is placed onto a waiting list. A mutex protects this operation, ensuring that only one thread modifies the queue at a time.

This list is called the **deferred probe pending list**.

```c
static LIST_HEAD(deferred_probe_pending_list);
```

The device is what enters the deferred queue, not the driver. This kernel's list tracks devices that haven't found a match, preserving them for another round of driver matching.

This is how Linux tries to impose order on randomness.

The list is mostly invisible, but a ledger exists in debugfs for the curious (***/sys/kernel/debug/devices_deferred***). Yet the system’s ability to bring itself online depends on its integrity.

Devices on this list are not broken. They are not misconfigured. They are simply early.

Linux makes a critical distinction between “no” and “not yet.”

Deferred probing exists entirely to preserve that distinction.

### The Signal to Wait

A device does not enter the deferred list by accident.

The probe function is the introduction between a device and a driver. If this probe handshake fails, the driver may return a special error code:

```c
ret = call_driver_probe(dev, drv);
if (ret == -EPROBE_DEFER)
    break;
```

***-EPROBE_DEFER***

This is not an error in the conventional sense. It is a statement.

*“I cannot work now, but I might work later.”*

The kernel records this request without judgment. A debug message may be logged. The device is marked* can_match = true*. It is eligible to match, just not now.

The decision to return *-EPROBE_DEFER* rests with the driver author. It is a human judgment encoded in C. The author must ask what is absolutely required to proceed.

A regulator? A clock? A GPIO? Another device?

Crashing is one option. Giving up is another. Deferring is a third.

Deferring is an **act of conditional optimism**. It assumes the system will eventually complete itself.

But the kernel does not trust optimism blindly.

The kernel maintains a dependency graph between devices. When Device A needs something from Device B (like a clock, power regulator, or GPIO controller), the kernel creates a "device link" marking B as A's supplier.

Before even calls your driver's probe function, it checks this graph:

```c
link_ret = device_links_check_suppliers(dev);
if (link_ret == -EPROBE_DEFER)
    return link_ret;
```

If a declared supplier device is not ready, the kernel may return -EPROBE_DEFER on the driver’s behalf.

The signal to **wait** therefore **comes from two places**: the **driver’s own discovery** that a resource is missing, and the **kernel’s abstract reasoning** that the dependency graph is incomplete.

Both paths converge on the same outcome.

Wait.

### The Timeout Mechanism

Waiting cannot be infinite. Hope, left unchecked, becomes deadlock.

To prevent this, the kernel enforces a timeout. Deferred probing is governed by a configurable limit set via the kernel command line:

```text
deferred_probe_timeout=
```

The default behavior is, Modular kernels wait, typically around ten seconds. Monolithic kernels do not wait at all.

When the timeout is set, a delayed work item is scheduled.

```c
schedule_delayed_work(&deferred_probe_timeout_work,
                      driver_deferred_probe_timeout * HZ);
```

When the timer expires, patience ends.

First, the kernel signals that firmware-based device links are finalized: *fw_devlink_drivers_done()*.

Second, it closes the window during which deferral is considered valid by setting: *driver_deferred_probe_timeout = 0*. No more extensions.

Third, it triggers a final, comprehensive retry: *driver_deferred_probe_trigger().* This moves every waiting device from the pending list to the active list, emptying the pending list entirely, and schedules the deferred probe workqueue to attempt one last round of matching.

Then comes crucial synchronization: *flush_work(&deferred_probe_work)*. This waits for that final retry wave to complete. During this retry, devices that fail again are added back to the pending list. The flush ensures all retries finish and the pending list stabilizes. Only after this, when the final matching is irrevocably complete, does the kernel examine what remains.

```text
Before timeout: [D1, D2, D3]
Retry: D1 succeeds, D2 defers, D3 succeeds
Final pending list: [D2]
```

It walks the pending list, now containing only devices that failed even this last chance and for each, prints the warning (in our case, D2):

```text
deferred probe pending: (reason unknown)
```

An epitaph.

The device had its final, uninterrupted opportunity.

Finally, the kernel signals that all probing is conclusively finished: *fw_devlink_probing_done()*.

The window of patience has closed.

This is a survival strategy.

A system that boots imperfectly is more useful than a system that never boots at all.

### The State Check

Sometimes the kernel knows waiting is futile from the start.

This insight comes from a function that serves as the kernel's own reality check: *driver_deferred_probe_check_state()*.

The function contains a cold, logical assessment:

```c
if (!IS_ENABLED(CONFIG_MODULES) && initcalls_done)
    return -ENODEV;
if (!driver_deferred_probe_timeout && initcalls_done)
    return -ETIMEDOUT;
return -EPROBE_DEFER;
```

When the kernel is compiled without module support (CONFIG_MODULES=n), every possible driver is already built into the kernel image. The flag *initcalls_done* becomes true after all built-in initialization functions have run. At this moment, the kernel knows the complete cast of drivers has taken the stage. If a device hasn't found its match by now, it never will. Waiting becomes delusion.

If the deferred probe timeout has reached zero and *initcalls_done* is true, the window of patience has officially closed:

In either case, the device is not re-queued. The probe fails permanently, and the system moves on.

This allows kernel to decide if the waiting is rational or not.

Only if both checks fail does the function return *-EPROBE_DEFER*. It defer only if there is some hope ahead.

### The Optimist’s Engine

Deferred probing is not just a mechanism. It is a worldview.

It assumes dependencies arrive out of order, readiness is local rather than global, and time can help, but only briefly.

From two linked lists, a mutex, a timer, and a workqueue, Linux constructs a decision engine capable of handling uncertainty.

```c
static LIST_HEAD(deferred_probe_pending_list);
static LIST_HEAD(deferred_probe_active_list);
static DEFINE_MUTEX(deferred_probe_mutex);
```

This is the Linux way.

Simple primitives. Clear states. Hard boundaries.

It waits precisely, retries deliberately, and lets go decisively.

That is the beauty of *dd.c*.

The act of structured patience, the optimism with an escape hatch.
