+++
date = "2026-04-02"
draft = false
title = "Inside the Live Update Orchestrator: How Linux Solved Kernel Updates for Running VMs"
+++

A critical CVE dropped. The host kernel needs a patch. It's running 200 virtual machines, all belonging to different customers, all running something that can't go down. The provider has two options: patch and reboot, killing and migrating every tenant workload, or leave the vulnerability sitting there. This choice has defined cloud operations for twenty years.

In February 2026, Linux merged a subsystem that solves it directly. The **Live Update Orchestrator, or LUO**, lets the kernel replace itself - entirely - while running virtual machines survive the transition. Not a patch. Not a migration. A full kernel swap, in place.

Here's how four failed attempts and one reframed question got us here.

## The four failures

The problem looks simple at first. Make some memory survive a **kexec** reboot. Kexec already **lets you jump from one kernel to another without touching BIOS**. The missing piece is state - when the new kernel boots, everything from the old one is gone. VM memory, device mappings, IOMMU tables, all wiped.

Between 2022 and 2024, at least four separate approaches tried to fix this. **Memory Pools** pre-allocated persistent regions that survived the reboot. **PRMEM** created resizable persistent regions with metadata pointers on the kernel command line. **Pkernfs** built an in-memory filesystem for kernel data. **PKRAM** preserved userspace pages via fixed metadata addresses.

Every one of them had the same flaw: they required the administrator to manually carve out a physical memory region before the kexec, because there was no mechanism outside the kernel command line to pass information between the old kernel and the new one. The administrator had to guess how much memory to reserve, where to put it, and hope the new kernel could find it. On a production hypervisor running hundreds of VMs with terabytes of RAM, that's operationally unworkable.

They failed not because the engineering was bad. They failed because they were answering the wrong question.

## The question that worked

Alexander Graf at Amazon, Mike Rapoport at Microsoft, and Changyuan Lyu at Google landed on a different framing: *what if the new kernel inherited the old kernel's memory, instead of the old kernel trying to preserve its own?*

The difference is subtle but it changes everything architecturally. Instead of the old kernel carving out special regions and hoping the new kernel finds them, the **old kernel produces** a manifest - a **Flattened Device Tree blob** - that describes everything worth keeping. The new kernel reads that manifest during boot and claims the preserved memory as its own. The preserved pages **look like they were allocated from** the **new kernel's buddy allocator**. No special regions. No command-line parameters. No guessing.

This became **Kexec Handover, or KHO**. It merged in Linux 6.16 in June 2025.

## The architecture

Three layers. Understanding them is the key to understanding why it works.

```text
┌─────────────────────────────────────────────────┐
│ Layer 3: LUO (Live Update Orchestrator)        │ Linux 6.19
│ State machine + subsystem callback orchestration│
│ Userspace interface: /dev/liveupdate            │
├─────────────────────────────────────────────────┤
│ Layer 2: KHO (Kexec Handover)                  │ Linux 6.16
│ Memory preservation + FDT metadata passing      │
│ Per-NUMA-node scratch regions (CMA-backed)      │
├─────────────────────────────────────────────────┤
│ Layer 1: kexec                                 │ (existing)
│ Direct kernel-to-kernel boot, bypasses BIOS     │
│ Standard: resets all devices, no state         │
└─────────────────────────────────────────────────┘
```

**Layer 1 is kexec** - it's been around for years. Jump from one kernel to another without touching BIOS or firmware, saving tens of seconds of POST time. Standard kexec is destructive, though. All state is gone.

**Layer 2 is KHO**, the memory layer. When a subsystem wants memory to survive the kexec, it calls `kho_preserve_folio()`. KHO tracks every preserved folio, coalesces them into contiguous regions where possible, and serializes metadata into an FDT blob. The new kernel reads the FDT during early boot, marks preserved regions as reserved, and subsystems call `kho_restore_folio()` to reclaim their pages. To the new kernel, those pages look like it allocated them.

But KHO has a bootstrap problem. The new kernel needs physically contiguous memory to boot - page tables, initial data structures, decompression buffers. That memory can't overlap with preserved pages. The solution is scratch regions: CMA-backed areas pre-allocated on each NUMA node. Because CMA guarantees only movable pages live inside, preserved pages never end up there. After the kexec, the scratch regions get reused, which means the kernel can be swapped again and again without accumulating overhead.

**Layer 3 is LUO**, the orchestration layer. KHO preserves memory. LUO decides when and in what order. It provides a state machine that coordinates the entire transition:

```text
NORMAL ──PREPARE──► PREPARED
  ▲                    │
  │                 FREEZE
  │                    ▼
  └── FINISH ── UPDATED ◄── FROZEN ── kexec
```

In NORMAL state, everything runs as usual. When an update starts, LUO moves to PREPARED: subsystems serialize their state while workloads keep running. Preparation happens while VMs are still active. The FREEZE transition - triggered by the reboot syscall - stops workloads and moves the system to FROZEN. That window is designed to be sub-second. Then kexec fires, the new kernel boots, reads the KHO manifest, and enters UPDATED. Subsystems restore their state, and LUO moves back to NORMAL.

If anything fails during the freeze, LUO calls  on every subsystem that already froze, rolling back to PREPARED. The workload resumes. The update can be retried or cancelled.

## What survives, what gets rebuilt

```text
Preserved across kexec       │ Rebuilt from scratch
─────────────────────────────┼──────────────────────
VM guest RAM                 │ Kernel page tables
IOMMU page tables             │ Scheduler state
VFIO device context           │ Userspace processes
DMA mappings                  │ Network stack
KVM virtual machine state     │ Filesystem mounts
```

The **left column is what matters.** VM guest RAM is preserved. The IOMMU page tables that map device DMA into guest memory are preserved. The VFIO device context - PCI config space, BAR mappings - for passthrough devices is preserved. DMA keeps running during the transition. The guest never knows the host kernel changed underneath it.

The right column - **Everything that makes the host kernel a host kernel gets rebuilt from scratch.** New page tables. New scheduler. New network stack. This is the real difference from live patching: LUO doesn't accumulate state from previous kernels. Every update starts clean.

## The device problem

Preserving memory is clean. Preserving devices is not.

When the old kernel kexecs, every PCIe device on the system is still running. NICs are processing packets. NVMe drives are completing I/O. GPUs are executing compute shaders. Reset those devices during the transition and you lose in-flight operations. Don't reset them and the new kernel inherits device state it never configured.

LUO's approach: don't touch the device. During the freeze, the IOMMU marks all Interrupt Remapping Table Entries as non-present. Interrupts from devices are silently dropped. DMA continues - the IOMMU page tables are preserved, so devices keep reading and writing guest memory through the same translations. After the new kernel boots, it restores interrupt vectors and blindly injects all previously configured interrupts into the guest. The guest sees a brief interrupt gap, not a device reset.

The constraint is that every device whose state must survive needs a KHO-aware driver. IOMMU and VFIO preservation patches landed in early 2026. NVMe and network card preservation are still in development. The architecture supports it - any subsystem can register with LUO and implement preserve/restore callbacks - but the ecosystem isn't there yet.

## Why VMs first

There's a reason LUO targets virtualized workloads, and it goes beyond the fact that cloud providers funded the work.

VMs are a good candidate because their state is already encapsulated. Guest RAM is backed by memfd - an anonymous memory-backed file descriptor - which means the kernel already tracks it as discrete, preservable folios. KVM holds virtual CPU state, memory mappings, and device emulation context in well-defined structures. VirtIO devices are entirely software-defined, living in kernel memory rather than hardware registers.

Compare that to a bare-metal workload. A running PostgreSQL instance has state in kernel page cache, file descriptors, network connections, shared memory segments, signal handlers, and process-local memory. Preserving a bare-metal process across a kernel swap would require preserving essentially the entire kernel. A VM's state is self-contained. The host kernel can be replaced around it like changing the engine fuel mid-flight while the passengers keep watching their movies.

## The numbers

No official end-to-end benchmark has been published yet. But the components are measurable. ACPI initialization normally takes over 100ms after a kexec. A fast handover optimization presented at Linux Plumbers Conference 2024 saves 100-300ms by caching ACPI state.

The goal: faster than live migration, with none of the operational overhead. No spare hosts, no network transfer, no capacity planning.

## The cost

LUO is not free.

The biggest risk is no automatic rollback. If the new kernel crashes during or after the handover, you can't go back. Preserved memory regions still exist in physical RAM, but if the new kernel panics before restoration completes, you're looking at a cold reboot. That's a deliberate call - building real rollback for a full kernel swap would mean keeping the old kernel image in memory, doubling the overhead. The trade-off makes sense. It still makes some operators uncomfortable.

## What changes

For cloud providers, this removes a trade-off that's defined operations since the first VM ran on shared hardware. A critical CVE drops on Monday. By Monday afternoon, every host in the fleet is running a patched kernel. No VM migrations. No customer-visible downtime. No spare capacity held in reserve.

For GPU cloud - AWS P5, Azure NDv5, GCP A3 - the stakes are higher. GPU VMs using VFIO passthrough can't be live-migrated. The GPU state is too complex, the device is too tightly coupled to the host. Today, patching the kernel on a GPU host means stopping the VM. With LUO and VFIO preservation, GPU VMs could survive kernel updates in place. For an AI training job running on 1024 GPUs, the difference between "restart your job" and "you didn't notice" is measured in hundreds of thousands of dollars.

## The roster

Four years, five companies, at least a dozen engineers:

Amazon, Microsoft, and Google co-authored the patches. Companies that compete on everything else. That alone tells you how bad the underlying problem was.

---

There's a pattern in systems engineering that repeats. A problem gets declared unsolvable. Workarounds accumulate. The workarounds get elaborate enough that they start looking like solutions. Then someone reframes the question, and the actual solution turns out to be simpler than all the workarounds combined.

Live patching was a workaround. Live migration was a workaround.

The kernel learned to swap itself. VM guest RAM survives. The kernel state gets rebuilt. Devices keep running. The VMs never notice.

Twenty years of "pick one" ended with a 2,200-line state machine and a Flattened Device Tree blob.

That's the thing about the right question - the answer tends to fit in a commit message even if the change is big!
