+++
date = "2026-05-01"
draft = false
title = "The authencesn Bug: How a 4-Byte Tag Write Became a Linux Root Exploit"
+++

Let’s start with what this exploit (***CVE-2026-31431***) actually does.

It allows an **unprivileged user to modify** a file without touching it on disk, without triggering integrity checks, and ultimately gain root access. The change happens only in memory - inside the **kernel’s page cache**.

So the system sees two different realities:
- Disk → unchanged
- Memory → modified

And the kernel trusts memory. That’s the entire exploit.

This article explains how something as small as a 4-byte write - safe for years - turned into a reliable root primitive across almost every Linux distribution.

## The Page Cache - The System’s Shared Reality

When a process reads a file, the kernel first checks if the data is already in RAM. If it is, the read is served from memory - no disk access needed. **Only when the cache is cold does the kernel load data from disk**.

This **cache is global**. It's shared across all processes. If two processes read /etc/passwd, the kernel serves them the same page cache pages - the same data from the same locations in RAM. Every process that reads that file gets identical bytes.

Under normal conditions, the kernel maintains consistency carefully. Writes go through write(), pages are marked dirty, and changes are flushed back to disk. That pipeline is what keeps memory and disk aligned.

But **if something modifies the page cache outside that path, the kernel doesn't notice**. No dirty flag is set, nothing is written back, and the disk remains unchanged - even though every process now sees modified data.

This exploit is about altering pages outside that path.

## The Scratch Write - A Bug That Wasn’t One

Inside the kernel **crypto subsystem** is a **template** called **authencesn**. It implements authenticated encryption for IPsec and needs to rearrange sequence number data inside buffers before computing the authentication tag.

To do that, it uses the **destination buffer as scratch space**. One step in that process writes four bytes of seqno_lo at: *assoclen + cryptlen.*

This offset lies just beyond the plaintext output - inside the authentication tag region.

The operation looks like:

This write happens before authentication is verified, and it executes even if the decryption ultimately fails. The overwritten bytes are never restored.

That detail is what makes this bug exploitable.

## Opening the Door - AF_ALG

In 2015, the kernel introduced **AF_ALG**, a **socket interface that exposes crypto operations to userspace**. Any unprivileged user could now trigger authencesn and other kernel crypto templates directly from a socket.

This made authencesn reachable from outside the kernel.

However, the system was still safe because operations were out-of-place: source (input) and destination (output) were separate memory regions, the scratch write landed in user-controlled buffers.

So even though the bug was reachable, it couldn’t affect anything sensitive.

## The Optimization - Changing the Target

In 2017, an optimization changed AEAD processing to **work in-place**. Instead of keeping source and destination separate, the kernel made them share the same scatterlist:

To support this, the tag pages from the TX scatterlist - which hold the authentication tag data - were chained onto the end of the RX scatterlist using sg_chain(). These tag pages are the pages delivered via splice(), which means they are references to the file's page cache, not copies.

The destination became a combined structure: a user buffer (containing copied AAD and ciphertext) followed by tag pages that referenced the file's page cache. And critically, this entire structure was writable.

Nothing about the original bug changed.

But the memory it wrote into did.

## The Intersection - Where It Breaks

The scratch write still targets: dst[assoclen + cryptlen].

But because of the in-place optimization and scatterlist chaining, that offset can extend beyond the user buffer and into the chained page cache pages. So the write lands not in user memory, but directly in the page cache of a file.

This is the moment the bug becomes an exploit.

## From 4 Bytes to Root

Once the attacker can write into the page cache, the rest is straightforward.

They use **splice()**, a zero-copy mechanism, to deliver file-backed pages directly into the crypto path. Then they repeatedly trigger the operation, each time writing four controlled bytes.

The value written comes from the input data - specifically, bytes 4-7 of the AAD, which authencesn interprets as seqno_lo. This gives the attacker precise control over what gets written.

By repeating this process across multiple iterations, they overwrite arbitrary locations in a file's page cache, byte by byte.

For setuid binaries like /usr/bin/su, this means injecting shellcode directly into the binary's text section in memory. The next execve("/usr/bin/su") loads the modified page cache version - and runs as UID 0.

The file on disk is unchanged. But when the system reads the binary, it reads from the page cache - and executes the attacker's code with root privileges.

The same principle applies to any file: overwrite instructions in memory, and the next read or execution sees the modified version.

## Why This Exploit Is Dangerous

This exploit stands out for three reasons.

First, it is deterministic. There are no races or timing dependencies - it works reliably.

Second, it is invisible. Because the disk is never modified, traditional integrity checks and monitoring tools detect nothing.

Third, it breaks isolation assumptions. The page cache is shared across containers and processes, so one unprivileged environment can influence another.

This is not just a local bug. It’s a cross-boundary violation.

## The Fix

The fix does not remove the bug.

It removes the path that made it dangerous.

The vulnerability depended on page cache pages being included in a writable destination scatterlist. This happened because the in-place AEAD path chained tag pages from the TX scatterlist into the destination using sg_chain() - and those tag pages, when delivered via splice(), were references to file-backed page cache pages.

The fix changes that construction.

File-backed pages from splice() stay in the source scatterlist - they are never chained into the destination, the destination buffer is restricted to the user's recvmsg buffer, page cache pages remain read-only inputs.

So even though the same write inside authencesn still occurs at dst[assoclen + cryptlen], it can no longer reach file-backed memory. It writes to the user's buffer instead - where it has always been harmless.

## The Lesson Taught by the Bug

This vulnerability was not introduced by a single mistake. The bug emerged from the interaction of three independent decisions:
1. A scratch write that was safe in isolation
2. A userspace interface that expanded access
3. An optimization that changed memory layout

Each decision made sense on its own. Together, they created an unexpected path to modify the page cache.

This is how **complex systems fail - not from obvious errors, but from the intersection of correct components operating under outdated assumptions.**

Because sometimes, all it takes…

is four bytes.
