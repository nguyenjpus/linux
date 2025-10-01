---
layout: post
title: "Multiple-choice - Questions 21-30 Explained"
date: 2025-09-30
tags: [Linux+, multiple-choice]
---

This document explains questions 21-30 from a set of 100 scenario-based multiple-choice questions on Linux system management, focusing on storage, LVM, RAID, and filesystems. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention.

## Question 21: Updating Module Dependency Database

**Question**: After installing new kernel modules, which command regenerates the `modules.dep` file?  
**Options**:

- modprobe -u
- depmod -a
- update-initramfs -u
- modules-update

**Correct Answer**: depmod -a  
**Why Correct**: `depmod -a` updates the module dependency database (`modules.dep`) in `/lib/modules/$(uname -r)/`, ensuring the kernel knows which modules depend on others for proper loading.  
**Why Others Wrong**:

- `modprobe -u` is invalid (`modprobe` loads/unloads modules).
- `update-initramfs -u` updates the initramfs image, not module dependencies.
- `modules-update` is not a valid command.  
  **Key Concept**: Run `depmod -a` after adding new modules to ensure `modprobe` works correctly.  
  **Memory Aid**: “depmod = Dependencies mapped.”

## Question 22: Viewing Kernel Ring Buffer Messages

**Question**: Which command is most useful for viewing kernel ring buffer messages for hardware detection and driver issues?  
**Options**:

- cat /var/log/syslog
- journalctl -k or dmesg
- lspci -v
- tail /proc/kmsg

**Correct Answer**: journalctl -k or dmesg  
**Why Correct**: Both `dmesg` and `journalctl -k` display kernel ring buffer messages, which log hardware detection, driver loading, and boot issues. `journalctl -k` is systemd’s equivalent to `dmesg`.  
**Why Others Wrong**:

- `/var/log/syslog` logs system events, but not always kernel-specific messages.
- `lspci -v` lists PCI devices, not logs.
- `/proc/kmsg` is raw kernel log (not user-friendly; requires root).  
  **Key Concept**: Use `dmesg --follow` for real-time logs.  
  **Memory Aid**: “dmesg = Debug messages, journalctl -k = Kernel journal.”

## Question 23: Extending an LVM Logical Volume

**Question**: To extend the /data filesystem (an LVM logical volume) using a new disk /dev/sdd, what is the correct sequence of steps?  
**Options**:

- Create a partition on /dev/sdd, run pvcreate, vgextend, lvextend, and then resize the filesystem.
- Run lvextend on the logical volume, then vgextend to add the new disk.
- Create a partition on /dev/sdd, format it, and then mount it inside /data.
- Run pvcreate on the new disk, then merge it with the existing volume group.

**Correct Answer**: Create a partition on /dev/sdd, run pvcreate, vgextend, lvextend, and then resize the filesystem.  
**Why Correct**: The sequence is:

1. Partition `/dev/sdd` (e.g., `/dev/sdd1` using `fdisk`).
2. `pvcreate /dev/sdd1` to initialize it as a physical volume (PV).
3. `vgextend vg_name /dev/sdd1` to add the PV to the volume group (VG).
4. `lvextend -L +size /dev/vg_name/lv_name` to extend the logical volume (LV).
5. Resize the filesystem (e.g., `resize2fs` for Ext4 or `xfs_growfs` for XFS).  
   **Why Others Wrong**:

- `lvextend` before `vgextend` fails because the VG needs more space first.
- Formatting and mounting `/dev/sdd1` bypasses LVM, creating a separate filesystem, not extending the existing one.
- `pvcreate` then “merge” is vague; `vgextend` is the correct term for adding PVs.  
  **Why No Formatting Before vgextend?** LVM uses raw partitions or disks as PVs, initialized with `pvcreate`, which adds LVM metadata but no filesystem. The new partition (`/dev/sdd1`) isn’t formatted separately; its raw storage is added to the VG’s pool. The existing LV’s filesystem (e.g., Ext4 or XFS on `/data`) is resized after `lvextend` to use the new space, without reformatting. Ext4/XFS filesystems are extended dynamically (using `resize2fs` or `xfs_growfs`), preserving data. The new disk’s storage is integrated into the LV’s filesystem, not formatted independently.  
  **Key Concept**: LVM enables dynamic storage expansion without data loss, abstracting raw storage into flexible LVs.  
  **Memory Aid**: “Partition, PV, VG, LV, Filesystem = P-P-V-L-F. No format for VG—raw PVs power LVM.”

## Question 24: Creating a New Logical Volume

**Question**: Which command creates a new 50 GB logical volume named `lv-web` from volume group `vg-main`?  
**Options**:

- vgcreate -n lv-web -L 50G vg-main
- lvcreate -n lv-web -L 50G vg-main
- pvcreate -n lv-web -L 50G vg-main
- lvextend -n lv-web -L 50G vg-main

**Correct Answer**: lvcreate -n lv-web -L 50G vg-main  
**Why Correct**: `lvcreate -n lv-web -L 50G vg-main` creates a logical volume named `lv-web` with 50GB from the `vg-main` volume group.  
**Why Others Wrong**:

- `vgcreate` creates volume groups, not logical volumes.
- `pvcreate` initializes physical volumes, not logical ones.
- `lvextend` extends existing logical volumes, not creates new ones.  
  **Key Concept**: LVM hierarchy: Physical Volume → Volume Group → Logical Volume.  
  **Memory Aid**: “lvcreate = Logical volume creation.”

## Question 25: Creating a RAID 1 Array

**Question**: To configure two disks (/dev/sdb, /dev/sdc) as a mirrored RAID 1 array, which command is the first step?  
**Options**:

- mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
- raid-create -l 1 -d 2 /dev/sdb /dev/sdc
- lvcreate --type raid1 -n mirror -L 100%FREE vg-main /dev/sdb /dev/sdc
- fdisk -t raid /dev/sdb /dev/sdc

**Correct Answer**: mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc  
**Why Correct**: `mdadm` creates software RAID arrays; this command sets up a RAID 1 (mirror) array on `/dev/md0` using `/dev/sdb` and `/dev/sdc`.  
**Why Others Wrong**:

- `raid-create` is not a valid command.
- `lvcreate --type raid1` is for LVM RAID, not standard software RAID.
- `fdisk -t raid` sets partition type, not creates RAID.  
  **Key Concept**: RAID 1 mirrors data for redundancy.  
  **Memory Aid**: “mdadm = Make disk array, mirror mode.”

## Question 26: Resizing an Ext4 Filesystem After lvextend

**Question**: After extending a logical volume with `lvextend`, which command resizes the Ext4 filesystem to use the new space?  
**Options**:

- remount -o resize /dev/vg-main/lv-data
- fsck.ext4 -f /dev/vg-main/lv-data
- resize2fs /dev/vg-main/lv-data
- mkfs.ext4 -S /dev/vg-main/lv-data

**Correct Answer**: resize2fs /dev/vg-main/lv-data  
**Why Correct**: `resize2fs` adjusts the Ext4 filesystem size to match the extended logical volume, making new space available.  
**Why Others Wrong**:

- `remount -o resize` is invalid.
- `fsck.ext4 -f` checks for errors, not resizes.
- `mkfs.ext4 -S` formats the filesystem, destroying data.  
  **Key Concept**: Use `xfs_growfs` for XFS filesystems instead.  
  **Memory Aid**: “resize2fs = Resize to fit space.”

## Question 27: Persistent NFS Mount

**Question**: To mount an NFS share from 192.168.1.50:/exports/data at /mnt/nfsdata persistently, which file must be edited?  
**Options**:

- /etc/exports
- /etc/fstab
- /etc/mtab
- /proc/mounts

**Correct Answer**: /etc/fstab  
**Why Correct**: Adding an entry like `192.168.1.50:/exports/data /mnt/nfsdata nfs defaults 0 0` to `/etc/fstab` ensures the NFS share mounts automatically at boot on the client system.  
**Why Others Wrong**:

- `/etc/exports` is used on the NFS server (e.g., 192.168.1.50) to define shared directories (e.g., `/exports/data 192.168.1.0/24(rw,sync)`), not for client-side mounting.
- `/etc/mtab` is a runtime mount table, not for persistent config.
- `/proc/mounts` is a kernel mount table, read-only.  
  **Why /etc/exports is Incorrect**: On the NFS server, `/etc/exports` specifies which directories are shared and to whom (e.g., IP ranges). The client (this system) uses `/etc/fstab` to mount those shares. The question focuses on the client’s persistent mount setup, not server export config.  
  **Key Concept**: `/etc/fstab` format: device, mount point, type, options, dump, pass. NFS server exports vs. client mounts are distinct roles.  
  **Memory Aid**: “Exports for server sharing, fstab for client grabbing.”

## Question 28: Checking RAID Array Status

**Question**: Which command provides detailed information about the status of a software RAID array /dev/md0?  
**Options**:

- cat /proc/mdstat
- mdadm --detail /dev/md0
- lsraid /dev/md0
- fdisk -l /dev/md0

**Correct Answer**: mdadm --detail /dev/md0  
**Why Correct**: `mdadm --detail /dev/md0` shows detailed RAID status, including state (clean, degraded), sync status, and disk roles.  
**Why Others Wrong**:

- `/proc/mdstat` shows basic RAID status, less detailed.
- `lsraid` is not a standard command.
- `fdisk -l` lists partitions, not RAID status.  
  **Key Concept**: RAID status is critical for redundancy monitoring.  
  **Memory Aid**: “mdadm --detail = Detailed array monitor.”

## Question 29: "Target is busy" Error When Unmounting

**Question**: Why does unmounting /mnt/data fail with a "target is busy" error?  
**Options**:

- The administrator does not have sudo privileges.
- The filesystem has disk errors and needs fsck.
- A user or process currently has a file open or is using a directory within /mnt/data.
- The /etc/fstab file has an incorrect entry.

**Correct Answer**: A user or process currently has a file open or is using a directory within /mnt/data.  
**Why Correct**: The “target is busy” error means a process is accessing files or directories on the mount point, preventing unmounting. Use `lsof /mnt/data` or `fuser -m /mnt/data` to find culprits.  
**Why Others Wrong**:

- Lack of sudo causes a permission error, not “busy.”
- Disk errors don’t cause this error (use `fsck` to check).
- Incorrect `/etc/fstab` affects mounting, not unmounting.  
  **Key Concept**: Unmounting requires no active file handles.  
  **Memory Aid**: “Busy = Blocked by users.”

## Question 30: Partitioning Scheme for Large Disks

**Question**: For a 2TB disk to support filesystems >2TB and more than four primary partitions, which partitioning scheme should be used?  
**Options**:

- Master Boot Record (MBR)
- GUID Partition Table (GPT)
- Extended Partition
- Logical Volume Manager (LVM)

**Correct Answer**: GUID Partition Table (GPT)  
**Why Correct**: GPT supports disks >2TB and up to 128 partitions, unlike MBR’s 2TB limit and four primary partitions.  
**Why Others Wrong**:

- MBR is limited to 2TB and four primaries.
- Extended partitions (within MBR) allow more logical partitions but still cap at 2TB.
- LVM is a volume management layer, not a partitioning scheme.  
  **Key Concept**: GPT is standard for modern, large disks.  
  **Memory Aid**: “GPT = Gigantic Partition Table.”

## Retention Tips for Questions 21-30

- **Themes**: Storage management (LVM, RAID, filesystems), kernel module dependencies, and mount troubleshooting.
- **Mnemonic for LVM Process**: “Partition, Physical Volume, Volume Group, Logical Volume, Filesystem = P-P-V-L-F.”
- **Practice**: In a Linux VM:
  - Create a RAID 1 array with `mdadm` using loop devices.
  - Add a disk to an LVM volume group and extend a logical volume.
  - Check `dmesg` for hardware logs and edit `/etc/fstab` for an NFS mount.
- **Spaced Repetition**: Review these in 24 hours, then in 3 days. Flashcards: “What does mdadm --detail show?” → “RAID status, clean/degraded.”
- **Quiz Yourself**: What’s the sequence to extend an LVM volume? Why does unmounting fail with “busy”?
