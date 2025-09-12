# Ext4 Filesystem Repair Practice Summary

This document summarizes the hands-on practice of simulating and repairing ext4 filesystem corruption, conducted on Ubuntu using tools like `dd`, `mkfs.ext4`, `e2fsck`, `tune2fs`, `debugfs`, and `dumpe2fs`. The goal was to understand filesystem corruption scenarios (superblock, journal, metadata) and recovery techniques, with clear steps and lessons learned.

## Objectives
- Create an ext4 filesystem image.
- Simulate corruption (superblock, journal, metadata).
- Detect and repair issues using `e2fsck`.
- Verify recovery by mounting and checking data.
- Explore advanced recovery (backup superblocks).

## Practice Steps and Outcomes

### 1. Initial Setup (100MB Image)
- **Commands**:
  ```
  dd if=/dev/zero of=~/ext4.img bs=1M count=100
  sudo mkfs.ext4 ~/ext4.img
  sudo mkdir /mnt/ext4
  sudo mount -o loop ~/ext4.img /mnt/ext4
  echo "Ext4 test data" | sudo tee /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  sudo cp ~/ext4.img ~/ext4.img.bk
  ```
- **Purpose**: Created a 100MB ext4 image, wrote a test file, and backed it up.
- **Outcome**: Successful creation/mount. File `test.txt` verified with content "Ext4 test data".
- **Issue**: 100MB size resulted in one block group (~25k blocks), so no backup superblocks (confirmed by `dumpe2fs | grep "Backup superblock"` returning empty).

### 2. Attempted Corruption (100MB Image)
- **Journal Removal (Option 1)**:
  ```
  sudo tune2fs -O ^has_journal ~/ext4.img
  sudo e2fsck -n ~/ext4.img
  sudo tune2fs -j ~/new.img
  ```
  - **Outcome**: Journal disabled and re-enabled. `e2fsck` reported "clean" because removing the journal isn't corruption—it converts ext4 to ext2-like behavior. File access remained intact.
  - **Lesson**: Journal removal doesn't destroy data; it reduces crash resilience.

- **Metadata Corruption via debugfs (Option 2)**:
  ```
  sudo debugfs -w ~/ext4.img
  stat /test.txt  # Showed inode 12
  unlink <12>     # Incorrect syntax
  zap_block 12    # Wrong command (targets block, not inode)
  quit
  ```
  - **Issue**: Incorrect commands (`unlink <12>`, `zap_block`) failed to corrupt. Correct commands are `rm /test.txt` (unlinks) or `clri <12>` (clears inode). `unlink` expects a path, and `zap_inode` isn't valid. Filesystem stayed clean.
  - **Lesson**: Debugfs requires precise commands. Journaling and checksums (e.g., `metadata_csum`) can mask minor changes.

- **Random Corruption (Option 3)**:
  ```
  sudo e2image -r -f ~/ext4.img ~/corrupt_ext4.img
  sudo dd if=/dev/urandom of=~/corrupt_ext4.img bs=512 count=10 seek=100 conv=notrunc
  sudo e2fsck -n ~/corrupt_ext4.img
  ```
  - **Issue**: `dd` failed due to missing `sudo` (Permission denied). `e2image` copy marked "not cleanly unmounted" due to metadata copy quirks, but no corruption applied. `e2fsck` reported clean.
  - **Lesson**: Always use `sudo` for root-owned files. Target `dd` at critical areas (e.g., seek=0 for superblock).

### 3. Revised Setup (500MB Image)
- **Commands**:
  ```
  dd if=/dev/zero of=~/ext4.img bs=1M count=500
  sudo mkfs.ext4 ~/ext4.img
  sudo mkdir /mnt/ext4
  sudo mount -o loop ~/ext4.img /mnt/ext4
  echo "Ext4 test data" | sudo tee /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  sudo cp ~/ext4.img ~/ext4.img.bk
  sudo dumpe2fs ~/ext4.img | grep "Backup superblock"
  ```
- **Outcome**: 500MB size created multiple block groups, enabling backups at blocks 32768 and 98304 (confirmed by `dumpe2fs`). File `test.txt` verified.
- **Why Better**: Larger size ensured backup superblocks, critical for recovery demos.

### 4. Successful Superblock Corruption and Recovery
- **Commands**:
  ```
  sudo cp ~/ext4.img ~/corrupt_ext4.img
  sudo dd if=/dev/urandom of=~/corrupt_ext4.img bs=512 count=20 seek=0 conv=notrunc
  sudo e2fsck -n ~/corrupt_ext4.img
  sudo e2fsck -b 32768 -fy ~/ext4.img
  sudo mount -o loop ~/ext4.img /mnt/ext4
  cat /mnt/ext4/test.txt
  sudo umount /mnt/ext4
  ```
- **Outcome**:
  - `dd seek=0` overwrote the primary superblock, causing "Bad magic number in super-block" and issues with inode 7 (resize inode). Dry-run (`-n`) listed "illegal blocks" and checksum errors.
  - Repair with `-b 32768 -fy` used backup superblock, recreated resize inode, fixed counts/checksums. Post-repair mount showed `test.txt` intact.
- **Lesson**: Backup superblocks are lifesavers for superblock corruption. `e2fsck -fy` automates most fixes. Target `seek=0` for guaranteed superblock damage.

### 5. Metadata Corruption via debugfs
- **Commands**:
  ```
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
  sudo umount /mnt/ext4
  ```
- **Outcome**:
  - `s_state 1` marked dirty (crash simulation). `rm /test.txt` and `clri <12>` removed file and inode. `s_rev_level 0` invalidated revision (ext4 expects 1).
  - Dry-run (`-n`) caught invalid journal and aborted. Repair (`-fy`) cleared journal, fixed revision, rebuilt root/lost+found, and recreated journal. No `test.txt` in `lost+found` because `clri` erased the inode entirely.
  - **Why No Recovery**: `clri` is destructive (clears inode metadata). `rm` alone would orphan the inode, recoverable to `lost+found`.
- **Lesson**: Debugfs can precisely break metadata. Use `rm` for recoverable corruption; `clri` for permanent loss. Checksums (`metadata_csum`) help detect issues.

### 6. Milder Debugfs Attempt (Orphan Recovery)
- **Commands**:
  ```
  cp ~/ext4.img.bk ~/orphan.img
  sudo debugfs -w ~/orphan.img
  stat /test.txt
  rm /test.txt
  quit
  sudo e2fsck -n ~/orphan.img
  sudo e2fsck -fy ~/orphan.img
  sudo mount -o loop ~/orphan.img /mnt/ext4
  sudo ls /mnt/ext4/lost+found/
  sudo umount /mnt/ext4
  ```
- **Issue**: `e2fsck` reported clean; no `test.txt` in `lost+found`. The `rm` was journaled, so ext4 treated it as a valid deletion, not corruption. No orphaned inode was left for recovery.
- **Lesson**: Journaling and `metadata_csum` auto-clean minor inconsistencies. To force orphan recovery, add `set_super_value s_state 1` or corrupt journal.

### Key Lessons Learned
1. **Image Size Matters**: Small filesystems (100MB) lack backup superblocks (single group). Use 500MB+ for realistic demos (multiple groups).
2. **Journal Behavior**: Disabling (`^has_journal`) doesn't corrupt; it reduces crash protection. Dirty state (`s_state 1`) or journal block corruption (via `dd`) better simulates issues.
3. **Debugfs Precision**: Use `rm /path` to orphan files (recoverable) or `clri <inode>` for permanent loss. Commands like `zap_block` or `unlink <inode>` are incorrect.
4. **Superblock Recovery**: `e2fsck -b <block>` (e.g., 32768) restores from backups. Use `dumpe2fs` to find backup locations.
5. **e2fsck Power**: `-fy` automates repairs (journal, inodes, bitmaps). Dry-run (`-n`) shows issues without changes.
6. **Mounting/Permissions**: Always use `sudo` for root-owned files (`corrupt_ext4.img`, `lost+found`). `-o loop` is required for image mounts.
7. **Resilience of Ext4**: Features like `metadata_csum` and journaling mask minor corruptions, requiring targeted attacks (e.g., superblock, inode table).

## Recommendations for Future Practice
- **Orphan Recovery**: Retry with `set_super_value s_state 1` after `rm /test.txt` to simulate crash and recover `test.txt` to `lost+found`.
- **Journal Corruption**: Use `dumpe2fs` to find journal blocks (e.g., 8193-12288), then `dd` to corrupt them: `sudo dd if=/dev/zero of=~/ext4.img bs=4096 count=1 seek=8193 conv=notrunc`.
- **Move to XFS**: Next filesystem for similar corruption/repair practice, using `xfs_db` and `xfs_repair`. Start with:
  ```
  dd if=/dev/zero of=~/xfs.img bs=1M count=500
  sudo mkfs.xfs ~/xfs.img
  sudo mkdir /mnt/xfs
  sudo mount -o loop ~/xfs.img /mnt/xfs
  echo "XFS test data" | sudo tee /mnt/xfs/test.txt
  sudo umount /mnt/xfs
  ```

## Conclusion
The practice successfully demonstrated ext4 superblock and metadata corruption/repair, with key insights into journaling, debugfs, and `e2fsck` capabilities. While the milder orphan recovery didn't show errors due to ext4's robustness, the superblock recovery and severe debugfs corruption were clear wins. This lays a strong foundation for tackling XFS and Btrfs next.