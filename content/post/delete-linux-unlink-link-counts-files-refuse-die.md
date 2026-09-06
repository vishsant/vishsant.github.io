+++
date = "2026-06-14"
draft = false
title = "There Is No Delete in Linux: unlink(), Link Counts, and the Files That Refuse to Die"
+++

Every programmer learns the same thing about rm: it deletes files. You point it at a file, the file is gone, the disk space comes back. It is one of the first commands anyone learns, and one of the few we never question.

It is also wrong in almost every detail. rm does not erase data. It does not reclaim disk space - not directly, and sometimes not for hours. The kernel has no operation called "delete a file." What actually happens is quieter, stranger, and once you see it, it explains a whole category of production mysteries.

## The Name

Start with what rm actually calls. It is not some delete_file() syscall. It is unlink(), and the name is the whole story: it removes a link, not a file. A filename, in Linux, is just a link - a directory entry pointing at an inode, the real object that owns the data. A single inode can have many names (that is what a hard link is), and the inode keeps a count of them.

When you unlink() a name, the filesystem's unlink operation decrements that count. Here is the entire mechanism, from fs/inode.c:

That is the "delete." One field, i_nlink, decremented by one. If the file had two names, it now has one, and nothing else happens at all. The data is untouched and still reachable by the other name. rm operates on names, and only names. It has no idea what "the file" is.

## The Two Counts

So when does the data get freed? This is the part nobody is taught, and it is the key to everything. An inode is kept alive by two independent counts, not one.

The first is i_nlink - how many names point at it, on disk. That is the one unlink() touches. The second is i_count - the inode's in-memory reference count. It counts every live in-kernel reference to the inode, but for a file you are trying to delete, the references that matter are the open handles: every process that has the file open is keeping i_count above zero. The kernel will not free the data until both counts reach zero - no names left, and nothing still holding it open. Removing the last name is necessary, but it is not sufficient.

When a process closes a file, the close path eventually reaches iput(), which drops one in-memory reference. Only when that reference count hits zero does the kernel even ask whether to free the inode - and the test it applies is this:

!inode->i_nlink - now the name count matters. If there are no names left, the inode is droppable, and iput_final() proceeds to evict() it, which is where the data blocks finally return to the filesystem. But iput() only ran in the first place because the last open handle was released. Two gates, in sequence: the last name goes, then the last handle goes, and only then does the data die.

rm only ever touches the first gate. The second one is held by whoever has the file open, and that might be a process that does not plan to close it any time soon.

## The Ghost

This is where the production mystery comes from. Watch what happens when you delete a file that something still has open. I created a 50MB file, opened it in one process, then deleted it by name:

By name, the file is gone. ls can't find it, no other process can open it, it has no path anymore. And yet:

There it is - (deleted), and still 52,428,800 bytes, every one of them readable. The name count hit zero, but the open handle kept the inode alive, so the data never went anywhere. The file has become a ghost: unreachable by name, fully present on disk. This is exactly why df (which counts real allocated blocks) and du (which sums the sizes of files it can reach by name) will flatly disagree on a busy server. df sees the ghost's blocks; du cannot, because the ghost has no name to walk to.

Now the war story writes itself. A disk hits 100%. Someone finds the giant log file, rms it, and watches df not move a single byte - because journald, or nginx, or your own application still has that log open and is happily writing to it. The name is gone; the data is pinned by an open handle that will not close until the process is restarted, or until you reach into /proc/<pid>/fd/ and truncate the ghost in place. The rm did precisely what it promised: it removed a name. It never promised to free anything.

## The Free

The actual freeing happens at evict(), and only after both gates have closed. In the demo, the moment I kill the holding process, its last reference drops, iput() runs, i_nlink is already zero, and the blocks return - instantly, with no rm involved at all. The disk space comes back not when you delete the file, but when the last process using it exits.

So what does evict() actually do to the data? Two things, to two different copies. First it calls truncate_inode_pages_final(), which drops every cached page of the file out of the page cache - the in-memory copy is released. Then the filesystem's own handler (ext4_evict_inode() on ext4) sets the size to zero and runs ext4_truncate(), which walks the inode's block map and hands every data block back to the filesystem's free-block bitmap; the inode itself returns to the free-inode pool. That is the moment df finally moves - the blocks are now marked available, and the inode number can be reused.

But look closely at what evict() does not do: it never overwrites those blocks. It marks them free; it does not wipe the bytes sitting in them. This is the whole reason rm is not a secure erase, and why undelete tools can sometimes recover a file long after it is "gone" - the data was abandoned, not destroyed. It stays perfectly legible in those blocks until something else allocates them and writes over the top. Even at the final stage, the kernel does not erase your file. It just stops protecting the ground it was standing on, and walks away.

One line worth drawing explicitly, because it is exactly where people argue past each other. Everything up to this point -  removing a name, the link count, and the rule that the data survives until both the name count and the open-handle count reach zero lives in the VFS layer, and it is the POSIX contract. Every compliant filesystem inherits it; an open file outliving its own  is guaranteed, not optional. What each filesystem decides for* *itself is the reclamation step inside : whether freed blocks are returned immediately or lazily, and whether they are scrubbed or merely marked available.  is the worked example here , . btrfs, XFS, or a log-structured filesystem differ in those mechanics. But none of them change the model this article is about:  removes a name, and the data is freed only when the last reference - name or handle.

Two reference counts, two gates, and the data survives until both are zero. The deletion you typed was only ever the first half.

## The Model

Once you internalize this, a pile of Unix behavior that seemed arbitrary becomes inevitable. Hard links stop being mysterious - they are just multiple names sharing one i_nlink, and rm removing one of them obviously cannot free data the other still points to. Temp-file tricks make sense - a program can create a file, open it, and immediately unlink() it, keeping a private scratch file with no name that the OS auto-frees the instant the process dies.

And the deepest part is the pattern itself. Linux did not implement "delete." It implemented reference counting - free the object when the last reference to it disappears, and "delete a file" is just one reference (a name) being dropped, decades before we started calling this idea garbage collection.

rm doesn't delete files. It lets go of one name, and waits with you to see if anything else was still holding on.
