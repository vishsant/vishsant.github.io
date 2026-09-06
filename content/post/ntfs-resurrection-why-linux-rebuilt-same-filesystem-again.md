+++
date = "2026-05-03"
draft = false
title = "The NTFS Resurrection: Why Linux Rebuilt the Same Filesystem Again and Again"
+++

There is a filesystem that lives on over a billion devices - every Windows laptop, every USB drive formatted on Windows, every dual-boot partition holding someone’s files. It belongs to Microsoft, has no complete public specification, and yet Linux has been trying to support it for thirty years.

This article traces four separate attempts to build NTFS support in Linux. Not to critique any single implementation, but to understand what this evolution reveals about how Linux kernel supports systems it does not control.

## The Filesystem Linux Cannot Ignore

**NTFS** was introduced **by Microsoft** in 1993 with Windows NT 3.1, **replacing FAT** as the primary filesystem. Unlike ext4 or XFS, **it was never designed for Linux**, and Microsoft has never published a complete public specification.

The on-disk format is complex. Files are stored as structured records inside the Master File Table (MFT), with attributes, B-tree indexed directories, compression, encryption, access control lists, and journaling. It **behaves less like** a **simple block-based filesystem** and **more like a structured database**.

Every piece of **Linux NTFS support** has been **built through reverse engineering**.

This creates a fundamental tension. Linux does not control the format, yet must maintain compatibility with it. The semantics don’t map cleanly: permissions differ, case sensitivity differs, and timestamp behavior differs.

And **yet Linux cannot ignore** NTFS.

USB drives, external disks, dual-boot partitions, and corporate file shares all depend on it. These are not optional use cases - they are user-critical. **Linux doesn’t get to choose which filesystems matter. Users already did.**

## The First Attempt: Read-Only and Barely Alive

The **first NTFS driver **was merged into Linux in 1997. It provided limited **read-only support** and was later rewritten in 2002 to improve stability and compatibility.

It could read files reliably, but** writing was unsafe**. Complex features like compression and encryption were only partially supported. For over a decade, it served as a safety net: you could access your files, but you couldn’t modify them with confidence.

In practice, this meant Linux could see NTFS data but couldn’t fully participate in it.

By 2024, the driver was removed entirely after 27 years in the kernel. It had done its job, but it never evolved into a complete solution.

## The FUSE Driver - A Tradeoff Solution

The real breakthrough came in 2006 with **NTFS-3G**, a **userspace driver** built on **FUSE (Filesystem in Userspace)**. It provided stable, **full read-write support** and quickly became the default solution across Linux distributions.

It worked - and for many users, that was enough.

But FUSE introduces a structural limitation. Each filesystem operation crosses the kernel-userspace boundary multiple times.

In a FUSE-based filesystem like NTFS-3G, the write() call takes a longer path. The request first enters the kernel, but instead of being handled there, it is passed to the FUSE module, which forwards it to a userspace daemon. This requires a context switch from kernel space to userspace.

The userspace daemon (NTFS-3G) then interprets the request figuring out where the data should go in the NTFS structure. Once it has done that, it makes another system call back into the kernel to perform the actual disk write. After the write completes, the response travels back through the same path in reverse.

**NTFS-3G made NTFS usable on Linux**. It did not make it fast. And it could not participate in the kernel's page cache, memory management, or I/O scheduling in the way an in-kernel filesystem can.

The compromise was real. But it was the best option available for fifteen years.

## The Corporate Contribution: NTFS3

In 2021, **Paragon Software** introduced **NTFS3, a fully in-kernel NTFS driver**. It eliminated FUSE overhead, delivered strong performance, and provided full read-write support.

For the first time, Linux had a native NTFS implementation that could compete with its own filesystems in terms of integration.

In an in-kernel filesystem, the flow is direct. When an application calls write(), the request crosses into the kernel once and stays there. The VFS resolves the file, the filesystem maps it to disk blocks, and the data is written into the page cache. From there, the kernel decides when to flush it to disk through the block layer.

Everything happens within a single address space, under one scheduler, using one page cache. There are no detours, no extra context switches, and no duplicated logic. The kernel has full visibility and control over the entire operation.

Most modern Linux distributions adopted NTFS3 as their default NTFS handler.

Then the maintainer went silent. Patches accumulated, bugs remained unresolved, and the project gradually lost momentum. Without active maintenance, even good code becomes fragile.

Technically, NTFS3 also lagged behind kernel evolution. It relied on older infrastructure like buffer_head and didn’t adopt newer abstractions like iomap or folios. Over time, this created a growing gap between NTFS3 and the rest of the kernel.

The code worked, but it stopped evolving - and in kernel development, that’s a slow failure.

## The Resurrection: A Modern Driver From Dead Code

The fourth attempt began with a clean slate.

**Namjae Jeon,** an experienced kernel developer, rebuilt NTFS support from scratch over four years. **He maintains the exFAT filesystem driver** - the filesystem that every SD card and USB drive uses when FAT32's 4GB file size limit is not enough.

Four years ago, Jeon looked at the dead NTFS codebase - the one that had just been removed from the kernel in 6.9 and decided it was worth rebuilding.

Not patching. Rebuilding.

The result is a modern driver aligned with how the Linux kernel works today. It was originally developed under the name **NTFSPlus** but was merged simply as the NTFS filesystem.

It uses iomap instead of buffer_head, enabling efficient extent-based I/O. It supports folios for improved memory management and implements delayed allocation for better write performance.

It also **introduces a full userspace toolchain**, including **fsck.ntfs**, allowing Linux to repair NTFS filesystems natively for the first time.

The driver passes more regression tests than previous implementations and was merged into Linux 7.1.

The key difference isn’t just effort. It’s alignment with modern kernel infrastructure.

The merge into Linux 7.1 in April 2026 carried this message from Linus Torvalds: **ntfs resurrection**. The word acknowledges history. The original NTFS driver lived in the kernel for 27 years. It was removed in 2024. And now it returns - not as it was, but rebuilt with modern infrastructure, modern testing, and an active maintainer.

Community discussion around the merge used stronger language. Some called it "the Nosferatu of filesystems" - the undead code that keeps returning. Others called it an "Easter miracle."

## What This Teaches About Maintaining What You Don't Own

For this rebuild to work, it need to have what the previous three implementation lacked: **sustained maintenance**.

Linux maintains dozens of filesystems. ext4, XFS, Btrfs, F2FS - these are Linux-native. The kernel community controls the specification, the tools, and the evolution.

Microsoft can release a new version of NTFS tomorrow. The driver maintainers must adapt to decisions made by organizations that have no obligation to consider Linux.

This creates a specific kind of engineering challenge. You are building infrastructure on a foundation you do not control. Your test suite validates behavior against a format that someone else defines.

The pattern is not unique to NTFS. It is the pattern of every piece of infrastructure that exists because users demand it, not because the community chose it.

The lesson is not about NTFS. It is about what sustains complex software over decades.

A driver without a maintainer is dead code with a pulse. It runs until it doesn't. It works until the world changes around it. The code does not rot. The context does.

Namjae Jeon spent four years rebuilding code that had been declared dead. Not because NTFS is glamorous. Not because it trends on social media. But because billions of devices store data on it, and Linux needed to read and write that data correctly.

NTFS on Linux has survived for thirty years not because it was elegant or easy, but because someone kept **rebuilding it**.

And **as long as users depend on it, someone always will**.

That is the part of open source that rarely gets celebrated.

It should be.

Because the filesystem your USB drive uses does not care about your kernel's release cycle. It just needs to work. And someone has to make it work.

For thirty years, someone always has.
