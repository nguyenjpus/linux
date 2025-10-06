---
layout: post
title: "Linux Lab Summary: Mastering the Boot Process and Recovery"
date: 2025-10-05 10:30:00 -0700
tags: [Linux+, Linuxlab]
---

This summary is a recreatable lab report addressing the original multiple-choice questions (1, 2, 4, 5, 7, 9, 11, 12, 86). For each question, I explain the correct answer, why it was chosen based on Linux fundamentals and lab experience, commands executed (to recreate the lab), key lessons learned (with critical ones in <span style="color:red">red</span>, emphasizing the `rescue.target` discovery), bonus insights (e.g., SSH/network inaccessibility, `fsck` behavior), and relevant user questions/insights to deepen understanding. The lab was conducted on Ubuntu 24.04.3 LTS (ARM64) in UTM, with an LVM root filesystem (`/dev/mapper/ubuntu--vg-ubuntu--lv`), `/boot` on `/dev/vda2`, and `/boot/efi` on `/dev/vda1`. We simulated failures, recovered using GRUB edits, and inspected components like initramfs. Only actions performed are included (e.g., no `grub-install` or kernel failure simulation).

The boot sequence learned: BIOS/UEFI → GRUB → kernel → initramfs → systemd → mounts /etc/fstab → services.

**Prerequisites to Recreate**:

- Run as root (`sudo -i`) or use `sudo` for privileged commands.
- Backup files before editing (e.g., `cp file file.bak`).
- Use UTM snapshots for safe failure simulations.
- Update system if needed: `apt update && apt upgrade`.

## Question 1: Bootloader Location in MBR Scheme

**Question**: A system administrator is troubleshooting a server that fails to start. After the BIOS/UEFI POST completes, the screen goes blank and nothing happens. The administrator suspects the very first stage of the bootloader is corrupt. On a system using a traditional MBR partitioning scheme, where is this initial bootloader stage located?

**Options**:

- In the /boot partition
- In the Master Boot Record (MBR)
- As a file within the root filesystem
- In the swap partition

**Correct Answer**: In the Master Boot Record (MBR).

**Why Chosen**: The MBR is the first 512 bytes of the disk, containing GRUB stage 1, which loads the full GRUB from `/boot`. A corrupt MBR causes a blank screen after POST, as the bootloader can't start. The other options are incorrect: `/boot` holds GRUB stage 2 and configs, the root filesystem holds `/etc/fstab`, and swap is for memory overflow.

**Commands Executed (Recreate)**:

- Verified boot logs to confirm bootloader activity:
  ```bash
  dmesg -T | grep -i initramfs
  # Output: [Sun Oct  5 17:47:05 2025] Trying to unpack rootfs image as initramfs...
  ```
- Checked kernel command line:
  ```bash
  cat /proc/cmdline
  # Output: BOOT_IMAGE=/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0
  ```

**Bonus Insights**:

- In UEFI (our setup), GRUB’s EFI binary is in `/boot/efi/EFI/ubuntu/grub.efi`, not MBR (Question 9).
- Use `ls -l /dev/disk/by-id/` to verify UUID.

**Lessons Learned**:

- The MBR holds GRUB stage 1; corruption leads to blank screens. In UEFI, `/boot/efi` is used instead.

**User Question**: “so the uuid did not lead directly to ubuntu--vg-ubuntu--lv” (Oct 5, 2025).

- **Insight**: UUIDs in `/etc/fstab` map to `/dev/mapper/ubuntu--vg-ubuntu--lv` via `/dev/dm-0`, ensuring stable device naming.

## Question 2: Interrupting Boot for Maintenance

**Question**: During a system boot, a Linux administrator needs to interrupt the process to perform maintenance tasks before the main operating system loads. Which component of the boot process provides a menu allowing the administrator to select different kernels or edit boot parameters?

**Options**:

- The systemd init process
- The BIOS/UEFI firmware
- The GRUB 2 bootloader
- The initramfs image

**Correct Answer**: The GRUB 2 bootloader.

**Why Chosen**: GRUB provides the boot menu for kernel selection and parameter edits (e.g., `verbose`). systemd starts later; BIOS/UEFI is hardware initialization; initramfs is for early filesystem mounting.

**Commands Executed (Recreate)**:

- Enabled GRUB menu:
  ```bash
  cp /etc/default/grub /etc/default/grub.bak
  nano /etc/default/grub  # Set GRUB_TIMEOUT=5, GRUB_TIMEOUT_STYLE=menu
  update-grub
  reboot
  # Press Shift/Esc to show menu
  ```
- Temporary edit:
  ```bash
  # At GRUB, press e, add verbose to linux line
  linux (hd0,gpt2)/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro verbose
  # Ctrl+X to boot
  cat /proc/cmdline  # Shows verbose
  reboot
  cat /proc/cmdline  # No verbose (temporary)
  ```

**Bonus Insights**:

- Add `systemd.unit=rescue.target` to GRUB `linux` line for single-user mode (Question 4).
- If menu hidden, use UTM console to access GRUB.

**Lessons Learned**:

- Temporary GRUB edits are non-persistent, ideal for testing parameters like `verbose` or `systemd.unit=rescue.target`.

**User Question**: “I cannot set it to single mode as the system froze” (Oct 5, 2025).

- **Insight**: UTM freezes on `single` due to virtual console issues; use `systemd.unit=rescue.target`.

## Question 4: Booting to Minimal Mode

**Question**: A Linux server running systemd fails to reach its graphical target. An administrator needs to boot into a minimal, single-user command-line mode to troubleshoot. Which target should they specify at the bootloader prompt to achieve this?

**Options**:

- emergency.target
- graphical.target
- rescue.target
- multi-user.target

**Correct Answer**: rescue.target.

**Why Chosen**: `rescue.target` provides a single-user shell with minimal services for maintenance (e.g., `fsck`, `/etc/fstab` fixes). `emergency.target` is more minimal (bare shell, no networking); `graphical.target` is full GUI; `multi-user.target` is network-enabled multi-user mode.

**Commands Executed (Recreate)**:

- Booted to `rescue.target`:
  ```bash
  reboot
  # At GRUB, press e, add to linux line:
  linux (hd0,gpt2)/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 systemd.unit=rescue.target
  # Ctrl+X to boot
  systemctl status  # Shows State: maintenance, rescue.target active
  ```
- Performed maintenance:
  ```bash
  systemctl stop boot.mount
  fsck /dev/vda2  # Output: /dev/vda2: clean
  systemctl start boot.mount
  systemctl default  # Returns to graphical.target
  ```
- Tested `fsck` in normal mode (failed):
  ```bash
  fsck /dev/vda2  # Error: device in use
  ```
- Retried in `rescue.target` (succeeded):
  ```bash
  systemctl stop boot.mount
  fsck /dev/vda2  # Output: clean
  systemctl start boot.mount
  ```

**Bonus Insights**:

- **We can perform this "live" without getting into grub> to perform fsck**:
  ```bash
  # check running mode
  systemctl status
  systemctl rescue # or systemctl emergency
  systemctl status # to reconfirm which mode is on
  systemctl stop boot.mount # sometimes, if this boot.mount is running, we can't run fsck
  fsck /dev/vda2 #Output: clean
  systemctl start boot.mount # Restart boot.mount
  systemctl default # getback to default mode.
  ```
- **SSH/Network Inaccessibility**: In `rescue.target` and `emergency.target`, networking (e.g., `enp0s1`) is disabled, causing SSH to fail (`Connection refused`). We used UTM’s console for access:
  ```bash
  # On Mac, after reboot to emergency.target:
  ssh ron@10.0.0.20  # Failed: Connection refused
  ```
  Access via UTM console, then:
  ```bash
  systemctl default  # Restored networking, SSH worked
  ```
- **Accessing SSH Despite Inaccessibility**: In some cases, an existing SSH session persisted briefly in `rescue.target` due to socket reuse or delayed termination. New connections failed until `graphical.target` restored networking.
- **fsck Requirement**: `fsck` requires unmounted filesystems, impossible in normal mode for `/boot` (`/dev/vda2`). Use `rescue.target` or `emergency.target` to stop `boot.mount`.
- Boot to `emergency.target`:
  ```bash
  # At GRUB:
  linux (hd0,gpt2)/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 systemd.unit=emergency.target
  # Ctrl+X to boot
  ```
- If stuck at GRUB prompt:
  ```bash
  set root=(hd0,gpt2)
  linux /vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 systemd.unit=rescue.target
  initrd /initrd.img-6.8.0-79-generic
  boot
  ```

**Lessons Learned**:

- <span style="color:red">**Booting to rescue.target via GRUB with `systemd.unit=rescue.target` works in UTM, unlike `single`, which freezes due to virtualization issues.**</span>
- `rescue.target` mounts essential filesystems, ideal for recovery; networking is disabled, requiring console access.

**User Question**: “So why do we turn on rescue here? Or is it always on and be ready whenever we need?” (Oct 4, 2025).

- **Insight**: `rescue.target` is activated only when requested via GRUB or `systemctl isolate rescue.target`; it’s for maintenance, not always active.

## Question 5: Initramfs Role

**Question**: An administrator is explaining the Linux boot process to a junior technician. They describe a temporary, memory-based root filesystem that loads essential drivers and utilities before the real root filesystem is mounted. What is this temporary filesystem called?

**Options**:

- The GRUB filesystem
- The swap space
- The initramfs (initial RAM filesystem)
- The /temp directory

**Correct Answer**: The initramfs (initial RAM filesystem).

**Why Chosen**: The initramfs is a RAM-based filesystem loaded by GRUB, containing tools (e.g., `lvm`) and modules (`dm-mod`) to mount complex roots like LVM. GRUB is the bootloader; swap is memory overflow; `/temp` is a directory.

**Commands Executed (Recreate)**:

- Located initramfs:
  ```bash
  ls -lh /boot/initrd.img-6.8.0-79-generic  # Output: 68M
  ```
- Checked format:
  ```bash
  file /boot/initrd.img-6.8.0-79-generic  # Output: ASCII cpio archive
  ```
- Attempted manual unpack (failed):
  ```bash
  zcat /boot/initrd.img-6.8.0-79-generic | cpio -idmv  # Error: not gzip
  ```
- Unpacked correctly:
  ```bash
  mkdir /tmp/initramfs-main
  unmkinitramfs /boot/initrd.img-6.8.0-79-generic /tmp/initramfs-main
  cd /tmp/initramfs-main/main
  ls  # Output: bin conf cryptroot etc init lib sbin scripts usr var
  ```
- Inspected contents:
  ```bash
  cat init  # Shows mounting root, lvm vgchange -ay
  ls sbin | grep lvm  # Output: lvm
  cat lib/modules/6.8.0-79-generic/modules.builtin | grep dm-mod  # Output: kernel/drivers/md/dm-mod.ko
  ```
- Verified in logs:
  ```bash
  dmesg -T | grep initramfs  # Output: Trying to unpack rootfs image as initramfs...
  ```

**Bonus Insights**:

- Regenerate initramfs: `update-initramfs -u -k 6.8.0-79-generic`.
- Check LVM activation: `lsblk` shows `/dev/mapper/ubuntu--vg-ubuntu--lv`.

**Lessons Learned**:

- <span style="color:red">**Ubuntu’s initramfs is concatenated (early microcode + main); use `unmkinitramfs` for extraction.**</span>
- The `init` script activates LVM for `/dev/mapper/ubuntu--vg-ubuntu--lv`.

**User Question**: “So we found main, and init, sbin, lvm, but we can't find drivers.” (Oct 5, 2025).

- **Insight**: Drivers like `dm-mod` are in `lib/modules/*/modules.builtin`, not a separate `/kernel/drivers/md`.

## Question 7: Kernel and Initramfs Location

**Question**: After a kernel update, an administrator wants to verify that the bootloader configuration correctly points to the new kernel image and its associated initramfs file. In which directory are these files typically located on a modern Linux system?

**Options**:

- /etc/
- /usr/src/
- /boot/
- /var/log/

**Correct Answer**: /boot/.

**Why Chosen**: `/boot` holds the kernel (`vmlinuz-*`) and initramfs (`initrd.img-*`), loaded by GRUB. `/etc` is configs; `/usr/src` is kernel sources; `/var/log` is logs.

**Commands Executed (Recreate)**:

- Verified files:
  ```bash
  ls -lh /boot  # Output: vmlinuz-6.8.0-79-generic, initrd.img-6.8.0-79-generic
  ```
- Checked GRUB config:
  ```bash
  cat /boot/grub/grub.cfg | grep linux  # References /vmlinuz-6.8.0-79-generic
  ```

**Bonus Insights**:

- Verify mounts: `lsblk`, `mount | grep /boot` (shows `/dev/vda2`).
- Check GRUB entries: `grep menuentry /boot/grub/grub.cfg`.

**Lessons Learned**:

- `/boot` is a separate ext4 partition for reliability in LVM setups, mounted by systemd (Question 86).

## Question 9: GRUB Config Location in UEFI

**Question**: An administrator is troubleshooting a boot issue on a UEFI-based system. They suspect a problem with the bootloader's configuration file. Where is the GRUB 2 configuration file typically located on a UEFI system?

**Options**:

- /boot/grub/grub.cfg
- /boot/efi/EFI/[distro]/grub.cfg
- /etc/grub.conf
- /etc/lilo.conf

**Correct Answer**: /boot/efi/EFI/[distro]/grub.cfg.

**Why Chosen**: In UEFI, GRUB’s EFI config is in the EFI System Partition (`/boot/efi/EFI/ubuntu/grub.cfg`), pointing to the full config in `/boot/grub/grub.cfg`. The other options are for BIOS or legacy bootloaders.

**Commands Executed (Recreate)**:

- Checked UEFI config:
  ```bash
  cat /boot/efi/EFI/ubuntu/grub.cfg  # Output: search.fs_uuid... configfile $prefix/grub.cfg
  ```

**Bonus Insights**:

- Verify EFI entries: `efibootmgr -v` shows boot order.
- Full GRUB config: `cat /boot/grub/grub.cfg`.

**Lessons Learned**:

- UEFI uses `/boot/efi/EFI/ubuntu/grub.cfg` for initial boot, linking to `/boot/grub/grub.cfg`.

**User Question**: “with these output, how can we confirmed that the GRUB 2 configuration file is typically located on /boot/efi/EFI/[distro]/grub.cfg?” (Oct 5, 2025).

- **Insight**: The output shows `/boot/efi/EFI/ubuntu/grub.cfg` loads `/boot/grub/grub.cfg`, confirming the UEFI location.

## Question 11: Kernel Panic Cause

**Question**: A system is failing to boot, and the error message "Kernel panic - not syncing: VFS: Unable to mount root fs" is displayed. What is the most likely cause of this issue?

**Options**:

- The BIOS/UEFI is configured with the wrong boot order.
- The kernel cannot find or mount the root filesystem specified in the bootloader configuration.
- The systemd service has crashed.
- The network interface card is not configured correctly.

**Correct Answer**: The kernel cannot find or mount the root filesystem specified in the bootloader configuration.

**Why Chosen**: The "VFS: Unable to mount root fs" panic occurs when the kernel/initramfs can't access the root (e.g., corrupted initramfs or wrong `root=`). BIOS boot order affects GRUB loading; systemd starts later; network is irrelevant for local boot.

**Commands Executed (Recreate)**:

- Simulated initramfs corruption:
  ```bash
  cp /boot/initrd.img-6.8.0-79-generic /boot/initrd.img-6.8.0-79-generic.bak
  truncate -s 1M /boot/initrd.img-6.8.0-79-generic
  reboot  # Kernel panic, unbootable
  ```
- Recovered via GRUB:
  ```bash
  # At GRUB prompt:
  set root=(hd0,gpt2)
  linux /vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 systemd.unit=rescue.target
  initrd /initrd.img-6.8.0-79-generic.bak
  boot
  ```
- Restored initramfs:
  ```bash
  cp /boot/initrd.img-6.8.0-79-generic.bak /boot/initrd.img-6.8.0-79-generic
  ls -lh /boot/initrd.img-6.8.0-79-generic  # Output: 68M
  ```

**Bonus Insights**:

- Check panic in logs (post-recovery): `dmesg -T | grep -i panic`.
- Use UTM snapshot to revert after failure simulations.
- **SSH in Rescue/Emergency**: After recovery, SSH worked in `graphical.target`:
  ```bash
  ssh ron@10.0.0.20  # Succeeded after systemctl default
  ```

**Lessons Learned**:

- <span style="color:red">**Kernel panics like "VFS: Unable to mount root fs" are often due to initramfs corruption or wrong `root=`, recoverable with GRUB backups and `systemd.unit=rescue.target`.**</span>

## Question 12: Persistent Kernel Parameters

**Question**: An administrator needs to modify the default kernel boot parameters to enable a specific hardware feature. Which file should be edited to add persistent kernel parameters that will be applied during the next bootloader configuration update?

**Options**:

- /boot/grub/grub.cfg
- /etc/default/grub
- /proc/cmdline
- /etc/fstab

**Correct Answer**: /etc/default/grub.

**Why Chosen**: `/etc/default/grub` defines kernel parameters (`GRUB_CMDLINE_LINUX_DEFAULT`), applied persistently via `update-grub`. `/boot/grub/grub.cfg` is auto-generated (don't edit); `/proc/cmdline` is runtime; `/etc/fstab` is for mounts.

**Commands Executed (Recreate)**:

- Edited for persistent `verbose`:
  ```bash
  cp /etc/default/grub /etc/default/grub.bak
  nano /etc/default/grub  # Set GRUB_CMDLINE_LINUX_DEFAULT="quiet verbose"
  update-grub
  reboot
  cat /proc/cmdline  # Output: ... verbose
  ```

**Bonus Insights**:

- Test parameters temporarily first (Question 2) to avoid UTM freezes (e.g., avoid `nomodeset`).

**Lessons Learned**:

- Persistent changes require `update-grub` to regenerate `/boot/grub/grub.cfg`.

## Question 86: /etc/fstab Responsibility

**Question**: A server fails to boot, dropping into an emergency shell. The administrator suspects a corrupted /etc/fstab file. During the boot process, which component is responsible for reading /etc/fstab and mounting the filesystems?

**Options**:

- The GRUB 2 bootloader
- The Linux kernel itself
- The systemd init process
- The mount command in /etc/rc.local

**Correct Answer**: The systemd init process.

**Why Chosen**: systemd (init PID 1) reads `/etc/fstab` via `systemd-fstab-generator` to create mount units (e.g., `boot.mount`). GRUB loads kernel/initramfs; kernel mounts initial root; `/etc/rc.local` is legacy.

**Commands Executed (Recreate)**:

- Verified `/etc/fstab`:
  ```bash
  cat /etc/fstab
  # Output: /dev/disk/by-id/dm-uuid-LVM-... / ext4, /dev/vda2 /boot ext4, /dev/vda1 /boot/efi vfat
  ```
- Simulated root failure (no effect):
  ```bash
  cp /etc/fstab /etc/fstab.bak
  nano /etc/fstab  # Change / to /dev/disk/by-id/dm-uuid-wrong
  reboot  # Booted normally
  ```
- Simulated `/boot` failure:
  ```bash
  nano /etc/fstab  # Change /boot to /dev/disk/by-uuid/wrong
  reboot  # Dropped to emergency.target
  ```
- Recovered:
  ```bash
  # In emergency.target (via UTM console) or rescue.target:
  cp /etc/fstab.bak /etc/fstab
  systemctl default
  ```
- Verified mounts:
  ```bash
  mount | grep -E '/ |/boot|/boot/efi'
  # Output: /dev/mapper/ubuntu--vg-ubuntu--lv on /, /dev/vda2 on /boot, /dev/vda1 on /boot/efi
  ```
- Checked logs:
  ```bash
  dmesg -T | grep -i "command line"  # Shows root=/dev/mapper/...
  journalctl -b | grep -i fstab  # Shows systemd processing
  ```

**Bonus Insights**:

- **SSH/Network Inaccessibility**: In `emergency.target`, SSH failed due to no networking:
  ```bash
  # On Mac, after reboot to emergency.target:
  ssh ron@10.0.0.20  # Failed: Connection refused
  ```
  Used UTM console for recovery. Existing SSH sessions may persist briefly due to socket reuse.
- **fsck Requirement**: `fsck` failed in normal mode for `/boot` due to mount:
  ```bash
  fsck /dev/vda2  # Error: device in use
  ```
  Succeeded in `rescue.target` after unmounting:
  ```bash
  systemctl stop boot.mount
  fsck /dev/vda2  # Clean
  ```
- Root mount uses GRUB’s `root=` (Question 5), ignoring `/etc/fstab` errors for `/`.
- Check UUID mappings:
  ```bash
  ls -l /dev/disk/by-id/dm-uuid-LVM-...  # Links to /dev/dm-0
  ls -l /dev/mapper/ubuntu--vg-ubuntu--lv  # Links to /dev/dm-0
  ```

**Lessons Learned**:

- <span style="color:red">**systemd mounts /etc/fstab entries; errors in critical mounts like /boot trigger emergency.target, recoverable with `systemd.unit=rescue.target`.**</span>
- <span style="color:red">**The kernel prioritizes GRUB’s `root=` over /etc/fstab for root mounting, so root errors may be ignored.**</span>

**User Questions**:

- “right here, we don’t have dev/mapper/ubuntu--vg-ubuntu--lv to edit. Do I miss anything?” (Oct 5, 2025).
  - **Insight**: `/etc/fstab` uses `/dev/disk/by-id/dm-uuid-LVM-...`, equivalent to `/dev/mapper/ubuntu--vg-ubuntu--lv`.
- “as you see, the disk was set wrong on purpose but the system runs normally” (Oct 5, 2025).
  - **Insight**: Root errors in `/etc/fstab` are ignored because the kernel uses `root=` from `/proc/cmdline`.

## Key Takeaways & Bonus Lessons

- <span style="color:red">**Booting to rescue.target via GRUB with `systemd.unit=rescue.target` works in UTM, unlike `single`, which freezes due to virtualization issues.**</span>
- <span style="color:red">**Ubuntu’s initramfs is concatenated (early + main); use `unmkinitramfs` for extraction.**</span>
- <span style="color:red">**The kernel prioritizes GRUB’s `root=` over /etc/fstab for root mounting, but /boot errors trigger emergency.target, recoverable with `systemd.unit=rescue.target`.**</span>
- <span style="color:red">**Kernel panics like "VFS: Unable to mount root fs" are often due to initramfs corruption or wrong `root=`, recoverable with GRUB backups and `systemd.unit=rescue.target`.**</span>
- GRUB edits are critical for recovery; use UTM snapshots for safety.
- `rescue.target` and `emergency.target` disable networking, requiring console access; existing SSH sessions may persist briefly.
- `fsck` requires unmounted filesystems, necessitating `rescue.target` or `emergency.target`.
- `dmesg` logs kernel/initramfs actions; `journalctl` logs systemd actions.

## Conclusion

This lab provided hands-on mastery of the Linux boot process, GRUB, initramfs, and recovery. Recreate it using the commands above, with UTM snapshots for safety. For further exploration, try system updates (`apt upgrade`) or test additional GRUB parameters.
