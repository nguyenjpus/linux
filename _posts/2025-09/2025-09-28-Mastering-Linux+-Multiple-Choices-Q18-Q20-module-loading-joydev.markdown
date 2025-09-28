---
layout: post
title: "Multiple-choice - Q18 - Q20 - Automatically Loading Kernel Modules at Boot"
date: 2025-09-28
tags: [Linux+, cmdline, modprobe, quiet, modules-load, joydev, blacklist]
---

This lesson addresses CompTIA+ Objective 2.6, covering two key tasks: configuring a kernel module to load automatically at boot (Question 18) and preventing a module from auto-loading via blacklisting (Question 20). Both are critical for data center operations to ensure hardware compatibility and system stability. We practiced with the `joydev` module (joystick driver) on Ubuntu 24.04.3 LTS (kernel 6.8.0-79-generic, aarch64), confirming it as a loadable module (unlike `loop`, which was builtin). We also troubleshooted `joydev` persisting after cleanup due to kernel/udev auto-loading, resolved via blacklisting.

**Question 18**:

> An administrator wants to ensure that a specific kernel module, `power_saver`, is loaded automatically every time the system boots. What is the recommended modern approach to configure this?
>
> - Add an `insmod` command to the `/etc/rc.local` file.
> - Create a new file in `/etc/modules-load.d/` containing the module name.
> - Edit the `/boot/grub/grub.cfg` file to include the module.
> - Manually run `modprobe power_saver` after each boot.

**Correct Answer (Q18)**: Create a new file in `/etc/modules-load.d/` containing the module name.

**Rationale for Q18**:
The goal is persistent, automatic module loading. `/etc/modules-load.d/` is the `systemd`-native method, where `.conf` files (e.g., `power_saver.conf`) list modules for `systemd-modules-load` to load at boot. Alternatives fail: `/etc/rc.local` is deprecated and error-prone; `/boot/grub/grub.cfg` is for bootloader config, not modules, and overwritten by updates; manual `modprobe` isn’t automatic. This approach scales for data centers, ensuring consistent hardware support. Analogy: It’s like adding a task to a digital to-do list that runs automatically.

**Question 20**:

> A specific kernel module is known to conflict with a piece of hardware. To prevent this module from ever being loaded automatically, the administrator needs to "blacklist" it. How can this be achieved persistently?
>
> - By adding a `blacklist [module name]` line to a configuration file in `/etc/modprobe.d/`.
> - By deleting the module’s `.ko` file from the `/lib/modules/` directory.
> - By running `mmod [module name]` every time the system boots.
> - By setting the module’s file permissions to 000.

**Correct Answer (Q20)**: By adding a `blacklist [module name]` line to a configuration file in `/etc/modprobe.d/`.

**Rationale for Q20**:
The goal is to persistently prevent auto-loading of a conflicting module. A `blacklist <module>` entry in `/etc/modprobe.d/` (e.g., `blacklist-joydev.conf`) stops the kernel from loading it via `udev` or `systemd-modules-load`, even if hardware triggers it. Alternatives are flawed: deleting `.ko` files is destructive and non-persistent; `mmod` (likely `rmmod`) isn’t persistent and requires manual scripting; setting permissions to 000 is non-standard and breaks updates. Blacklisting is safe and standard for data centers. Analogy: It’s like adding a “do not call” entry to a phone list.

### Why This Matters in Real Life (Especially for Data Center Jobs)

Kernel modules enable critical hardware or features (e.g., RAID drivers, input devices) or can cause conflicts if loaded inappropriately. In a data center, automating module loading ensures servers boot correctly, while blacklisting prevents issues like hardware crashes, vital for scaling across thousands of machines. Troubleshooting persistent modules (like `joydev`) preps you for real-world scenarios where virtual or physical hardware triggers unwanted loads. Analogy: It’s like setting your coffee maker to brew automatically but knowing how to stop it if it malfunctions. These skills are key for CompTIA+ and data center roles.

### Hands-On Practice Summary

We simulated a data center task: Ensuring the `joydev` module loads at boot (Q18) and preventing its auto-loading via blacklisting (Q20).

1. **Checked loaded modules**: `lsmod | grep joydev` → Empty (not loaded initially).
2. **Backed up configs**: `cp -r /etc/modules-load.d /root/backups/modules-load.d.bak`.
3. **Created config**: `echo "joydev" | sudo tee /etc/modules-load.d/joydev.conf` → Added module to auto-load.
4. **Tested manually**: `sudo modprobe joydev` → Confirmed in `lsmod` (`joydev 36864 0`).
5. **Rebooted and verified**: `lsmod | grep joydev` → Showed loaded; `journalctl -b | grep joydev` confirmed `systemd-modules-load` inserted it.
6. **Cleaned up**: Removed `/etc/modules-load.d/joydev.conf`, unloaded module (`modprobe -r joydev`), restored backup (`cp -r /root/backups/modules-load.d.bak/* /etc/modules-load.d/`), rebooted.
7. **Troubleshooting persistence**: `joydev` persisted post-cleanup due to kernel/udev auto-loading in UTM. Checked `/etc/modules-load.d/*.conf` (empty), `lsinitramfs` (no `joydev`), `udevadm monitor` (no triggers). Fixed by blacklisting: `echo "blacklist joydev" | sudo tee /etc/modprobe.d/blacklist-joydev.conf` (after fixing typo `blacklilst`), `update-initramfs -u`, rebooted. Verified: `lsmod | grep joydev` empty.

**System Details**: Ubuntu 24.04.3 LTS on UTM (aarch64), kernel 6.8.0-79-generic, root on LVM (`/dev/mapper/ubuntu--vg-ubuntu--lv`). Note: `joydev` confirmed loadable via `modinfo` (filename: `/lib/modules/.../joydev.ko.zst`).

**Pro-Tips for Data Center Readiness**:

- Verify module loading with `journalctl -b | grep modules` or `systemctl status systemd-modules-load`.
- Use `find /lib/modules/$(uname -r) -name 'module*'` to locate compressed modules (.ko.zst on arm64).
- Test in staging VMs to avoid production issues; check builtins with `/boot/config-$(uname -r) | grep <CONFIG>`.
- Best practice: Document changes in change management tools (e.g., ServiceNow). Blacklist unneeded modules to reduce resource use and attack surfaces.

## FAQ Section

From this and prior lessons (GRUB parameters and initial `loop` attempt):

- **Q: Which file shows the current kernel boot parameters?**  
  **A:** `/proc/cmdline`

- **Q: What’s the difference between /proc/cmdline and /etc/default/grub?**  
  **A:** `/proc/cmdline` shows current parameters; `/etc/default/grub` sets future boot parameters.

- **Q: What causes an (initramfs) prompt?**  
  **A:** Kernel can’t mount the root filesystem, often due to incorrect `root=` parameters.

- **Q: How do you recover from (initramfs)?**  
  **A:** Manually mount the root filesystem or correct GRUB parameters and update `/etc/default/grub`.

- **Q: Why do we need this pci=pcie_bus_safe?**  
  **A:** Sets PCIe Max Payload Size to a safe value for compatibility, preventing device errors.

- **Q: What does the 'quiet' parameter do?**  
  **A:** Suppresses verbose kernel boot messages for a cleaner boot.

- **Q: What does the 'splash' parameter do?**  
  **A:** Enables a graphical boot screen (e.g., via Plymouth).

- **Q: Why do we have this vt.handoff=7?**  
  **A:** Auto-added with `splash` for smooth virtual terminal handoff to userspace.

- **Q: Why was the loop module "loaded" 2 hours ago when I was asleep? No one touched my laptop.**  
  **A:** Timestamps reflect boot time (~1.5 hours ago). Builtins like `loop` initialize automatically at boot.

- **Q: Why does modules.conf have different font and background colors in original /etc/modules-load.d compared to the backup?**  
  **A:** Terminal coloring via `ls --color=auto`; no functional difference—files are identical.

- **Q: Why does `joydev` persist in lsmod after cleanup?**  
  **A:** Likely auto-loaded by kernel/udev due to virtual hardware in UTM. Fixed by blacklisting in `/etc/modprobe.d/blacklist-joydev.conf` and updating initramfs.

- **Q: If we don’t use blacklist, how do we stop `joydev` from loading?**  
  **A:** Options include manual unloading (`modprobe -r joydev` post-boot, non-persistent), modifying initramfs to skip it, or disabling `udev` rules triggering input devices. Blacklisting is simplest and persistent.

## Practice Questions

For review and CompTIA+ prep:

- **Q: Why doesn’t a builtin module like `loop` appear in lsmod?**  
  **A:** lsmod lists only loadable modules from `/proc/modules`; builtins are compiled in and always active. _Explanation_: Check with `modinfo <module>` for `filename: (builtin)`.

- **Q: What does `filename: (builtin)` in modinfo mean?**  
  **A:** The module is compiled directly into the kernel, not a separate .ko file. _Explanation_: No need for modprobe; it’s auto-initialized at boot.

- **Q: How can you find all loadable modules in your kernel?**  
  **A:** Use `find /lib/modules/$(uname -r) -name '*.ko*'` or `lsmod` for loaded ones. _Explanation_: Catches .ko and compressed .ko.zst files.

- **Q: In a data center, why blacklist unneeded modules?**  
  **A:** Prevents auto-loading, reducing resource use and security risks. _Explanation_: Use `/etc/modprobe.d/` and `update-initramfs` for persistence; saves troubleshooting time.
