# Lesson Learned: Filesystem Corruption with Clean Slate Setup in Ubuntu

## Overview

This document summarizes a hands-on exercise (conducted on August 23, 2025, in an Ubuntu VM, kernel 6.14.0-28-generic, aarch64, running in UTM) to simulate and attempt repair of filesystem corruption on a 100 MB ext4 image (`test.img`). It emphasizes a robust clean slate setup to prevent issues like multiple loop devices or sticky `$LOOPDEV` values, demonstrates `fsck.ext4` on a healthy filesystem, and clarifies `e2fsck`. It supports Linux+ studies and builds on previous exercises (e.g., `parport` blacklisting, swap file troubleshooting).

- This post was upated on 8/23

## Clean Slate Setup

To ensure a fresh start and prevent multiple loop devices or sticky `$LOOPDEV`:

- **Remove** `test.img`:

  ```bash
  sudo rm -f test.img
  ```

- **Detach All Loop Devices**:

  ```bash
  sudo losetup -l | grep test.img | awk '{print $1}' | xargs -r sudo losetup -d
  ```

- **Clear and Unset** `$LOOPDEV`:

  ```bash
  LOOPDEV=''
  unset LOOPDEV
  echo $LOOPDEV  # Should output nothing
  ```

- **Verify**:

  ```bash
  sudo losetup -l | grep test.img  # Should output nothing
  ```

- **Check for Persistent** `$LOOPDEV`:

  ```bash
  grep LOOPDEV ~/.bashrc ~/.profile
  ```

  - Remove any lines setting `$LOOPDEV` if found.

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
- **Why** `lost+found`**?**
  - **Answer**: Created by `mkfs.ext4` for `fsck` to store orphaned files during repair.
  - **Lesson**: Normal for ext4 filesystems.

### Step 2: Verify Healthy Filesystem with `fsck.ext4`

- **Commands**:

  ```bash
  sudo umount /mnt/test3
  sudo losetup -fP test.img
  LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}' | head -n 1)
  sudo fsck.ext4 -y $LOOPDEV
  sudo dumpe2fs $LOOPDEV | grep -E "Block size|blocks per group|superblock"
  sudo losetup -d $LOOPDEV
  unset LOOPDEV
  echo $LOOPDEV #to confirm LOOPDEV is clean now.
  ```

- **Outcome**:

  - Expected `fsck.ext4` output:

    ```
    e2fsck 1.47.2 (1-Jan-2025)
    /dev/loopX: clean, 11/25600 files, 2586/25600 blocks
    ```

  - Expected `dumpe2fs` output:

    ```
    Block size:               4096
    Blocks per group:         8192
    Primary superblock at 0, Group descriptors at 1-1
    Backup superblock at 8193, Group descriptors at 8194-8194
    Backup superblock at 24576, Group descriptors at 24577-24577
    ```

- **Previous Issue**: `fsck.ext4 -b 8193` and `-b 32768` failed on a healthy `test.img`:

  ```
  fsck.ext4: Invalid argument while trying to open /dev/loop5
  The superblock could not be read ... try running e2fsck with an alternate superblock:
      e2fsck -b 8193 <device>
   or
      e2fsck -b 32768 <device>
  ```

  - **Why Failed?**: `-b` forced invalid superblock locations. `mkfs.ext4` set 32,768 blocks per group, reducing backup superblocks.

- **Lesson**: Use `fsck.ext4` without `-b` for healthy filesystems. Verify superblocks with `dumpe2fs`.

### Step 3: Corrupt `test.img`

- **Commands**:

  ```bash
  sudo umount /mnt/test3
  sudo dd if=/dev/urandom of=test.img bs=1M count=1 seek=10
  ```

- **Outcome**: Overwrote 1 MB at offset 10 MB (\~2,500–2,750 blocks), corrupting metadata.
- **Lesson**: Random data breaks ext4 structure.

### Step 4: Set Up a Loop Device

- **Commands**:

  ```bash
  sudo losetup -fP test.img
  LOOPDEV=$(sudo losetup -l | grep test.img | awk '{print $1}' | head -n 1)
  echo $LOOPDEV  # Output: e.g., /dev/loop5
  sudo losetup -l | grep test.img #recheck to see if multiple loop devices accumulated or not
  ```

- **Issue**: If multiple loop devices accumulated (`/dev/loop5`, `/dev/loop14`, `/dev/loop16`) as follows:

  ```
  /dev/loop16         0      0         0  0 /home/ron/test.img                                        0     512
  /dev/loop14         0      0         0  0 /home/ron/test.img                                        0     512
  /dev/loop5          0      0         0  0 /home/ron/test.img                                        0     512
  ```

- **Then Fix, or else move to step 5**:

  ```bash
  sudo losetup -l | grep test.img | awk '{print $1}' | xargs -r sudo losetup -d
  LOOPDEV=''
  unset LOOPDEV
  ```

- **Lesson**: Detach loop devices and clear `$LOOPDEV` after each step.

### Step 5: Attempt Repair with `fsck.ext4`/`e2fsck`

- **Commands**:

  ```bash
  sudo fsck.ext4 -b 8193 -y $LOOPDEV
  sudo fsck.ext4 -b 16385 -y $LOOPDEV
  sudo fsck.ext4 -b 24576 -y $LOOPDEV
  sudo e2fsck -b 32768 -y $LOOPDEV
  ```

- **Outcome**: All failed due to metadata damage.
- **Is** `e2fsck` **Another Tool?**:

  - **Answer**: No, `fsck.ext4` is a symbolic link to `e2fsck`. Verify:

    ```bash
    ls -l /sbin/fsck.ext4  # Output: lrwxrwxrwx ... fsck.ext4 -> e2fsck
    ```

  - Block 32,768 exceeds `test.img`’s size (25,600 blocks).

- **Lesson**: Use `-b` only for corrupted superblocks with valid locations.

### Step 6: Inspect with `dumpe2fs`

- **Commands**:

  ```bash
  sudo dumpe2fs $LOOPDEV | grep -E "Block size|blocks per group|superblock"
  ```

- **Previous Output**:

  ```
  Block size:               4096
  Blocks per group:         32768
  Primary superblock at 0, Group descriptors at 1-1
  ```

- **Why 32,768?**: `mkfs.ext4` chose a larger block group size, reducing backup superblocks. Use `-g 8192` to enforce standard size.
- **Lesson**: Verify block group size and superblocks with `dumpe2fs`.

### Step 7: Checking Mounted Drives

- **Commands**:

  ```bash
  sudo mount test.img /mnt/test3 # mount will fail because test.img has been corrupted.
  sudo losetup -l | grep test.img
  findmnt /mnt/test3
  lsblk -f | grep -E 'loop|test.img'
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

1. **Clean Slate**: Remove `test.img`, detach all loop devices, clear `$LOOPDEV`.
2. **fsck.ext4**: Use without `-b` for healthy filesystems. Use `-b` for recovery.
3. **e2fsck**: Same as `fsck.ext4`.
4. **Block Groups**: Typically 8,192 blocks, but may be 32,768 for small filesystems.
5. **Loop Devices**: Detach after use to prevent sticky `$LOOPDEV`.
