+++
date = "2026-08-16"
draft = false
title = "Inside Linux's USB Stack: The Abstraction That Made Cables Irrelevant"
+++

A USB serial adapter is plugged into a Windows laptop. Inside WSL2, separated from the host by a Hyper-V virtual switch and a TCP connection, a driver loads for the device. /dev/ttyUSB0 appears. dmesg prints standard USB enumeration: bus number, device number, endpoints. The driver reads and writes the serial port exactly as it would if the adapter were plugged directly into a Linux machine. The adapter is not local. The driver does not know.

That is not a hack or a special case. The driver that loaded, ftdi_sio, one of the most common USB-to-serial drivers in embedded development, runs the same code path whether the device is connected by a five-centimeter cable or routed through a network socket to a machine across the room. Through this interface, it cannot tell the difference. The reason is a function pointer table - a decades-old compatibility abstraction that was clean enough to make physical proximity irrelevant.

## The Contract

Every OS with a USB stack faces the same problem: device drivers need to submit requests without knowing which controller silicon sits underneath. Every major OS solves it with some form of host controller abstraction - NetBSD has its own native VHCI, Windows supports virtual host controllers through frameworks such as UCX. Linux's USB/IP project was among the earliest to use the pattern for device sharing over IP.

In Linux, the contract is a function pointer table called , defined at . The driver describes what it wants - read these bytes, write this configuration, check this endpoint. The contract accepts the request and delivers it. The driver never learns how. Every USB host controller in the system must fill in this table:

The pattern is identical to VFS's struct file_operations - a table descended from SunOS's vnode operation tables in 1985, that decouples the subsystem from any specific backend. The USB driver submits a request. The table routes it. Everything below the table is replaceable.

## The Function Pointer

Follow a USB request from the moment a driver submits it. The driver calls usb_submit_urb() at drivers/usb/core/urb.c. That function validates the request and hands it to usb_hcd_submit_urb() at drivers/usb/core/hcd.c, which resolves which host controller owns this device:

Then it dispatches.

One function pointer call. That is the abstraction boundary. The normal USB request path eventually crosses this critical function-pointer boundary. The obvious reading is straightforward: the call lands in the host controller's enqueue function, which programs silicon, sets up a DMA transfer, and fires the USB transaction down the cable.

Except hcd->driver is a pointer. It calls whatever urb_enqueue the registered controller provided. If the controller is xHCI, the call programs registers and schedules DMA. If the controller is something else, something that has never seen a USB cable, the call does something entirely different. The driver that submitted the request cannot distinguish the two. It handed a request to a function pointer and walked away.

## Why the Contract Existed

The contract was not designed for abstraction's sake. It solved a concrete hardware problem that every operating system faced.

USB 1.0 shipped in January 1996 with a five-meter cable limit. Intel designed UHCI as a host controller interface. Compaq, Microsoft, and National Semiconductor published OHCI - a different register layout, a different scheduling model, a different DMA mechanism. USB 2.0 brought EHCI. USB 3.0 brought xHCI. Four standards across a decade, each incompatible at the register level. The USB specification deliberately left the host controller interface undefined - each vendor designed its own. Every OS needed a way to route USB requests through any of them without forcing device drivers to care which one was present.

 is how Linux solved it. One table, filled in differently by each controller. The USB core submits a request, the table routes it to whichever silicon is registered. The constraint that shaped this design was hardware incompatibility - Intel and Compaq could not agree on a register interface. The kernel needed one interface so USB core could work across four chipsets. The kernel developers who wrote  solved a compatibility problem. They did not know they had also solved a location problem.

## Two Implementations

### Silicon

When xHCI fills in the table, urb_enqueue points at xhci_urb_enqueue(). That function programs memory-mapped registers, configures transfer descriptor rings, and initiates DMA transfers that move data over a physical USB cable to a physical device. This is the path every engineer imagines when they think about USB.

### No Silicon

In 2005, Takahiro Hirofuchi at the Nara Institute of Science and Technology published "USB/IP - a Peripheral Bus Extension for Device Sharing over IP Network". The paper demonstrated USB device sharing over IP using a virtual host controller - a complete hc_driver implementation with no hardware at all.

vhci_hc_driver, defined at drivers/usb/usbip/vhci_hcd.c, fills in the same table.

No physical IRQ line. No DMA setup. No register access. This is a host controller that controls no hardware. When a request arrives at vhci_urb_enqueue, it does not touch a register. It calls vhci_tx_urb(), which does two things:

The request goes onto a linked list. A kernel thread wakes up, serializes the request into a USB/IP network message, and writes it to a TCP socket - a connection to the Windows machine where the physical device is actually plugged in. On the Windows side, usbipd receives the request and passes it to the USB/IP stub driver, which turns it back into a real USB request for the Windows USB stack and controller. The response travels back through the same path: Windows USB stack → usbipd → TCP → VHCI. The Linux driver that originally submitted the request then receives the completion callback it would have received from a locally attached device.

USB/IP began as Hirofuchi's 2003–2005 work on sharing USB devices over IP. It eventually became part of the Linux kernel, and the same VHCI architecture is what usbipd-win uses to attach Windows USB devices into WSL2 today.

## Beyond USB

This pattern is not unique to USB. The VFS uses the same structure, struct file_operations to make a pipe look like a file and an NFS mount look like a local directory. Network stacks use virtual interfaces to route packets through tunnels instead of physical NICs. Virtual machines present emulated hardware that programs behind device registers. Containers change what processes believe exists around them.

The strongest systems abstractions make physical assumptions irrelevant.

VFS made "the file is on this disk" irrelevant. Virtual NICs made "the network is on this wire" irrelevant. The USB contract made "the device is on this machine" irrelevant.

None of the developers who wrote these abstractions set out to erase a physical boundary. They built a clean interface for a compatibility problem, and the interface turned out to be more powerful than the problem it solved.
