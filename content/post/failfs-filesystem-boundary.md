+++
date = "2026-08-30"
draft = false
title = "failfs: The Filesystem as a Boundary"
+++

Some security boundaries are much simpler than the mechanisms we usually reach for.

A process has finished opening what it needs. Now you want to prevent it from opening anything else. The approaches involve namespaces, chroot(), or a seccomp policy. Each works. Each is also much larger than the requirement.

According to the failfs documentation and source code (was merged into Linux 7.3-rc1 recently), there is now a new option: a filesystem designed to refuse everything.

## What failfs Is Supposed To Do

The entire implementation fits in a tiny *fs/failfs.c*. Per the source, every filesystem operation returns  before the kernel parses a path component.

A process enters failfs via a single syscall: *fchroot(FD_FAILFS_ROOT)*, where FD_FAILFS_ROOT is a magic constant (like FD_NSFS_ROOT), not a real file descriptor. According to the documentation, after entry:
- Absolute paths fail
- Working directory relative paths fail
- Execution fails (interpreter resolution requires path lookup)
- File descriptors opened before entry continue to work

The design inverts typical filesystem machinery. Most filesystems ask, “What should I do with this request?” failfs asks, “Can this request go any further?” The answer is always no.

## The Trick: Pathname vs. Object

This is where the design becomes clear.

Pathnames and file descriptors are not the same thing. When a process opens , the kernel walks the pathname, locates the inode, and returns a file descriptor - a handle to the already-located object. The process then uses that fd for subsequent reads and writes.

Once the fd exists, operations on it don't require re-doing the pathname lookup. The filesystem view is no longer involved.

So when a process enters failfs:
- New  calls fail (pathname lookup is blocked)
- Existing fds continue to work (they point to objects already located)

The important distinction is that failfs blocks new pathname resolution. It doesn't revoke references the process already holds. A process with its root in failfs must anchor every path lookup at an explicit file descriptor. That's why it works.

## Why This Matters

The mechanism is minimal because it holds no policy. According to the source, the permission check returns  unconditionally. There is no table to consult, no policy to evaluate, no ongoing work to maintain. The refusal happens at the first checkpoint, then the walk stops.

Compare this to the alternatives:
- **Mount namespace:** Changes the filesystem view for a group of processes; heavier machinery that typically involves unshare() or clone().
- **chroot():** Changes the process's root, but doesn't cleanly separate already-acquired objects from future pathname lookups.
- **seccomp:** Restricts syscalls, so the policy has to account for the different ways a process can acquire or access filesystem objects. Restricting open() alone, for example, doesn't capture every path-based access mechanism.

failfs doesn't build an allowlist. It removes the lookup mechanism and leaves existing references intact.

## The Trade-Off

According to the doc, entering is effectively one-way. Getting out requires CAP_SYS_ADMIN, a mount namespace file descriptor, and setns(). Without those capabilities and resources and with the relevant syscalls blocked by seccomp, the process has no way back out.

The documentation states this deliberately: the restrictive version ships first. A one-way door can be widened later. A door that turned out to be two-way cannot be narrowed once something depends on it.

The interesting thing about failfs isn't that it rejects everything.

It's that it doesn't need to know what the process is allowed to access. It simply makes acquiring anything new through pathname lookup impossible, while leaving existing references untouched.

Sometimes the cleanest security boundary isn't another policy.

It's removing the path to everything else.
