---
layout: post
title: "Linux Lab Summary: Mastering Linux Storage Management"
date: 2025-10-11 11:55:00 -0700
tags:
  [
    Linux+,
    Linuxlab,
    100 Multiple Choices,
    Storage,
    LVM,
    RAID,
    Xfs,
    Backups,
    Wipefs,
    Resize2fs,
    Mdadm,
  ]
---

This document summarizes the CompTIA Linux+ Filesystem Management Lab, covering Logical Volume Manager (LVM), RAID, backups, disk space management, and XFS filesystems. It details the environment, steps, outputs, addressed questions, configuration files, insights, bookmarks, and good practices, ensuring a beginner-friendly guide for Linux+ certification preparation (XK0-005 objectives).

## Lab Environment

- **Operating System**: Ubuntu Server 24.04.3 LTS (arm64)
- **Architecture**: aarch64
- **Virtualization**: UTM Virtual Machine
- **Kernel**: 6.8.0-85-generic
- **Disk Layout** (from `lsblk` and `blkid`):
  - `/dev/vda`: 32GB (system disk)
    - `/dev/vda1`: 1GB, vfat, `/boot/efi` (UUID: C6B8-2DBC)
    - `/dev/vda2`: 2GB, ext4, `/boot` (UUID: 910ca4fc-68bb-4d17-a6ad-faa654339112)
    - `/dev/vda3`: 28.9GB, LVM2_member, hosts `ubuntu-vg/ubuntu-lv` (14.5GB, ext4, UUID: 97788617-ae17-4940-ab7d-d488fffba8ab, mounted on `/`)
  - `/dev/vdb`, `/dev/vdc`, `/dev/vdd`: 10GB each, used for LVM, RAID, backups, and XFS
  - `/dev/sr0`: 2.8GB, iso9660 (Ubuntu installation media)
- **Tools Used**: `mdadm`, `parted`, `wipefs`, `mkfs.ext4`, `mkfs.xfs`, `dd`, `tar`, `fallocate`, `lsof`, `lsblk`, `blkid`, `partprobe`, `e2fsck`, `resize2fs`, `xfs_growfs`, `xfs_info`, `lvextend`, `lvreduce`

## Lab Objectives

The lab covers CompTIA Linux+ objectives for filesystem and storage management, including:

- **LVM** (Questions 25, 27-28, 30-32): Creating, extending, reducing, and removing LVM setups.
- **RAID** (Questions 25, 28, 30-32): Configuring RAID 1, handling failures, and cleanup.
- **Backups** (Questions 62, 68, 81, 94): Using `dd` for disk images, `tar` for file archives, and setting up swap.
- **Disk Space Issues** (Questions 34, 81): Managing full filesystems and log archiving.
- **Filesystem Management** (Questions 23, 24, 27, 29, 30-32, 33, 66, 83, 90, 94, 96, 100): Partitioning, formatting (ext4, XFS), mounting, resizing, and cleanup.

## Hands-On Steps, Purposes, Outputs, Bookmarks, and Good Practices

### Section 1: Logical Volume Manager (LVM) Setup with ext4

**Purpose**: Learn to create, extend, reduce, and remove LVM volumes with ext4 filesystems (Questions 25, 27-28, 30-32).

1. **Wipe Disk**:

   - **Command**:
     ```bash
     wipefs -a /dev/vdc
     ```
   - **Purpose**: Clear any existing filesystem or RAID metadata to prepare `/dev/vdc` for LVM (Question 30).
   - **Output**: `ext4` signatures erased (`53 ef`).
   - **Bookmarks**:
     - Use `wipefs -a` to clear metadata before LVM or RAID setup to avoid conflicts (Questions 30, 32).
   - **Good Practice**:
     - Use `wipefs -a` before LVM/RAID setup; verify with `blkid` and `lsblk`.

2. **Partition Disk**:

   - **Command**:
     ```bash
     parted /dev/vdc mklabel gpt mkpart primary ext4 1MiB 100%
     parted /dev/vdc set 1 lvm on
     partprobe /dev/vdc
     ```
   - **Purpose**: Create a GPT partition table and a single partition (`/dev/vdc1`) flagged for LVM (Questions 30, 31).
   - **Output**: Partition created; `lsblk` shows `/dev/vdc1` (~10GB); `blkid` shows `PARTLABEL="primary"`, `PARTUUID="7aa333cb-7910-474b-91f3-b2012451efe6"`.
   - **Bookmarks**:
     - In `parted mkpart`, the `ext4` parameter sets a partition type hint,`not the filesystem; use`mkfs.ext4`or`mkfs.xfs` to create the filesystem (Questions 30, 32).
     - Rescan with `partprobe` if partitions don’t appear; check `lsblk`, `blkid` (Questions 30, 31).
   - **Good Practices**:
     - Flag partitions for LVM with `set 1 lvm on` in `parted`.
     - Check partition tables with `parted <disk> print`.
     - Rescan disks with `partprobe` after `parted`; verify with `lsblk`, `blkid`.

3. **Create Physical Volume (PV)**:

   - **Command**:
     ```bash
     pvcreate /dev/vdc1
     ```
   - **Purpose**: Initialize `/dev/vdc1` as an LVM PV (Question 25).
   - **Output**: `Physical volume "/dev/vdc1" successfully created.`
   - **Bookmarks**:
     - `pvcreate` initializes a partition for LVM; verify with `pvs` (Question 25).
   - **Good Practice**:
     - Verify LVM setup with `pvs`, `vgs`, `lvs`.

4. **Create Volume Group (VG)**:

   - **Command**:
     ```bash
     vgcreate my_vg /dev/vdc1
     ```
   - **Purpose**: Group PVs into a VG named `my_vg` (Question 25).
   - **Output**: `Volume group "my_vg" successfully created.`
   - **Bookmarks**:
     - `vgcreate` groups PVs for logical volume allocation (Question 25).
   - **Good Practice**:
     - Verify with `vgs`.

5. **Create Logical Volume (LV)**:

   - **Command**:
     ```bash
     lvcreate -L 7G -n my_lv my_vg
     ```
   - **Purpose**: Create a 7GB LV named `my_lv` in `my_vg` (Question 25).
   - **Output**: `Logical volume "my_lv" created.`
   - **Bookmarks**:
     - `lvcreate -L` specifies LV size; `-n` sets the name (Question 25).
   - **Good Practice**:
     - Verify with `lvs`.

6. **Format and Mount LV**:

   - **Command**:
     ```bash
     mkfs.ext4 /dev/my_vg/my_lv
     mkdir /mnt/test
     mount /dev/my_vg/my_lv /mnt/test
     ```
   - **Purpose**: Format LV as ext4, mount at `/mnt/test` for use (Questions 27, 32).
   - **Output**: Filesystem created (UUID: `37606edd-7748-4d60-9ffd-e5f2e1c7aecb`); `df -h /mnt/test` shows ~6.8GB, 6.5GB available; `lvs` shows `my_lv` (7.00g).
   - **Bookmarks**:
     - `mkfs.ext4` creates an ext4 filesystem; mount with `mkdir` and `mount` (Questions 27, 32).
   - **Good Practices**:
     - Unmount before modifying LVM setup.
     - Test mounts with file read/write (e.g., `echo "test" > /mnt/test/file.txt`).

7. **Extend LV**:

   - **Command**:
     ```bash
     lvextend -L +2G /dev/my_vg/my_lv
     resize2fs /dev/my_vg/my_lv
     ```
   - **Purpose**: Increase LV size by 2GB (not performed here as LV was created at 7GB), resize filesystem (Question 28).
   - **Output (if run)**: `lvextend` extends LV; `resize2fs` expands filesystem; `df -h` shows ~7GB.
   - **Bookmarks**:
     - Use `lvextend -L +<size>` to grow LVs; `resize2fs` for ext4 filesystems (Question 28).
   - **Good Practice**:
     - Run `resize2fs` after `lvextend`; verify with `df -h`.

8. **Reduce LV**:

   - **Command**:
     ```bash
     umount /mnt/test
     e2fsck -f /dev/my_vg/my_lv
     resize2fs /dev/my_vg/my_lv 4G
     lvreduce -L 4G /dev/my_vg/my_lv
     mount /dev/my_vg/my_lv /mnt/test
     ```
   - **Purpose**: Shrink filesystem to 4GB, reduce LV to match, remount to verify (Question 28).
   - **Output**:
     ```
     umount: /mnt/test: unmounted
     e2fsck 1.47.0 (5-Feb-2023)
     Pass 1: Checking inodes, blocks, and sizes
     ...
     /dev/my_vg/my_lv: 11/458752 files (0.0% non-contiguous), 53247/1835008 blocks
     resize2fs 1.47.0 (5-Feb-2023)
     Resizing the filesystem on /dev/my_vg/my_lv to 1048576 (4k) blocks.
     The filesystem on /dev/my_vg/my_lv is now 1048576 (4k) blocks long.
     Size of logical volume my_vg/my_lv changed from 7.00 GiB (1792 extents) to 4.00 GiB (1024 extents).
     Logical volume my_lv successfully resized.
     ```
     `lvs` shows `my_lv` as 4.00g; `df -h /mnt/test` shows 3.9G total, 3.7G available; `lsblk` shows `/dev/my_vg/my_lv` (4G) on `/mnt/test`.
   - **Key Insights**:
     - Reducing the LV before the filesystem risks data loss because the filesystem’s metadata may extend beyond the reduced LV size, causing corruption. The correct order—unmount, `e2fsck -f`, `resize2fs`, then `lvreduce`—ensures the filesystem fits within the new LV size (Question 28).
     - Focusing on ext4 covered most Linux+ filesystem objectives, but XFS is also important for its scalability and performance in enterprise environments. Adding XFS experience (e.g., `mkfs.xfs`, `xfs_growfs`) ensures comprehensive preparation for Questions 27, 28, and 32, addressing filesystem creation, mounting, and resizing.
   - **Bookmarks**:
     - Before reducing, unmount, check with `e2fsck -f`, resize with `resize2fs`, then `lvreduce` (Question 28).
     - Always run `resize2fs` before `lvreduce` when shrinking an LV to prevent data loss; unmount and check with `e2fsck -f` first (Question 28).
   - **Good Practices**:
     - Unmount before reducing LVs.
     - Run `resize2fs` before `lvreduce`; verify with `df -h`.
     - When reducing an LV, unmount, run `e2fsck -f`, `resize2fs`, then `lvreduce` in that order to avoid filesystem corruption (Question 28).

9. **Remove LV, VG, PV**:
   - **Command**:
     ```bash
     umount /mnt/test
     lvremove /dev/my_vg/my_lv
     vgremove my_vg
     pvremove /dev/vdc1
     wipefs -a /dev/vdc
     ```
   - **Purpose**: Clean up LVM setup (Questions 30, 32).
   - **Output**:
     ```
     umount: /mnt/test: unmounted
     Logical volume "my_lv" successfully removed.
     Volume group "my_vg" successfully removed.
     Physical volume "/dev/vdc1" successfully removed.
     /dev/vdc: GPT signatures erased.
     ```
     `pvs`, `vgs`, `lvs`, `blkid /dev/vdc` show no entries.
   - **Bookmarks**:
     - Remove LVM with `lvremove`, `vgremove`, `pvremove` in that order (Questions 30, 32).
     - Use `wipefs -a` to clear metadata (Questions 30, 32).
   - **Good Practices**:
     - Verify VG/PV removal with `vgs`, `pvs`, `blkid`.
     - Use `wipefs -a`; verify with `blkid`, `lsblk`.

### Section 2: Logical Volume Manager (LVM) Setup with XFS

**Purpose**: Create, format, mount, and resize an XFS filesystem on an LVM Logical Volume (Questions 25, 27-28, 30-32).

1. **Prepare Disk**:

   - **Command**:
     ```bash
     wipefs -a /dev/vdc
     parted /dev/vdc mklabel gpt mkpart primary 1MiB 100%
     partprobe /dev/vdc
     ```
   - **Purpose**: Clear `/dev/vdc`, create GPT and `/dev/vdc1` for LVM (Questions 30, 31).
   - **Output**: Signatures erased; `lsblk` shows `/dev/vdc1` (~10GB); `blkid` shows `PARTLABEL="primary"`.
   - **Bookmarks**:
     - Use `wipefs -a` to clear metadata (Questions 30, 32).
     - `parted mkpart` sets partition type; use `mkfs.xfs` for filesystem (Questions 30, 32).
     - Rescan with `partprobe`; check `lsblk`, `blkid` (Questions 30, 31).
   - **Good Practices**:
     - Use `wipefs -a` before LVM; verify with `blkid`, `lsblk`.
     - Rescan with `partprobe` after `parted`; verify with `lsblk`, `blkid`.

2. **Set Up LVM**:

   - **Command**:
     ```bash
     pvcreate /dev/vdc1
     vgcreate xfs_vg /dev/vdc1
     lvcreate -L 4G -n xfs_lv xfs_vg
     ```
   - **Purpose**: Create PV, VG (`xfs_vg`), and 4GB LV (`xfs_lv`) for XFS (Question 25).
   - **Output**:
     ```
     Physical volume "/dev/vdc1" successfully created.
     Volume group "xfs_vg" successfully created.
     Logical volume "xfs_lv" created.
     ```
     `lvs` shows `xfs_lv` (4.00g).
   - **Bookmarks**:
     - `pvcreate` initializes a partition for LVM; verify with `pvs` (Question 25).
     - `vgcreate` groups PVs (Question 25).
     - `lvcreate -L` specifies LV size; `-n` sets the name (Question 25).
   - **Good Practice**:
     - Verify LVM setup with `pvs`, `vgs`, `lvs`.

3. **Create XFS Filesystem**:

   - **Command**:
     ```bash
     mkfs.xfs /dev/xfs_vg/xfs_lv
     ```
   - **Purpose**: Format the LV as XFS (Question 32).
   - **Output**:
     ```
     meta-data=/dev/xfs_vg/xfs_lv     isize=512    agcount=4, agsize=262144 blks
     ...
     data     =                       bsize=4096   blocks=1048576, imaxpct=25
     ...
     Writing superblock and filesystem accounting information: done
     ```
     `blkid /dev/xfs_vg/xfs_lv` shows `TYPE="xfs"`, UUID.
   - **Bookmarks**:
     - Use `mkfs.xfs` to create an XFS filesystem; verify with `blkid` (Question 32).
   - **Good Practice**:
     - Verify XFS filesystem creation with `blkid` and `xfs_info` (Question 32).

4. **Mount and Test XFS**:

   - **Command**:
     ```bash
     mkdir /mnt/xfs_test
     mount /dev/xfs_vg/xfs_lv /mnt/xfs_test
     echo "XFS test" > /mnt/xfs_test/test.txt
     df -h /mnt/xfs_test
     lsblk
     ```
   - **Purpose**: Mount XFS filesystem, test read/write, verify (Questions 27, 32).
   - **Output**:
     ```
     Filesystem               Size  Used Avail Use% Mounted on
     /dev/mapper/xfs_vg-xfs_lv 4.0G   33M  3.9G   1% /mnt/xfs_test
     NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
     ...
     vdc                       253:32   0   10G  0 disk
     └─vdc1                    253:33   0   10G  0 part
       └─xfs_vg-xfs_lv         252:2    0    4G  0 lvm  /mnt/xfs_test
     ```
     `ls /mnt/xfs_test` shows `test.txt`.
   - **Bookmarks**:
     - Mount XFS filesystems with `mount`; test with read/write operations (Questions 27, 32).
   - **Good Practices**:
     - Test mounts with file read/write.
     - Verify mounts with `df -h`, `lsblk`.

5. **Resize XFS Filesystem (Grow)**:

   - **Command**:
     ```bash
     lvextend -L +2G /dev/xfs_vg/xfs_lv
     xfs_growfs /mnt/xfs_test
     df -h /mnt/xfs_test
     ```
   - **Purpose**: Extend LV by 2GB to 6GB, grow XFS filesystem online, verify (Question 28).
   - **Output**:
     ```
     Size of logical volume xfs_vg/xfs_lv changed from 4.00 GiB to 6.00 GiB.
     meta-data=/dev/mapper/xfs_vg-xfs_lv isize=512    agcount=4, agsize=393216 blks
     data     =                       bsize=4096   blocks=1572864, imaxpct=25
     ...
     Filesystem               Size  Used Avail Use% Mounted on
     /dev/mapper/xfs_vg-xfs_lv 6.0G   33M  5.9G   1% /mnt/xfs_test
     ```
   - **Bookmarks**:
     - Use `xfs_growfs` to resize XFS filesystems online after `lvextend`; XFS does not support shrinking (Question 28).
   - **Good Practices**:
     - Run `xfs_growfs` after `lvextend` for XFS; verify with `df -h`.
     - Use `xfs_info` to check XFS filesystem details after resizing.

6. **Verify XFS Filesystem**:

   - **Command**:
     ```bash
     xfs_info /mnt/xfs_test
     lvs
     ```
   - **Purpose**: Check XFS metadata and LV size (Questions 23, 29, 96).
   - **Output**:
     ```
     meta-data=/dev/mapper/xfs_vg-xfs_lv isize=512    agcount=4, agsize=393216 blks
     ...
     data     =                       bsize=4096   blocks=1572864, imaxpct=25
     ...
     LV       VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
     xfs_lv   xfs_vg  -wi-ao----  6.00g
     ```
   - **Bookmarks**:
     - Use `xfs_info` to inspect XFS filesystem details; verify LV with `lvs` (Questions 23, 29, 96).
   - **Good Practice**:
     - Verify XFS filesystems with `xfs_info`, `df -h`, `lsblk`.

7. **Clean Up XFS and LVM**:
   - **Command**:
     ```bash
     umount /mnt/xfs_test
     lvremove /dev/xfs_vg/xfs_lv
     vgremove xfs_vg
     pvremove /dev/vdc1
     wipefs -a /dev/vdc
     ```
   - **Purpose**: Remove XFS filesystem, LV, VG, PV, and clear disk metadata (Questions 30, 32).
   - **Output**:
     ```
     umount: /mnt/xfs_test: unmounted
     Logical volume "xfs_lv" successfully removed.
     Volume group "xfs_vg" successfully removed.
     Physical volume "/dev/vdc1" successfully removed.
     /dev/vdc: GPT signatures erased.
     ```
     `pvs`, `vgs`, `lvs`, `blkid /dev/vdc` show no entries.
   - **Bookmarks**:
     - Remove LVM with `lvremove`, `vgremove`, `pvremove` in that order (Questions 30, 32).
     - Use `wipefs -a` to clear metadata (Questions 30, 32).
   - **Good Practices**:
     - Verify VG/PV removal with `vgs`, `pvs`, `blkid`.
     - Use `wipefs -a`; verify with `blkid`, `lsblk`.
     - Unmount and clean up at lab end.

### Section 3: RAID Configuration

**Purpose**: Set up RAID 1, simulate failure, replace disk, and clean up (Questions 25, 28, 30-32).

1. **Wipe Disks**:

   - **Command**:
     ```bash
     wipefs -a /dev/vdb
     wipefs -a /dev/vdc
     ```
   - **Purpose**: Clear metadata for RAID setup (Question 30).
   - **Output**: Signatures erased; `blkid /dev/vdb /dev/vdc` empty.
   - **Bookmarks**:
     - Use `wipefs -a` to clear metadata before LVM or RAID setup (Questions 30, 32).
   - **Good Practice**:
     - Use `wipefs -a` before RAID/LVM; verify with `blkid`, `lsblk`.

2. **Create RAID 1**:

   - **Command**:
     ```bash
     mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdb /dev/vdc
     ```
   - **Purpose**: Create RAID 1 array (mirroring) on `/dev/vdb` and `/dev/vdc` (Questions 25, 28).
   - **Output**: `mdadm: array /dev/md0 started.`; `cat /proc/mdstat` shows sync.
   - **Bookmarks**:
     - `mdadm --create --level=1` sets up RAID 1; check with `cat /proc/mdstat` (Questions 25, 28).
   - **Good Practice**:
     - Check RAID status with `mdadm --detail /dev/md0`, `cat /proc/mdstat`.

3. **Format and Mount RAID**:

   - **Command**:
     ```bash
     mkfs.ext4 /dev/md0
     mkdir /mnt/raid
     mount /dev/md0 /mnt/raid
     ```
   - **Purpose**: Format RAID as ext4, mount for use (Questions 27, 32).
   - **Output**: Filesystem created; `df -h /mnt/raid` shows ~10GB.
   - **Bookmarks**:
     - Format RAID with `mkfs.ext4`; mount to test (Questions 27, 32).
   - **Good Practice**:
     - Test mounts with file read/write.

4. **Simulate Disk Failure**:

   - **Command**:
     ```bash
     mdadm /dev/md0 -f /dev/vdc
     ```
   - **Purpose**: Mark `/dev/vdc` as failed to test RAID resilience (Question 28).
   - **Output**: `mdadm: set /dev/vdc faulty in /dev/md0`; `mdadm --detail /dev/md0` shows degraded.
   - **Bookmarks**:
     - Use `mdadm -f` to simulate RAID failure; verify with `mdadm --detail` (Question 28).
   - **Good Practice**:
     - Check RAID status post-failure.

5. **Replace Failed Disk**:

   - **Command**:
     ```bash
     mdadm /dev/md0 -r /dev/vdc
     mdadm /dev/md0 -a /dev/vdd
     ```
   - **Purpose**: Remove failed `/dev/vdc`, add `/dev/vdd` to rebuild (Question 28).
   - **Output**: `mdadm: hot removed /dev/vdc`; `mdadm: hot added /dev/vdd`; `cat /proc/mdstat` shows rebuilding.
   - **Bookmarks**:
     - Use `mdadm -r` to remove, `mdadm -a` to add disks; monitor with `cat /proc/mdstat` (Question 28).
   - **Good Practice**:
     - Verify RAID data integrity post-failure; save snapshot.

6. **Save RAID Config**:

   - **Command**:
     ```bash
     mdadm --detail --scan >> /etc/mdadm/mdadm.conf
     ```
   - **Purpose**: Persist RAID configuration (Question 28).
   - **Output**: Array details appended to `/etc/mdadm/mdadm.conf`.
   - **Bookmarks**:
     - Save RAID config to `/etc/mdadm/mdadm.conf` for boot-time assembly (Question 28).
   - **Good Practice**:
     - Test RAID integrity; save config to `/etc/mdadm/mdadm.conf`.

7. **Clean Up RAID**:
   - **Command**:
     ```bash
     umount /mnt/raid
     mdadm --stop /dev/md0
     mdadm --zero-superblock /dev/vdb
     mdadm --zero-superblock /dev/vdc
     mdadm --zero-superblock /dev/vdd
     ```
   - **Purpose**: Dismantle RAID, clear metadata (Questions 30-32).
   - **Output**: Array stopped; `blkid` shows no RAID metadata.
   - **Bookmarks**:
     - Use `mdadm --stop` and `--zero-superblock` to clean RAID (Questions 30-32).
   - **Good Practice**:
     - Verify cleanup with `blkid`, `lsblk`.

### Section 4: Disk Cleanup

**Purpose**: Ensure disks are clean for backup tasks (Questions 30-32).

1. **Wipe RAID Metadata**:
   - **Command**:
     ```bash
     mdadm --zero-superblock /dev/vdc
     ```
   - **Purpose**: Clear RAID superblock to prepare `/dev/vdc` (Question 30).
   - **Output**: No RAID metadata; `blkid /dev/vdc` empty.
   - **Bookmarks**:
     - `mdadm --zero-superblock` removes RAID metadata (Questions 30-32).
   - **Good Practice**:
     - Use `wipefs -a` or `mdadm --zero-superblock`; verify with `blkid`, `lsblk`.

### Section 5: Backups

**Purpose**: Create disk images, file archives, partition table backups, and swap (Questions 62, 68, 81, 94).

1. **Wipe and Partition Disk**:

   - **Command**:
     ```bash
     wipefs -a /dev/vdb
     parted /dev/vdb mklabel gpt mkpart primary ext4 1MiB 100%
     partprobe /dev/vdb
     ```
   - **Purpose**: Clear `/dev/vdb`, create GPT and `/dev/vdb1` for backups (Questions 30-32).
   - **Output**: Signatures erased; `lsblk` shows `/dev/vdb1` (~10GB); `blkid` shows `UUID="079fd50b-45c4-4e3a-8b4b-00da75617509" TYPE="ext4"`.
   - **Issue**: Initially, `/dev/vdb1` not detected; fixed with `partprobe /dev/vdb`.
   - **Bookmarks**:
     - `parted mkpart ext4` sets partition type; use `mkfs.ext4` (Questions 30, 32).
     - Rescan with `partprobe` if partitions don’t appear; check `lsblk`, `blkid` (Questions 30, 31).
   - **Good Practices**:
     - Use `wipefs -a` before setup; verify with `blkid`, `lsblk`.
     - Rescan disks with `partprobe` after `parted`; verify with `lsblk`, `blkid`.

2. **Format and Mount**:

   - **Command**:
     ```bash
     mkfs.ext4 /dev/vdb1
     mkdir -p /mnt/backup
     mount /dev/vdb1 /mnt/backup
     ```
   - **Purpose**: Format `/dev/vdb1` as ext4, mount for backup storage (Questions 27, 32).
   - **Output**: Filesystem created; `df -h /mnt/backup` shows ~9.8GB; `blkid /dev/vdb1` shows UUID.
   - **Bookmarks**:
     - `mkfs.ext4` creates filesystem; mount with `mkdir` and `mount` (Questions 27, 32).
   - **Good Practice**:
     - Test mounts with file read/write (e.g., `echo "Backup test" > /mnt/backup/test.txt`).

3. **Create Test Files**:

   - **Command**:
     ```bash
     echo "Backup test" > /mnt/backup/test.txt
     mkdir /mnt/backup/data
     echo "Data file" > /mnt/backup/data/data.txt
     ```
   - **Purpose**: Add files to test backups (Question 81).
   - **Output**: `ls -R /mnt/backup` shows `test.txt`, `data/data.txt`.
   - **Bookmarks**:
     - Test backups with sample files; verify with `ls -R` (Question 81).
   - **Good Practice**:
     - Test mounts with file read/write.

4. **Disk Image Backup with `dd`**:

   - **Command**:
     ```bash
     dd if=/dev/vdb1 of=/root/vd1_backup.img bs=4M status=progress
     ```
   - **Purpose**: Create a bit-for-bit image of `/dev/vdb1` (Question 62).
   - **Output**: Copied ~8.3GB, stopped due to root filesystem full; `ls -lh /root/vd1_backup.img` shows 8.3GB; `file` confirms ext4.
   - **Issue**: Root filesystem (`/`) filled (14.5GB LV); fixed by `rm /root/vd1_backup.img`.
   - **Bookmarks**:
     - `dd` creates disk images with `bs=4M` for efficiency; verify with `file` (Question 62).
     - "No space left" errors require `df -h`, `rm`, `lsof | grep deleted` (Question 34).
   - **Good Practices**:
     - Use `dd` with `bs=4M` for backups; verify restores.
     - Use `lsof | grep deleted` for full filesystems.

5. **Restore and Verify `dd` Backup**:

   - **Command**:
     ```bash
     dd if=/root/vd1_backup.img of=/dev/vdc bs=4M status=progress
     mkdir /mnt/vdc-restore
     mount /dev/vdc /mnt/vdc-restore
     ```
   - **Purpose**: Restore image to `/dev/vdc`, verify data (Question 62).
   - **Output**: Restored ~8.3GB; `ls -R /mnt/vdc-restore` shows `test.txt`, `data/data.txt`.
   - **Bookmarks**:
     - Test `dd` restores by mounting and checking files (Question 62).
   - **Good Practice**:
     - Verify restores with `ls -R`.

6. **File Archive with `tar`**:

   - **Command**:
     ```bash
     cd /mnt/backup
     tar -czvf /root/backup.tar.gz .
     ```
   - **Purpose**: Archive `/mnt/backup` contents (Question 81).
   - **Output**: Archived `test.txt`, `data/data.txt`; `tar -tzvf /root/backup.tar.gz` lists files.
   - **Issue**: Initial `tar -czvf /root/backup.tar.gz /mnt/backup` caused nested paths (`mnt/backup/`); fixed by using `cd` and `.`.
   - **Bookmarks**:
     - `tar -C` sets extraction directory; check paths with `tar -tzvf` (Question 81).
     - Avoid nested `tar` paths with `cd <dir>; tar -czvf <archive> .` or `--strip-components` (Question 81).
     - `tar` fails if directory empty; check `ls` first (Question 81).
   - **Good Practices**:
     - Verify `tar` backups with `tar -tzvf`.
     - Check `ls` before `tar`.

7. **Test `tar` Restore**:

   - **Command**:
     ```bash
     rm -rf /mnt/backup/*
     tar -xzvf /root/backup.tar.gz -C /mnt/backup/
     ```
   - **Purpose**: Restore and verify files (Question 81).
   - **Output**: `ls -R /mnt/backup` shows `test.txt`, `data/data.txt`.
   - **Bookmarks**:
     - Use `tar -xzvf -C` for restores; verify with `ls -R` (Question 81).
   - **Good Practice**:
     - Test restores with `ls -R`.

8. **Partition Table Backup**:

   - **Command**:
     ```bash
     dd if=/dev/vdb of=/root/vdb_mbr.bak bs=512 count=1
     ```
   - **Purpose**: Back up GPT protective MBR (Question 68).
   - **Output**: 512 bytes copied; `ls -lh /root/vdb_mbr.bak` shows 512B; `hexdump -C` shows `55 aa` (PMBR signature), `ee` (GPT partition type).
   - **Issue**: Failed due to full root filesystem; fixed by freeing space.
   - **Bookmarks**:
     - `dd bs=512 count=1` backs up MBR/PMBR; use `count=34` for full GPT (Question 68).
     - MBR backup shows PMBR (`55 aa`, type `ee`) for GPT; use `count=34` for full GPT (Question 68).
   - **Good Practice**:
     - Use `dd` for partition table backups; verify with `hexdump -C`.

9. **Swap File Setup**:
   - **Command**:
     ```bash
     fallocate -l 1G /swapfile
     chmod 600 /swapfile
     mkswap /swapfile
     swapon /swapfile
     echo '/swapfile none swap sw 0 0' >> /etc/fstab
     ```
   - **Purpose**: Create and activate 1GB swap file, persist in `/etc/fstab` (Question 94).
   - **Output**: `swapon --show` lists `/swapfile` (1GB) and `/swap.img` (3GB); `free -h` shows 4GB total swap, ~12KiB used.
   - **Note**: Total swap is 4GB due to existing `/swap.img`.
   - **Bookmarks**:
     - `fallocate` creates swap files efficiently; verify with `swapon --show` (Question 94).
     - Swap size sums all active areas; check with `swapon --show` (Question 94).
   - **Good Practices**:
     - Verify swap with `swapon --show`.
     - Verify swap in `/etc/fstab`.

### Section 6: Disk Space Issues

**Purpose**: Simulate and resolve full filesystem issues, archive logs (Questions 34, 81).

1. **Simulate Full Filesystem**:

   - **Command**:
     ```bash
     dd if=/dev/zero of=/mnt/backup/large.log bs=1M count=9000
     ```
   - **Purpose**: Fill `/mnt/backup` (~9GB) to simulate full filesystem (Question 34).
   - **Output**: Copied 9.4GB; `df -h /mnt/backup` shows 96% usage (450M available).
   - **Bookmarks**:
     - Simulate full filesystem with `dd`; check with `df -h` (Question 34).
   - **Good Practice**:
     - Monitor disk usage with `df -h` during large operations.

2. **Simulate Held File**:

   - **Command**:
     ```bash
     tail -f /mnt/backup/large.log &
     ```
   - **Purpose**: Hold file open to prevent space reclamation (Question 34).
   - **Output**: Background process started (PID 4302).
   - **Bookmarks**:
     - `tail -f <file> &` holds files open; use `lsof` and `kill -9` to free space (Question 34).
   - **Good Practice**:
     - Use `lsof | grep deleted` for full filesystems.

3. **Delete File and Check**:

   - **Command**:
     ```bash
     rm /mnt/backup/large.log
     df -h /mnt/backup
     ```
   - **Purpose**: Delete file, check if space is freed (Question 34).
   - **Output**: Still 96% usage due to `tail` holding file.
   - **Bookmarks**:
     - "No space left" errors require `df -h`, `rm`, `lsof | grep deleted` (Question 34).
   - **Good Practice**:
     - Use `lsof | grep deleted` for held files.

4. **Free Space**:

   - **Command**:
     ```bash
     lsof | grep large.log
     kill -9 4302
     df -h /mnt/backup
     ```
   - **Purpose**: Identify and kill process holding file, free space (Question 34).
   - **Output**: `lsof` shows `tail` (PID 4302) holding `large.log (deleted)`; after `kill -9`, `df -h` shows 1% usage (9.3G free).
   - **Bookmarks**:
     - Simulate full filesystem with `dd`; free held files with `lsof`, `kill -9` (Question 34).
     - `tail -f <file> &` holds files open; use `lsof` and `kill -9` to free space (Question 34).
   - **Good Practice**:
     - Use `lsof | grep deleted` and `kill -9` to free space.

5. **Archive Logs**:

   - **Command**:
     ```bash
     tar -czvf /mnt/backup/log-backup.tar.gz /var/log
     ```
   - **Purpose**: Archive system logs for backup (Question 81).
   - **Output**: Archived `/var/log` (e.g., `auth.log`, `syslog`); `tar -tzvf /mnt/backup/log-backup.tar.gz` lists files.
   - **Bookmarks**:
     - Archive logs with `tar -czvf` (Question 81).
   - **Good Practices**:
     - Archive logs with `tar -czvf`.
     - Clean logs after archiving.

6. **Clean Logs**:
   - **Command**:
     ```bash
     find /var/log -type f -name "*.log" -delete
     ```
   - **Purpose**: Remove log files to free space (Question 81).
   - **Output**: `.log` files deleted; `ls /var/log` shows remaining files.
   - **Bookmarks**:
     - Archive logs with `tar -czvf`; clean with `find ... -delete` (Question 81).
   - **Good Practice**:
     - Clean logs after archiving to save space.

### Cleanup Steps

**Purpose**: Reset system to clean state (Questions 30-32, 66, 83).

1. **Unmount Filesystems**:

   - **Command**:
     ```bash
     umount /mnt/backup
     umount /mnt/vdc-restore
     umount /mnt/xfs_test
     ```
   - **Purpose**: Unmount test filesystems (Question 27).
   - **Output**: No mounts shown in `df -h`.
   - **Good Practice**:
     - Unmount filesystems at lab end; verify with `df -h`.

2. **Disable Swap**:

   - **Command**:
     ```bash
     swapoff /swapfile
     swapoff /swap.img
     ```
   - **Purpose**: Deactivate swap files (Question 94).
   - **Output**: `swapon --show` shows no active swap.
   - **Good Practice**:
     - Disable swap at lab end.

3. **Remove Test Files**:

   - **Command**:
     ```bash
     rm /mnt/backup/*
     rm /root/vdb_mbr.bak
     rm /root/backup.tar.gz
     rm /swapfile
     rm /swap.img
     ```
   - **Purpose**: Delete temporary files (Question 66).
   - **Output**: `ls /mnt/backup /root` empty; `ls /swapfile /swap.img` fails.
   - **Good Practice**:
     - Remove test files at lab end.

4. **Clean `/etc/fstab`**:

   - **Command**:
     ```bash
     vim /etc/fstab  # Remove /swapfile, /swap.img entries
     ```
   - **Purpose**: Reset persistent mounts and swap (Questions 83, 94).
   - **Output**: `/etc/fstab` contains only system mounts.
   - **Good Practice**:
     - Verify `/etc/fstab` at lab end.

5. **Wipe Disks**:

   - **Command**:
     ```bash
     wipefs -a /dev/vdb
     wipefs -a /dev/vdc
     wipefs -a /dev/vdd
     ```
   - **Purpose**: Clear all metadata from test disks (Questions 30-32).
   - **Output**: `blkid /dev/vdb /dev/vdc /dev/vdd` empty.
   - **Bookmarks**:
     - Use `wipefs -a` to clear metadata (Questions 30, 32).
   - **Good Practice**:
     - Use `wipefs -a`; verify with `blkid`, `lsblk`.

6. **Final Verification**:
   - **Command**:
     ```bash
     lsblk
     blkid
     cat /etc/fstab
     df -h
     free -h
     swapon --show
     ```
   - **Purpose**: Confirm clean state (Questions 23, 24, 29, 33, 96, 100).
   - **Output**:
     - `lsblk`: Shows `/dev/vda` with system partitions; `/dev/vdb`, `/dev/vdc`, `/dev/vdd` empty.
     - `blkid`: No filesystems on `/dev/vdb`, `/dev/vdc`, `/dev/vdd`.
     - `/etc/fstab`: Only system mounts.
     - `df -h`: No test mounts.
     - `free -h`: No swap usage.
   - **Bookmarks**:
     - Verify lab completion with `lsblk`, `blkid`, `/etc/fstab`, snapshots (All Questions).
   - **Good Practice**:
     - Verify with `lsblk`, `blkid`, `df -h` at lab end.

## Configuration Files

- **/etc/mdadm/mdadm.conf** (RAID Configuration, Question 28):

  ```bash
  # Initial state (after RAID creation)
  ARRAY /dev/md0 metadata=1.2 name=usv:0 UUID=...
  ```

  - **Purpose**: Persists RAID array configuration for boot-time assembly.
  - **Bookmarks**:
    - Save RAID config to `/etc/mdadm/mdadm.conf` for boot-time assembly (Question 28).
  - **Good Practice**:
    - Save and verify RAID config.

- **/etc/fstab** (Mounts and Swap, Questions 83, 94):
  ```bash
  # Initial state (system mounts)
  /dev/mapper/ubuntu--vg-ubuntu--lv / ext4 defaults 0 0
  UUID=C6B8-2DBC /boot/efi vfat defaults 0 0
  UUID=910ca4fc-68bb-4d17-a6ad-faa654339112 /boot ext4 defaults 0 0
  # Added during lab
  /swap.img none swap sw 0 0
  /swapfile none swap sw 0 0
  ```
  - **Purpose**: Defines persistent mounts and swap areas.
  - **Note**: `/swap.img` and `/swapfile` added for swap (removed during cleanup).
  - **Bookmarks**:
    - Swap entries in `/etc/fstab` persist swap; verify with `swapon --show` (Question 94).
  - **Good Practices**:
    - Verify swap in `/etc/fstab`.
    - Clean `/etc/fstab` at lab end.

## Questions Addressed

- **Question 23**: Use `lsblk` to list block devices and their hierarchy.
- **Question 24**: Use `blkid` to display filesystem UUIDs and types.
- **Question 25**: Manage LVM (PV, VG, LV creation with `pvcreate`, `vgcreate`, `lvcreate`) and RAID (creation with `mdadm --create`).
- **Question 27**: Mount filesystems (`mount`, `/mnt/test`, `/mnt/raid`, `/mnt/backup`, `/mnt/xfs_test`).
- **Question 28**: Extend/reduce LVM (`lvextend`, `lvreduce`, `resize2fs`, `xfs_growfs`); configure RAID 1 (`mdadm --create`, `--level=1`).
- **Question 29**: Use `lsblk`, `blkid` for device information.
- **Question 30**: Partition disks with `parted`; wipe with `wipefs -a`.
- **Question 31**: Create partitions with `parted mkpart`; rescan with `partprobe`.
- **Question 32**: Format filesystems with `mkfs.ext4`, `mkfs.xfs`; clean with `wipefs -a`.
- **Question 33**: Use `lsblk`, `blkid` for verification.
- **Question 34**: Handle full filesystems with `df -h`, `lsof`, `kill -9`.
- **Question 62**: Create disk images with `dd` (e.g., `dd if=/dev/vdb1`).
- **Question 66**: Recover filesystems by restoring `dd` images.
- **Question 68**: Back up partition tables with `dd bs=512 count=1`.
- **Question 81**: Archive files with `tar -czvf`; clean logs with `find ... -delete`.
- **Question 83**: Manage `/etc/fstab` for mounts and swap.
- **Question 90**: Use `lsblk`, `blkid` for device management.
- **Question 94**: Create swap with `fallocate`, `mkswap`, `swapon`.
- **Question 96**: Verify configurations with `lsblk`, `blkid`.
- **Question 100**: Use `lsblk`, `blkid` for system verification.

## Bookmarks

- **B1**: `pvcreate` initializes a partition for LVM; verify with `pvs` (Question 25).
- **B2**: `vgcreate` groups PVs for logical volume allocation (Question 25).
- **B3**: `lvcreate -L` specifies LV size; `-n` sets the name (Question 25).
- **B4**: `mkfs.ext4` creates an ext4 filesystem; mount with `mkdir` and `mount` (Questions 27, 32).
- **B5**: Use `lvextend -L +<size>` to grow LVs; `resize2fs` for ext4 filesystems (Question 28).
- **B6**: Before reducing, unmount, check with `e2fsck -f`, resize with `resize2fs`, then `lvreduce` (Question 28).
- **B7**: Remove LVM with `lvremove`, `vgremove`, `pvremove` in that order (Questions 30, 32).
- **B8**: `mdadm --create --level=1` sets up RAID 1; check with `cat /proc/mdstat` (Questions 25, 28).
- **B9**: Format RAID with `mkfs.ext4`; mount to test (Questions 27, 32).
- **B10**: Use `mdadm -f` to simulate RAID failure; verify with `mdadm --detail` (Question 28).
- **B11**: Use `mdadm -r` to remove, `mdadm -a` to add disks; monitor with `cat /proc/mdstat` (Question 28).
- **B12**: Use `wipefs -a` to clear metadata before LVM or RAID setup (Questions 30, 32).
- **B13**: Save RAID config to `/etc/mdadm/mdadm.conf` for boot-time assembly (Question 28).
- **B14**: Use `mdadm --stop` and `--zero-superblock` to clean RAID (Questions 30-32).
- **B29**: `parted mkpart ext4` sets partition type, not filesystem; use `mkfs.ext4` or `mkfs.xfs` (Questions 30, 32).
- **B30**: Rescan with `partprobe` if partitions don’t appear; check `lsblk`, `blkid` (Questions 30, 31).
- **B31**: `dd` creates disk images (`bs=4M`), `tar -czvf` archives files, `fallocate` creates swap; verify with `file`, `tar -tzvf`, `swapon --show` (Questions 62, 81, 94).
- **B32**: `tar -C` sets extraction directory; check paths with `tar -tzvf` (Question 81).
- **B33**: "No space left" errors require `df -h`, `rm`, `lsof | grep deleted` (Question 34).
- **B34**: Avoid nested `tar` paths with `cd <dir>; tar -czvf <archive> .` or `--strip-components` (Question 81).
- **B35**: Archive logs with `tar -czvf`; clean with `find ... -delete` (Question 81).
- **B36**: MBR backup shows PMBR (`55 aa`, type `ee`) for GPT; use `count=34` for full GPT (Question 68).
- **B37**: `tar` fails if directory empty; check `ls` first (Question 81).
- **B38**: Simulate full filesystem with `dd`; free held files with `lsof`, `kill -9` (Question 34).
- **B39**: Use `cd <dir>; tar -czvf <archive> .` or `--strip-components` to avoid nested paths (Question 81).
- **B40**: `tail -f <file> &` holds files open; use `lsof` and `kill -9` to free space (Question 34).
- **B41**: Swap size sums all active areas; check with `swapon --show` (Question 94).
- **B42**: Verify lab completion with `lsblk`, `blkid`, `/etc/fstab`, snapshots (All Questions).
- **B43**: Always run `resize2fs` before `lvreduce` when shrinking an LV to prevent data loss; unmount and check with `e2fsck -f` first (Question 28).
- **B44**: Use `mkfs.xfs` to create an XFS filesystem; verify with `blkid` (Question 32).
- **B45**: Mount XFS filesystems with `mount`; test with read/write operations (Questions 27, 32).
- **B46**: Use `xfs_growfs` to resize XFS filesystems online after `lvextend`; XFS does not support shrinking (Question 28).
- **B47**: Use `xfs_info` to inspect XFS filesystem details; verify LV with `lvs` (Questions 23, 29, 96).

## Good Practices

- **GP1**: Flag partitions for LVM with `set 1 lvm on` in `parted` (Section 1).
- **GP2**: Verify LVM setup with `pvs`, `vgs`, `lvs` (Sections 1, 2).
- **GP5**: Unmount before modifying LVM setup (Sections 1, 2).
- **GP6**: Test LVM/RAID mounts with file read/write (Sections 1, 2, 3, 5).
- **GP7**: Check RAID status with `mdadm --detail`, `cat /proc/mdstat` (Section 3).
- **GP8**: Run `resize2fs` after `lvextend` or before `lvreduce` for ext4; use `xfs_growfs` for XFS; verify with `df -h` (Sections 1, 2).
- **GP9**: Verify VG/PV removal with `vgs`, `pvs`, `blkid` (Sections 1, 2).
- **GP10**: Check partition tables with `parted <disk> print` (Section 1).
- **GP11**: Test RAID integrity; save config to `/etc/mdadm/mdadm.conf` (Section 3).
- **GP12**: Use `wipefs -a` before RAID/LVM; verify with `blkid`, `lsblk` (Sections 1, 2, 3, 4, 5).
- **GP13**: Verify RAID data integrity post-failure; save snapshot (Section 3).
- **GP14**: Use `dd` with `bs=4M`, `tar -cvf` for backups; verify restores (Section 5).
- **GP15**: Rescan disks with `partprobe` after `parted`; verify with `lsblk`, `blkid` (Sections 1, 2, 5).
- **GP16**: Verify backups with `file`, `tar -tzvf`; test swap with `swapon --show` (Section 5).
- **GP17**: Use `lsof | grep deleted` for full filesystems; archive logs with `tar -czvf` (Section 6).
- **GP18**: Check `ls` before `tar`; verify swap in `/etc/fstab` (Section 5).
- **GP19**: Monitor `df -h` for disk usage; clean logs after archiving (Section 6).
- **GP20**: At lab end, unmount, disable swap, remove test files, verify with `lsblk`, `blkid`, `df -h` (Cleanup).
- **GP21**: When reducing an LV, unmount, run `e2fsck -f`, `resize2fs`, then `lvreduce` in that order to avoid filesystem corruption (Section 1, Question 28).
- **GP22**: Verify XFS filesystems with `xfs_info`, `df -h`, `lsblk`; use `xfs_growfs` for resizing (Section 2, Questions 27, 28, 32).

## Final Notes

This lab provided hands-on experience with Linux storage management, covering LVM (with ext4 and XFS), RAID, backups, and disk space issues. Your insights about `resize2fs`/`lvreduce` and the need for XFS experience highlight critical skills for Linux+ certification. Including XFS ensures comprehensive coverage of filesystem objectives (Questions 27, 28, 32). All tasks were completed, and cleanup ensures a reset system. Save a final UTM snapshot named "Lab-Final" and review outputs to confirm readiness for the XK0-005 exam.
