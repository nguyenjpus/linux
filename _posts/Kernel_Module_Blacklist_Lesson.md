# Lesson Learned: Troubleshooting Kernel Module Blacklisting in Ubuntu

## Overview

This document summarizes the lessons learned from a practice exercise (Sub-Part 4.1: Kernel Module Fails to Load) conducted on August 21, 2025, in an Ubuntu VM (kernel 6.14.0-28-generic, aarch64) running in UTM. The goal was to simulate a kernel module failing to load by blacklisting the `parport` module (since `floppy` was unavailable), identify the cause, resolve it, and explore a variation with a non-existent module. The exercise revealed nuances in Linux kernel module blacklisting and required a stricter approach to enforce it.

## Steps and Key Lessons

### Step 1: Verify Module Availability

- **Action**: Checked if the `parport` module exists using:
  ```bash
  modinfo parport
  ```
  Output confirmed the module is available in `/lib/modules/6.14.0-28-generic/kernel/drivers/parport/parport.ko.zst`.
- **Lesson**: Always verify a module exists with `modinfo` before blacklisting. Use non-critical modules (e.g., `parport`, `joydev`) for safe testing, especially in VMs without hardware like floppy drives.

### Step 2: Simulate Issue by Blacklisting

- **Action**: Added `blacklist parport` to `/etc/modprobe.d/blacklist.conf`, updated initramfs (`sudo update-initramfs -u`), and rebooted.
  ```bash
  sudo nano /etc/modprobe.d/blacklist.conf
  # Added: blacklist parport
  sudo update-initramfs -u
  sudo reboot
  ```
- **Lesson**: The `blacklist` directive in `/etc/modprobe.d/` prevents automatic module loading (e.g., by udev). Ensure file permissions are `644` (`-rw-r--r--`) and syntax is correct (`blacklist <module>`).

### Step 3: Identify the Cause

- **Actions**:
  - Checked if `parport` was loaded: `lsmod | grep parport` (empty, expected, as no parallel port hardware in UTM).
  - Checked kernel logs: `sudo dmesg | grep parport` (empty, due to no hardware probing).
  - Tried manual loading: `sudo modprobe parport` unexpectedly succeeded, showing:
    ```
    parport                73728  0
    ```
- **Issue**: The `blacklist parport` directive didn’t prevent manual loading with `sudo modprobe`.
- **Troubleshooting**:
  - Added `modprobe.blacklist=parport` to `/etc/default/grub` (`GRUB_CMDLINE_LINUX_DEFAULT`), updated GRUB (`sudo update-grub`), and rebooted.
  - Still, `sudo modprobe parport` succeeded, indicating `blacklist` alone wasn’t enough.
  - Used a stricter blacklist: `install parport /bin/false` in `/etc/modprobe.d/blacklist.conf`, updated initramfs, and rebooted.
  - Result: `sudo modprobe parport` failed with:
    ```
    modprobe: ERROR: ../libkmod/libkmod-module.c:1072 command_do() Error running install command '/bin/false' for module parport: retcode 1
    modprobe: ERROR: could not insert 'parport': Invalid argument
    ```
    `lsmod | grep parport` was empty, confirming the blacklist worked.
- **Lessons**:
  - The `blacklist` directive prevents automatic loading but may allow manual `modprobe` with root privileges in some kernels (e.g., 6.14.0-28-generic).
  - The `install <module> /bin/false` directive is stricter, blocking all loading attempts by running `/bin/false`.
  - Use `sudo dmesg` for kernel logs, as non-root `dmesg` is restricted (`Operation not permitted`).
  - Check `/proc/cmdline` to verify kernel parameters like `modprobe.blacklist=<module>`.

### Step 4: Resolve the Issue

- **Action**:
  - Removed `install parport /bin/false` from `/etc/modprobe.d/blacklist.conf` and `modprobe.blacklist=parport` from `/etc/default/grub`.
  - Updated configurations:
    ```bash
    sudo update-grub
    sudo update-initramfs -u
    sudo reboot
    ```
  - Verified: `sudo modprobe parport` succeeded, and `lsmod | grep parport` showed:
    ```
    parport                73728  0
    ```
- **Lesson**: Removing blacklist entries and updating initramfs/GRUB restores normal module loading. Always verify with `lsmod` and `modprobe`.

### Step 5: Practice Variation - Missing Module

- **Action**:
  - Tried loading a non-existent module: `sudo modprobe fake_module`.
  - Expected output:
    ```
    modprobe: FATAL: Module fake_module not found in directory /lib/modules/6.14.0-28-generic
    ```
  - Checked logs: `sudo dmesg | grep fake_module` (likely empty).
- **Lesson**: A “not found” error indicates a missing module, distinct from a blacklist’s “Operation not permitted” or “Invalid argument”. This simulates real-world issues like typos or outdated configs.

## UTM and System Context

- **Environment**: Ubuntu VM in UTM (aarch64, kernel 6.14.0-28-generic)
- **Impact**: UTM’s virtualized environment (no parallel port hardware) meant `parport` didn’t auto-load, focusing the exercise on manual loading. The blacklist issue was kernel-related, not UTM-specific.

## Key Takeaways

1. **Blacklisting Modules**:
   - Use `blacklist <module>` in `/etc/modprobe.d/` for automatic loading prevention.
   - Use `install <module> /bin/false` for stricter blocking of all loading attempts.
   - Add `modprobe.blacklist=<module>` to GRUB for boot-time enforcement. << this is not very effective - just optional>>
2. **Diagnostics**:
   - Use `lsmod`, `modinfo`, and `sudo dmesg` to check module status and errors.
   - Verify configurations with `grep -r <module> /etc/modprobe.d/` and `/proc/cmdline`.
3. **Updates**:
   - Run `sudo update-initramfs -u` and `sudo update-grub` after changing module or GRUB configs, followed by a reboot.
4. **Error Types**:
   - Blacklist failure: “Operation not permitted” or “Invalid argument”.
   - Missing module: “Module not found”.
