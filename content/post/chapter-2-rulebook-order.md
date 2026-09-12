+++
date = "2025-11-02"
draft = false
title = "The Udev in Linux: The Rulebook of Order"
+++

Your smartphone knows what to do when you plug in headphones: pause the speaker, route audio, maybe launch your music app. When you get in your car, it connects to Bluetooth and opens your navigation.

Different triggers, different responses - all automatic.

Your phone isn't psychic.

Someone wrote rules for each one of those behavior.

Udev operates on the same principle, but with infinite flexibility. If the [Name Problem](/post/chapter-1-name-problem/) was about identity, this chapter is about purpose.

Think about how you've automated your life:
- "When my alarm goes off, turn on the coffee maker"
- "When I arrive home, unlock the door and turn on the lights"
- "When someone emails me with [URGENT], send a whatsapp notification"

These are all rules in the form: **Condition → Action**

Udev rules follow the exact same pattern, just for hardware instead of your daily routine. Based on the available pre-defined rules, udev manages each device dynamically.

## Understanding the Tools: Your udev Detective Kit

Before we write rules, let's look into some of the important tools when working with udev.

### Step 1: Discover Your Device

```bash
lsblk
```

### Step 2: Investigate with udevadm

```bash
udevadm info --query=all --name=/dev/sdb1
udevadm info --query=all --name=/dev/sdb1 | grep -E "(MODEL|SERIAL|VENDOR|UUID|ID_FS)"
```

This command reveals everything udev knows about your device: manufacturer, model, serial number, filesystem type, and much more.

## The Udev Rules System: Anatomy of Your First Rule

Rules live in a carefully organized hierarchy:
- **/lib/udev/rules.d/** - System defaults (don't edit these!)
- **/etc/udev/rules.d/ **- Your custom rules (they take priority)
- **/run/udev/rules.d/ **- Runtime rules (temporary)

Let's solve a real problem: **"My SD card should always be accessible at /dev/mybackup"**

### Step 1: Find Unique Identifiers

After running udevadm info, you might see:

```text
E: ID_MODEL=Extreme_Pro
E: ID_SERIAL=SanDisk_Extreme_Pro_1234567890
E: ID_FS_UUID=abcd-1234
E: ID_VENDOR=SanDisk
```

### Step 2: Create Your Rule

```bash
sudo nano /etc/udev/rules.d/99-mybackup.rules
```

Add this content:

```udev
SUBSYSTEM=="block", ATTRS{serial}=="SanDisk_Extreme_Pro_1234567890", SYMLINK+="mybackup"
```

Let's break this down word by word:
- **SUBSYSTEM=="block"** - Only match block devices (disks, USB drives)
- **ATTRS{serial}=="SanDisk_Extreme_Pro_1234567890"** - Find the device with this exact serial number
- **SYMLINK+="mybackup"** - Create a symbolic link called "mybackup" in /dev

### Step 3: Activate Your Rule

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -l /dev/mybackup
```

## Rule Language Cheat Sheet

### Match Conditions (The "WHEN"):

- **SUBSYSTEM=="usb"** - USB devices
- **SUBSYSTEM=="block"** - Storage devices
- **KERNEL=="sd*"** - Device names starting with "sd"
- **ATTRS{idVendor}=="1234"** - Match vendor ID
- **ENV{ID_FS_UUID}=="abcd" -** Match filesystem UUID

### Actions (The "DO"):

- **SYMLINK+="mydrive"** - Create permanent name
- **RUN+="/path/script"** - Run a script
- **OWNER="john"** - Set device owner
- **GROUP="users"** - Set device group
- **MODE="0660"** - Set permissions

## Common Beginner Mistakes & Solutions

- **[Mistake 1] Rules not working**: Check syntax with sudo udevadm test /sys/block/sdb1
- **[Mistake 2] Scripts not running: **Ensure scripts are executable (chmod +x)
- **[Mistake 3] Multiple rules conflicting: **Rules run in numerical order (00-first to 99-last)
