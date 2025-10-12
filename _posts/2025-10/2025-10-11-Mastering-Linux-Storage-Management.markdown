---
layout: post
title: "Linux Lab Summary: Mastering Linux Storage Management"
date: 2025-10-11 10:10:00 -0700
tags:
  [Linux+, Linuxlab, storage, LVM, RAID, backups, parted, mdadm, wipefs, tar]
---

This document summarizes the CompTIA Linux+ Filesystem Management Lab, covering Logical Volume Manager (LVM), RAID, backups, and disk space management. It details the environment, steps, outputs, addressed questions, configuration files, insights, bookmarks, and good practices, ensuring a beginner-friendly guide for Linux+ certification preparation (XK0-005 objectives).

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
  - `/dev/vdb`, `/dev/vdc`, `/dev/vdd`: 10GB each, used for LVM, RAID, and backups
  - `/dev/sr0`: 2.8GB, iso9660 (Ubuntu installation media)
- **Tools Used**: `mdadm`, `parted`, `wipefs`, `mkfs.ext4`, `dd`, `tar`, `fallocate`, `lsof`, `lsblk`, `blkid`, `partprobe`

## Lab Objectives

The lab covers CompTIA Linux+ objectives for filesystem and storage management, including:

- **LVM** (Questions 25, 27-28, 30-32): Creating, extending, reducing, and removing LVM setups.
- **RAID** (Questions 25, 28, 30-32): Configuring RAID 1, handling failures, and cleanup.
- **Backups** (Questions 62, 68, 81, 94): Using `dd` for disk images, `tar` for file archives, and setting up swap.
- **Disk Space Issues** (Questions 34, 81): Managing full filesystems and log archiving.
- **Filesystem Management** (Questions 23, 24, 29, 30-32, 33, 66, 83, 90, 96, 100): Partitioning, formatting, mounting, and cleanup.

## Hands-On Steps, Purposes, Outputs, Bookmarks, and Good Practices

### Section 1: Logical Volume Manager (LVM) Setup

**Purpose**: Learn to create, extend, reduce, and remove LVM volumes (Questions 25, 27-28, 30-32).

1. **Wipe Disk**:

   - **Command**: `wipefs -a /dev/vdc`
   - **Purpose**: Clear any existing filesystem or RAID metadata to prepare `/dev/vdc` for LVM (Question 30).
   - **Output**: Removed ext4, GPT, or RAID signatures.
   - **Bookmark**: Use `wipefs -a` to clear metadata before LVM or RAID setup to avoid conflicts (Questions 30, 32).
   - **Good Practice**: Always use `wipefs -a` before LVM/RAID setup; verify with `blkid` and `lsblk`.

2. **Partition Disk**:

   - **Command**: `parted /dev/vdc mklabel gpt mkpart primary ext4 1MiB 100%; parted /dev/vdc set 1 lvm on`
   - **Purpose**: Create a GPT partition table and a single partition (`/dev/vdc1`) flagged for LVM (Questions 30, 31).
   - **Output**: Partition created; `lsblk` shows `/dev/vdc1`.
   - **Bookmark**: In `parted mkpart`, the `ext4` parameter sets a partition type hint, not the filesystem; use `mkfs.ext4` to create the filesystem (Questions 30, 32).
   - **Good Practice**: Flag partitions for LVM with `set 1 lvm on` in `parted`.
   - **Good Practice**: Check partition tables with `parted <disk> print`.

3. **Create Physical Volume (PV)**:

   - **Command**: `pvcreate /dev/vdc1`
   - **Purpose**: Initialize `/dev/vdc1` as an LVM PV (Question 25).
   - **Output**: `Physical volume "/dev/vdc1" successfully created.`
   - **Bookmark**: `pvcreate` initializes a partition for LVM; verify with `pvs` (Question 25).
   - **Good Practice**: Verify LVM setup with `pvs`, `vgs`, `lvs`.

4. **Create Volume Group (VG)**:

   - **Command**: `vgcreate my_vg /dev/vdc1`
   - **Purpose**: Group PVs into a VG named `my_vg` (Question 25).
   - **Output**: `Volume group "my_vg" successfully created.`
   - **Bookmark**: `vgcreate` groups PVs for logical volume allocation (Question 25).
   - **Good Practice**: Verify with `vgs`.

5. **Create Logical Volume (LV)**:

   - **Command**: `lvcreate -L 5G -n my_lv my_vg`
   - **Purpose**: Create a 5GB LV named `my_lv` in `my_vg` (Question 25).
   - **Output**: `Logical volume "my_lv" created.`
   - **Bookmark**: `lvcreate -L` specifies LV size; `-n` sets the name (Question 25).
   - **Good Practice**: Verify with `lvs`.

6. **Format and Mount LV**:

   - **Command**: `mkfs.ext4 /dev/my_vg/my_lv; mkdir /mnt/test; mount /dev/my_vg/my_lv /mnt/test`
   - **Purpose**: Format LV as ext4, mount at `/mnt/test` for use (Questions 27, 32).
   - **Output**: Filesystem created; `df -h /mnt/test` shows ~5GB.
   - **Bookmark**: `mkfs.ext4` creates an ext4 filesystem; mount with `mkdir` and `mount` (Questions 27, 32).
   - **Good Practice**: Unmount before modifying LVM setup.
   - **Good Practice**: Test mounts with file read/write (e.g., `echo "test" > /mnt/test/file.txt`).

7. **Extend LV**:

   - **Command**: `lvextend -L +2G /dev/my_vg/my_lv; resize2fs /dev/my_vg/my_lv`
   - **Purpose**: Increase LV size by 2GB to 7GB, resize filesystem (Question 28).
   - **Output**: `lvextend` extends LV; `resize2fs` expands filesystem; `df -h` shows ~7GB.
   - **Bookmark**: Use `lvextend -L +<size>` to grow LVs; `resize2fs` for ext4 filesystems (Question 28).
   - **Good Practice**: Run `resize2fs` after `lvextend`; verify with `df -h`.

8. **Reduce LV**:

   - **Command**: `umount /mnt/test; e2fsck -f /dev/my_vg/my_lv; resize2fs /dev/my_vg/my_lv 4G; lvreduce -L 4G /dev/my_vg/my_lv`
   - **Purpose**: Shrink filesystem to 4GB, reduce LV to match (Question 28).
   - **Output**: Filesystem checked and resized; LV reduced; `lvs` shows 4GB.
   - **Bookmark**: Before reducing, unmount, check with `e2fsck -f`, resize with `resize2fs`, then `lvreduce` (Question 28).
   - **Good Practice**: Unmount before reducing LVs.

9. **Remove LV, VG, PV**:
   - **Command**: `umount /mnt/test; lvremove /dev/my_vg/my_lv; vgremove my_vg; pvremove /dev/vdc1`
   - **Purpose**: Clean up LVM setup (Questions 30, 32).
   - **Output**: LV, VG, PV removed; `pvs`, `vgs`, `lvs` show no entries.
   - **Bookmark**: Remove LVM with `lvremove`, `vgremove`, `pvremove` in that order (Questions 30, 32).
   - **Good Practice**: Verify removal with `vgs`, `pvs`, `blkid`.

### Section 2: RAID Configuration

**Purpose**: Set up RAID 1, simulate failure, replace disk, and clean up (Questions 25, 28, 30-32).

1. **Wipe Disks**:

   - **Command**: `wipefs -a /dev/vdb /dev/vdc`
   - **Purpose**: Clear metadata for RAID setup (Question 30).
   - **Output**: Signatures erased; `blkid /dev/vdb /dev/vdc` empty.
   - **Bookmark**: Use `wipefs -a` to clear metadata before LVM or RAID setup (Questions 30, 32).
   - **Good Practice**: Use `wipefs -a` before RAID/LVM; verify with `blkid`, `lsblk`.

2. **Create RAID 1**:

   - **Command**: `mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdb /dev/vdc`
   - **Purpose**: Create RAID 1 array (mirroring) on `/dev/vdb` and `/dev/vdc` (Questions 25, 28).
   - **Output**: `mdadm: array /dev/md0 started.`; `cat /proc/mdstat` shows sync.
   - **Bookmark**: `mdadm --create --level=1` sets up RAID 1; check with `cat /proc/mdstat` (Questions 25, 28).
   - **Good Practice**: Check RAID status with `mdadm --detail /dev/md0`, `cat /proc/mdstat`.

3. **Format and Mount RAID**:

   - **Command**: `mkfs.ext4 /dev/md0; mkdir /mnt/raid; mount /dev/md0 /mnt/raid`
   - **Purpose**: Format RAID as ext4, mount for use (Questions 27, 32).
   - **Output**: Filesystem created; `df -h /mnt/raid` shows ~10GB.
   - **Bookmark**: Format RAID with `mkfs.ext4`; mount to test (Questions 27, 32).
   - **Good Practice**: Test mounts with file read/write.

4. **Simulate Disk Failure**:

   - **Command**: `mdadm /dev/md0 -f /dev/vdc`
   - **Purpose**: Mark `/dev/vdc` as failed to test RAID resilience (Question 28).
   - **Output**: `mdadm: set /dev/vdc faulty in /dev/md0`; `mdadm --detail /dev/md0` shows degraded.
   - **Bookmark**: Use `mdadm -f` to simulate RAID failure; verify with `mdadm --detail` (Question 28).
   - **Good Practice**: Check RAID status post-failure.

5. **Replace Failed Disk**:

   - **Command**: `mdadm /dev/md0 -r /dev/vdc; mdadm /dev/md0 -a /dev/vdd`
   - **Purpose**: Remove failed `/dev/vdc`, add `/dev/vdd` to rebuild (Question 28).
   - **Output**: `mdadm: hot removed /dev/vdc`; `mdadm: hot added /dev/vdd`; `cat /proc/mdstat` shows rebuilding.
   - **Bookmark**: Use `mdadm -r` to remove, `mdadm -a` to add disks; monitor with `cat /proc/mdstat` (Question 28).
   - **Good Practice**: Verify RAID data integrity post-failure; save snapshot.

6. **Save RAID Config**:

   - **Command**: `mdadm --detail --scan >> /etc/mdadm/mdadm.conf`
   - **Purpose**: Persist RAID configuration (Question 28).
   - **Output**: Array details appended to `/etc/mdadm/mdadm.conf`.
   - **Bookmark**: Save RAID config to `/etc/mdadm/mdadm.conf` for boot-time assembly (Question 28).
   - **Good Practice**: Test RAID integrity; save config to `/etc/mdadm/mdadm.conf`.

7. **Clean Up RAID**:
   - **Command**: `umount /mnt/raid; mdadm --stop /dev/md0; mdadm --zero-superblock /dev/vdb /dev/vdc /dev/vdd`
   - **Purpose**: Dismantle RAID, clear metadata (Questions 30-32).
   - **Output**: Array stopped; `blkid` shows no RAID metadata.
   - **Bookmark**: Use `mdadm --stop` and `--zero-superblock` to clean RAID (Questions 30-32).
   - **Good Practice**: Verify cleanup with `blkid`, `lsblk`.

### Section 3: Disk Cleanup

**Purpose**: Ensure disks are clean for backup tasks (Questions 30-32).

1. **Wipe RAID Metadata**:
   - **Command**: `mdadm --zero-superblock /dev/vdc`
   - **Purpose**: Clear RAID superblock to prepare `/dev/vdc` (Question 30).
   - **Output**: No RAID metadata; `blkid /dev/vdc` empty.
   - **Bookmark**: `mdadm --zero-superblock` removes RAID metadata (Questions 30-32).
   - **Good Practice**: Use `wipefs -a` or `mdadm --zero-superblock`; verify with `blkid`, `lsblk`.

### Section 4: Backups

**Purpose**: Create disk images, file archives, partition table backups, and swap (Questions 62, 68, 81, 94).

1. **Wipe and Partition Disk**:

   - **Command**: `wipefs -a /dev/vdb; parted /dev/vdb mklabel gpt mkpart primary ext4 1MiB 100%`
   - **Purpose**: Clear `/dev/vdb`, create GPT and `/dev/vdb1` for backups (Questions 30-32).
   - **Output**: Signatures erased; `lsblk` shows `/dev/vdb1` (~10GB).
   - **Issue**: Initially, `/dev/vdb1` not detected; fixed with `partprobe /dev/vdb`.
   - **Bookmark**: `parted mkpart ext4` sets partition type; use `mkfs.ext4` for filesystem (Questions 30, 32).
   - **Bookmark**: Rescan with `partprobe` if partitions don’t appear; check `lsblk`, `blkid` (Questions 30, 31).
   - **Good Practice**: Use `wipefs -a` before setup; verify with `blkid`, `lsblk`.
   - **Good Practice**: Rescan disks with `partprobe` after `parted`; verify with `lsblk`, `blkid`.

2. **Format and Mount**:

   - **Command**: `mkfs.ext4 /dev/vdb1; mkdir -p /mnt/backup; mount /dev/vdb1 /mnt/backup`
   - **Purpose**: Format `/dev/vdb1` as ext4, mount for backup storage (Questions 27, 32).
   - **Output**: Filesystem created; `df -h /mnt/backup` shows ~9.8GB; `blkid /dev/vdb1` shows UUID (e.g., 079fd50b-45c4-4e3a-8b4b-00da75617509).
   - **Bookmark**: `mkfs.ext4` creates filesystem; mount with `mkdir` and `mount` (Questions 27, 32).
   - **Good Practice**: Test mounts with file read/write (e.g., `echo "Backup test" > /mnt/backup/test.txt`).

3. **Create Test Files**:

   - **Command**: `echo "Backup test" > /mnt/backup/test.txt; mkdir /mnt/backup/data; echo "Data file" > /mnt/backup/data/data.txt`
   - **Purpose**: Add files to test backups (Question 81).
   - **Output**: `ls -R /mnt/backup` shows `test.txt`, `data/data.txt`.
   - **Bookmark**: Test backups with sample files; verify with `ls -R` (Question 81).
   - **Good Practice**: Test mounts with file read/write.

4. **Disk Image Backup with `dd`**:

   - **Command**: `dd if=/dev/vdb1 of=/root/vd1_backup.img bs=4M status=progress`
   - **Purpose**: Create a bit-for-bit image of `/dev/vdb1` (Question 62).
   - **Output**: Copied ~8.3GB, stopped due to root filesystem full; `ls -lh /root/vd1_backup.img` shows 8.3GB; `file` confirms ext4.
   - **Issue**: Root filesystem (`/`) filled (14.5GB LV); fixed by `rm /root/vd1_backup.img`.
   - **Bookmark**: `dd` creates disk images with `bs=4M` for efficiency; verify with `file` (Question 62).
   - **Bookmark**: "No space left" errors require `df -h`, `rm`, `lsof | grep deleted` (Question 34).
   - **Good Practice**: Use `dd` with `bs=4M` for backups; verify restores.
   - **Good Practice**: Use `lsof | grep deleted` for full filesystems.

5. **Restore and Verify `dd` Backup**:

   - **Command**: `dd if=/root/vd1_backup.img of=/dev/vdc bs=4M status=progress; mkdir /mnt/vdc-restore; mount /dev/vdc /mnt/vdc-restore`
   - **Purpose**: Restore image to `/dev/vdc`, verify data (Question 62).
   - **Output**: Restored ~8.3GB; `ls -R /mnt/vdc-restore` shows `test.txt`, `data/data.txt`.
   - **Bookmark**: Test `dd` restores by mounting and checking files (Question 62).
   - **Good Practice**: Verify restores with `ls -R`.

6. **File Archive with `tar`**:

   - **Command**: `cd /mnt/backup; tar -czvf /root/backup.tar.gz .`
   - **Purpose**: Archive `/mnt/backup` contents (Question 81).
   - **Output**: Archived `test.txt`, `data/data.txt`; `tar -tzvf /root/backup.tar.gz` lists files.
   - **Issue**: Initial `tar -czvf /root/backup.tar.gz /mnt/backup` caused nested paths (`mnt/backup/`); fixed by using `cd` and `.`.
   - **Bookmark**: `tar -C` sets extraction directory; check paths with `tar -tzvf` (Question 81).
   - **Bookmark**: Avoid nested `tar` paths with `cd <dir>; tar -czvf <archive> .` or `--strip-components` (Question 81).
   - **Bookmark**: `tar` fails if directory empty; check `ls` first (Question 81).
   - **Bookmark**: Use `cd <dir>; tar -czvf <archive> .` or `--strip-components` to avoid nested paths (Question 81).
   - **Good Practice**: Verify `tar` backups with `tar -tzvf`.
   - **Good Practice**: Check `ls` before `tar`.

7. **Test `tar` Restore**:

   - **Command**: `rm -rf /mnt/backup/*; tar -xzvf /root/backup.tar.gz -C /mnt/backup/`
   - **Purpose**: Restore and verify files (Question 81).
   - **Output**: `ls -R /mnt/backup` shows `test.txt`, `data/data.txt`.
   - **Bookmark**: Use `tar -xzvf -C` for restores; verify with `ls -R` (Question 81).
   - **Good Practice**: Test restores with `ls -R`.

8. **Partition Table Backup**:

   - **Command**: `dd if=/dev/vdb of=/root/vdb_mbr.bak bs=512 count=1`
   - **Purpose**: Back up GPT protective MBR (Question 68).
   - **Output**: 512 bytes copied; `ls -lh /root/vdb_mbr.bak` shows 512B; `hexdump -C` shows `55 aa` (PMBR signature), `ee` (GPT partition type).
   - **Issue**: Failed due to full root filesystem; fixed by freeing space.
   - **Bookmark**: `dd bs=512 count=1` backs up MBR/PMBR; use `count=34` for full GPT (Question 68).
   - **Bookmark**: MBR backup shows PMBR (`55 aa`, type `ee`) for GPT; use `count=34` for full GPT (Question 68).
   - **Good Practice**: Use `dd` for partition table backups; verify with `hexdump -C`.

9. **Swap File Setup**:
   - **Command**: `fallocate -l 1G /swapfile; chmod 600 /swapfile; mkswap /swapfile; swapon /swapfile; echo '/swapfile none swap sw 0 0' >> /etc/fstab`
   - **Purpose**: Create and activate 1GB swap file, persist in `/etc/fstab` (Question 94).
   - **Output**: `swapon --show` lists `/swapfile` (1GB) and `/swap.img` (3GB); `free -h` shows 4GB total swap, ~12KiB used.
   - **Note**: Total swap is 4GB due to existing `/swap.img`.
   - **Bookmark**: `fallocate` creates swap files efficiently; verify with `swapon --show` (Question 94).
   - **Bookmark**: Swap size sums all active areas; check with `swapon --show` (Question 94).
   - **Good Practice**: Verify swap with `swapon --show`.
   - **Good Practice**: Verify swap in `/etc/fstab`.

### Section 5: Disk Space Issues

**Purpose**: Simulate and resolve full filesystem issues, archive logs (Questions 34, 81).

1. **Simulate Full Filesystem**:

   - **Command**: `dd if=/dev/zero of=/mnt/backup/large.log bs=1M count=9000`
   - **Purpose**: Fill `/mnt/backup` (~9GB) to simulate full filesystem (Question 34).
   - **Output**: Copied 9.4GB; `df -h /mnt/backup` shows 96% usage (450M available).
   - **Bookmark**: Simulate full filesystem with `dd`; check with `df -h` (Question 34).
   - **Good Practice**: Monitor disk usage with `df -h` during large operations.

2. **Simulate Held File**:

   - **Command**: `tail -f /mnt/backup/large.log &`
   - **Purpose**: Hold file open to prevent space reclamation (Question 34).
   - **Output**: Background process started (PID 4302).
   - **Bookmark**: `tail -f <file> &` holds files open; use `lsof` and `kill -9` to free space (Question 34).
   - **Good Practice**: Use `lsof | grep deleted` for full filesystems.

3. **Delete File and Check**:

   - **Command**: `rm /mnt/backup/large.log; df -h /mnt/backup`
   - **Purpose**: Delete file, check if space is freed (Question 34).
   - **Output**: Still 96% usage due to `tail` holding file.
   - **Bookmark**: "No space left" requires `df -h`, `rm`, `lsof | grep deleted` (Question 34).
   - **Good Practice**: Use `lsof | grep deleted` for held files.

4. **Free Space**:

   - **Command**: `lsof | grep large.log; kill -9 4302; df -h /mnt/backup`
   - **Purpose**: Identify and kill process holding file, free space (Question 34).
   - **Output**: `lsof` shows `tail` (PID 4302) holding `large.log (deleted)`; after `kill -9`, `df -h` shows 1% usage (9.3G free).
   - **Bookmark**: Free held files with `lsof` and `kill -9` (Question 34).
   - **Bookmark**: `tail -f &` holds files; use `lsof` and `kill -9` (Question 34).
   - **Good Practice**: Use `lsof | grep deleted` and `kill -9` to free space.

5. **Archive Logs**:

   - **Command**: `tar -czvf /mnt/backup/log-backup.tar.gz /var/log`
   - **Purpose**: Archive system logs for backup (Question 81).
   - **Output**: Archived `/var/log` (e.g., `auth.log`, `syslog`); `tar -tzvf /mnt/backup/log-backup.tar.gz` lists files.
   - **Bookmark**: Archive logs with `tar -czvf` (Question 81).
   - **Good Practice**: Archive logs with `tar -czvf`.
   - **Good Practice**: Clean logs after archiving.

6. **Clean Logs**:
   - **Command**: `find /var/log -type f -name "*.log" -delete`
   - **Purpose**: Remove log files to free space (Question 81).
   - **Output**: `.log` files deleted; `ls /var/log` shows remaining files.
   - **Bookmark**: Clean logs with `find ... -delete` after archiving (Question 81).
   - **Good Practice**: Clean logs after archiving to save space.

### Cleanup Steps

**Purpose**: Reset system to clean state (Questions 30-32, 66, 83).

1. **Unmount Filesystems**:

   - **Command**: `umount /mnt/backup /mnt/vdc-restore`
   - **Purpose**: Unmount test filesystems (Question 27).
   - **Output**: No mounts shown in `df -h`.
   - **Good Practice**: Unmount filesystems at lab end; verify with `df -h`.

2. **Disable Swap**:

   - **Command**: `swapoff /swapfile; swapoff /swap.img`
   - **Purpose**: Deactivate swap files (Question 94).
   - **Output**: `swapon --show` shows no active swap.
   - **Good Practice**: Disable swap at lab end.

3. **Remove Test Files**:

   - **Command**: `rm /mnt/backup/* /root/vdb_mbr.bak /root/backup.tar.gz /swapfile /swap.img`
   - **Purpose**: Delete temporary files (Question 66).
   - **Output**: `ls /mnt/backup /root` empty; `ls /swapfile /swap.img` fails.
   - **Good Practice**: Remove test files at lab end.

4. **Clean `/etc/fstab`**:

   - **Command**: `vim /etc/fstab` (remove `/swapfile`, `/swap.img` entries)
   - **Purpose**: Reset persistent mounts and swap (Questions 83, 94).
   - **Output**: `/etc/fstab` contains only system mounts.
   - **Good Practice**: Verify `/etc/fstab` at lab end.

5. **Wipe Disks**:

   - **Command**: `wipefs -a /dev/vdb /dev/vdc /dev/vdd`
   - **Purpose**: Clear all metadata from test disks (Questions 30-32).
   - **Output**: `blkid /dev/vdb /dev/vdc /dev/vdd` empty.
   - **Bookmark**: Use `wipefs -a` to clear metadata (Questions 30, 32).
   - **Good Practice**: Use `wipefs -a`; verify with `blkid`, `lsblk`.

6. **Final Verification**:
   - **Command**: `lsblk; blkid; cat /etc/fstab; df -h; free -h; swapon --show`
   - **Purpose**: Confirm clean state (Questions 23, 24, 29, 33, 96, 100).
   - **Output**:
     - `lsblk`: Shows `/dev/vda` with system partitions; `/dev/vdb`, `/dev/vdc`, `/dev/vdd` empty.
     - `blkid`: No filesystems on `/dev/vdb`, `/dev/vdc`, `/dev/vdd`.
     - `/etc/fstab`: Only system mounts.
     - `df -h`: No test mounts.
     - `free -h`: No swap usage.
   - **Bookmark**: Verify lab completion with `lsblk`, `blkid`, `/etc/fstab`, snapshots (All Questions).
   - **Good Practice**: Verify with `lsblk`, `blkid`, `df -h` at lab end.

## Configuration Files

- **/etc/mdadm/mdadm.conf** (RAID Configuration, Question 28):

  ```bash
  # Initial state (after RAID creation)
  ARRAY /dev/md0 metadata=1.2 name=usv:0 UUID=...
  ```

  - **Purpose**: Persists RAID array configuration for boot-time assembly.
  - **Bookmark**: Save RAID config to `/etc/mdadm/mdadm.conf` (Question 28).
  - **Good Practice**: Save and verify RAID config.

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
  - **Bookmark**: Swap entries in `/etc/fstab` persist swap; verify with `swapon --show` (Question 94).
  - **Good Practice**: Verify swap in `/etc/fstab`.
  - **Good Practice**: Clean `/etc/fstab` at lab end.

## Questions Addressed

- **Question 23**: Use `lsblk` to list block devices and their hierarchy.
- **Question 24**: Use `blkid` to display filesystem UUIDs and types.
- **Question 25**: Manage LVM (PV, VG, LV creation with `pvcreate`, `vgcreate`, `lvcreate`) and RAID (creation with `mdadm --create`).
- **Question 27**: Mount filesystems (`mount`, `/mnt/test`, `/mnt/raid`, `/mnt/backup`).
- **Question 28**: Extend/reduce LVM (`lvextend`, `lvreduce`, `resize2fs`); configure RAID 1 (`mdadm --create`, `--level=1`).
- **Question 29**: Use `lsblk`, `blkid` for device information.
- **Question 30**: Partition disks with `parted`; wipe with `wipefs -a`.
- **Question 31**: Create partitions with `parted mkpart`; rescan with `partprobe`.
- **Question 32**: Format filesystems with `mkfs.ext4`; clean with `wipefs -a`.
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
- **B15**: Use `lsblk` to visualize disk layout, including partitions, LVM, RAID, and mounts (Questions 23, 24, 29, 33, 96, 100).
- **B29**: `parted mkpart ext4` sets partition type, not filesystem; use `mkfs.ext4` (Questions 30, 32).
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

## Good Practices

- **GP1**: Flag partitions for LVM with `set 1 lvm on` in `parted` (Section 1).
- **GP2**: Verify LVM setup with `pvs`, `vgs`, `lvs` (Section 1).
- **GP5**: Unmount before modifying LVM setup (Section 1).
- **GP6**: Test LVM/RAID mounts with file read/write (Sections 1, 2, 4).
- **GP7**: Check RAID status with `mdadm --detail`, `cat /proc/mdstat` (Section 2).
- **GP8**: Run `resize2fs` after `lvextend`; verify with `df -h` (Section 1).
- **GP9**: Verify VG/PV removal with `vgs`, `pvs`, `blkid` (Section 1).
- **GP10**: Check partition tables with `parted <disk> print` (Section 1).
- **GP11**: Test RAID integrity; save config to `/etc/mdadm/mdadm.conf` (Section 2).
- **GP12**: Use `wipefs -a` before RAID/LVM; verify with `blkid`, `lsblk` (Sections 1, 2, 3, 4).
- **GP13**: Verify RAID data integrity post-failure; save snapshot (Section 2).
- **GP14**: Use `dd` with `bs=4M`, `tar -cvf` for backups; verify restores (Section 4).
- **GP15**: Rescan disks with `partprobe` after `parted`; verify with `lsblk`, `blkid` (Section 4).
- **GP16**: Verify backups with `file`, `tar -tzvf`; test swap with `swapon --show` (Section 4).
- **GP17**: Use `lsof | grep deleted` for full filesystems; archive logs with `tar -czvf` (Section 5).
- **GP18**: Check `ls` before `tar`; verify swap in `/etc/fstab` (Section 4).
- **GP19**: Monitor `df -h` for disk usage; clean logs after archiving (Section 5).
- **GP20**: At lab end, unmount, disable swap, remove test files, verify with `lsblk`, `blkid`, `df -h` (Cleanup).
