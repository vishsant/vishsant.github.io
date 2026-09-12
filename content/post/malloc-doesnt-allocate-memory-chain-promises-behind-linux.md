+++
date = "2026-03-15"
draft = false
title = "malloc() Doesn’t Allocate Memory: The Chain of Promises Behind Linux Memory"
+++

There is a line of code that appears in almost every C program:

```c
void *p = malloc(1024);
```

Every programmer learns the same idea:

malloc allocates memory.

But this belief is wrong.

It has always been wrong.

malloc does not allocate physical memory.

It allocates virtual address space.

And the difference between those two things explains some of the strangest behaviors in Linux systems.

It explains why:
- a program can allocate 2GB and use 200MB
- a 50GB process can fork instantly
- a machine with 64GB RAM can promise 300GB

And sometimes…

why a critical service disappears like a ghost.

This article traces the path from a malloc call to physical RAM.

By the end, you will see the full chain.

And you will never think about memory allocation the same way again.

## The Allocation

In the early days of computing, malloc meant exactly what people think it means today.

If you asked for memory, the operating system gave you physical memory.

Simple.

But modern systems work differently.

When you call malloc, the C runtime may ask the kernel for more memory using:

```text
brk()
mmap()
```

The kernel responds by extending the process's virtual address space.

But it does not assign physical RAM yet.

Instead it updates a structure called the **page table**.

The entry simply says:

This address range is valid.

But the physical page behind it does not exist yet.

Your pointer is real.

The memory is not.

Think of it like a restaurant reservation.

You have a table number.

But the food has not been cooked yet.

## The Address

To understand what malloc gives you, you need to understand what an address is.

Every process on a modern Linux system believes it owns an enormous, private memory space.

This is a fiction. A productive one.

The hardware enforces this fiction through a mechanism called the **Memory Management Unit, or MMU**.

Every memory access your program makes goes through the MMU.

The program says: I want the byte at address 0x7f3a00001000. The MMU translates that virtual address into a physical address that points to actual RAM.

The **translation table that connects virtual addresses to physical locations is the page table.**

The **kernel maintains one for each process**. When the kernel grants your malloc request, it adds entries to your page table.

These entries say: this virtual address range is valid. But many of those entries point to nothing.

**An entry with no physical backing is not an error.** It is a deliberate state.

The kernel creates it on purpose.

The entry exists so that the MMU knows the address is legal.

But the physical page has not been assigned.

```text
Virtual Address Space (per process):
  0x0000000000000000
  ├── [text]   mapped to physical pages
  ├── [data]   mapped to physical pages
  ├── [heap]   partly mapped, partly empty  ← malloc lives here
  ├── [ ...vast unmapped gap... ]
  ├── [stack]  mapped on demand
  0x00007FFFFFFFFFFF
```

The heap region grows when malloc asks the kernel for more space. But "grows" means the kernel extends the valid address range. It does not fill that range with physical memory.

Your process sees a flat, continuous space. The kernel sees a sparse map.

Most of the entries in your page table lead nowhere.

This is **virtual memory**.

A fundamental layer of **indirection between your code and the hardware**. Every pointer you hold is a virtual address. Every access goes through **translation**. And the translation can fail.

When it fails, something important happens.

## The Fault

Now imagine the first write:

```c
buf[0] = 'A';
```

The CPU sends the virtual address to the MMU.

The page table says:

Address is valid. But no physical page exists.

So the hardware raises a **page fault**.

Despite its name, this is not an error.

It is a request for help.

The kernel handles the fault:
1. find free physical page
2. zero the page
3. update page table
4. restart instruction

```text
CPU: load [virtual address]
  ↓
MMU: page-table walk → valid, but no physical page
  ↓
page fault → kernel allocates and maps a page
  ↓
CPU retries the instruction → succeeds
```

The program never notices.

Demand paging has just turned a promise into real memory.

## The Clone

The fork() system call creates a copy of a process.

In theory, copying a process with 30GB of memory should require copying 30GB of RAM.

But fork() returns in microseconds.

Because Linux does something clever.

Instead of copying pages, it shares them.

Both processes reference the same physical pages.

They are marked read-only.

If either process writes to the page:
1. hardware raises a fault
2. kernel allocates new page
3. data is copied

This is called **Copy-On-Write**.

**Memory is copied only when needed.**

## The Gamble

Now imagine a machine with:
- 16GB RAM
- 8GB swap

Total memory: 24GB

A strict system would reject allocations beyond that.

Linux usually does not.

By default it overcommits memory.

It approves allocations larger than available RAM.

Why?

Because **most programs never use everything they allocate.**

Java might reserve 8GB but touch 2GB.

A database might allocate buffers it rarely fills.

Linux bets that not everyone will collect their promises.

Most of the time, the bet works.

## The Reckoning

Sometimes the bet fails.

Processes start touching pages simultaneously.

Page faults explode.

RAM fills.

Swap fills.

The kernel cannot satisfy new page faults.

Now it must make a choice:
1. freeze the system
2. crash the machine
3. kill a process

Linux chooses the third.

The **OOM Killer** activates.

It calculates a score for each process based on:
- memory usage
- priority adjustments
- ownership

This is the "badness" score of the process.

Then it selects a victim with the highest score.

It sends SIGKILL.

The process dies without cleanup.

Immediate termination.

In theory, the OOM killer should choose the process consuming the most memory. In practice, the scoring produces surprises.

A database with a large but stable memory footprint scores high.

A small, transient script that caused the spike scores low.

The script triggered the crisis. The database pays the price.

So, sometimes the wrong process dies.

Because of this, Linux provides a way to make a process immune to the OOM killer.

```bash
# Check current OOM score
cat /proc/$(pidof postgres)/oom_score

echo -1000 > /proc/$(pidof postgres)/oom_score_adj
echo 1000 > /proc/$(pidof backup-script)/oom_score_adj

[Service]
OOMScoreAdjust=-500
```

With this, even if a process sacrifice, the system survives.

## The Chain

Everything begins with one line:

```c
void *p = malloc(1024);
```

And unfolds like this:

```text
malloc() → virtual address → first write → page fault
→ physical page allocation → fork() copy-on-write
→ overcommit promises → OOM killer collects the debt
```

Every step defers cost.

Every step optimizes the common case.

Every step makes a promise about future memory.

Most of the time those promises hold.

But sometimes they do not.

And that is when systems engineers learn how memory really works.

The next time you write that line, you will feel the weight of the chain behind it. Not as fear. As clarity.

You will know that the pointer is a promise, that the promise unfolds across time, and that the cost of that promise is never zero.

It is only deferred.
