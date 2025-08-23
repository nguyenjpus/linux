# Lesson Learned: Filesystem Corruption, Loop Devices, and Superblocks in Ubuntu

## Overview

This document summarizes a hands-on exercise (conducted on August 22, 2025, in an Ubuntu VM, kernel 6.14.0-28-generic, aarch64, running in UTM) to simulate and attempt repair of filesystem corruption on a 100 MB ext4 image (`test.img`). The exercise explored loop devices, the `lost+found` directory, superblock recovery, and mount point checks, building on previous exercises (e.g., `parport` blacklisting).

## Exercise Steps and Lessons

### Step 1: Create and Mount `test.img`

- **Commands**:
  ```bash
  sudo losetup -d /dev/loop14 #or any other dev/loopxx that this test.img was mapped on
  sudo rm test.img
  sudo dd if=/dev/zero of=test.img bs=1M count=100
  sudo mkfs.ext4 test.img
  sudo mount test.img /mnt/test3
  ls /mnt/test3  # Output: lost+found
  ```
- **Outcome**:
  - Created a 100 MB ext4 filesystem image (`test.img`) with 25,600 4 KB blocks.
  - Mounted on `/mnt/test3` (via `/dev/loop14` initially, later `/dev/loop5`).
  - `ls /mnt/test3` showed `lost+found`.
- **Question 1: Why `lost+found`?**
  - **Answer**: `lost+found` is created by `mkfs.ext4` in every ext4 filesystem. It’s a directory where `fsck` stores orphaned files/fragments during repair (e.g., after corruption). It’s empty in a fresh filesystem, as no files were added.
  - **Lesson**: `lost+found` is a normal recovery directory, not a sign of corruption.

### Step 2: Corrupt `test.img`

- **Commands**:
  ```bash
  sudo umount /mnt/test3
  sudo dd if=/dev/urandom of=test.img bs=1M count=1 seek=10
  ```
- **Outcome**:
  - Overwrote 1 MB at offset 10 MB (~2,500–2,750 blocks) with random data, corrupting ext4 metadata (e.g., inode table, group descriptors, journal).
  - Used `/dev/urandom` (correct, non-blocking) unlike earlier `/dev/random`.
- **Lesson**: Random data overwrites break ext4 structure, causing mount and `fsck` failures.

### Step 3: Set Up a Loop Device

- **Command**:
  ```bash
  sudo losetup -fP test.img
  LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}')
  echo $LOOPDEV  # Output: /dev/loop5
  ```
- **Question 3: Explain `sudo losetup -fP test.img`**
  - **Answer**:
    - `losetup`: Manages loop devices, mapping regular files to virtual block devices.
    - `-f`: Finds an available loop device (e.g., `/dev/loop5`).
    - `-P`: Scans for partitions (not needed for single-filesystem `test.img` but harmless).
    - **Purpose**: Makes `test.img` accessible as a block device for `fsck`, `mount`, or `dumpe2fs`.
  - **Lesson**: Loop devices bridge regular files to block device interfaces, essential for filesystem operations on images.

### Step 4: Attempt Repair with `fsck.ext4`

- **Commands**:
  ```bash
  sudo fsck.ext4 -b 8193 -y $LOOPDEV
  sudo fsck.ext4 -b 16385 -y $LOOPDEV
  sudo fsck.ext4 -b 24576 -y $LOOPDEV
  ```
- **Outcome**:
  - All failed: `fsck.ext4: Invalid argument ... superblock could not be read`.
  - **Issue**: The 1 MB overwrite at 10–11 MB damaged critical metadata, making all superblocks unusable.
  - **Typo**: Tried `fsck.ext` (non-existent) and `-Y` (invalid, should be `-y`). Corrected commands still failed.
- **Question 4: What are superblock offsets 8,193, 16,385, 24,576?**
  - **Answer**:
    - **Superblocks**: Store filesystem metadata (e.g., block size, inode count). Primary superblock is at block 0 (offset 1,024 bytes). Backups are at specific blocks for recovery.
    - **Offsets**: For 4 KB blocks (your `test.img`):
      - Block 8,193 = 33,558,528 bytes (~32 MB).
      - Block 16,385 = 67,117,056 bytes (~64 MB).
      - Block 24,576 = 100,663,296 bytes (~96 MB).
    - **Why These?**: Ext4 places backups in block groups (often every 8,192 or 32,768 blocks). Your 25,600-block filesystem typically has backups at 8,193, 24,576, etc.
    - **Why Failed**: Corruption at 10–11 MB (~2,500–2,750 blocks) damaged inode table or group descriptors, rendering superblocks unusable.
  - **Lesson**: Use `fsck.ext4 -b <block>` for backup superblocks. Severe corruption may prevent recovery.

### Step 5: Inspect with `dumpe2fs`

- **Commands**:
  ```bash
  sudo dumpe2fs $LOOPDEV | grep -i superblock
  dumpe2fs 1.47.2 (1-Jan-2025)
  Primary superblock at 0, Group descriptors at 1-1
  ```
- **Question 2: Can `dumpe2fs` confirm superblock status?**
  - **Answer**:
    - `dumpe2fs` dumps filesystem metadata, including superblock locations.
    - **Output**: Shows primary superblock (block 0) and group descriptors (block 1). Missing backup superblocks (e.g., 8,193, 24,576) indicate corruption prevents reading the full superblock table.
    - **When to Run**: Use on a healthy filesystem to list superblocks:
      ```bash
      sudo losetup -fP test.img
      LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}')
      sudo dumpe2fs $LOOPDEV | grep -i superblock
      ```
      - Expected on fresh `test.img`: Backups at 8,193, 24,576, etc.
  - **Lesson**: `dumpe2fs` confirms superblock locations but may show incomplete data if corrupted.

### Step 6: Checking Mounted Drives and Mount Points

- **Context**: Previously, `mount`, `cat /etc/fstab`, and `findmnt -l` didn’t clearly show `test.img` on `/dev/loop14`.
- **Commands**:
  ```bash
  sudo mount test.img /mnt/test3
  sudo losetup -l | grep test.img
  findmnt /mnt/test3
  lsblk -f | grep -E 'loop|test.img'
  sudo umount /mnt/test3
  ```
- **Explanation**:
  - `losetup -l`: Shows loop devices and backing files (e.g., `/dev/loop5` for `test.img`).
  - `findmnt /mnt/test3`: Shows mount details, including `/dev/loop5`.
  - `lsblk -f`: Lists devices, filesystems, and mount points, linking `test.img` to loop devices.
- **Lesson**: Use `losetup -l` and `lsblk -f` to trace file-to-loop-device mappings. `/etc/fstab` only shows persistent mounts.

### Step 7: Resolution

- **Outcome**: `test.img` was unrepairable due to extensive metadata damage.
- **Recreate**:
  ```bash
  sudo losetup -d $LOOPDEV
  sudo rm test.img
  sudo dd if=/dev/zero of=test.img bs=1M count=100
  sudo mkfs.ext4 test.img
  ```
- **Lesson**: If `fsck` fails with all superblocks, recreate the filesystem.

## UTM and System Context

- **Environment**: Ubuntu VM in UTM (aarch64, kernel 6.14.0-28-generic) with RAID (`/mnt/raid1`), swap file (`/swap.img`), and EFI boot.
- **Impact**: Exercise confined to `test.img` on `/dev/vda2`, not affecting other components.

## Key Takeaways

1. **Loop Devices**: Use `losetup -fP` to map files to block devices for `fsck`, `mount`, etc.
2. **lost+found**: Created by `mkfs.ext4` for `fsck` recovery, normal in ext4 filesystems.
3. **Superblocks**: Primary at block 0, backups at 8,193, 24,576, etc. (4 KB blocks). Corruption can render all unusable.
4. **dumpe2fs**: Lists superblock locations; truncated output indicates corruption.
5. **Mount Checks**: Use `losetup -l`, `lsblk -f`, and `findmnt` for loop device mappings.
