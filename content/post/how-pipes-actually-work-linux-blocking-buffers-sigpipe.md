+++
date = "2026-01-04"
draft = false
title = "How Pipes Actually Work in Linux: Blocking, Buffers, and SIGPIPE"
+++

The problem isn't that pipes are complicated.

They're not.

The problem is that the simple examples work too well.

They hide the interesting parts.

This is not a tutorial. You won't find installation instructions or a list of common commands.

This is an examination of what actually happens when you connect two programs with a vertical bar.

## The Command That Lies

Take a moment and think about what happens when you run the below command:

```bash
ls | wc -l
```

Two programs ran. One listed files. One counted lines.

Somehow, without touching the disk, without any visible connection, the output of the first became the input of the second.

When examples work perfectly, they hide their own operation.

What actually happened here is:
- The shell created a pipe
- The kernel allocated a buffer
- File descriptors were duplicated and redirected
- Blocking rules were established
- The lifecycle of both processes was coordinated

All of this happened before either program executed its instructions.

The tutorial moves on.

But learning stops exactly where it should begin.

Consider a different command:

```bash
cat /dev/urandom | head -n 1
```

This also works.

But /dev/urandom produces an infinite stream.

But why doesn't cat run forever?

Something stopped it.

Something sent a signal.

Something managed the lifecycle of both processes in a coordinated way.

The tutorial doesn't mention this. It can't. The example is too simple to reveal the complexity. And so the students moves forward, confident and ignorant.

This pattern repeats throughout computing education. The simplest examples demonstrate syntax while hiding behavior.

The students learns to recognize patterns without understanding mechanics. They can write *command1 | command2* all day long. They cannot predict when it will block, when it will buffer, or when it will fail.

## When Programs Wait

Pipes are often explained using a water metaphor.

That metaphor is convenient, but wrong.

**Programs don't flow. They block.**

Consider below command:

```bash
yes | head -n 3
```

*yes* writes endlessly.

*head* reads three lines and exits.

The pipeline finishes instantly.

But *yes* did not stop willingly. It attempted to write again and failed. But no crash.

Conceptual sequence:
1. *head* exits
2. The kernel closes its read-end file descriptor
3. The pipe now has no readers
4. *yes* calls *write()*
5. The kernel sends SIGPIPE
6. *yes* terminates (default signal behavior)

```c
/* fs/pipe.c in kernel: anon_pipe_write() */
if (!pipe->readers) {
    if (!(iocb->ki_flags & IOCB_NOSIGNAL))
        send_sig(SIGPIPE, current, 0);
    ret = -EPIPE;
    goto out;
}
```

The rule is explicit:

> Writing to a pipe with no readers is illegal. So, the writers receive SIGPIPE

Writing to a pipe with no readers is illegal. So, the writers receive SIGPIPE

*yes* dies because the kernel kills it.

Now reverse the roles:

```bash
sleep 3 | cat
```

*sleep* writes nothing and exits.

*cat* waits for input.

Why doesn’t *cat* block forever?

Because EOF is not a timeout. It’s a condition.

Conceptual sequence:
1. *sleep* exits
2. Kernel closes the pipe’s write end
3. No writers remain
4. *cat* calls *read()*
5. Kernel returns 0 (EOF)

*cat* source code reference:

```c
while (true) {
    ssize_t n_read = read(input_desc, buf, bufsize);
    if (n_read < 0) { /* error */ }
    if (n_read == 0) return true; /* EOF */
    if (full_write(STDOUT_FILENO, buf, n_read) != n_read)
        write_error();
}
```

kernel code reference:

```c
/* fs/pipe.c : anon_pipe_read() */
if (pipe_empty(head, tail)) {
    if (!pipe->writers)
        return 0; /* EOF */
    /* otherwise: block */
}
```

EOF is returned only when:
- The buffer is empty
- No writers exist

Nothing else triggers it.

These are not edge cases. This is normal pipe behavior.

Programs block waiting for data. Programs die when their readers vanish.

>

Programs receive EOF when their writers close.

All of this coordination happens through the kernel, invisible to both processes.

## The Buffer Between Worlds

Processes connected by a pipe never communicate directly.

Between them sits a kernel-managed buffer.

This buffer is small.

On most systems, 64 kilobytes. Sometimes less. This matters more than you'd think.

What the buffer enforces:
- Backpressure
- Fair scheduling
- Memory safety
- Speed mismatch tolerance

When the buffer fills, writers block.

```c
#define pipe_full(head, tail, limit) \
    ((head) - (tail) >= (limit))
```

This single condition decides whether a writer proceeds or sleeps.

When a writer hits a full pipe, the kernel does not spin.

It puts the process to sleep.

writer blocking:

```c
wait_event_interruptible_exclusive(
    pipe->wr_wait,
    pipe_writable(pipe)
);
```

The process is removed from the run queue.

It resumes only when a reader consumes data.

When that happens:

```c
wake_up_interruptible_sync_poll(
    &pipe->wr_wait,
    EPOLLOUT | EPOLLWRNORM
);
```

No polling

Only explicit wakeups.

### A Stress Case That Reveals the Design

- Writer: 1 MB chunks
- Reader: 1 byte at a time
- Buffer: 64 KB

```bash
dd if=/dev/zero bs=1M count=100 | dd of=/dev/null bs=1
```

Here's what happens at the kernel level:
1. **First write**: dd calls write() with 1MB of zeros
2. **Kernel copies** 64KB into the pipe buffers (16 pages)
3. **Buffer full**: pipe_full() returns true
4. **Process blocks**: dd sleeps in pipe->wr_wait
5. **Reader consumes** 1 byte, freeing one buffer slot
6. **Kernel wakes writer**: wake_up_interruptible_sync_poll() called
7. **Writer resumes**: copies another 4KB (one page)
8. **Repeat** until all data transfers

This continues until all data passes through.

The pipeline works. But it's not smooth. It's a series of writes, blocks, reads, and resumptions.

The buffer mediates every interaction.

Why this design matters?

The buffer isn't an implementation detail. It's the mechanism that makes pipes work. Without it, programs would need to run at exactly the same speed. With it, they can run at different speeds and the kernel manages the coordination.

The 64KB limit creates **backpressure**: a fast writer cannot overwhelm a slow reader. It must wait. This prevents memory exhaustion and ensures fair scheduling.

Most pipelines never fill the buffer. *ls | wc -l* produces a few hundred bytes. The buffer never comes close to full. This is why tutorial examples hide the buffer's existence. It's there, doing its job, but never tested.

## How File Descriptors Are Used in Between

A file descriptor is a number.

That's the technical definition.

That definition is accurate, but useless.

What matters is what it indexes:
- A per-process table
- Pointing to kernel objects
- Representing streams of bytes

Files, pipes, sockets, terminals - all share this abstraction.

Every process starts with three file descriptors already open.

Zero is standard input. One is standard output. Two is standard error.

These aren't special numbers. They're just the first three slots. The kernel could have used different numbers. Zero, one, and two are conventions.

When a program opens a file, the kernel returns an integer. That integer is the file descriptor.

When you create a pipe, the kernel allocates a buffer and returns two file descriptors. One for reading. One for writing. These are just numbers, but they point to opposite ends of the same kernel object.

```c
int pipefd[2];
pipe(pipefd); /* pipefd[0]: read, pipefd[1]: write */
```

**Both ends reference the same pipe object.**

That’s the entire trick.

The critical part: **file descriptors are per-process**. Each process has its own table of them. When you fork a process, the child inherits copies of the parent's file descriptors. Now two processes have file descriptors pointing to the same underlying kernel object.

## How the Shell Builds a Pipeline

When you type:

```bash
ls | wc -l
```

The shell performs this exact sequence:
1. Create pipe
2. Fork writer
3. Redirect *stdout* → pipe write end
4. Fork reader
5. Redirect *stdin* → pipe read end
6. Close unused descriptors
7. *exec()* both programs

The close operations matter.

If the shell keeps a write descriptor open, EOF never arrives.

> All file descriptor setup — *dup2()* and *close()* — happens *before* *exec()*.



Once exec() runs, you lose the chance to fix file descriptors.

## Lifecycle: From Creation to Free

When the writer exits:

```c
/* fs/pipe.c : pipe_release() */
if (file->f_mode & FMODE_WRITE)
    pipe->writers--;
```

When writers reach zero, readers wake.

When readers reach zero, buffers are freed.

```c
void free_pipe_info(struct pipe_inode_info *pipe)
{
    kfree(pipe->bufs);
    kfree(pipe);
}
```

Nothing leaks.

Everything is accounted for.

## The Moment Everything Connects

Once pipes make sense, other abstractions fall into place.

You realize that network sockets work the same way. They're file descriptors. They block on reads and writes. They use kernel buffers. The same mechanisms, applied to network communication instead of local processes.

You realize that files are different. When you write to a file, it rarely blocks. The kernel buffers the write and returns immediately. The data reaches disk later, asynchronously. Pipes can't do this. They must coordinate two active processes, so blocking is necessary.

You realize that standard input, output, and error are just file descriptors. They're not special. They're often connected to a terminal, but they could be connected to files, pipes, or anything else. The program doesn't need to know.

You realize that redirection, pipes, and process substitution are all variations on the same theme: file descriptor manipulation. The shell is connecting programs by rearranging where their I/O goes.

This is when pipes stop being a syntax trick and become a window into how Unix systems work.

So next time you see a pipe, don’t just think about what you type, think about the kernel rules coordinating independent readers and writers under the hood.
