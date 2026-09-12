+++
date = "2025-12-13"
draft = false
title = "The Predictable Outcome of Blurry Ownership in Linux Kernel."
+++

You lend your friend ₹2000.

A week later, they pay you back.

You thank them, put that money in your pocket and move on.

Two weeks later, the same friend walks up and says: "Hey, here's that ₹2000 I owe you."

You're confused.

"You already paid me back," you say.

They insist they haven't.

You check your wallet - the money is there. They check theirs - they have ₹2000 available to give you again.

Both of you trust your own belief.

Something fundamental went wrong here: **the boundary of ownership.**

In the Linux kernel, memory management works the same way.

When you allocate memory, the kernel promises: *"This space is yours."*

When you free that memory, you're saying: *"I'm done, take it back."*

The kernel trusts you completely.

It doesn't keep a ledger of who's paid what. It assumes you're keeping track.

**A double free is when you return the same memory twice.**

Like paying back the same debt twice when you only borrowed once.

Let’s walk through a common scenario:

```c
void *ptr = kmalloc(256, GFP_KERNEL);
kfree(ptr);
void *ptr2 = kmalloc(256, GFP_KERNEL);
kfree(ptr); /* BUG: ptr2 may be using this memory. */
```

The kernel did nothing wrong. It followed its side of the contract faithfully.

But **your pointer didn’t know its owner changed.** It still pointed to an address that now belonged to someone else.

This is **not a memory problem. **It’s a** boundary problem.**

## The Library That Trusts Too Much

Think of the kernel's memory system as a library.

When you check out a book, you promise to return it once.

If you somehow return the same book twice:
- Maybe you photocopied the barcode
- Maybe you walked through checkout twice
- Doesn't matter how

The library's system breaks:
- It might assign that book to two people
- It might lose track of inventory
- It might mark other books as returned when they're not

Is the library's system buggy?

No.

It's operating under the assumption that you won't deliberately or accidentally break the protocol.

The Linux kernel does the same.

Not because it is naive, but because the cost of enforcing ownership would destroy its performance model.

## The 4 Invisible Ownership Models

Boundary violations occur because different parts of your kernel code silently assume **different ownership semantics**.

Here are the four models constantly interacting inside the kernel:

**1. Exclusive Ownership:**

One creator. One destroyer. Simple until you pass the object somewhere.

**2. Reference-Counted Ownership:**

Multiple owners. Last one out turns off the lights. Complex because everyone needs to remember to decrement.

**3. Delegated Ownership:**

"Here, you deal with this now.". Dangerous because both parties might think they still own it.

**4. Borrowed Ownership:**

"I'm just looking, not freeing.". Safe unless you forget you're just looking.

**The problem:** Your code lives in one model. The function you're calling lives in another. The boundary between them is invisible.

A Real Example: The USB MIDI Double Free:

```c
midi = kmalloc(...);
if (register_audio_device(&midi->audio) < 0)
    goto error;
if (register_usb_interface(midi) < 0)
    goto error;
return 0;
error:
    kfree(midi); /* WRONG: ownership was transferred */
```

**The boundary violation:** Registration transferred ownership. Error handler didn't respect that transfer.

## Debugging Double Free: The Archaeology of Corruption

An archaeologist uncovers a tomb dated to 1000 BCE.

But the artifacts tell a confusing story:

A Roman coin.

A shard of medieval pottery.

All point to events centuries after the tomb was sealed.

Impossible.

The archaeologist realizes:

The site is a place written over by time, holding multiple, overlapping histories.

The coin? Left by a later explorer.

The pottery? Debris from a settlement above.

Understanding it requires disentangling time itself.

This is what debugging a double free corruption feels like.

When you investigate a crash caused by a double free:

You're doing digital archaeology.

Trying to find where the timeline actually fractured, not where the fracture is first observed.

The crash shows you the final collapse:
- Corrupted data structure
- Segfault
- Kernel panic

But the original logic error, happened earlier.

Somewhere else entirely.

The memory has been overwritten by the program's own subsequent history.

Double free shows symptoms far away from the cause:
- A network packet structure crashes
- But the real mistake was freeing a driver object
- 200 microseconds earlier
- On a different CPU
- In a different subsystem

This is why stack traces lie:

But the driver still thinks it owns the object:

Later, the network subsystem tries to free the same object:

Stack trace shows network code. But the bug is in driver code.

Crash scene lies about the cause.

Because of this, traditional debugging fails:
- Stack traces point to innocent victims
- Core dumps show aftermath, not cause
- Reproduction is inconsistent (depends on allocation patterns)

This is why double free is so dangerous.

The code that calls *kfree()* twice might be perfectly valid C.

It compiles.

It might even run successfully 99% of the time.

>

***The "bug" exists in the semantic layer: In the meaning of what you're doing, not the mechanism.***

**Spatial separation:** Allocation, frees, and corruption in completely different parts of the codebase.

**Temporal separation:** Corruption might not manifest until much later.

**Logical separation:** Crash occurs in code that had nothing to do with your mistake.

You crossed a boundary.

The consequences rippled through spacetime in your program's execution.

## Tools That Turn Archaeology Into Engineering

You need specialized tools to debug since the logging lies.

### KASAN (Kernel Address Sanitizer)

The cost?
- Memory overhead (shadow memory)
- Performance impact (instrumentation)

This is why you can't run it in production.

It's archaeological equipment, not everyday tools.

### Temporal Markers

Leave breadcrumbs in your code:

Later, if you see MAGIC_FREED in active memory:

You've found corrupted remains.

### Poison Patterns

Explicitly mark freed memory:

When you see 0xDEADBEEF in a crash dump:

You know that memory was freed.

### Assertion Guards

Make invisible rules visible:

The above statement says,* "When this function is called, the caller must be holding exactly one reference to this object. If not, something is wrong."*

Because refcounts represent how many owners currently exist.

**If refcount == 1 **means:
- The caller is the unique owner
- Nobody else is holding the device
- It is safe to transfer ownership, increment, or move the object into a global list

**If refcount != 1 **means:
- Someone else also holds a reference
- You’re not the exclusive owner anymore
- Freeing, unregistering, or modifying the object now becomes unsafe
- A double free or use-after-free is brewing

The assertion catches this **the moment it happens**, not after the memory gets corrupted.

This approach catch boundary violations early, not after a kernel panic.

### Why The Kernel Can't (And Shouldn't) Protect You

Some or you might ask: *"Why doesn't the kernel just check? Why kernel add guards instead of asking dev to do all these?"*

Sounds reasonable, until you actually run the numbers.

Consider the cost:

**Memory overhead:** Every allocation needs a tracking entry. Linux systems do millions of allocations per second.

Let’s imagine a lightweight tracking entry of just 32 bytes:
- 1 Million allocations/sec = 32MB/sec of metadata for just one second worth of live allocations.
- Over a minute: 1.9GB of RAM consumed purely for tracking metadata.
- Multiply by number of CPUs, or by subsystems with per-CPU allocators, and the cost becomes absurd.

Even if you had the RAM, you now need machinery to maintain that metadata:
- Lock acquisition (contention on multi-core)
- Hash lookup (CPU cycles)
- What about tracking partial frees?
- How long do you track freed pointers?

So, kernel made a **philosophical choice**:

*"I choose to trust the programmer to be disciplined enough to maintain boundaries, to build a fast, efficient system."*

## Systems Consciousness: Seeing the Boundaries

When you learn to see boundaries, you stop writing “fixes” and start writing systems.

**You see:**
- Where ownership is transferred
- Where responsibility ends
- Where assumptions change context
- Where invisible contracts exist

**You ask:**
- Who owns this now?
- What happens on failure?
- Is this boundary explicit or assumed?
- Does this function take or observe?

That’s systems consciousness.

It’s what keeps large, shared, high-speed systems sane.

## Five Practices to Make Boundaries Visible

Building reliable systems isn't about eliminating all bugs.

That's impossible.

It's about embedding system consciousness:

**1. Name Ownership in APIs**

**2. Null Out Pointers After Transfer:**

**3. Document Failure Contracts Clearly:**

Functions must say:
- On success: who owns
- On failure: who frees

**4. Use Scope-Based Cleanup:**

Automatically clean resources on exit paths.

__cleanup is a GNU C extension.

It tells the compiler:

*“When this variable goes out of scope, whether by return, goto, or error, automatically call this cleanup function on it.”*

**5. Assert Invariants:**

Catch boundary violations at the border, not at the crash site.

## Why Boundary Violations Matter Beyond Code

You're driving on a two-lane road.

No physical barrier between you and oncoming traffic. Just a painted yellow line.

Every car approaching at 60 mph is:
- Piloted by a stranger
- Whose skill you don't know
- Whose attention you can't verify
- Whose judgment you can't predict

You're separated from a head-on collision by:
- Paint
- Convention
- Mutual agreement
- Trust

This system works because everyone understands the boundary. Everyone has personal responsibility not to cross it.

We could build physical barriers between lanes. The cost would be astronomical. The efficiency loss massive.

Instead, we chose: Trust + accountability + clear boundaries.

Linux kernel development operates the same way.

When you call *kfree()*, you're entering a social contract with kernel and other developers contributing to the kernel.

The kernel's side:

"I promise to:"
- Accept your memory return
- Make it available for others
- Manage fragmentation efficiently
- Use minimal metadata

"I require you to:"
- Free only what you allocated
- Free each allocation exactly once
- Not access memory after freeing
- Pass valid pointers or NULL

"I do NOT promise to:"
- Validate your pointers
- Track whether you've freed before
- Prevent you from shooting yourself
- Detect your boundary violations

This contract is explicit, documented, and non-negotiable.

Breaking it doesn't make the kernel buggy.

It makes you a contract violator.

---

The kernel doesn't protect you from double free.

Because it's teaching you something more valuable: *How to see boundaries and maintain them yourself.*

That's not a bug. That's a feature.

Because the most reliable systems aren't built on protection. They're built on clarity, discipline, and conscious practice.

And if you are not disciplined, every boundary crossed has consequences.

Some immediate. Some delayed. Some obvious. Some hidden.

So, pull up a module you wrote six months ago. Find every function that allocates memory. Ask yourself:

***"If someone called this function, would they know whether they own the result?"* *"If they free it, would they know if someone else might too?"* *"If this function fails halfway through, is ownership unambiguous?"***

If the answer is "I'm not sure," you have invisible boundaries.

And invisible boundaries always get crossed.
