+++
date = "2026-08-09"
draft = false
title = "mmap in Linux: The First Thing It Does Is Nothing"
+++

Almost every description of mmap starts the same way: it maps a file into memory. The phrase is so universal that it has become the definition. You call mmap, and the file appears in your address space, ready to be accessed like an array. The implication, whether stated or not, is that the file's contents are loaded into your process's memory.

This mental model breaks in exactly the situations where understanding mmap matters most. It breaks when you're debugging why a process's RSS is smaller than the file it mapped. It breaks when two processes sharing the same mapping see each other's writes instantly. It breaks when mmap turns out to be slower than the read() call it was supposed to replace.

What mmap does is not a load. It is a wiring operation. And the thing it wires your process to is not the file on disk. It is the kernel's page cache.

## The Call

When you call mmap on a file, the kernel does remarkably little. The system call enters ksys_mmap_pgoff(), which calls do_mmap(), and the critical work happens in mmap_region(). What that function creates is a vm_area_struct, a VMA which is a bookkeeping structure describing a range of virtual addresses in your process.

The VMA records the starting address, the length, permissions, and a pointer to the file's vm_operations_struct, which contains the fault handler the kernel will call later. It gets inserted into the process's VMA tree. Under the default flags, that is the entire operation. No pages are allocated. No data is read from disk. No physical memory is consumed.

The return value you receive, that pointer you can now use like an array, is a virtual address. The page table entries for that address range either don't exist yet or are marked invalid. The MMU will fault on the first access, and that fault is the whole point.

## The Fault

The first time your code reads or writes an address inside the mapped range, the CPU's MMU walks the page table and finds no valid translation. It raises a page fault. This is not an error. It is the kernel's invitation to do work it deliberately deferred.

The fault enters handle_mm_fault() in mm/memory.c, walks the multi-level page table, and reaches do_fault(). For a file-backed mapping, this calls the VMA's ->fault() handler. Which handler depends on the filesystem. For ext4, that handler is filemap_fault() in mm/filemap.c. And filemap_fault is where the page cache enters the picture.

The function calls filemap_get_folio() to look up the page in the page cache using the file's address space mapping and the fault's page offset. Two outcomes are possible, and the difference between them is the single most important performance characteristic of mmap.

If the page is already in the cache, because someone read this region recently, or another process already faulted on the same page, the lookup succeeds without disk I/O. The kernel installs a page table entry pointing at the cached page and returns. This is a minor fault. The process resumes and the data is waiting, with no disk access. Cost: microseconds.

If the page is not in the cache, the kernel reads it from the filesystem, inserts the new page into the page cache, installs the page table entry, and returns. This is a major fault. Cost: milliseconds, bounded by storage latency.

The ratio of minor to major faults tells you how well your mmap workload is served by the page cache. You can see it live:

A process with 84,721 minor faults and 3 major faults is living almost entirely in the page cache. A process with major faults climbing steadily is fighting for memory.

## The Cache

The page cache is the detail that makes everything else about mmap make sense. It is a system-wide cache of file data, indexed by (inode, offset) pairs, and it is the same cache that read() and write() use. When you call read() on a file, the kernel copies data from the page cache into your user-space buffer. When you mmap the same file, the kernel skips the copy and points your page table directly at the page cache pages.

This is the fundamental difference. read() gives you a private copy of the data in a buffer you own. mmap gives you a view of the data where it already lives. One involves a memcpy from kernel space to user space on every call. The other involves a page table update once, on the first fault, and then direct memory access from that point forward.

The page cache page is the same physical page in both paths. The only question is whether your process gets its own copy or looks at the original. This is why mmap is sometimes called "zero-copy" - not because nothing was read from disk, but because no data was copied between kernel space and user space.

## The Window

When two processes mmap the same file with MAP_SHARED, their page table entries point to the same physical pages in the page cache. Process A writes a byte at offset 1000 and process B sees it immediately - no IPC mechanism, no system call, no explicit synchronization - because both processes are looking at the same page.

Shared libraries get there by a different route. The dynamic linker maps libc MAP_PRIVATE, not MAP_SHARED. But .text and .rodata are never written, so no copy-on-write fault ever fires, and every process's PTEs keep pointing at the same page cache pages. Two hundred processes share one copy of libc's read-only segments. The writable one, .data, do get copied per process during relocation, which is the part the folk version leaves out.

The sharing is not a feature of MAP_SHARED or of shared libraries. There is one page cache, each file page exists in it once, and any read-only mapping points there by default.

## The Cost

There is a persistent belief that mmap is always faster than read(). The mechanism says otherwise, and the reason is the TLB.

A virtual-to-physical translation that the TLB can answer costs nothing extra. That is what a TLB is for. Only a miss forces the hardware to walk the page tables.

What matters is reach. The TLB holds a fixed number of entries, and 4 KB each is not much ground: this machine's /proc/cpuinfo reports TLB size: 2560 4K pages, which covers about 10 MB. Whatever the exact figure, a large mapping accessed randomly will not fit, and every miss past that point costs tens of nanoseconds when the page tables are cached and hundreds when they are not.

read() sidesteps this entirely. The kernel reads data into the page cache and copies it into your user-space buffer. Your code accesses a small, fixed buffer, whose pages stay hot in the TLB. You pay for the memcpy, but you avoid the TLB storm.

So the honest version is conditional. Sequential streaming usually favours read(). Random access over data already in the page cache usually favours mmap.

## The Boundary

One set of lines worth drawing explicitly, because they are exactly where knowledgeable critics push back. Everything above describes the default: mmap without special flags, on a filesystem backed by block storage, using the kernel's standard page cache. Three cases shift the boundaries. MAP_POPULATE and MAP_LOCKED tell the kernel to prefault the entire range inside the mmap() call itself, pages are read from disk eagerly, not lazily. This is a deliberate opt-in for applications that need deterministic first-access latency.

**NOTE:** The design - demand-paging file data through a unified cache, with the same pages backing both read() and mmap() - originates in the SunOS 4.0 VM system (Gingell, Moran, Shannon, USENIX '87; Moran, EUUG'88). Linux reimplemented the mechanism; the architecture was Sun's. Refer to this paper [http://kos.enix.org/pub/sunos-vi.pdf](http://kos.enix.org/pub/sunos-vi.pdf), [http://kos.enix.org/pub/gingell8.pdf](http://kos.enix.org/pub/gingell8.pdf) for more.

## The Principle

The deeper pattern behind mmap is indirection as a resource multiplier. Virtual memory doesn't give each process its own RAM. It gives each process the illusion of owning RAM by interposing a translation layer. mmap doesn't give your process a copy of a file. It gives your process the illusion of owning the file by reusing the same translation layer.

The power is that indirection turns physical resources into logical claims that can overlap. One copy of libc serves a hundred processes. One page cache page satisfies both mmap readers and read() callers. One file on disk appears simultaneously as memory in a dozen address spaces. The physical resource is scarce, but virtual claims are cheap, and the translation layer resolves them lazily, only when someone actually touches a page.

And that is the beauty of mmap!
