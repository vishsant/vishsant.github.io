+++
date = "2026-07-19"
draft = false
title = "How glibc's free() Really Works on Linux"
+++

Every programmer learns the same idea. malloc takes memory from the system, free gives it back. It is clean, symmetric, but incomplete. free() does not give your memory back to the system. malloc and free, both are bookkeeping against a warehouse of memory the kernel already handed to glibc.

## The Return

Start with the belief, because it is almost reasonable. You asked the allocator for memory and it grew your heap to satisfy you. Now you are done, so you call free, and it seems obvious that the heap should shrink back. The symmetry is too clean to question.

The measurement destroys it. I allocated 200,000 small objects, wrote to every one so the pages were truly resident, then freed the first 199,999 objects while intentionally keeping the final allocation - the highest-address chunk, alive.

The result:

Freeing 99.9% of the heap returned nothing. RSS, the amount of physical RAM the kernel currently counts as backing your process, reported in /proc/PID/status, did not fall. It rose slightly, from the accounting glibc did to track the free chunks. The memory only came back when I explicitly asked for it with malloc_trim. To understand why, you have to follow the freed chunk to where it actually goes.

### The Bin

When you call free, the chunk does not leave your process. glibc drops it onto a free list so the next malloc of a similar size can reuse it without ever talking to the kernel. There are several tiers of these lists, the fastest a per-thread cache that needs no locks at all, but the names matter less than the rule they all share: a freed chunk stays in your address space, waiting to be handed back out.

The design goal is speed, and it is the right goal. A syscall costs hundreds of nanoseconds, and the page fault the next time you touch fresh memory costs more. Reusing a warm chunk from a bin costs almost nothing. So glibc bets that a program which freed a 512-byte object will soon want another one, and it keeps the memory close. Think of it like a library that does not ship returned books back to the publisher. It reshelves them, because someone will want them again tomorrow.

The consequence is that free is a local operation. It moves a chunk from "in use" to "available" inside a pool your process already owns. The kernel is not involved and RSS does not move. Returning memory to the operating system is a separate, rarer event. It only happens when glibc can give back free memory from the top of the heap.

### The Wilderness

There is one region glibc can easily give back to the kernel: the top of the heap. The heap grows upward as your program allocates memory, so the highest addresses are the newest ones. The unused space at that end is called the top chunk, historically the wilderness. If that free space becomes large enough, glibc can shrink the heap and return those pages to the kernel. The function that does this is systrim() in malloc/malloc.c:

Read that comment carefully because it explains the entire mechanism. Memory goes back to the operating system only when there is unused space at the high end of the heap, and only after that free space exceeds a configurable threshold. By default, glibc starts trimming when the top chunk grows beyond 128 KB, though that threshold can be tuned with M_TRIM_THRESHOLD. The mallopt(3) man page states it plainly: free() releases memory to the system only "when the amount of contiguous free memory at the top of the heap grows sufficiently large."

Everything hinges on two words. Contiguous, and top.

### The Pin

Now the trap becomes obvious. glibc can only shrink the heap from the top down. If a single live allocation sits at the highest address, everything below it, even gigabytes of freed memory stays trapped inside the process.

That is exactly what my experiment did. I freed the first 199,999 allocations and intentionally kept the last one alive. Because it was the highest-address allocation, the free space below it could never reach the top of the heap. glibc happily merged the freed chunks together, but it still had nothing it could return to the kernel. RSS stayed flat.

Had I kept the first allocation alive instead, the result would have been completely different. As the later allocations were freed, they would have merged into the top chunk, allowing glibc to shrink the heap automatically. RSS would have fallen without any explicit trimming.

This is heap fragmentation in its simplest form. It takes only one long-lived allocation in the wrong place to strand everything beneath it. The memory is freed. The allocator can reuse it immediately. But until the top of the heap becomes free, the kernel cannot reclaim it.

malloc_trim(0) escapes this trap because it does more than shrink the heap. Modern glibc also scans for pages inside the heap that are completely free and releases them with madvise(MADV_DONTNEED). Any page that still contains a live allocation must remain, but pages made entirely of freed chunks can finally be reclaimed.

### The Exceptions

There is one important exception to everything you've just read.

Large allocations often bypass the heap entirely. When a request exceeds glibc's  (128 KB initially), glibc allocates it with a dedicated  instead of carving space from the heap. That allocation lives in its own mapping, completely independent of the heap. When you free one of these large blocks, glibc simply calls . The mapping disappears immediately, and the kernel reclaims the pages. No top chunk. No trimming. No fragmentation trap.

That exception is narrower than it first appears. The  threshold is dynamic: as a program repeatedly allocates and frees large blocks, glibc raises the threshold up to 32 MB on 64-bit systems. Over time, allocations that once received their own mappings may instead come from the heap, where all the rules you've just learned apply again.

 releases memory to the allocator. Returning memory to the operating system is a separate decision, driven by the allocator's policy rather than your call to .
