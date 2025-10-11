---
layout: post
title: "Linux Lab Summary: Mastering the Boot Process and Recovery"
date: 2025-10-10 10:10:00 -0700
tags: [Linux+, Linuxlab, kernel, modules, troubleshooting]
---

This lab guided me through kernel module management on an Ubuntu 24.04 LTS (`aarch64`) virtual machine (VM) running in UTM on an M1 MacBook, using kernel `6.8.0-85-generic` (with `6.8.0-79-generic` as fallback, verified via `dpkg --list | grep linux-image`). The lab addressed specific questions (3, 6, 8, 10, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 93) across five sections, covering kernel basics, hardware detection, building custom modules (`hello.ko` and `dep.ko`), advanced module management, and troubleshooting scenarios like driver conflicts and hardware failures. You worked as `root` (via `sudo -i`) for most commands, used UTM snapshots for safety, and configured two network interfaces: `enp0s1` (`virtio_net`, `10.0.0.20/24` for SSH) and `enp0s2` (`e100`, `10.0.0.21/24`). Key tools included `lsmod`, `modprobe`, `rmmod`, `depmod`, `lspci`, `ethtool`, `dmesg`, and `journalctl`. This summary includes all hands-on steps, their purposes, your outputs, the creation of `hello.ko` and `dep.ko`, questions addressed, and all ten bookmarks with explanations.

## Lab Setup

- **Environment**:
  - **OS**: Ubuntu 24.04, `aarch64`.
  - **Kernel**: `6.8.0-85-generic`.
  - **Network**: `enp0s1` (`virtio_net`, `10.0.0.20/24`), `enp0s2` (`e100`, `10.0.0.21/24`).
  - **VM**: UTM, 2 GB RAM, 32 GB disk, 2 CPUs.
- **Packages**:
  - Installed: `vim`, `hwinfo`, `usbutils`, `pciutils`, `build-essential`, `linux-headers-$(uname -r)`, `dkms`.
  - Command: `sudo apt update && sudo apt install -y vim hwinfo usbutils pciutils build-essential linux-headers-$(uname -r) dkms`.
  - Why: Provides tools for editing, hardware inspection, and kernel module compilation.
- **Safety**:
  - Took UTM snapshots before risky operations (e.g., module loading).
  - Why: Protects against crashes from unstable modules.
- **Questions Addressed**:
  - None directly, but sets the stage for hardware and module management.

## Section 1: Kernel and Module Basics

- **Goal**: Understand kernel version, module listing, Virtual File System (VFS), and kernel logs.
- **Questions Addressed**:
  - **Question 3**: What is the kernel version and architecture?
  - **Question 6**: How to list loaded kernel modules?
  - **Question 8**: What is the role of VFS in Linux?
  - **Question 10**: What is the kernel’s role in hardware management?
- **Steps and Why**:
  1. **Check Kernel Version (Question 3)**:
     - Command: `uname -r`.
     - Output: `6.8.0-85-generic`.
     - Why: Confirms kernel for module compatibility.
     - Command: `arch` or `uname -m`.
     - Output: `aarch64`.
     - Why: Verifies 64-bit architecture for memory handling.
  2. **List Loaded Modules (Question 6)**:
     - Command: `lsmod | grep virtio`.
     - Output: `virtio_net`, `virtio_gpu`, `virtio_rng`.
     - Why: Shows modules managing hardware (e.g., network, GPU).
  3. **Explore VFS (Question 8)**:
     - Command: `sudo mkdir /mnt/tmpfs; sudo mount -t tmpfs none /mnt/tmpfs; touch /mnt/tmpfs/testfile; cat /mnt/tmpfs/testfile`.
     - Output: Empty file (no errors).
     - Cleanup: `sudo umount /mnt/tmpfs; sudo rmdir /mnt/tmpfs`.
     - Why: Demonstrates `tmpfs` (RAM-based) and VFS abstraction for uniform filesystem access.
  4. **View Kernel Logs (Question 10)**:
     - Command: `dmesg -T | grep virtio`.
     - Output: `[Thu Oct 9 20:23:03 2025] virtio_net virtio1 enp0s1: renamed from eth0`.
     - Why: Shows kernel’s role in hardware detection (e.g., network interface renaming).
- **Outputs**:
  - `uname -m`: `aarch64`.
  - `lsmod | grep virtio`: `virtio_net`, `virtio_gpu`, `virtio_rng`.

## Section 2: Hardware Detection

- **Goal**: Inspect hardware (PCI, USB), verify network drivers, and integrate logs.
- **Questions Addressed**:
  - **Question 15**: How to inspect USB devices?
  - **Question 19**: How to identify hardware and drivers?
  - **Question 22**: How to use kernel logs for hardware diagnosis?
- **Steps and Why**:

  1. **List PCI Devices (Question 19)**:

     - Command: `lspci`.
     - Output:

       ```
       00:02.0 Ethernet controller: Intel Corporation 82801BA/BAM/CA/CAM Ethernet Controller (rev 03)
       00:04.0 Audio device: Intel Corporation 82801FB/FBM/FR/FW/FRW (ICH6 Family) High Definition Audio Controller (rev 01)
       ```

     - Why: Identifies hardware for driver troubleshooting.

  2. **Check Drivers (Question 19)**:

     - Command: `lspci -k`.
     - Output:

       ```
       00:02.0 Ethernet controller: ... Kernel driver in use: e100
       00:04.0 Audio device: ... Kernel driver in use: snd_hda_intel
       ```

     - Command: `ethtool -i enp0s2`.
     - Output:

       ```
       driver: e100
       bus-info: 0000:00:02.0
       ```

     - Why: Confirms drivers (`e100` for `enp0s2`, `snd_hda_intel` for audio).

  3. **Inspect USB Devices (Question 15)**:
     - Command: `lsusb`.
     - Output: QEMU Tablet.
     - Why: Verifies USB hardware (e.g., for webcam troubleshooting).
  4. **Check Logs (Question 22)**:

     - Command: `dmesg -T | grep -i e100`.
     - Output:

       ```
       [Thu Oct 9 23:00:24 2025] e100 0000:00:02.0 enp0s2: renamed from eth1
       [Thu Oct 9 23:00:41 2025] e100 0000:00:02.0 enp0s2: NIC Link is Up 100 Mbps Full Duplex
       ```

     - Why: Diagnoses hardware detection issues.

  5. **Configure Network**:
     - Command: `ip addr add 10.0.0.21/24 dev enp0s2; ip link set enp0s2 up; ping 10.0.0.21`.
     - Output: `64 bytes from 10.0.0.21: icmp_seq=1 ttl=64 time=0.046 ms`.
     - Why: Ensures `enp0s2` is functional (preserved `enp0s1` for SSH).

## Section 3: Building `hello.ko`

- **Goal**: Create, compile, and manage a simple kernel module (`hello.ko`).
- **Questions Addressed**:
  - **Question 13**: How to compile and load a kernel module?
  - **Question 14**: How to verify loaded modules?
  - **Question 16**: How to unload a module?
  - **Question 17**: How to inspect module metadata?
- **Steps and Why**:

  1. **Create** `hello.c`:

     - Command: `nano /root/hello.c`.
     - Content:

       ```c
       #include <linux/module.h>
       #include <linux/kernel.h>
       static int myparam = 0;
       module_param(myparam, int, 0644);
       MODULE_PARM_DESC(myparam, "An integer parameter");
       int hello_function(void) {
           printk(KERN_INFO "Hello function called\n");
           return myparam;
       }
       EXPORT_SYMBOL(hello_function);
       static int __init hello_init(void) {
           printk(KERN_INFO "Hello loaded, myparam=%d\n", myparam);
           return 0;
       }
       static void __exit hello_exit(void) {
           printk(KERN_INFO "Hello unloaded\n");
       }
       module_init(hello_init);
       module_exit(hello_exit);
       MODULE_LICENSE("GPL");
       MODULE_AUTHOR("Your Name");
       MODULE_DESCRIPTION("A simple hello world kernel module");
       ```

     - Why: Defines a module with a parameter (`myparam`), an exported function (`hello_function`) for dependencies, and logs for load/unload events.

  2. **Create** `Makefile`:

     - Command: `nano /root/Makefile`.
     - Content:

       ```makefile
       obj-m += hello.o
       ```

     - Why: Instructs `make` to build `hello.ko` as a kernel module.

  3. **Compile**:
     - Command: `make -C /lib/modules/$(uname -r)/build M=$(pwd) modules`.
     - Output: Creates `hello.ko` in `/root`.
     - Why: Uses kernel headers to compile for `6.8.0-85-generic`.
  4. **Load Module (Question 13, 14)**:

     - Command: `insmod /root/hello.ko myparam=999`.
     - Verify: `lsmod | grep hello`, `dmesg -T | tail`.
     - Output:

       ```
       hello                  12288  0
       [Thu Oct 9 20:16:13 2025] Hello loaded, myparam=999
       ```

     - Why: Tests module loading and parameter passing.

  5. **Unload Module (Question 16)**:
     - Command: `rmmod hello`.
     - Output: `[Thu Oct 9 20:16:20 2025] Hello unloaded`.
     - Why: Verifies clean unloading.
  6. **Inspect Metadata (Question 17)**:
     - Command: `modinfo /root/hello.ko`.
     - Output: Shows `license: GPL`, `author: Your Name`, `parm: myparam`.
     - Why: Confirms module details for troubleshooting.

## Section 4: Advanced Module Management

- **Goal**: Configure auto-loading, persistent parameters, blacklisting, and dependencies.
- **Questions Addressed**:
  - **Question 17**: How to view module parameters and metadata?
  - **Question 18**: How to configure auto-loading?
  - **Question 20**: How to blacklist a module?
  - **Question 21**: How to manage module dependencies?
  - **Question 93**: How to set persistent module parameters?
- **Steps and Why**:

  1. **Install** `hello.ko` **(Question 21)**:
     - Command: `cp /root/hello.ko /lib/modules/$(uname -r)/kernel/drivers/misc/; depmod -a`.
     - Verify: `cat /lib/modules/$(uname -r)/modules.dep | grep hello`.
     - Output: `kernel/drivers/misc/hello.ko:`.
     - Why: Enables `modprobe` by updating dependency database.
  2. **Auto-Load on Boot (Question 18)**:
     - Command: `echo "hello" > /etc/modules-load.d/hello.conf`.
     - Reboot, verify: `lsmod | grep hello`.
     - Output: `hello 12288 0`.
     - Why: Loads `hello` at boot via `systemd-modules-load.service`.
  3. **Persistent Parameters (Question 93)**:
     - Command: `echo "options hello myparam=666" > /etc/modprobe.d/hello.conf`.
     - Command: `modprobe hello`.
     - Output: `[Wed Oct 8 23:02:35 2025] Hello loaded, myparam=666`.
     - Why: Applies `myparam=666` on every load.
  4. **Blacklist** `hello` **(Question 20)**:
     - Command: `echo "blacklist hello" > /etc/modprobe.d/blacklist-hello.conf`.
     - Reboot, verify: `lsmod | grep hello` (empty).
     - Manual bypass: `modprobe hello myparam=555`.
     - Output: `[Thu Oct 9 20:16:13 2025] Hello loaded, myparam=555`.
     - Why: Prevents auto-loading but allows manual loading.
  5. **Verify Configurations (Question 17)**:

     - Command: `modprobe -c | grep hello`.
     - Output:

       ```
       blacklist hello
       options hello myparam=666
       ```

     - Why: Shows module configurations for debugging.

## Section 5: Troubleshooting Scenarios

- **Goal**: Simulate and resolve module issues, hardware failures, and driver conflicts.
- **Questions Addressed**:
  - **Question 13**: How to recover from a missing driver?
  - **Question 16**: How to handle module unload failures?
  - **Question 17**: How to inspect module details during troubleshooting?
  - **Question 19**: How to diagnose hardware issues?
  - **Question 20**: How to resolve driver conflicts?
  - **Question 22**: How to use logs for troubleshooting?
- **Task 5.1: Module Instability**:
  - **Steps**:
    - Blacklist `hello`, reboot (no auto-load).
    - Manual load: `modprobe hello myparam=555`.
    - Verify: `cat /sys/module/hello/parameters/myparam`.
  - **Output**: `555`.
  - **Why**: Tests bypassing blacklist to simulate instability (Question 20).
- **Task 5.2: Unload Busy**:

  - **Create** `dep.c`:

    - Command: `nano /root/dep.c`.
    - Content:

      ```c
      #include <linux/module.h>
      #include <linux/kernel.h>
      extern int hello_function(void);
      static int __init dep_init(void) {
          int ret;
          request_module("hello");
          printk(KERN_INFO "Dep loaded, calling hello_function\n");
          ret = hello_function();
          printk(KERN_INFO "hello_function returned %d\n", ret);
          return 0;
      }
      static void __exit dep_exit(void) {
          printk(KERN_INFO "Dep unloaded\n");
      }
      module_init(dep_init);
      module_exit(dep_exit);
      MODULE_LICENSE("GPL");
      MODULE_AUTHOR("Your Name");
      MODULE_DESCRIPTION("Module depending on hello");
      ```

    - Why: Creates `dep.ko` that depends on `hello.ko`, using `hello_function` and `request_module` (Question 21).

  - **Update** `Makefile`:

    - Content:

      ```makefile
      obj-m += hello.o dep.o
      ```

    - Why: Compiles both `hello.ko` and `dep.ko`.

  - **Compile and Install**:
    - Command: `make -C /lib/modules/$(uname -r)/build M=$(pwd) modules`.
    - Command: `cp /root/hello.ko /root/dep.ko /lib/modules/$(uname -r)/kernel/drivers/misc/`.
    - Command: `depmod -a`.
    - Verify: `cat /lib/modules/$(uname -r)/modules.dep | grep dep`.
    - Output: `kernel/drivers/misc/dep.ko: kernel/drivers/misc/hello.ko`.
    - Why: Ensures `modprobe dep` loads `hello.ko` first (Question 21).
  - **Test Dependency (Question 16)**:

    - Command: `modprobe dep`.
    - Output:

      ```
      lsmod | grep -iE "hello|dep"
      dep                    12288  0
      hello                  12288  1 dep
      [Fri Oct 10 20:48:21 2025] Hello loaded, myparam=666
      [Fri Oct 10 20:48:21 2025] Dep loaded, calling hello_function
      [Fri Oct 10 20:48:21 2025] Hello function called
      [Fri Oct 10 20:48:21 2025] hello_function returned 666
      ```

    - Command: `rmmod hello`.
    - Output: `rmmod: ERROR: Module hello is in use by: dep`.
    - Fix: `rmmod dep; rmmod hello` or `modprobe -r dep`.
    - Output:

      ```
      modprobe -r dep
      lsmod | grep -iE "hello|dep"
      ```

    - Why: Demonstrates dependency blocking unload; `modprobe -r` unloads both (Question 16).

- **Task 5.3: Hardware Not Detected**:

  - **Steps (Questions 13, 19, 22)**:

    - Command: `modprobe -r e100`.
    - Verify: `ethtool -i enp0s2`.
    - Output: `Cannot get driver information: No such device`.
    - Recover: `modprobe e100; ip addr add 10.0.0.21/24 dev enp0s2; ip link set enp0s2 up`.
    - Output:

      ```
      ethtool -i enp0s2
      driver: e100
      ping 10.0.0.21
      64 bytes from 10.0.0.21: icmp_seq=1 ttl=64 time=0.046 ms
      dmesg -T | grep -i e100
      [Thu Oct 9 23:00:24 2025] e100 0000:00:02.0 enp0s2: renamed from eth1
      ```

    - Why: Simulates network card failure by unloading driver, then restores (Questions 13, 19, 22).

- **Task 5.4: Driver Conflict**:

  - **Steps (Questions 17, 20, 22)**:

    - Command: `modprobe snd_hda_intel`.
    - Verify: `lsmod | grep snd_hda_intel`.
    - Output:

      ```
      snd_hda_intel          57344  0
      snd_hda_codec         208896  2 snd_hda_codec_generic,snd_hda_intel
      ```

    - Blacklist: `echo "blacklist snd_hda_intel" > /etc/modprobe.d/blacklist-snd-hda-intel.conf`.
    - Unload: `modprobe -r snd_hda_intel`.
    - Reboot, verify: `lsmod | grep snd_hda_intel` (empty).
    - Info: `modinfo snd_hda_intel`.
    - Output: Shows parameters like `power_save`.
    - Cleanup: `rm /etc/modprobe.d/blacklist-snd-hda-intel.conf`.
    - Why: Simulates resolving a driver conflict by blacklisting (Question 20); logs and metadata aid diagnosis (Questions 17, 22).

- **Task 5.5: Wrap-Up**:

  - **Steps (Questions 17, 22)**:

    - Command: `modprobe -c | grep hello`.
    - Output:

      ```
      blacklist hello
      options hello myparam=666
      softdep dep pre: hello
      ```

    - Reboot, verify: `journalctl -k -b | grep -i hello` (empty), `journalctl -k -b | grep -i e100`.
    - Output:

      ```
      Oct 10 21:20:23 usv kernel: e100 0000:00:02.0 enp0s2: renamed from eth1
      ```

    - Cleanup:

      ```bash
      rmmod dep
      rmmod hello
      rm /etc/modprobe.d/hello.conf
      rm /etc/modprobe.d/blacklist-hello.conf
      rm /etc/modules-load.d/hello.conf
      rm /lib/modules/$(uname -r)/kernel/drivers/misc/hello.ko
      rm /lib/modules/$(uname -r)/kernel/drivers/misc/dep.ko
      depmod -a
      make -C /lib/modules/$(uname -r)/build M=$(pwd) clean
      apt autoremove --purge
      ls /lib/modules
      6.8.0-79-generic  6.8.0-85-generic
      ```

    - Why: Verifies configurations, ensures clean system (Questions 17, 22).

## Bookmarks

1.  **Kernel Basics**:
    - **Concept**: Use `uname -r` to check kernel version, `lsmod` to list loaded modules.
    - **Explanation**: Essential for confirming system environment and active modules (e.g., `virtio_net` for `enp0s1`). Addresses Question 3 (kernel version) and Question 6 (module listing).
    - **Example**: `uname -r` → `6.8.0-85-generic`.
2.  **Virtual File Systems**:
    - **Concept**: VFS unifies filesystems like `ext4`, `tmpfs`, `/proc`, `/sys`.
    - **Explanation**: Provides a consistent interface for applications to access different filesystems. Addresses Question 8 (VFS role).
    - **Example**: `mount -t tmpfs none /mnt/tmpfs` creates a RAM-based filesystem.
3.  **tmpfs Details**:
    - **Concept**: `tmpfs` is a RAM-based filesystem, cleared on reboot.
    - **Explanation**: Used for temporary storage (e.g., `/run`). Addresses Question 8 by showing VFS in action.
    - **Example**: `mount | grep tmpfs` shows `/run` as `tmpfs`.
4.  **Pseudo-Filesystems**:
    - **Concept**: `/proc` and `/sys` provide kernel interfaces, not tied to physical storage.
    - **Explanation**: Allows inspection of processes (`/proc`) and kernel parameters (`/sys`). Addresses Question 8.
    - **Example**: `cat /proc/modules` mirrors `lsmod`.
5.  **Creating/Using** `hello.ko`:
    - **Concept**: Build `hello.ko` with parameters, use persistent settings, and blacklist.
    - **Explanation**: Demonstrates module lifecycle: creation, loading, configuration, and control. Addresses Questions 13 (compile/load), 14 (verify), 16 (unload), 17 (metadata), 18 (auto-load), 20 (blacklist), 93 (parameters).
    - **Example**: `modprobe hello myparam=555` → `[Thu Oct 9 20:16:13 2025] Hello loaded, myparam=555`.
6.  **Blacklist Behavior**:
    - **Concept**: `blacklist` in `/etc/modprobe.d/` prevents auto-loading but allows manual loading.
    - **Explanation**: Useful for avoiding conflicting drivers. Addresses Question 20.
    - **Example**: `echo "blacklist hello" > /etc/modprobe.d/blacklist-hello.conf; modprobe hello` (still loads manually).
7.  **Kernel Versions**:
    - **Concept**: Use `dpkg --list | grep linux-image` to list installed kernels.
    - **Explanation**: Helps manage kernel versions for module compatibility. Addresses setup and cleanup.
    - **Example**: `dpkg --list | grep linux-image` → `6.8.0-79-generic`, `6.8.0-85-generic`.
8.  **Module Dependencies**:

    - **Concept**: Use `request_module`, `EXPORT_SYMBOL`, `depmod -a`, and `modprobe -c` for dependency management and configuration inspection.
    - **Explanation**: Ensures modules load in the correct order; `modprobe -c` shows settings like `blacklist` and `options`. Addresses Questions 17 (configurations), 21 (dependencies), 93 (parameters). Your insight: “`modprobe -c | grep hello` is also a good one to know since it loads the configuration of the modules.”
    - **Example**:

      ```
      cat /lib/modules/$(uname -r)/modules.dep | grep dep
      kernel/drivers/misc/dep.ko: kernel/drivers/misc/hello.ko
      modprobe -c | grep hello
      blacklist hello
      options hello myparam=666
      ```

9.  **Driver Identification**:

    - **Concept**: Use `lspci -k` and `ethtool -i` to identify drivers for hardware.
    - **Explanation**: Critical for troubleshooting hardware issues (e.g., network interfaces). Addresses Question 19. Your insight: “I do want to note these 'lspci -k' and 'ethtool -i enp0s2' to verify the module/controler/driver.”
    - **Example**:

      ```
      lspci -k
      00:02.0 Ethernet controller: ... Kernel driver in use: e100
      ethtool -i enp0s2
      driver: e100
      ```

10. **Unloading Modules**:

    - **Concept**: `rmmod` unloads a single module; `modprobe -r` unloads a module and its dependencies.
    - **Explanation**: Tests dependency handling during unload. Addresses Question 16. Your insight: “let’s retest the difference between rmmod and modprobe -r.”
    - **Example**:

      ```
      rmmod hello
      rmmod: ERROR: Module hello is in use by: dep
      modprobe -r dep
      lsmod | grep -iE "hello|dep"
      ```

## Key Insights

- **Your Observations**:
  - “`depmod -a` should always be executed before I can use a module” (Task 5.2).
  - “`modprobe -c | grep hello` is also a good one to know since it loads the configuration of the modules” (Task 5.5).
  - “I do want to note these 'lspci -k' and 'ethtool -i enp0s2' to verify the module/controler/driver” (Task 5.3).
- **Lessons**:
  - `depmod -a` updates the module dependency database, critical for `modprobe`.
  - `modprobe -c` reveals module configurations (e.g., blacklist, parameters).
  - Blacklisting prevents auto-loading but allows manual loading for testing.
  - `modprobe -r` handles dependencies, unlike `rmmod`, which fails if a module is in use.
  - Logs (`dmesg`, `journalctl`) are essential for diagnosing hardware and module issues.

## Recommendations

- **Purge Old Kernel**:

  ```bash
  sudo apt purge linux-image-6.8.0-79-generic linux-headers-6.8.0-79-generic
  reboot
  ls /lib/modules
  ```

- **Apply Updates**:

  ```bash
  sudo apt update
  sudo apt upgrade
  ```

- **Verify Network**:

  ```bash
  ip link set enp0s2 up
  ip addr add 10.0.0.21/24 dev enp0s2
  ping 10.0.0.21
  ```
