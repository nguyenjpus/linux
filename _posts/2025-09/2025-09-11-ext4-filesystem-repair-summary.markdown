---
layout: post
title: "Ext4 Filesystem Repair Practice Summary"
date: 2025-09-11
tags: [Basic, filesystem, ext4, e2fsck, tune2fs]
---

This document summarizes hands-on practice with ext4 filesystem corruption and repair on Ubuntu, using `dd`, `mkfs.ext4`, `e2fsck`, `tune2fs`, `debugfs`, and `dumpe2fs`. The focus was on simulating superblock, journal, and metadata corruption, with a final attempt to recover an orphaned file.

## Objectives

- Create ext4 filesystem images (100MB and 500MB).
- Simulate corruption (superblock, journal, metadata).
- Detect and repair issues using `e2fsck`.
- Verify recovery by mounting and checking data.
- Recover an orphaned file to `lost+found`.

## Practice Steps and Outcomes

### 1. Initial Setup (100MB Image)

- **Commands**:
  ```bash
  dd if=/dev/zero of=~/ext4.img bs=1M count=100
  sudo mkfs.ext4 ~/ext4.img
  sudo mkdir /mnt/ext4
  sudo mount -o loop ~/ext4.img /mnt/ext4
  echo "Ext4 test data" | sudo tee /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  sudo cp ~/ext4.img ~/ext4.img.bk
  ```
- **Outcome**: Created 100MB ext4 image with `test.txt`. No backup superblocks (single block group, ~25k blocks, confirmed by `dumpe2fs | grep "Backup superblock"`).
- **Lesson**: Small filesystems lack backup superblocks, limiting recovery options.

### 2. Attempted Corruption (100MB Image)

- **Journal Removal**:

  ```bash
  sudo tune2fs -O ^has_journal ~/ext4.img
  sudo e2fsck -n ~/ext4.img
  sudo tune2fs -j ~/new.img
  ```

  - **Outcome**: Journal disabled/re-enabled. `e2fsck` reported clean (journal removal isn’t corruption).
  - **Lesson**: Journal toggling affects crash resilience, not data integrity.

- **Metadata Corruption (debugfs)**:

  ```bash
  sudo debugfs -w ~/ext4.img
  stat /test.txt
  unlink <12>  # Incorrect
  zap_block 12  # Incorrect
  quit
  ```

  - **Issue**: Wrong commands (`unlink <12>`, `zap_block`). Correct: `rm /test.txt` or `clri <12>`. Filesystem stayed clean.
  - **Lesson**: Debugfs requires precise syntax. Journaling masks minor changes.

- **Random Corruption**:
  ```bash
  sudo e2image -r -f ~/ext4.img ~/corrupt_ext4.img
  sudo dd if=/dev/urandom of=~/corrupt_ext4.img bs=512 count=10 seek=100 conv=notrunc
  sudo e2fsck -n ~/corrupt_ext4.img
  ```
  - **Issue**: `dd` failed (no `sudo`). `e2image` marked unclean, but no corruption applied.
  - **Lesson**: Use `sudo` for root-owned files. Target critical areas (e.g., superblock).

### 3. Revised Setup (500MB Image)

- **Commands**:
  ```bash
  dd if=/dev/zero of=~/ext4.img bs=1M count=500
  sudo mkfs.ext4 ~/ext4.img
  sudo mkdir /mnt/ext4
  sudo mount -o loop ~/ext4.img /mnt/ext4
  echo "Ext4 test data" | sudo tee /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  sudo cp ~/ext4.img ~/ext4.img.bk
  ```
- **Outcome**: 500MB image with backups at 32768/98304 (via `dumpe2fs`). `test.txt` verified.
- **Why Better**: Multiple block groups enabled superblock backups.

### 4. Superblock Corruption and Recovery

- **Commands**:
  ```bash
  sudo cp ~/ext4.img ~/corrupt_ext4.img
  sudo dd if=/dev/urandom of=~/corrupt_ext4.img bs=512 count=20 seek=0 conv=notrunc
  sudo e2fsck -n ~/corrupt_ext4.img
  sudo e2fsck -b 32768 -fy ~/ext4.img
  sudo mount -o loop ~/ext4.img /mnt/ext4
  cat /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  ```
- **Outcome**: Zeroed superblock (block 0), causing "Bad magic number". `e2fsck -b 32768` used backup, fixed metadata (resize inode, counts). `test.txt` intact.
- **Lesson**: Backup superblocks enable recovery. Target `seek=0` for superblock damage.

### 5. Metadata Corruption (debugfs)

- **Commands**:
  ```bash
  cp ~/ext4.img.bk ~/new.img
  sudo tune2fs -O ^has_journal ~/new.img
  sudo tune2fs -j ~/new.img
  sudo debugfs -w ~/new.img
  set_super_value s_state 1
  rm /test.txt
  clri <12>
  set_super_value s_rev_level 0
  quit
  sudo e2fsck -n ~/new.img
  sudo e2fsck -fy ~/new.img
  sudo mount -o loop ~/new.img /mnt/ext4
  sudo ls /mnt/ext4/lost+found/
  ```
- **Outcome**: Marked dirty, removed `test.txt`, cleared inode 12, set invalid revision. `e2fsck` rebuilt root, journal, and structure, but no `test.txt` (due to `clri`).
- **Lesson**: `rm` orphans files (recoverable); `clri` deletes permanently. `metadata_csum` aids detection.

### 6. Orphan Recovery Attempts

- **First Attempt**:

  ```bash
  cp ~/ext4.img.bk ~/orphan.img
  sudo debugfs -w ~/orphan.img
  stat /test.txt
  rm /test.txt
  quit
  sudo e2fsck -n ~/orphan.img
  sudo e2fsck -fy ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  sudo ls /mnt/ext4/lost+found/
  ```

  - **Issue**: `e2fsck` reported clean; no `test.txt` in `lost+found`. Journal replay during mount finalized `rm`.
  - **Lesson**: Avoid mounts before `e2fsck`. Corrupt journal to preserve orphans.

- **Second Attempt**:

  ```bash
  sudo dd if=/dev/zero of=~/orphan.img bs=4096 count=1 seek=8 conv=notrunc
  sudo debugfs -w ~/orphan.img
  stat /test.txt
  rm /test.txt
  set_super_value s_state 1
  quit
  sudo e2fsck -n ~/orphan.img
  sudo e2fsck -fy ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  sudo ls /mnt/ext4/lost+found/
  ```

  - **Issue**: Block 8 (GDT) didn’t hit journal. Mount replayed journal, finalizing `rm`. No recovery.
  - **Lesson**: Target correct journal block (e.g., 65536, via `debugfs stat <8>`).

- **Third Attempt**:

  ```bash
  rm ~/orphan.img
  dd if=/dev/zero of=~/orphan.img bs=1M count=500
  sudo mkfs.ext4 ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  echo "Orphan test data" | sudo tee /mnt/ext4/testOrphan.txt
  sudo umount /mnt/ext4
  sudo dd if=/dev/zero of=~/orphan.img bs=4096 count=1 seek=0 conv=notrunc
  sudo debugfs -w ~/orphan.img
  stat /testOrphan.txt
  set_super_value s_state 1
  quit
  sudo e2fsck -n ~/orphan.img
  sudo e2fsck -fy ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  sudo ls /mnt/ext4/  # Showed lost+found, testOrphan.txt
  ```

  - **Issue**: Zeroed superblock (block 0), blocking `debugfs` unlink. `e2fsck` repaired superblock; `testOrphan.txt` remained intact.
  - **Lesson**: Unlink before corrupting superblock.

- **Final Attempt**:
  ```bash
  rm ~/orphan.img
  dd if=/dev/zero of=~/orphan.img bs=1M count=500
  sudo mkfs.ext4 ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  echo "Orphan test data" | sudo tee /mnt/ext4/testOrphan.txt
  sudo umount /mnt/ext4
  sudo cp ~/orphan.img ~/orphan.img.bk
  sudo debugfs -w ~/orphan.img
  stat /testOrphan.txt
  rm /testOrphan.txt
  set_super_value s_state 1
  quit
  sudo dd if=/dev/zero of=~/orphan.img bs=4096 count=1 seek=65536 conv=notrunc
  sudo e2fsck -n ~/orphan.img
  sudo e2fsck -fy ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  sudo ls /mnt/ext4/lost+found/
  sudo debugfs ~/orphan.img
  stat <8>
  ```
  - **Outcome**: Unlinked `testOrphan.txt` (inode 12), corrupted journal at block 65536 (from `stat <8>`). `e2fsck -n` detected journal corruption but aborted. `e2fsck -fy` cleared journal, freed blocks 65536–69631, recreated journal. No `testOrphan.txt` in `lost+found`.
  - **Why**: Journal logged `rm` as valid. Corruption allowed `e2fsck` to finalize deletion, freeing inode 12. No orphaned inode detected.
  - **Lesson**: Severe journal corruption (block 65536) caused `e2fsck` to discard journal, treating `rm` as complete. Corrupting more blocks or disabling journal post-unlink may preserve orphans.

### Key Lessons Learned

1. **Image Size**: 500MB+ ensures backup superblocks (32768, 98304).
2. **Superblock Recovery**: `e2fsck -b <block>` restores from backups. Target `seek=0` for damage.
3. **Journal Behavior**: Journal (inode 8, blocks 65536–69631) logs metadata. Corruption requires precise block targeting (via `debugfs stat <8>`).
4. **Orphan Recovery**: `rm` orphans inodes, but journal replay finalizes deletions unless fully corrupted. Avoid mounts before `e2fsck`.
5. **Debugfs**: `rm /file` for recoverable corruption; `clri <inode>` for permanent loss.
6. **Ext4 Resilience**: `metadata_csum` and journaling clean minor issues, requiring severe corruption for demos.
7. **Permissions**: Use `sudo` for root-owned files (`lost+found`, images).

### Conclusion

The ext4 practice demonstrated superblock and metadata recovery, with challenges in orphan recovery due to journaling. The final attempt confirmed journal corruption but didn’t recover `testOrphan.txt` due to logged deletion. This prepares for XFS and LVM exercises.
