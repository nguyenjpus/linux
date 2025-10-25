---
layout: post
title: "Multiple-choices - Questions 11-20 Explained"
date: 2025-09-30 05:15:00 -0700
tags: [Linux+, Multiple-choices]
---

This document explains questions 11-20 from a set of 100 scenario-based multiple-choices questions on Linux system management, focusing on boot troubleshooting, kernel modules, and hardware detection. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention.

## Question 11: Cause of "Kernel panic - not syncing: VFS: Unable to mount root fs"

**Question**: A system fails to boot, displaying "Kernel panic - not syncing: VFS: Unable to mount root fs." What is the most likely cause?  
**Options**:

- The BIOS/UEFI is configured with the wrong boot order.
- The kernel cannot find or mount the root filesystem specified in the bootloader configuration.
- The systemd service has crashed.
- The network interface card is not configured correctly.

**Correct Answer**: The kernel cannot find or mount the root filesystem specified in the bootloader configuration.  
**Why Correct**: This error occurs when the kernel, after loading, cannot locate or access the root filesystem (/) specified in the bootloader (e.g., GRUB’s `root=` parameter). Common causes include a wrong UUID or device path in GRUB, a missing initramfs, or a corrupted filesystem.  
**Why Others Wrong**:

- Wrong boot order (BIOS/UEFI) would prevent the bootloader from loading, not cause a kernel panic.
- Systemd starts after the root filesystem is mounted, so it’s not relevant here.
- Network issues don’t affect mounting the root filesystem unless it’s an NFS root (rare, not implied).  
  **Key Concept**: Kernel panic halts the system when critical components fail. Check GRUB’s `grub.cfg` or `/etc/fstab` for root FS specs.  
  **Memory Aid**: “Kernel panic = Can’t find home (root FS)!”

## Question 12: File to Edit for Persistent Kernel Boot Parameters

**Question**: Which file should be edited to add persistent kernel parameters applied during the next bootloader configuration update?  
**Options**:

- /boot/grub/grub.cfg
- /etc/default/grub
- /proc/cmdline
- /etc/fstab

**Correct Answer**: /etc/default/grub  
**Why Correct**: Editing `/etc/default/grub` (specifically the `GRUB_CMDLINE_LINUX` variable) sets kernel parameters (e.g., `quiet splash`) that persist across boots. After editing, run `update-grub` to apply changes to `/boot/grub/grub.cfg`.  
**Why Others Wrong**:

- /boot/grub/grub.cfg is auto-generated; direct edits are overwritten.
- /proc/cmdline shows current boot parameters (read-only).
- /etc/fstab defines filesystem mounts, not kernel params.  
  **Key Concept**: Use `update-grub` or `grub-mkconfig` to regenerate GRUB config.  
  **Memory Aid**: “/etc/default/grub = Default boot tweaks.”

## Question 13: Loading a Kernel Module Immediately

**Question**: A new network card driver is provided as `new_net.ko`. Which command loads this module into the running kernel immediately?  
**Options**:

- modprobe new_net
- insmod /path/to/new_net.ko
- lsmod | grep new_net
- depmod new_net

**Correct Answer**: insmod /path/to/new_net.ko  
**Why Correct**: `insmod` directly loads a specific kernel module file (`.ko`) into the kernel, requiring the full path to the file (e.g., `/lib/modules/.../new_net.ko`).  
**Why Others Wrong**:

- `modprobe new_net` loads a module by name, resolving dependencies, but doesn’t specify a file path.
- `lsmod | grep new_net` lists loaded modules, not loads them.
- `depmod new_net` updates module dependencies, not loads modules.  
  **Key Concept**: `insmod` is low-level; `modprobe` is preferred for automatic dependency handling.  
  **Memory Aid**: “insmod = Insert module straight in.”

## Question 14: Listing Loaded Kernel Modules

**Question**: Which command lists all currently loaded kernel modules, their memory usage, and dependencies?  
**Options**:

- modinfo
- lsmod
- depmod -a
- dmesg

**Correct Answer**: lsmod  
**Why Correct**: `lsmod` displays a table of loaded kernel modules, including their size (in bytes) and modules that depend on them.  
**Why Others Wrong**:

- `modinfo` shows details about a module (not necessarily loaded).
- `depmod -a` updates module dependency files.
- `dmesg` shows kernel logs, not a module list.  
  **Key Concept**: `lsmod` reads from `/proc/modules`.  
  **Memory Aid**: “lsmod = List system modules.”

## Question 15: Listing USB Devices

**Question**: To troubleshoot a USB webcam, which utility lists all connected USB devices with their buses and device IDs?  
**Options**:

- lspci
- lsusb
- lsblk
- lshw

**Correct Answer**: lsusb  
**Why Correct**: `lsusb` lists USB devices, showing bus and device IDs, vendor/product IDs, and device names, ideal for diagnosing USB hardware issues.  
**Why Others Wrong**:

- `lspci` lists PCI devices (e.g., GPUs, not USB).
- `lsblk` lists block devices (e.g., disks).
- `lshw` lists all hardware but is less specific for USB details.  
  **Key Concept**: Use `lsusb -v` for verbose output.  
  **Memory Aid**: “lsusb = List USB stuff.”

## Question 16: Unloading a Kernel Module

**Question**: To unload a kernel module `buggy_driver` causing instability (no dependencies), which command should be used?  
**Options**:

- insmod -r buggy_driver
- rmmod buggy_driver
- modinfo -r buggy_driver
- blacklist buggy_driver

**Correct Answer**: rmmod buggy_driver  
**Why Correct**: `rmmod` removes a specified kernel module from the running kernel, assuming no dependencies or active use.  
**Why Others Wrong**:

- `insmod -r` is invalid (`insmod` loads, doesn’t unload).
- `modinfo -r` doesn’t exist (`modinfo` shows module info).
- `blacklist` prevents auto-loading, not unloads.  
  **Key Concept**: Use `modprobe -r` for dependency-aware unloading.  
  **Memory Aid**: “rmmod = Remove module now.”

## Question 17: Viewing Kernel Module Metadata

**Question**: Which command displays metadata (author, description, license, parameters) for a module `special_driver`?  
**Options**:

- modprobe --show-info special_driver
- lsmod special_driver
- modinfo special_driver
- dmesg | grep special_driver

**Correct Answer**: modinfo special_driver  
**Why Correct**: `modinfo` shows detailed metadata for a kernel module, including author, license, description, and parameters, even if not loaded.  
**Why Others Wrong**:

- `modprobe --show-info` is invalid.
- `lsmod` lists loaded modules, not metadata.
- `dmesg | grep` shows kernel logs, not structured metadata.  
  **Key Concept**: Find module files in `/lib/modules/$(uname -r)/`.  
  **Memory Aid**: “modinfo = Module info.”

## Question 18: Auto-Loading a Kernel Module at Boot

**Question**: What’s the modern approach to ensure the `power_saver` module loads automatically at boot?  
**Options**:

- Add an insmod command to /etc/rc.local
- Create a file in /etc/modules-load.d/ with the module name
- Edit /boot/grub/grub.cfg to include the module
- Manually run modprobe power_saver after each boot

**Correct Answer**: Create a file in /etc/modules-load.d/ with the module name  
**Why Correct**: Creating a file (e.g., `/etc/modules-load.d/power_saver.conf`) with `power_saver` ensures systemd loads the module at boot.  
**Why Others Wrong**:

- `/etc/rc.local` is outdated and not always used.
- `/boot/grub/grub.cfg` is for boot params, not modules.
- Manual `modprobe` isn’t persistent.  
  **Key Concept**: Systemd’s `modules-load.d` is the standard for modern distros.  
  **Memory Aid**: “modules-load.d = Load modules at dawn (boot).”

## Question 19: Listing PCI Devices

**Question**: Which command lists all PCI devices to identify a storage controller model?  
**Options**:

- lsusb -v
- lspci
- dmidecode -t storage
- lsblk

**Correct Answer**: lspci  
**Why Correct**: `lspci` lists all PCI devices (e.g., storage controllers, GPUs) with vendor and device IDs, perfect for identifying hardware.  
**Why Others Wrong**:

- `lsusb -v` is for USB devices.
- `dmidecode -t storage` shows system hardware info, less specific.
- `lsblk` lists block devices, not PCI details.  
  **Key Concept**: Use `lspci -v` for verbose output.  
  **Memory Aid**: “lspci = List PCI components.”

## Question 20: Blacklisting a Kernel Module

**Question**: How to persistently prevent a kernel module from loading automatically?  
**Options**:

- Add a blacklist [module_name] line to a file in /etc/modprobe.d/
- Delete the module’s .ko file from /lib/modules/
- Run rmmod [module_name] every boot
- Set the module’s file permissions to 000

**Correct Answer**: Add a blacklist [module_name] line to a file in /etc/modprobe.d/  
**Why Correct**: Adding `blacklist [module_name]` to a file like `/etc/modprobe.d/blacklist.conf` prevents the module from auto-loading at boot.  
**Why Others Wrong**:

- Deleting the `.ko` file is destructive and not recommended.
- `rmmod` every boot isn’t persistent.
- Setting permissions to 000 may not work and causes issues.  
  **Key Concept**: Blacklisting doesn’t affect manual loading with `modprobe`.  
  **Memory Aid**: “Blacklist in modprobe.d = Ban module from booting.”

## Retention Tips for Questions 11-20

- **Themes**: Boot troubleshooting (kernel panic, GRUB config), kernel module management (`insmod`, `rmmod`, `modprobe`, `modinfo`, `lsmod`), and hardware detection (`lsusb`, `lspci`).
- **Mnemonic for Kernel Modules**: “Insmod Inserts, Rmmod Removes, Modprobe Manages, Modinfo Mentions, Lsmod Lists” (I-R-M-M-L).
- **Practice**: In a Linux VM:
  - Check loaded modules with `lsmod`.
  - Run `lsusb` and `lspci` to see devices.
  - Simulate a blacklist by creating `/etc/modprobe.d/test.conf` with `blacklist floppy`.
  - Edit `/etc/default/grub` to add a parameter (e.g., `quiet`), then run `sudo update-grub`.
- **Spaced Repetition**: Review these explanations in 24 hours, then in 3 days. Create flashcards (e.g., “What does lsusb show?” → “USB devices, bus/device IDs”).
- **Quiz Yourself**: What causes a kernel panic about VFS? How do you blacklist a module?
