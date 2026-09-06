+++
date = "2025-11-30"
draft = false
title = "Importance of Reference Counters (krefs) For Kernel Objects in Linux"
+++

When you are writing code to handle concurrency, you might think:

*“But I never free the object while it’s still in use. I’m careful.”*

The thing is:

In concurrent code, “I’m careful” is not a strategy.

If your object can be shared, passed around, enqueued, handed to another thread, or looked up in a table… …then without ref counts, you are relying on human memory to enforce object lifetime.

And that does not scale.

This is exactly where ***krefs**** *come in.

***struct kref***, was created to provide a simple, and hopefully failproof method of adding proper reference counting to any kernel data structure.

You embed this reference counter inside your object:

That’s it.

It can live anywhere in the struct.

Then, right after you allocate the object:

That call to ***kref_init()*** sets the reference count to 1.

From this point on, you no longer “guess” who owns the object.

Instead, you follow a **simple discipline**:
- Whoever *holds* a usable pointer has **a reference**.
- When you *share* that pointer, you **increment** the refcount.
- When you’re done, you **decrement** it.
- And when the count hits zero… *one* well-defined cleanup function runs and frees the object.

No more “I hope this isn’t freed yet.”

It’s either referenced… or it’s gone.

And the API is tiny:
- ***kref_init()*** – start with 1 ref
- ***kref_get()*** – add a reference
- ***kref_put()*** – drop a reference, and maybe free

But the magic is not in the functions.

It’s in **the rules** you follow around them.

## Core Rules to Follow:

### Rule 1: If you hand off a pointer, you must get a ref first

If you make a non-temporary copy of a pointer, especially one that can be:
- Stored somewhere, or
- Used by another thread, or
- Enqueued on a list or queue,

you **must** call:

**before** handing it off.

Why “before”?

Because the moment you pass that pointer away, you’ve promised:

>

“This object will stay alive at least until that other code drops its ref.”

Instead, if you do:

you’ve introduced a race: the new thread might *already* be using or dropping the pointer before you’ve bumped the refcount.

So the pattern should be:

Now the new thread owns that extra reference. When it’s done, it calls:

…and your original code later does its own ***kref_put()***.

When the *last* one hits zero, **and only then**,  runs and frees the memory.

### Rule 2: When you’re done, you must call kref_put()

Whenever your code is finished with an object, you call:
- If you’re *not* the last one, nothing is freed.
- If you *are* the last one,  runs.

A typical release function looks like this:

This is the **only place** that  should ever be called for that object.

Everywhere else, you just call **kref_put()** and trust the refcount.

### Rule 3: If you don’t already have a valid ref, you must serialize get vs. put

This one is the trickiest, and it’s where people usually get bitten.

If you don’t already hold a valid reference, you must protect the  with a lock **while finding the object** - otherwise the object might disappear during lookup.\

Imagine This Scenario:

We have one global pointer to a shared object:

Only one object.

No lists.

No threads.

Just a global pointer that can be set to NULL.

### What we want to do

We want a function that:
1. Finds the object
2. Gets a reference to it (so it cannot be freed while in use)
3. Returns it

Let’s try the **WRONG** version first.

Looks harmless, but here's the race:

The problem:

You **looked up the pointer without ensuring** it stayed alive.

At that moment, you **did not own a ref** yet → Rule #3 violated.

**CORRECT** method - **Protect Lookup + Get Together**

You must **hold a lock** during the entire critical action:

Now no one can:
- drop the last ref
- free the object

between lookup and increment.

So when you return , you **definitely** own a reference.

And when you’re done?

So, let’s recap the mental model:
- Your kernel objects aren’t “owned” by one magical place.
- They’re owned by whoever holds a **reference**.
- **kref_get()** to extend it.
- **kref_put()** ends it and when the last one drops, your release function runs and only then is the object really gone.

If you follow the three rules…
1. **Get a ref before you hand off a pointer**,
2. **Put your ref when you’re done**,
3. **Serialize lookup + get when you don’t already hold a ref**,

…you dramatically reduce use-after-free bugs, weird crashes, and subtle race conditions.

Now here’s your challenge:

**Look at one subsystem you’ve worked on - kernel or user space - and ask yourself:**

“*Where am I secretly relying on people to remember object lifetimes instead of using refcounts?*”
