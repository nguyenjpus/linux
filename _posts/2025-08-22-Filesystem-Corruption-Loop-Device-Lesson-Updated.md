# Lesson Learned: Filesystem Corruption with Clean Slate Setup in Ubuntu

## Overview

This document summarizes a hands-on exercise (conducted on August 23, 2025, in an Ubuntu VM, kernel 6.14.0-28-generic, aarch64, running in UTM) to simulate and attempt repair of filesystem corruption on a 100 MB ext4 image (`test.img`). It includes testing `fsck.ext4` on a healthy filesystem, clarifying `e2fsck`, and using a clean slate setup to avoid issues like stale loop devices or variable accumulation. It builds on previous exercises (e.g., `parport` blacklisting, swap file troubleshooting).

- This post was updated on 8/23/25

## Clean Slate Setup

To ensure a fresh start:

- **Remove `test.img`**:
  ```bash
  sudo rm -f test.img
  ```
- **Detach All Loop Devices**:
  ```bash
  sudo losetup -l | grep test.img | awk '{print $1}' | xargs -r sudo losetup -d
  ```
- **Clear and Unset `$LOOPDEV`**:
  ```bash
  LOOPDEV=''
  unset LOOPDEV
  echo $LOOPDEV  # Should output nothing
  ```
- **Verify**:
  ```bash
  sudo losetup -l | grep test.img  # Should output nothing
  ```

## Exercise Steps and Lessons

### Step 1: Create and Mount `test.img`

- **Commands**:
  ```bash
  sudo dd if=/dev/zero of=test.img bs=1M count=100
  sudo mkfs.ext4 -g 8192 test.img  # Force 8,192 blocks per group
  sudo mount test.img /mnt/test3
  ls /mnt/test3  # Output: lost+found
  ```
- **Outcome**: Created a 100 MB ext4 filesystem (25,600 4 KB blocks), mounted on `/mnt/test3`.
- **Why `lost+found`?**
  - **Answer**: Created by `mkfs.ext4` for `fsck` to store orphaned files during repair.
  - **Lesson**: Normal for ext4 filesystems.

### Step 2: Verify Healthy Filesystem with `fsck.ext4`

- **Commands**:
  ```bash
  sudo losetup -fP test.img
  LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}' | head -n 1)
  sudo fsck.ext4 -y $LOOPDEV
  ```
- **Outcome**:
  - Expected output:
    ```
    e2fsck 1.47.2 (1-Jan-2025)
    /dev/loopX: clean, 11/25600 files, 2586/25600 blocks
    ```
  - **Issue**: Running `fsck.ext4 -b 8193` or `-b 32768` on a healthy `test.img` failed:
    ```
    fsck.ext4: Invalid argument while trying to open /dev/loop5
    The superblock could not be read ... try running e2fsck with an alternate superblock:
        e2fsck -b 8193 <device>
     or
        e2fsck -b 32768 <device>
    ```
  - **Why Failed?**:
    - The `-b` option forces `fsck.ext4` to use a specific superblock, but 8,193 and 32,768 were invalid for this filesystem (32,768 exceeds 25,600 blocks).
    - `mkfs.ext4` set 32,768 blocks per group, reducing backup superblocks.
- **Lesson**: Use `fsck.ext4` without `-b` for healthy filesystems. Check superblock locations with `dumpe2fs`.

### Step 3: Corrupt `test.img`

- **Commands**:
  ```bash
  sudo umount /mnt/test3
  sudo dd if=/dev/urandom of=test.img bs=1M count=1 seek=10
  ```
- **Outcome**: Overwrote 1 MB at offset 10 MB (~2,500–2,750 blocks), corrupting metadata.
- **Lesson**: Random data breaks ext4 structure.

### Step 4: Set Up a Loop Device

- **Commands**:
  ```bash
  sudo losetup -fP test.img
  LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}' | head -n 1)
  echo $LOOPDEV  # Output: e.g., /dev/loop5
  ```
- **Explain `losetup -fP`**:
  - Maps `test.img` to a block device for `fsck`, `mount`, etc.
  - `-f`: Finds an available loop device.
  - `-P`: Scans for partitions (not needed but harmless).
- **Lesson**: Clear `$LOOPDEV` and detach loop devices for a clean slate.

### Step 5: Attempt Repair with `fsck.ext4`/`e2fsck`

- **Commands**:
  ```bash
  sudo fsck.ext4 -b 8193 -y $LOOPDEV
  sudo fsck.ext4 -b 16385 -y $LOOPDEV
  sudo fsck.ext4 -b 24576 -y $LOOPDEV
  sudo e2fsck -b 32768 -y $LOOPDEV
  ```
- **Outcome**: All failed due to severe metadata damage.
- **Is `e2fsck` Another Tool?**:
  - **Answer**: No, `fsck.ext4` is a symbolic link to `e2fsck`. Verify:
    ```bash
    ls -l /sbin/fsck.ext4  # Output: lrwxrwxrwx ... fsck.ext4 -> e2fsck
    ```
  - Block 32,768 exceeds `test.img`’s size (25,600 blocks), so it fails.
- **Lesson**: `fsck.ext4` and `e2fsck` are the same. Use `-b` only for corrupted superblocks.

### Step 6: Inspect with `dumpe2fs`

- **Commands**:
  ```bash
  sudo dumpe2fs $LOOPDEV | grep -E "Block size|blocks per group|superblock"
  # Output on healthy filesystem:
  #   Block size:               4096
  #   Blocks per group:         8192
  #   Primary superblock at 0, Group descriptors at 1-1
  #   Backup superblock at 8193, Group descriptors at 8194-8194
  #   Backup superblock at 24576, Group descriptors at 24577-24577
  ```
- **Your Output**:
  ```
  Blocks per group:         32768
  Primary superblock at 0, Group descriptors at 1-1
  ```
  - **Why 32,768?**: `mkfs.ext4` chose a larger block group size, reducing backup superblocks.
- **Lesson**: Use `dumpe2fs` to confirm block size (4 KB) and block group size (typically 8,192, but 32,768 here).

### Step 7: Checking Mounted Drives

- **Commands**:
  ```bash
  sudo mount test.img /mnt/test3
  sudo losetup -l | grep test.img
  findmnt /mnt/test3
  lsblk -f | grep -E 'loop|test.img'
  sudo umount /mnt/test3
  ```
- **Lesson**: Use `losetup -l` and `lsblk -f` for loop device mappings.

### Step 8: Clean Up After Experiment

- **Commands**:
  ```bash
  sudo losetup -d $LOOPDEV
  sudo rm -f test.img
  LOOPDEV=''
  unset LOOPDEV
  ```

## UTM and System Context

- **Environment**: Ubuntu VM in UTM (aarch64, kernel 6.14.0-28-generic) with RAID (`/mnt/raid1`), swap file (`/swap.img`), and EFI boot.

## Key Takeaways

1. **Clean Slate**: Remove `test.img`, detach loop devices, clear `$LOOPDEV`.
2. **fsck.ext4**: Use without `-b` for healthy filesystems. Use `-b` for corrupted superblocks.
3. **e2fsck**: Same as `fsck.ext4`.
4. **Block Groups**: Typically 8,192 blocks, but may be 32,768 for small filesystems.
5. **Superblocks**: Verify with `dumpe2fs` before corruption.
