+++
date = "2026-05-17"
draft = false
title = "Your Terminal Is Not Your Shell In Linux"
+++

There is a thing every developer believes they understand. It is the rectangle of text on their screen where they type commands. They call it "the terminal," or "the shell," or "the command line," and most of the time those words are used interchangeably.

They are not the same thing. They have never been the same thing. The line between them is one of the cleanest, most consequential abstractions in the entire Unix design - and almost nobody can draw it.

This article traces the entire path - from the moment you press a key in your terminal window to the moment a process in your shell sees that key as input and back again. By the end, you will see why every containerized shell, every SSH session, and every weird thing your terminal has ever done is a direct consequence of one design decision the kernel made in 1998.

## The Two Halves

Open two terminal windows on a Linux system. In each one, run the command:

```console
$ tty
/dev/pts/0
```

You will see two different paths. Something like ***/dev/pts/0*** in one and ***/dev/pts/1*** in the other. Those are real files. You can ***stat*** them. You can read their inode numbers. They are character devices, the same class of file that the kernel uses to represent ***/dev/null, /dev/random***, and every serial port and hardware terminal it has ever supported.PTY master slave dev nodes

These files are the **slave** sides of two pseudo-terminal pairs. The slave is the half of the PTY that your shell is holding. When your shell reads from the slave, it sees the characters you typed in the terminal window. When your shell writes to the slave, the characters appear in the terminal window.

The other half of each pair is the master. The master is what your terminal emulator - xterm, kitty, gnome-terminal, the VS Code integrated terminal, whatever - is holding. The master does not have a path under /dev/pts/. It is anonymous. The kernel will not let you find it by listing directories. You can only obtain a **master file** descriptor by opening ***/dev/ptmx***, the pseudo-terminal master multiplexer. Every time a process opens /dev/ptmx, the kernel allocates a brand-new master/slave pair, returns the master fd, and creates a new node under /dev/pts/ for the slave.

So when your terminal emulator launched, here is what it did:

It opened /dev/ptmx. The kernel gave it back a master file descriptor and conjured up a /dev/pts/N for the corresponding slave. The emulator then forked, the child became a session leader, opened the slave as its standard input, output, and error, and executed your shell. The parent - your terminal emulator - kept the master.

From that moment on, your terminal emulator and your shell are not connected. They are each holding one end of a kernel object. The kernel is the connection.

## The Line Discipline

If a PTY were just a pipe - a kernel buffer with one end for writing and another for reading - none of the things you expect from a terminal would work. Ctrl+C would not send a signal; it would just appear as the byte 0x03 in your shell's input. Backspace would not erase characters; it would also just appear as a byte. Echo would not happen; you would type a character and see nothing on screen unless your shell explicitly wrote it back. Job control would not exist; there would be no concept of foreground, background, or "the terminal that owns this process group."

The reason all of those things work is the **line discipline**. The line discipline is a piece of kernel code that sits between the master and the slave and transforms the bytes flowing between them. The default line discipline on Linux is called ***n_tty***, and it lives in drivers ***/tty/n_tty.c***. It is one of the most quietly important pieces of code in the kernel.

When your terminal emulator writes a byte into the master, the line discipline catches it on its way to the slave's read queue. If the byte is 0x03 (ASCII ETX, what Ctrl+C produces), the line discipline does not put it into the queue. Instead, it sends SIGINT to the foreground process group of the terminal. If the byte is the configured erase character (usually 0x7F, backspace), it removes the previous character from a line buffer it is maintaining internally. If the byte is a printable character, it adds it to the line buffer *and* writes it back to the master so your terminal emulator can render it on screen - this is what "echo" actually means.

The line buffer is held in the kernel, not in your shell. Only when you press Enter does the line discipline release the buffered line into the slave's read queue and your shell's read() finally returns. This is called **canonical mode**, and it is why you can edit a half-typed command with backspace before pressing Enter - your shell never saw the half-typed version, because the kernel was buffering it on the shell's behalf.

You can turn all of this off. Programs that need to handle every keystroke immediately - vim, less, htop, fish's interactive editor - call ***tcsetattr()*** to disable canonical mode and echo, putting the line discipline into raw mode. In **raw mode**, every byte the terminal emulator writes goes straight into the slave's queue. The program reads keystrokes one at a time and handles its own line editing, its own screen rendering, its own everything.

This is the why behind the PTY's existence. PTYs were not invented to forward bytes - pipes already did that. **PTYs were invented to keep the line-discipline subsystem, the session-management subsystem, and the job-control subsystem alive when terminal emulation moved out of the kernel and into userspace**. The kernel kept being the teletype. The terminal emulator just took over rendering it.

## The Five-Step Mechanism

When a process wants to create a new PTY pair, it follows a five-step protocol:

```text
1. fd = posix_openpt(O_RDWR | O_NOCTTY)
       → allocates new master/slave pair, returns master fd

2. grantpt(fd)
       → sets ownership/permission on the slave device

3. unlockpt(fd)
       → unlocks the slave (default state is locked, prevents races)

4. slave_path = ptsname(fd)
       → returns /dev/pts/N for this pair's slave

5. slave_fd = open(slave_path, O_RDWR)
       → opens the slave
```

After step 5, the process has both ends of the pair. It can now do something like fork(), give the slave end to the child as stdin/stdout/stderr, and keep the master in the parent. The parent becomes the "terminal emulator" for the child - it reads what the child writes by reading from the master, and it injects input into the child by writing to the master.

## Where the Slaves Live: devpts

The slave devices in /dev/pts/ do not exist on any disk. The directory /dev/pts/ is mounted with a **virtual filesystem **called** devpts**.

devpts is also the answer to a question almost nobody asks until they hit it: *how does a container have its own /dev/pts/0 independent of the host's /dev/pts/0?*

In Linux 4.7, the kernel made each mount of devpts an independent filesystem instance automatically. Container runtimes mount devpts inside the container's mount namespace, and the kernel gives that mount its own numbering - /dev/pts/0, /dev/pts/1, etc. totally isolated from the host's allocations. Container PTY 0 is not host PTY 0. They are separate kernel objects in separate devpts instances.

This is also why ***docker exec -it*** works. The* -t* flag tells the Docker daemon to allocate a new PTY *inside the container's devpts instance*, give the master to containerd-shim, and connect the slave to the new process. The shim proxies bytes between the container's PTY master and a Unix domain socket that ultimately ends up back at your local terminal. You are typing into your local terminal's master, those bytes go down the socket, the shim writes them into the container's PTY master, the container's line discipline transforms them, the container's shell reads them from the slave. Every keystroke crosses two PTYs and one socket.

## Why vim Does Not Work Over a Pipe

If you have ever tried something like ***cat file.txt | vim*** and watched vim refuse to start with the message *"Vim: Warning: Input is not from a terminal,"* you have run into the consequence of the design decision the kernel made in 1998.

*vim* and *less*, and *htop*, and *nano*, and every other full-screen interactive terminal program needs to do three things that a pipe cannot provide. It needs to find out the size of the terminal window (the TIOCGWINSZ ioctl). It needs to put the line discipline into raw mode so it can handle every keystroke directly (the TCSETS ioctl). And it needs to receive SIGWINCH when the terminal is resized.

None of those operations make sense on a pipe. A pipe has no concept of "terminal size" - it is just a kernel buffer. It has no line discipline - bytes flow through untouched. It does not generate SIGWINCH because there is no terminal to be resized.

---

So, the entire architecture can be consolidated to:

```text
Keyboard
   ↓
Terminal Emulator
(kitty, gnome-terminal)
   ↓
PTY Master
   ↓
Kernel Line Discipline
   ↓
PTY Slave
   ↓
Shell (bash/zsh)
   ↓
Programs (vim, gcc, python)
```

The next time you open a terminal, remember what is actually happening. Your terminal emulator launched, opened /dev/ptmx, received a master file descriptor, and asked the kernel for a fresh slave device. The kernel created /dev/pts/N and handed back the path. The emulator forked, the child became a session leader, opened the slave as its controlling terminal, and executed your shell. From that point on, every byte you type travels through a 50-year-old contract maintained by a kernel module that nobody talks about.

Your terminal is not connected to your shell. It never has been. The kernel has been the connection the whole time.
