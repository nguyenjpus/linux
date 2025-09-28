---
layout: post
title: "Using Loop and loseupt"
date: 2025-09-07
tags: [Basic, extundelete, photorec, Recovery Tools]
---

This document summarizes the hands-on practices for data recovery on ext4, XFS, and Btrfs filesystems, based on CompTIA Linux+ objectives. It incorporates lessons from attempts where recovery failed, highlighting tool limitations and pitfalls. Expanded from Fermin O. Goetz's book.

## Goal

Practice recovering deleted files using journal-based (`extundelete`), carving (`photorec`), and repair tools (`e2fsck`, `xfs_repair`, `btrfs check`). Understand why recovery isn't always successful.

## Prerequisites

- Tools: `sudo apt install extundelete testdisk photorec xfsprogs btrfs-progs`.
- Test in VM with snapshots.
- Key Lesson: Recovery is best-effort; small text files are hard to recover.

## Summary of Lesson

- **extundelete**: Ext4-specific, relies on journal for inode recovery. Partial success in tests (e.g., recovered `file.txt` but not `lost_ext.txt`). Fails if journal flushed or inodes reused. Use `sync`/`sleep` to aid, but limitations persist on small images.
- **photorec**: Carving tool, scans for signatures. Failed in tests (recovered metadata like `f0000000.xfs`, not text files). Better for media; weak for small/plain text. Special: Choose filesystem type (e.g., "P XFS 5") and enable [txt] for best chance.
- **Btrfs Specialties**: Uses CoW for data integrity, snapshots for easy recovery (not used here). Without snapshots, deletions are hard to reverse. `btrfs check`/`rescue` fix corruption, not deletions—tests showed "no error found."
- **Partial Recovery**: Some files (e.g., `file.txt`) recovered due to order/timing (later deletions logged better). Others (e.g., `lost_ext.txt`) lost to journal flushing or overwrite.
- **XFS/Btrfs Failures**: No journal for undelete; repair tools only fix structure. Nothing recovered as deletions were clean.
- **Pitfalls**: Small images/metadata overhead; rapid operations clear journals; tool bugs (e.g., "double free" crashes); missing `sudo`/options; deletions before full write/sync.
- **Overall**: Don't rely on recovery—use backups. Tests showed 0% success for small text files across tools.

## Reasons for Recovery Failure

1. **Small File Size**: Tiny text files (~20-30 bytes) lack distinct signatures for `photorec` carving.
2. **Journal Flushing**: ext4 journal entries cleared despite `sync`/`sleep`, due to commit intervals or rapid ops.
3. **No Journal in XFS/Btrfs**: Lack ext4-style undelete support; deletions permanent without snapshots.
4. **Small Image Size**: 400MB-1GB images have high metadata overhead, leading to quick overwrite.
5. **Deletion Timing**: Immediate deletions may not log fully; order affects partial recovery.
6. **Tool Limitations**: `extundelete` ext4-only/buggy; `photorec` weak for text; repair tools not for undelete.
7. **Overwrite Behavior**: Free blocks reused by metadata updates post-deletion.
8. **Mount Options**: `data=journal` helps but not enough if journal wraps.
9. **No Corruption**: Tests were clean deletions; repair tools found "no need to recover."

## Step-by-Step Practices (With Notes on Failures)

### ext4

```bash
dd if=/dev/zero of=~/ext4.img bs=1M count=1000
sudo mkfs.ext4 -J size=64 ~/ext4.img
sudo mount -o loop,data=journal ~/ext4.img /mnt/ext4
sudo bash -c 'mkdir /mnt/ext4/dir; echo "ext4 recover me" > /mnt/ext4/lost_ext.txt; sync; sleep 5; echo "Dir content" > /mnt/ext4/dir/file.txt; sync; sleep 5; rm /mnt/ext4/lost_ext.txt; sync; sleep 5; rm -rf /mnt/ext4/dir; sync; sleep 5'
sudo umount /mnt/ext4
sudo extundelete ~/ext4.img --restore-all  # Failed in tests
sudo photorec ~/ext4.img  # Failed, empty recovery
```

### XFS

```bash
dd if=/dev/zero of=~/xfs.img bs=1M count=400
sudo mkfs.xfs -f ~/xfs.img
sudo mount -o loop ~/xfs.img /mnt/xfs
sudo bash -c 'echo "XFS recover me" > /mnt/xfs/lost_xfs.txt; sync; rm /mnt/xfs/lost_xfs.txt; sync'
sudo umount /mnt/xfs
xfs_repair ~/xfs.img  # No recovery
sudo photorec ~/xfs.img  # Failed, metadata only
```

### Btrfs

```bash
dd if=/dev/zero of=~/btrfs.img bs=1M count=400
sudo mkfs.btrfs ~/btrfs.img
sudo mount -o loop ~/btrfs.img /mnt/btrfs
sudo bash -c 'echo "Btrfs recover me" > /mnt/btrfs/lost_btrfs.txt; sync; rm /mnt/btrfs/lost_btrfs.txt; sync'
sudo umount /mnt/btrfs
sudo btrfs check ~/btrfs.img  # No error
sudo btrfs rescue super-recover ~/btrfs.img  # No need
sudo photorec ~/btrfs.img  # Failed, empty
```

### Cleanup

```bash
rm ~/ext4.img ~/xfs.img ~/btrfs.img ./RECOVERED_FILES ~/recup_dir.* ~/xfs_recovered ~/btrfs_recovered
sudo umount /mnt/*
sudo rmdir /mnt/*
```

### Deepen Understanding

- Use backups over recovery.
- Test tools in real scenarios.
- For Btrfs, use snapshots for better recovery.
- CompTIA Tip: Know tool scopes—repair vs. undelete.
