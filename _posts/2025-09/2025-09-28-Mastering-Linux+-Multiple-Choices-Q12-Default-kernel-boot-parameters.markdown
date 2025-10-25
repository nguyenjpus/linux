---
layout: post
title: "Multiple-choices - Q12 - Modifying Default Kernel Boot Parameters"
date: 2025-09-28
tags: [Linux+, Multiple-choices, Cmdline, PCIe, Quiet]
---

This lesson covers how to persistently modify kernel boot parameters by editing `/etc/default/grub` and updating GRUB. We focused on the file `/etc/default/grub` as the correct choice for adding parameters like hardware enablers (e.g., `pci=pcie_bus_safe`). Other options like `/boot/grub/grub.cfg` (generated file, don't edit directly), `/proc/cmdline` (read-only current params), and `/etc/fstab` (filesystem mounts) were explained as incorrect.

### Why This Matters in Real Life (Especially for Data Center Jobs)

Kernel parameters fine-tune low-level system behavior, such as hardware detection or boot debugging. In a data center, you'd use this to fix server boot issues (e.g., enabling drivers for RAID controllers in a rack of servers) without re-imaging, minimizing downtime. Imagine troubleshooting a fleet of 500 servers—automate with tools like Ansible to push changes. Analogy: Parameters are "startup cheats" for hardware, like adjusting a car's ECU for better performance in varying conditions.

### Hands-On Practice Summary

We simulated a data center scenario: Adding `pci=pcie_bus_safe` to handle PCIe device stability.

1. **Checked current params**: `cat /proc/cmdline` → Showed baseline like `BOOT_IMAGE=/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0`.
2. **Backed up**: `cp /etc/default/grub /etc/default/grub.bak`.
3. **Edited**: Used `nano /etc/default/grub` to change `GRUB_CMDLINE_LINUX_DEFAULT="console=tty0"` to `"console=tty0 quiet splash pci=pcie_bus_safe"`.
4. **Updated GRUB**: `sudo update-grub` → Regenerated `/boot/grub/grub.cfg`.
5. **Rebooted and verified**: `cat /proc/cmdline` now includes new params, plus auto-added `vt.handoff=7`.
6. **Cleanup**: Restore backup, update GRUB, reboot to revert.

**System Details**: Ubuntu 24.04.3 LTS on UTM (aarch64), kernel 6.8.0-79-generic, root on LVM (`/dev/mapper/ubuntu--vg-ubuntu--lv`).

**Pro-Tips for Data Center Readiness**:

- Always snapshot VMs or back up before changes.
- Use `grub-editenv` for advanced tweaks. (Have not tried)
- In production, test on staging servers; monitor with `dmesg | grep pci` for PCIe logs.
- Best practice: Script checks (e.g., Bash to verify params post-boot) for automated deployments.
- With "quiet", we will not see options to get into GRUB> in current UTM settings.

## FAQ Section

This collects questions asked during the lesson for quick reference.

- **Q: Which file shows the current kernel boot parameters?**  
  **A:** `/proc/cmdline`

- **Q: What’s the difference between /proc/cmdline and /etc/default/grub?**  
  **A:** `/proc/cmdline` shows current parameters; `/etc/default/grub` sets future boot parameters.

- **Q: What causes an (initramfs) prompt?**  
  **A:** The kernel can’t mount the root filesystem, often due to incorrect root= parameters.

- **Q: How do you recover from (initramfs)?**  
  **A:** Manually mount the root filesystem or correct GRUB parameters and update `/etc/default/grub`.

- **Q: Why do we need this pci=pcie_bus_safe? What does this parameter do?**  
  **A:** Added for practice to simulate hardware enabling; it sets PCIe device Max Payload Size (MPS) to the largest compatible value for all devices, ensuring safe data transfers in mixed hardware setups.

- **Q: What does the 'quiet' parameter do?**  
  **A:** Suppresses verbose kernel boot messages for a cleaner boot process.

- **Q: What does the 'splash' parameter do?**  
  **A:** Enables a graphical loading screen (e.g., via Plymouth) to hide boot text.

- **Q: Why do we have this vt.handoff=7?**  
  **A:** Ubuntu auto-adds it with 'splash' for smooth virtual terminal handoff from kernel to userspace, preventing display glitches.

## Practice Questions

Here are a few more for review (closed-ended style, with explanations):

Q: What command regenerates GRUB config after editing /etc/default/grub?  
A: `sudo update-grub` (or `sudo grub-mkconfig -o /boot/grub/grub.cfg`). Explanation: This applies changes persistently by rebuilding the boot menu from templates.

Q: Why avoid editing /boot/grub/grub.cfg directly?  
A: It's auto-generated and gets overwritten on updates; changes aren't persistent. Explanation: Always use /etc/default/grub for safety—analogy: Edit the recipe, not the cooked meal.

Q: In a data center, how might you apply kernel params to multiple servers?  
A: Use automation like Ansible playbooks to edit files and run update-grub remotely. Explanation: Scales fixes across racks without manual SSH; essential for efficiency.

Q: What does 'ro' in /proc/cmdline mean?  
A: Mounts the root filesystem read-only initially for fsck safety. Explanation: Kernel switches to read-write later; common in all boots to prevent corruption.
