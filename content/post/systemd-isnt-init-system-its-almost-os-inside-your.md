+++
date = "2026-04-19"
draft = false
title = "Systemd Isn’t an Init System - It’s Almost an OS Inside Your OS"
+++

I used to think systemd was solving the wrong problem.

It felt like an init system that refused to stay in its lane - pulling in logging, networking, resource control, and more. If you’ve worked with Linux long enough, you’ve probably felt that discomfort too. By the standards of the Unix philosophy, it looks like scope creep.

And yet, the same system is quietly running most modern Linux distributions - starting services faster, recovering from crashes more gracefully, and providing visibility that previously required an entire observability stack.

So which is it? A design mistake, or a necessary evolution?

Both sides are correct. They’re just evaluating different systems.

One side evaluates systemd against what an init system should be. The other evaluates it against what it actually does. The mismatch between those two perspectives is where most of the confusion and the debate comes from.

Once you shift your lens, the picture becomes clearer.

**systemd** is **no longer just an init system.**

It behaves like a userspace management platform - **an orchestration layer between the kernel and applications** that organizes resources, logs, and dependencies into something coherent.

The init system is just the entry point. The platform is the real story.

## The Init System That Isn’t

In 2013, Lennart Poettering described systemd as "69 individual binaries.". As of recent releases, that number is well over one hundred.

If you judge systemd purely as an init system, the criticism is valid.

An init system traditionally starts processes and manages their lifecycle. systemd does that -but it also includes a DNS resolver, logging system, network manager, container runtime, time synchronization daemon, and more. This is not a small tool anymore.

But that expansion is not accidental. It reflects a design thesis: t**he kernel provides mechanisms, but not policy.** systemd fills that gap by organizing those mechanisms into a consistent userspace model.

The key shift is this: **the kernel sees processes. systemd makes the system see services**.

That distinction unlocks everything else.

## The Socket Trick

Imagine a traditional Linux system booting up. The** logging daemon (syslog)** must start before the **message bus (dbus)**, because dbus wants to log messages. The message bus service must start before the Bluetooth daemon, and so on.

This creates a **serial dependency chain**. Each service waits for the one before it to finish initializing. On a modern server with dozens of services, this sequential waiting adds seconds - or tens of seconds - to boot time. Worse, if one service in the chain fails, everything behind it stalls.

systemd solved this problem with an impressive idea: **socket activation - separate the socket from the service.**

The mechanism is surprisingly simple: **file descriptor inheritance.**

Every Unix process has a small table of open files, numbered 0, 1, 2, 3, and so on. These numbers are **file descriptors**.
- 0 is standard input (your keyboard).
- 1 is standard output (your terminal).
- 2 is standard error (also your terminal).

When systemd starts a service, it can pass already-open sockets to it—pre-bound, already listening. Instead of creating its own socket, the service simply receives one.

The handoff looks like this:

systemd opens the socket. Your service just consumes it.

This small shift has large consequences.

systemd can create all required sockets upfront - TCP, UDP, Unix sockets, pipes - before starting any services. Once those sockets exist, services no longer need to start in a strict order. They can all start in parallel.

If a service isn’t ready yet, incoming requests wait in the kernel buffer.

If a service crashes, the socket survives. New requests queue up until the service restarts.

If a service is rarely used, it doesn’t need to run at all. The first incoming connection starts it on demand.

The socket becomes the stable interface.

The process behind it becomes replaceable.

## The Resource Governor

The Linux kernel exposes resource control through cgroups, but the interface is low-level and difficult to manage directly.

systemd turns this into a policy layer by mapping services to cgroups automatically and allowing limits to be defined declaratively:

These limits apply to the entire service, not just individual processes. Any child processes inherit the same constraints, which eliminates tracking issues across forks.

Without systemd, achieving the same result would require manually creating and managing cgroup hierarchies.

systemd compresses that complexity into a single abstraction: the service.

## The Logging Engine

Traditional Unix logging is text. A daemon writes a string to syslog. Syslog appends it to a file. Humans and scripts process it with grep, awk, and sed. This model has survived since the 1980s. It is simple, portable, and universal.

journald replaces this with binary structured logging. Every log entry is stored as a set of typed key-value fields. The human-readable message is just one field among dozens. The journal automatically attaches metadata that the logging process never provided and cannot forge:

Each field is indexed in a hash table. This makes queries fast without external infrastructure:

No parsing. No pipelines. Just direct queries.

The tradeoff is losing plain text simplicity. The benefit is gaining indexed, trustworthy logs without external infrastructure.

## The Nervous System

systemd’s components are not just co-located - they are connected. This connection happens through **D-Bus**, which **acts as the control plane for the entire system**.

When you run ***systemctl start sshd.service, ***systemctl does not directly manipulate any process. It sends a **D-Bus method call to PID 1**: *StartUnit("sshd.service", "replace")*. PID 1 executes the request and emits a signal when the unit's state changes. systemctl is a thin client. All intelligence lives in PID 1.

**Every unit is exposed as a D-Bus object**. The service **sshd.service** becomes an object at the path ***/org/freedesktop/systemd1/unit/sshd_2eservice***.

Without this shared context, each subsystem would operate in isolation. With it, they behave as a coordinated system.

That coordination is the defining characteristic of a platform.

## The Platform Layer

At a high level, systemd mirrors the kernel’s responsibilities - but in userspace.

The kernel provides mechanisms: processes, memory, communication, and devices. systemd provides structure: services, policies, and relationships between those mechanisms.

This mapping is what makes the “platform” model useful. **systemd** doesn’t replace the kernel -it **organizes** it.

And that organization enables features that loosely coupled tools struggle to provide: service-level resource accounting, consistent logging attribution, coordinated startup and recovery, shared system-wide context.

These are not individual features. They are emergent properties of integration.

## The Tradeoff

The criticism of systemd is not wrong.

Integration creates dependency. Replacing one component often means replacing many. The system becomes harder to decompose into independent parts.

But the alternative has its own cost.

Loose coupling provides flexibility, but it also fragments context. Each tool operates with partial information, and stitching them together requires additional layers of complexity.

systemd chooses the opposite tradeoff: **tighter integration in exchange for shared understanding.**

---

Some engineers value replaceability and minimalism. Others value integration and capability.

systemd is not an init system that grew too large.

It is an answer to a question the Unix philosophy never asked:

*What happens when the space between the kernel and applications becomes complex enough to need its own organizing layer?*

Whether that answer is correct depends on what you value more.

The freedom to replace any part. Or the power that comes from parts that understand each other.

The kernel manages hardware.

systemd manages the space in between.

And that space is where most of the real complexity lives.
