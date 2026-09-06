+++
date = "2026-02-25"
draft = false
title = "The Physics of Software Time: The Hidden Complexity of Linux Timekeeping"
+++

There is a moment most engineers experience at least once.

Two services.

Two log files.

One sequence of events.

Event B happened six milliseconds before Event A.

Impossible.

Except it isn’t.

The logs are not lying.

The clocks are.

And every process on that host - every service, every application, every line of code - relies on those same clocks.

When one asks the time, they all hear the same approximation.

This article is about that approximation.

And about why trusting clocks blindly is one of the most subtle design mistakes in distributed systems.

By the end, you will not trust timestamps the way you used to.

That loss of trust will make you a better engineer.

---

## The Assumption Nobody Questions

We grow up with a simple model of time:
- It moves forward.
- It moves at the same rate for everyone.
- It is a number.

Engineering inherits that assumption.

Physics disagrees.

Every computer keeps time using a crystal oscillator. The crystal vibrates. The chip counts vibrations.

But crystals drift.

Temperature changes them. Age changes them. Manufacturing variance changes them.

Two machines started at the same time will not stay synchronized. They can be milliseconds apart within hours.

Now scale that across continents.

Then ask those machines to agree on:
- Payment ordering
- Log sequencing
- Conflict resolution
- Cache invalidation

You are asking drifting hardware to produce a coherent history.

**Linux** knows this. That is why it does not **give you** one **clock**. It gives you **four**.

---

## Four Clocks, Four Different Questions

Time is not one thing.

It depends on what you’re measuring.

**1. CLOCK_REALTIME**

**Wall clock time**.

Seconds since Jan 1, 1970.

This is what ***date*** command shows. What humans mean by "time."

It can move forward.

It can move move backward if NTP steps.

It can forward instantly if NTP steps.

An admin can change it.

Leap seconds affect it.

**Never use this for durations.**

Here is what happens when you do:

Why can it go backward?

NTP (Network Time Protocol) can step the clock, just like you manually rotate your watch time by adjusting the dial - to correct large errors. We'll explore exactly how this works later.

This is not theoretical. It happens in production.

**2. CLOCK_MONOTONIC**

**Elapsed time since boot** (roughly).

It never moves backward.

It's unaffected by NTP adjustments or admin changes.

Perfect for:
- Timeouts
- Intervals
- Measuring durations

But it ignores suspend.

A laptop sleeps for 8 hours. CLOCK_MONOTONIC pretends nothing happened.

What every engineer actually uses it for:

**3. CLOCK_MONOTONIC_RAW**

It reads **directly from the hardware counter (TSC).**

No NTP adjustments. No smoothing.

It's always increasing.

Useful for:
- Hardware profiling with calibration tools
- Scientists measuring oscillator behavior

What breaks when you use it wrong:

Almost never correct for application logic.

**4. CLOCK_BOOTTIME**

Like MONOTONIC, but includes suspend time.

It's always increasing. Not affected by NTP.

Useful when uptime must include sleep. Rarely correct otherwise.

When you actually need it:

So why not always use BOOTTIME? Because it's slightly slower to read (more kernel work).

When you actually need it:
- Services that must run after suspend (cron jobs, backup schedulers)
- Uptime monitoring that should include sleep
- Anything that needs "total time since boot including naps"

Use CLOCK_BOOTTIME only when your code must account for system sleep. For most timeouts and intervals, MONOTONIC is what you want.

---

Four clocks. Four different answers to “what time is it?”

Most code never asks which question it is asking.

---

## The Kernel’s Shortcut (vDSO)

Reading time millions of times per second is expensive.

A syscall costs roughly one to four microseconds on modern hardware. For most operations this is fine. For reading time millions of times per second, it adds up fast.

So Linux cheats.

The vDSO (virtual Dynamic Shared Object) is a clever optimization used for that.

When a Linux process starts, the kernel maps a small region of its own memory into the process's address space. This region is read-only from the process's perspective. Inside it, the kernel continuously writes updated time information.

When your code calls *clock_gettime()*, the C library does not make a system call. Instead, it reads from this mapped region directly. The time is already there, in your process's own memory, maintained by the kernel behind the scenes.

50–100x faster.

Elegant.

But the value inside that page is built on something else.

The TSC.

---

## The Crystal That Drifts (TSC)

The **Time Stamp Counter** (TSC) is a special register in your CPU that increments on every clock cycle. On a 3 GHz processor, it increments three billion times per second. Reading it takes a single instruction.

Fast. Simple. But Dangerous.

**Early CPUs had a problem:** The TSC frequency was tied to the CPU's current clock speed.

Speed up CPU → time moves faster.

Slow down CPU → time moves slower.

If the CPU slowed down to save power, the TSC would also slow down.

This made it unreliable for measuring real-world time.

**Modern CPUs solved this with invariant TSC.** This is a hardware feature that guarantees the TSC increments at a fixed frequency, usually tied to the processor's maximum speed or the system's reference clock. It no longer fluctuates with the CPU's dynamic clock speed.

This constant rate holds true whether the CPU is:
- Running at full speed (P-state)
- Idle in a sleep state (C-state)
- Throttling for thermal management (T-state)

This means you can trust it for timekeeping. Mostly.

---

## The Art of Controlled Correction (NTP)

Even if every machine had a perfect hardware clock, synchronized clocks would still drift. Quartz crystals and oscillators are physical objects in a physical environment. They age. They respond to temperature. They simply do not stay exactly right forever.

NTP is the service that corrects them.

It runs on every production server, quietly doing its job that most engineers forget exists.

NTP periodically asks a trusted time server:

"What time is it, really?"

It measures how long the question and answer took to travel across the network, accounts for that delay, and then compares the answer to its own clock.

If your machine is 50 milliseconds ahead, NTP knows. Then it must decide how to fix it.

Two ways to correct time: Slewing and Stepping.

Normally, NTP uses** slewing.** It gradually speeds up or slows down your system clock - just a tiny bit -until it matches the real time.

From your application's perspective:

Time never jumps backward.

Time never jumps forward.

It just runs slightly faster or slower for a while.

A clock running at 110% speed for a few minutes to catch up is invisible to your code. Timeouts still fire. Logs stay ordered. Everything works.

This is the safe way to correct time.

But slewing has limits.

The kernel will not slew a clock by more than about 500 milliseconds by default. Why? Because fixing a large error by slewing would take too long. A clock that's 5 seconds slow would need to run at 110% speed for 50 seconds to catch up. That's 50 seconds of weirdness.

If the error is larger than ~500ms, NTP does something more drastic: it **steps** the clock.

That means instantly jumping forward or backward to the correct time.

This is the dangerous mode.

Which means: CLOCK_REALTIME can jump. Even in production. Even while your code is running.

## When the Network Disappears

NTP needs to talk to its time servers. It needs the network.

**Network partition** is when the network breaks into pieces that cannot communicate.

Imagine your office building has two wings. At 2:00 PM, someone locks the door between them.
- You can still talk to people in your wing
- You cannot reach anyone in the other wing
- Both wings continue working, but they cannot share updates

That is a network partition.

During a network partition:
- Your machine is running fine
- NTP clients cannot reach their servers
- No corrections happen at all
- Your clock continues drifting, uncorrected
- If the partition lasts an hour, your clock may be seconds off

When connectivity returns, NTP discovers the error. If it's large enough, it may **step** the clock -causing that sudden jump we just discussed.

---

## When Clocks Travel (Distributed Systems)

So far, we've talked about time on a single machine.

But modern systems are not single machines. They are dozens, hundreds, sometimes thousands of machines working together. And they all need to agree on what happened when.

This is where time really starts to break.

Imagine two machines in the same data center:
- Machine A in rack 1
- Machine B in rack 2

Both run NTP. Both are synchronized to within 3 milliseconds of the true time. This is considered good. Many data centers operate with 1–5ms skew between machines.

Now, what does "within 3ms of true time" actually mean?

It means:
- Machine A's clock might be 2ms ahead of reality
- Machine B's clock might be 1ms behind reality
- The difference between them: 3ms

They disagree by 3 milliseconds.

For a human, this is irrelevant. You cannot perceive 3ms. Your brain doesn't register it.

In practice, 3ms delay can cause serious issues.

Let's take a banking example.

Two transactions hit the same account at nearly the same moment, in an account with an initial balance of ₹50:
- Transaction X: Check if balance ≥ ₹100, then withdraw ₹100
- Transaction Y: Deposit ₹100

Now, the order matters:

If Y happens first (deposit then withdraw):
1. Y deposits ₹100 → balance becomes ₹50 + ₹100 = ₹150
2. X checks: ₹150 ≥ ₹100 → withdraws ₹100 → balance becomes ₹50
3. Result: X succeeds, final balance ₹50

If X happens first (withdraw then deposit):
1. X checks: ₹50 < ₹100 → withdrawal fails, balance remains ₹50
2. Y deposits ₹100 → balance becomes ₹150
3. Result: X fails, final balance ₹150

Same two transactions, different order, completely different outcomes.

This is a race condition. And when machines disagree on timestamps due to clock skew, the database may apply the transactions in the wrong order - leading to incorrect business logic.

## How Google Solved This

Google runs some of the largest distributed systems in the world. **Spanner** is their globally distributed database. It spans data centers across continents.

They needed a way to order transactions correctly, even with clock skew.

Their solution is called **TrueTime**.

TrueTime does not return a single number. It returns an **interval**.

Instead of saying: "The current time is 10:00:05.000"

TrueTime says: "The current time is somewhere between 10:00:04.997 and 10:00:05.003"

**The width of that interval is the uncertainty**. Google knows how much their clocks might be wrong (thanks to GPS and atomic clocks in each data center), and they expose that uncertainty directly.

How TrueTime Orders Events:

When Spanner wants to commit a transaction, it does something clever:
1. Read the current TrueTime interval
2. Use the **latest** possible time as the commit timestamp
3. Wait until that timestamp is guaranteed to be in the past

The rule is: **Wait until earliest(commit) > latest(previous)**

Let's see this in action:

**Transaction 1 commits:**
- Reads TrueTime: [10:00:04.997, 10:00:05.003]
- Chooses commit timestamp: 10:00:05.003 (the latest)
- Waits until real time > 10:00:05.003
- Then commits

**Transaction 2 starts after Transaction 1 finishes:**
- Reads TrueTime: [10:00:05.004, 10:00:05.010]
- Earliest possible time (10:00:05.004) is greater than Transaction 1's latest (10:00:05.003)
- Guaranteed: Transaction 2 happened after Transaction 1

The waiting ensures that even if clocks are wrong, the ordering is correct.

TrueTime forces you to think about time differently.

Now time is not a point, it is a range.

You probably don't have atomic clocks in your data centers. You probably can't implement TrueTime exactly.

But you can absorb its lesson:

**If your system orders events by comparing timestamps from different machines, you are relying on luck.**

Because even in the best-managed data centers, clocks disagree by milliseconds. And milliseconds are enough to corrupt order.

---

## The Timestamp Fallacy

The most dangerous assumption in distributed systems:

“A timestamp represents reality.”

It does not.

A **timestamp** represents **what** **one machine believed** at one moment, based on its own imperfect clock, after whatever NTP corrections happened to have been applied, with whatever drift it had accumulated since the last sync.

Timestamps are **evidence**, not truth.

Correct systems treat time as:
- **Approximate** - Never assume it's exact
- **Local** - Don't compare across machines without accounting for uncertainty
- **Bounded by uncertainty** - Know how wrong your clocks might be

If you need strict ordering across machines, don't use wall-clock timestamps. Use tools designed for ordering:
- **Logical clocks** (Lamport timestamps, vector clocks) that track causality, not wall time
- **Consensus algorithms** (Paxos, Raft) that agree on order through communication
- **Sequencers** that assign monotonically increasing IDs
- **CRDTs** that make order irrelevant

These tools do not trust the clock. They trust math and communication.

---

## The Four Clocks Cheat Sheet

---

The number your system returns when you ask for the time is not the time.

It is a carefully maintained estimate. Built from:
- Crystal physics
- CPU counters
- Kernel math
- Network corrections
- Distributed compromise

The clock is almost always right.

Almost always is not always.

And "almost always" is not good enough for systems that depend on ordering events across machines.

The clock is always running.

It is just not always telling the truth.
