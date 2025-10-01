---
layout: post
title: "Q1: GRUB 2 Rescue Prompt - Hands-On Reflections Summary"
date: 2025-09-14
tags: [Linux+, GRUB, initramfs]
---

This document summarizes our in-depth discussion on CompTIA Linux+ Question 1 (Q1) focusing on troubleshooting boot issues using GRUB 2 rescue mode. We explored simulating a GRUB failure on an Ubuntu Server VM (running on UTM for Mac, aarch64, kernel 6.8.0-79-generic), adapting the generic Q1 commands to an LVM-based setup, and debugging boot freezes/panics. The conversation evolved from basic command practice to advanced discovery techniques (e.g., inferring root filesystems without `lsblk`), GRUB limitations, and verification methods.

This hands-on reinforced Linux+ objectives like boot troubleshooting, emphasizing practical application over rote memorization. The practice was conducted on a system (`ron@usv`) with `vda` disks, an LVM root on `/dev/mapper/ubuntu--vg-ubuntu--lv`, and a RAID5 array on `/raiddata`.

## What We Discussed

We covered a step-by-step simulation of a GRUB rescue scenario, starting from the boot menu (options like "Boot from next volume") and progressing through failures and recoveries:

1. **Accessing GRUB Rescue Prompt**:

   - Interrupted boot (Esc/Shift in UTM) to reach GRUB menu.
   - Edited entry with `c`, to force the `grub>` prompt (or `e` then `Ctrl+C` to force the `grub>` prompt).
   - Explored GRUB's `ls` command for disks/partitions (e.g., `(hd0,gpt1)` for EFI `/boot/efi`, `(hd0,gpt2)` for `/boot` with `vmlinuz-6.8.0-79-generic` and `initrd.img-6.8.0-79-generic`).

2. **Q1 Command Sequence and Adaptations**:

   - Original Q1 (generic example for simple partition):
     ```
     set root=(hd0,gpt1)
     linux (hd0,gpt1)/vmlinuz root=/dev/sda1 ro
     initrd (hd0,gpt1)/initrd.img
     boot
     ```
     - Mnemonic: "Set the table (root), cook the main dish (linux), add the side (initrd), serve (boot)."
   - Adapted to the system: Used `(hd0,gpt2)` for `/boot` (vda2), `root=/dev/mapper/ubuntu--vg-ubuntu--lv ro` for the LVM root (vda3).
   - Tested variations: Incorrect `root=` parameters (e.g., `/dev/sda1`, `/dev/vda3`), UUID searches (`search --no-floppy --fs-uuid c3a5ce35-aad2-4cca-bc8a-fc770d3b0087` for `/boot`), and single-user mode (`ro single`).

3. **Key Challenges and Insights**:
   - `linux` command: GRUB-specific for loading the kernel and passing parameters (e.g., `root=` specifies the kernel’s root filesystem; `ro` mounts it read-only initially).
   - GRUB `ls` limitations: Lists partitions/files (e.g., `ls (hd0,gpt2)/`), but cannot access `/dev/` (a Linux runtime directory) or run `lsblk`.
   - Boot freezes: Non-LVM attempts (e.g., `root=/dev/sda1`, `root=/dev/vda3`, or missing `root=`) caused kernel panics (unable to mount root). The correct LVM path (`/dev/mapper/ubuntu--vg-ubuntu--lv`) worked.
   - UUIDs from `/etc/fstab`: `/boot` UUID `c3a5ce35-aad2-4cca-bc8a-fc770d3b0087`; root used an LVM UUID (`dm-uuid-LVM-...`).

## Key Learnings

- **GRUB vs. Linux Environment**: GRUB operates pre-kernel, seeing disks as `(hdX,Y)` (e.g., `(hd0,gpt2)`). The `root=` parameter in `linux` uses Linux device names (e.g., `/dev/mapper/...` for LVM), passed to the kernel after `initrd` loads necessary modules (like LVM drivers).
- **LVM Specifics**: Root on a logical volume requires a precise `root=/dev/mapper/ubuntu--vg-ubuntu--lv`; `initrd` handles LVM activation. The generic Q1 (`/dev/sda1`) assumes a simple partition—real systems often need adaptation.
- **Trial-and-Error Troubleshooting**: Incorrect `root=` causes panics; `search --fs-uuid` helps find partitions by UUID (more reliable than device names, which can change).
- **Boot Sequence Deep Dive**: `set root` defines GRUB’s file access point; `linux` loads the kernel with parameters; `initrd` provides early drivers (e.g., for LVM); `boot` starts the kernel.
- **Exam Relevance**: Mirrors Linux+ scenarios (e.g., "locate root and boot latest kernel"). Builds confidence in manual recovery without tools like `lsblk`.
- **VM Quirks**: UTM/aarch64 uses `vda` (not `sda`); EFI/GPT partitioning is common; snapshots ensure safe experimentation.

## How to Verify Success/Failures

After a boot attempt (successful or failed), confirm outcomes using logs or system checks:

1. **Post-Boot Verification (Working System)**:

   - Kernel loaded: `uname -a` (expect `Linux usv 6.8.0-79-generic ...`).
   - Root mounted: `df -h /` (shows `/dev/mapper/ubuntu--vg-ubuntu--lv` at `/`, ~14.5GB).
   - Parameters passed: `cat /proc/cmdline | grep root` (shows `root=/dev/mapper/ubuntu--vg-ubuntu--lv ro ...`).

2. **Failure Tracing (After Freeze/Reboot)**:

   - Previous boot logs:
     ```
     sudo journalctl -k -b -1 | grep -iE 'panic|mount|root|lv|vg|unable|error|fail'
     ```
   - Current boot grep: `dmesg -T | grep -iE 'panic|mount|root'`.
   - Enable persistent logs: `sudo mkdir -p /var/log/journal` (survives reboots).

   - Sadly, none of the above helps tracing the failures.

## Preparation for This Practice

Before simulating GRUB rescue, gather system information and set up safeguards—treat it like a real outage scenario:

1. **System Recon (Run in Working Session)**:

   - Device Layout: `lsblk -a` (note LVM path like `/dev/mapper/ubuntu--vg-ubuntu--lv`, partitions like `vda2` for `/boot`).
   - UUIDs/Labels: `blkid` (e.g., root UUID); `cat /etc/fstab` (mount points, UUIDs like `c3a5ce35-aad2-4cca-bc8a-fc770d3b0087` for `/boot`).
   - Kernel Files: `ls /boot` (confirm `vmlinuz-6.8.0-79-generic`, `initrd.img-6.8.0-79-generic`).
   - GRUB Config: `cat /etc/default/grub` (check timeout/menu settings).

2. **VM Safeguards**:

   - **Snapshot**: In UTM, save a VM snapshot (VM Settings > Snapshots) before edits—revert instantly.
   - **Backup**: Export VM disk if paranoid.
   - **Access Tweaks**: Edit `/etc/default/grub` for a visible menu (`GRUB_TIMEOUT=5`, `GRUB_TIMEOUT_STYLE=menu`), then run `sudo update-grub` and reboot. (only if needed, a bit overdo)

3. **Quick Cheat Sheet** (Print/Save):
   | Command | Purpose | Your Example |
   |---------|---------|--------------|
   | `ls` | List disks/partitions | `(hd0,gpt2)` for `/boot` |
   | `set root=(hd0,gpt2)` | Set GRUB root for files | Load from vda2 |
   | `linux ... root=...` | Load kernel + params | `linux root=/dev/mapper/ubuntu--vg-ubuntu--lv ro` |
   | `search --fs-uuid <UUID>` | Auto-set root by UUID | For `/boot`: `c3a5ce35-aad2-4cca-bc8a-fc770d3b0087` |

## Recommendations

- **Practice Iteratively**: Start with the working sequence, then introduce failures (e.g., wrong `root=`) to build debugging skills. Vary kernels (`ls (hd0,gpt2)/` for older `vmlinuz-*`) or modes (`ro single` for recovery shell).
- **Exam Prep Tips**: Memorize the mnemonic and generic Q1 sequence for Linux+, but understand adaptations for LVM/UUIDs. Practice on varied setups (e.g., a non-LVM VM) to handle "unknown" systems.
- **Safety First**: Always use snapshots; avoid practicing on production hardware. If LVM/RAID is complex, create a minimal Ubuntu ISO VM for simple partition practice.
- **Extensions**:
  - Permanent Fix: After rescue, chroot and run `grub-install /dev/vda` + `update-grub`.

## Step-by-Step Setup Guide for GRUB Rescue Practice

To safely replicate and practice the GRUB 2 rescue prompt on your Ubuntu Server VM in UTM (Mac, aarch64), follow this guide. It builds on the preparation above, ensuring you can simulate failures, recover, and debug without risk. Total time: 20–30 minutes for initial setup, 10–15 minutes per practice run.

### Prerequisites

- **Hardware/Software**: Mac with UTM installed; Ubuntu Server VM running (your `ron@usv` setup with kernel 6.8.0-79-generic).
- **Access**: SSH or console access to the VM (e.g., via UTM's VNC/Spice viewer).
- **Backup**: Always work on a snapshot—UTM makes this easy.

### Step 1: Duplicate the VM and all its data (5 minutes)

### Step 2: Gather System Intelligence (5 minutes)

Boot the VM normally and run these commands to note your setup (copy-paste into a notes file). This is your "cheat sheet" for GRUB (since no `lsblk` in rescue mode).

1. Boot the VM and log in (`ron@usv`).
2. Run and record:

   ```
   lsblk -a
   ```

   - Note: Root on `/dev/mapper/ubuntu--vg-ubuntu--lv` (LVM in vda3); `/boot` on vda2.

   ```
   cat /etc/fstab
   ```

   - Note: Root entry (`/dev/disk/by-id/dm-uuid-LVM-... / ext4 defaults 0 1`); `/boot` UUID (`c3a5ce35-aad2-4cca-bc8a-fc770d3b0087`).

   ```
   blkid /dev/mapper/ubuntu--vg-ubuntu--lv
   ```

   - Note: Root UUID (e.g., `UUID="abc123..." TYPE="ext4"`—use for `search --fs-uuid`).

   ```
   ls /boot | grep vmlinuz
   ```

   - Note: Kernel files (e.g., `vmlinuz-6.8.0-79-generic`).

   ```
   uname -a
   ```

   - Confirm: `Linux usv 6.8.0-79-generic ...`.

3. Shut down: `sudo shutdown -h now`.

### Step 3: Configure UTM for Easier GRUB Access and Debugging (5 minutes)

1. Start the VM, but interrupt boot immediately (hold Esc/Shift) to test GRUB menu visibility.
   - If menu doesn't appear, boot normally, then edit `/etc/default/grub`:
     ```
     sudo nano /etc/default/grub
     ```
     Add/Set:
     ```
     GRUB_TIMEOUT=5
     GRUB_TIMEOUT_STYLE=menu
     ```
     Run `sudo update-grub`, reboot, and confirm menu shows (e.g., "Ubuntu Server", "Boot from next volume").
2. Shut down again.

### Step 4: Practice the GRUB Rescue Sequence (10–15 minutes per run)

Now simulate and recover—repeat 3–5 times, introducing variations.

1. **Power On and Access GRUB**:

   - Start VM in UTM.
   - Hold Esc/Shift during boot to enter GRUB menu.
   - Select "Boot from next volume" (your usual), press `e` to edit.
   - Press Ctrl+C to drop to `grub>` prompt (GNU GRUB 2.12).

2. **Explore and Discover (GRUB-Only)**:

   ```
   ls  # Lists: (hd0,gpt1) (hd0,gpt2) etc.
   ls (hd0,gpt2)/  # Confirms: vmlinuz-6.8.0-79-generic, initrd.img-6.8.0-79-generic
   ```

3. **Working Recovery (Baseline)**:

   ```
   set root=(hd0,gpt2)
   linux (hd0,gpt2)/vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro
   initrd (hd0,gpt2)/initrd.img-6.8.0-79-generic
   boot
   ```

   - Boots to login. Verify: `uname -a`, `df -h /`.

4. **Simulate Failure and Debug**:

   - Reboot to `grub>`.
   - Try wrong root (e.g., Attempt 1):
     ```
     set root=(hd0,gpt2)
     linux (hd0,gpt2)/vmlinuz-6.8.0-79-generic root=/dev/sda1 ro verbose debug
     initrd (hd0,gpt2)/initrd.img-6.8.0-79-generic
     boot
     ```
     - VM freezes. Force reboot (UTM: VM > Shut Down > Force).
   - Variation: UUID for `/boot` (wrong for root):
     ```
     search --no-floppy --fs-uuid --set=root c3a5ce35-aad2-4cca-bc8a-fc770d3b0087
     linux /vmlinuz-6.8.0-79-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro
     initrd /initrd.img-6.8.0-79-generic
     boot
     ```

### Step 5: Clean Up and Iterate (2 minutes)

- Revert to "Pre-GRUB-Practice" snapshot in UTM (VM > Snapshots > Restore).
- If permanent changes (e.g., GRUB timeout), reboot and test normal boot.
- Iterate: Practice 2–3 failures, then review logs. Time yourself for exam speed (under 5 minutes per recovery).

### Tips and Troubleshooting

- **Stuck?**: Revert snapshot; check UTM logs (VM > Logs).
- **No GRUB Menu**: Increase timeout as in Step 3.
- **aarch64/UTM Issues**: If keys fail, use UTM's "Send Key" (Ctrl+Alt+Esc).
- **Scale Up**: Create a second VM without LVM (Ubuntu minimal install) for Q1's generic `/dev/sda1` practice.
- **Track Progress**: Add notes to this Markdown file after each run (e.g., "Attempt 2: Panic due to LVM not activated").

This practice turned a "weird" GRUB command into a superpower—boot failures are now demystified!
