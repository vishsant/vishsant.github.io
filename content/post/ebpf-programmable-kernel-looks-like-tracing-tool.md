+++
date = "2026-04-09"
draft = false
title = "eBPF: The Programmable Kernel That Looks Like a Tracing Tool"
+++

There is a category of tools that engineers learn twice. The first time, they learn the surface: what it does, how to run it, what output it produces. The second time, they learn what it actually is - and the first understanding falls apart entirely.

eBPF is that kind of tool. Most engineers encounter it through **bpftrace** or **perf**. They come away with a reasonable mental model: **eBPF is for observability**. You attach programs to tracepoints. You watch syscalls. You build flamegraphs. It is powerful, and in capable hands, it surfaces things no other tool can.

But **that description mistakes the application for the architecture**. eBPF is **not an observability tool**. It is a **programmable runtime embedded in the Linux kernel** - one that lets you inject verified, JIT-compiled programs at hundreds of points in the kernel's execution path, with no recompilation, no reboot, and with strong safety checks designed to substantially reduce the risk of destabilizing the system you're running on. The observability tools are one application of that. So is Cloudflare's DDoS mitigation, Meta's load balancer, Cilium's Kubernetes networking, and the ability to replace the CPU scheduler entirely.

The path from a packet filter written in 1992 to a runtime that can replace the scheduler is not obvious. This is the explanation of how it happened.

## The Filter

The original **BPF - Berkeley Packet Filter** - was described by Steven McCanne and Van Jacobson in their 1992 paper "***The BSD Packet Filter: A New Architecture for User-level Packet Capture.***" The problem they were solving was specific: tcpdump needed to decide which packets to copy to userspace, but copying every packet and filtering there was prohibitively expensive. The question was whether filtering could happen inside the kernel, before the copy.

Their answer was a small pseudo-machine. You expressed your filter as bytecode - "*capture only TCP packets on port 80*" and the kernel evaluated the bytecode against each arriving packet. If the filter matched, the packet got copied to your tool. If not, the kernel discarded it and never paid the copy cost. The VM itself was deliberately minimal: two 32-bit registers, a small scratch memory area, about 30 instructions. Comparisons, arithmetic, packet field loads, conditional jumps. Just enough to express the class of questions a packet capture tool would ask.

This was enough for two decades. tcpdump, libpcap, socket filters, seccomp - all built on the same VM. But the engineering world had changed. Engineers wanted to observe scheduler events, trace memory allocations, attach logic to arbitrary kernel functions, implement custom security policies in production without modifying the kernel source. The two-register packet filter was not equipped for any of it.

## The Extension

In 2014, Alexei Starovoitov submitted patches that transformed BPF into what became **eBPF (extended BPF)**. The changes were fundamental rather than incremental.

The instruction set expanded from ~30 to over 100. The register width moved from 32-bit to 64-bit. The register count grew from two to eleven - R0 through R10. A 512-byte stack was added. Programs could now call each other (bounded call depth), and they could call a growing set of approved kernel helper functions.

But the most consequential additions were not architectural. They were the verifier and the maps.

## The Verifier

The verifier is the **reason eBPF can exist**.

Loading arbitrary programs into the kernel is dangerous by default. A program that loops indefinitely can freeze a CPU core. A program that accesses memory out of bounds can corrupt kernel state or leak secrets. A null pointer dereference triggers a kernel panic. None of these are acceptable in code that runs in ring 0 on a production machine.

The verifier prevents all of this statically, before the program ever executes. When you load an eBPF program via the bpf() syscall, the **verifier runs** abstract interpretation against the bytecode - a **data flow analysis that simulates every possible execution path**, tracking the state of every register and memory region at every instruction.

It checks four things.

First, **bounded execution:** the program must terminate. eBPF programs may not contain unbounded loops. The verifier rejects any back edge in the control flow graph where termination cannot be proven. Second, **memory safety:** every pointer dereference must be provably within bounds. If the program looks up a value in a map, the verifier requires that the return value is tested for NULL before it is dereferenced. Packet buffer accesses are checked against the known packet length. Stack accesses are bounded to the 512-byte frame. Third, **register initialization:** no register may be read before it is written, through any code path. Fourth, helper **call validity:** every call to a kernel helper must pass arguments of the correct type - the verifier tracks whether a register holds a scalar value, a pointer to a map, a pointer to the stack, or a pointer to packet data, and rejects calls that pass the wrong types.

If the verifier passes, the program is JIT-compiled to native machine code for the host architecture. It does not run as interpreted bytecode - it executes at native speed inside the kernel. The bytecode is the verification artifact; native code is what runs.

The safety guarantee this creates is unusual: a program you wrote, loaded at runtime, executing in kernel space, that the verifier statically checked so the kernel can reject unsafe ones before they run. Those checks are not a total correctness proof - kernel bugs and JIT edge cases have existed - but they are a strong safety gate that makes whole classes of dangerous programs rejectable at load time. That gate is what unlocked every application of eBPF that followed.

## The Maps

**eBPF programs** are event-driven and short-lived. They **run when something happens** - a packet arrives, a syscall fires, a function is entered - and they return. **They do not retain state** between invocations on their own.

**Maps provide the persistence layer.**

An eBPF map is a kernel data structure with a typed key and value, created by userspace via **bpf(BPF_MAP_CREATE, ...**) and held in kernel memory for the lifetime of its file descriptor. Both eBPF programs and userspace processes can read and write the same map simultaneously - eBPF programs through helper functions, userspace through the bpf() syscall or, for the ring buffer type, through direct memory-mapped access.

The ring buffer map type (`**BPF_MAP_TYPE_RINGBUF**`) follows the same shared-memory principle as io_uring: the kernel and userspace share physical memory, so events written by the eBPF program appear in userspace without a syscall per read. A program tracing every context switch can write timestamps directly to the ring; the userspace consumer reads from the same memory.

## The Hooks

An eBPF program does nothing by itself. It must be attached to a hook - a point in the kernel's execution path where the kernel will invoke it. The variety of available hooks is where the architecture's scope becomes concrete.

**XDP (eXpress Data Path)** attaches at the NIC driver level, before the kernel allocates a socket buffer for the arriving packet. The packet exists only as raw memory in the driver ring. The XDP program examines it and returns one of four verdicts: XDP_DROP (discard immediately), XDP_PASS (hand to the network stack), XDP_TX (retransmit out the same interface), or XDP_REDIRECT (forward to another interface or CPU). No socket buffer allocation, no protocol parsing, no netfilter traversal. Cloudflare's DDoS mitigation uses XDP to drop traffic at rates exceeding 10 million packets per second per core - the attack traffic is discarded before the network stack is involved.

**TC (Traffic Control)** attaches after the socket buffer exists, in the tc subsystem on ingress or egress. Programs here can parse the full packet structure, modify headers, redirect flows, and enforce policy. Cilium attaches eBPF at TC to implement Kubernetes network policy and service load balancing without iptables. The iptables rules that kube-proxy would have installed - potentially thousands of rules traversed linearly per packet - are replaced by a hash table lookup in a BPF map.

**Tracepoints** are static hooks defined in the kernel source at semantically meaningful locations: syscall entry and exit, scheduler wakeup and sleep, memory allocation, block I/O submission and completion. They use a kernel-defined tracepoint interface - the hook names and argument layouts are generally stable across kernel versions, though "stable" here is a practical convention rather than a strict ABI guarantee. bpftrace attaches to tracepoints.

**LSM hooks** are the same attachment points used by SELinux and AppArmor, now available to eBPF programs. Custom security policies - written in eBPF, loaded at runtime - run at every security decision point in the kernel.

## The Portable Kernel

Early eBPF tooling compiled programs at runtime on the target machine. The **BCC (BPF Compiler Collection) framework** - which underlies many bpftrace use cases - ships programs as source code and compiles them on the host using LLVM, referencing local kernel headers for struct layouts. Each tool carries a compiler dependency. Running a BCC tool on a production machine requires kernel headers and an LLVM installation.

This worked. But it was operationally fragile: a tool that needs a compiler on every host it touches, using struct layouts that change between kernel versions.

The structural problem is that eBPF programs often access kernel data structures by field - reading task->pid, walking a linked list, examining a socket's state. If you hard-code the byte offset of pid in task_struct, your program reads the right field on the kernel version you compiled against. It may read garbage on a kernel where someone added a field above it.

**CO-RE - Compile Once, Run Everywhere** - solves this through **BTF: BPF Type Format.**

BTF is type information - similar to what DWARF encodes for debuggers, but far more compact - embedded in the kernel image itself at ***/sys/kernel/btf/vmlinux***. Every struct, every field, every typedef, with their offsets for the running kernel. When you compile an eBPF program with CO-RE support using libbpf and Clang, the compiler emits relocation records alongside the bytecode. These records encode intent, not offsets: "this instruction reads field pid of struct task_struct."

When **libbpf** loads the program on the target machine, it reads the running kernel's BTF, resolves the actual offset of task_struct.pid on this kernel version, and rewrites the instruction with the correct offset before loading. The adjustment happens at load time.

The practical consequence: ship your eBPF tool as a compiled binary. It self-relocates on load to work correctly on whatever kernel version it finds - as long as that kernel exposes BTF and the program types it uses are available - without requiring headers or a compiler on the target host.

## The Programmable Kernel

The distinction that matters is not observability versus performance, or kernel versus userspace. It is about where the extension boundary sits. Before eBPF, extending what a production kernel did at runtime required accepting serious operational risk: a kernel module with full kernel privileges and no crash protection, or a userspace workaround that paid the boundary-crossing cost on every operation, or months of waiting for a patch to land upstream and ship in a distribution. The kernel did what it was compiled to do.

eBPF moved that boundary.

A packet filter from 1992 became the extension API for every Linux system running in production today. The mechanism that made it possible was not a new instruction set, not a faster VM, not a better JIT. It was a proof system that lets the kernel say: *this code is safe to run inside me.*

That is what eBPF actually is.
