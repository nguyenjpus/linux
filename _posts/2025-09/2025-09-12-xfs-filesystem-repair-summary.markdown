# XFS Filesystem Repair Practice Summary (Sep 12, 2025)

This document summarizes hands-on practice with XFS filesystem corruption and repair on Ubuntu 24.04, using `dd`, `mkfs.xfs`, `xfs_db`, and `xfs_repair`. The focus was on superblock corruption, repair, and an attempt at orphan file recovery.

## Objectives

- Create a 500MB XFS filesystem.
- Corrupt the superblock and repair it.
- Attempt to orphan a file and recover it to `lost+found`.
- Verify data integrity post-repair.

## Practice Steps and Outcomes

### 1. Initial Setup

- **Commands**:
  ```bash
  dd if=/dev/zero of=~/xfs.img bs=1M count=500
  sudo mkfs.xfs ~/xfs.img
  sudo mkdir /mnt/xfs
  sudo mount -o loop ~/xfs.img /mnt/xfs/
  echo "xfs test data" | sudo tee /mnt/xfs/xfstest.txt
  sudo umount /mnt/xfs
  sudo cp ~/xfs.img ~/xfs.img.bk
  sudo mount -o loop ~/xfs.img /mnt/xfs/
  cat /mnt/xfs/xfstest.txt
  ```
- **Outcome**: Created 500MB XFS V5 filesystem (128000 4k blocks, 16384-block log, `crc=1`, `reflink=1`). Wrote `xfstest.txt` with “xfs test data,” verified, and backed up.
- **Lesson**: XFS V5 features (e.g., CRC, reflink) require modern `xfsprogs` (6.6.0).

### 2. Superblock Corruption Attempts with `xfs_db`

- **Commands**:
  ```bash
  sudo xfs_db -x ~/xfs.img
  sb 0
  write version 0
  write sb_versionnum 0
  write versionnum 0
  write versionnum 0x0
  sudo xfs_db -x -u ~/xfs.img
  ```
- **Outcome**: All `write` commands failed:
  - `version`, `sb_versionnum`: “field not found” (incorrect field names).
  - `versionnum 0`, `versionnum 0x0`: “Superblock has unknown features enabled” and “write error: Invalid argument” due to V5 verifier checks (CRC, feature masks).
  - `-u` flag: “invalid option” (not supported in `xfsprogs` 6.6.0).
- **Lesson**: `xfs_db` write mode enforces strict V5 superblock checks. Correct field is `versionnum`, but verifiers block invalid values. Use `dd` for reliable corruption.

### 3. Superblock Corruption with `dd`

- **Commands**:
  ```bash
  sudo dd if=/dev/zero of=~/xfs.img bs=512 count=10 seek=0 conv=notrunc
  sudo xfs_repair -n ~/xfs.img
  sudo xfs_repair ~/xfs.img
  sudo mount -o loop ~/xfs.img /mnt/xfs
  ls /mnt/xfs/
  cat /mnt/xfs/xfstest.txt
  sudo umount /mnt/xfs
  sudo xfs_db ~/xfs.img
  sb 0
  print
  ```
- **Outcome**:
  - `dd`: Zeroed 5KB (superblock, partial AGF/AGI), breaking magic number (`XFSB`).
  - `xfs_repair -n`: Detected “bad magic number,” found secondary superblock (AG 1, ~block 32000).
  - `xfs_repair`: Rebuilt superblock, fixed AG 0’s AGF/AGI (bad CRCs, magic, UUIDs), corrected inode 131’s map, rebuilt root inode. Warned about “host filesystem geometry” (irrelevant for loop image).
  - Mount: Showed `xfstest.txt` with “xfs test data.” No `lost+found` (no orphaned inodes).
  - `xfs_db print`: Confirmed restored superblock (`magicnum = 0x58465342`, `versionnum = 0xb4a5`, `crc` correct).
- **Lesson**: `dd` reliably corrupts superblock. `xfs_repair` rebuilds from AG metadata, preserving data blocks. No unlink meant no orphans.

### 4. Orphan File Recovery Attempt

- **Commands**:
  ```bash
  sudo cp ~/xfs.img.bk ~/xfs.img
  sudo mount -o loop ~/xfs.img /mnt/xfs
  sudo rm /mnt/xfs/xfstest.txt
  sync # with or without this, the results were the same (i.e: no lost+found under /mnt/xfs/)
  sudo umount /mnt/xfs
  sudo xfs_db -x ~/xfs.img
  logformat
  quit
  sudo xfs_repair -n ~/xfs.img
  sudo xfs_repair -L ~/xfs.img
  sudo mount -o loop ~/xfs.img /mnt/xfs
  sudo ls /mnt/xfs/
  ```
- **Outcome**:
  - Unlinked `xfstest.txt` (inode ~131), corrupted log with `logformat` (cycle 1).
  - `xfs_repair -n`: Detected log issue (“LSN 1:13 ahead of 1:2”), no orphaned inodes reported.
  - `xfs_repair -L`: Cleared log (cycle 4), rebuilt root directory, but `/mnt/xfs/` was empty (no `lost+found` or `xfstest.txt`).
- **Why**: Log clearing finalized the `rm`, freeing inode 131. XFS’s log ensures atomicity, and `xfs_repair -L` discarded the unlinked inode rather than recovering it.
- **Lesson**: XFS is less likely to recover orphaned inodes than ext4’s `e2fsck`. Log corruption must catch an uncommitted `rm`, and `sync` may help. Root directory reset can empty the filesystem.

### Key Lessons Learned

1. **XFS V5 Features**: CRC, reflink, and `finobt` require modern `xfsprogs`. Verifiers block invalid `xfs_db` writes.
2. **Superblock Repair**: `xfs_repair` rebuilds from AG superblocks, preserving data (e.g., `xfstest.txt` survived superblock corruption).
3. **Log Corruption**: `xfs_db logformat` disrupts log, but `xfs_repair -L` clears it, often finalizing deletions.
4. **Orphan Recovery**: Requires unlinked inode with intact data blocks and disrupted log. XFS is conservative, unlike ext4.
5. **Tools**: `dd` is reliable for corruption; `xfs_db` needs precise fields (`versionnum`) and lacks `-u` in 6.6.0.
6. **Verification**: Always mount and check files post-repair (`ls`, `cat`).

### Conclusion

XFS practice demonstrated superblock repair success and challenges with orphan recovery due to log atomicity. `dd` overcame `xfs_db` limitations, and `xfs_repair` proved robust for metadata reconstruction. Ready for LVM practice next.
