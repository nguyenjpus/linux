---
layout: post
title: "Rsync and ddrescue"
date: 2025-09-06
tags: [Basic, Rsync, Ddrescue]
---

Detailed hands-on practices for mastering file-level backups using `rsync` and block-level backups using `ddrescue`, based on the CompTIA Linux+ certification objectives. It expands on the original book content by Fermin O. Goetz, incorporating additional scenarios, troubleshooting tips, and best practices.

## Goal

Master file-level backups using `rsync` for efficient synchronization and incremental updates, and block-level backups using `ddrescue` for cloning disks or images while handling errors gracefully. This covers scenarios like local backups, remote transfers, resuming interrupted operations, and dealing with faulty hardware.

## Prerequisites

- Ensure `rsync` and `ddrescue` are installed (e.g., on Ubuntu: `sudo apt install rsync ddrescue`; on Fedora: `sudo dnf install rsync gddrescue`).
- Work in a non-root directory to avoid permissions issues, but use `sudo` where necessary for mounting.
- Snapshot your VM or use a test environment to prevent data loss.
- Optional: Install `tree` for directory visualization (`sudo apt install tree`).

## Step-by-Step Expanded Practices

### Prepare Test Data (Enhanced Setup)

1. Create a dedicated test directory:
   ```bash
   mkdir ~/backup_test && cd ~/backup_test
   ```
2. Generate diverse files to simulate real data:
   - Text file: `echo "Important confidential data" > data.txt`
   - Binary file: `dd if=/dev/urandom of=binary.dat bs=1M count=5` (creates a 5MB random binary file)
   - Subdirectory with nested files: `mkdir -p sub/nested && echo "Nested file content" > sub/nested/deep.txt`
   - Symlink: `ln -s data.txt symlink.txt`
   - Hard link: `ln data.txt hardlink.txt`
   - File with special permissions: `touch special.txt && chmod 600 special.txt` (owner-only read/write)
   - Large file for transfer testing: `dd if=/dev/zero of=largefile.img bs=1M count=50` (50MB zero-filled file)
3. Verify structure: `tree .` or `ls -R`

### File-Level Backup with rsync (Local and Remote Scenarios)

1. Create a local backup destination:
   ```bash
   mkdir ~/backup_dest
   ```
2. Perform initial sync preserving attributes:
   ```bash
   rsync -aAXH --delete --numeric-ids --info=progress2 ~/backup_test/ ~/backup_dest/
   ```
   - Flags explained: `-a` (archive mode: recursive, preserve symlinks, permissions, times, etc.), `-A` (ACLs), `-X` (extended attributes), `-H` (hard links), `--delete` (remove files in dest not in source), `--numeric-ids` (use numeric user/group IDs), `--info=progress2` (show progress).
   - Note: Trailing slashes on directories ensure content sync, not the dir itself.
3. Verify backup integrity:
   - Compare: `diff -r ~/backup_test/ ~/backup_dest/` (should show no differences)
   - Check permissions: `ls -l data.txt` and `ls -l ~/backup_dest/data.txt` (should match)
   - For hard links: `ls -i hardlink.txt data.txt` (inode numbers should match in backup if preserved)
4. Simulate data changes:
   - Modify: `echo "Updated important data" > data.txt`
   - Add: `echo "New file added" > newfile.txt`
   - Delete: `rm sub/nested/deep.txt`
   - Move: `mv symlink.txt moved_symlink.txt`
5. Resync and observe incremental behavior: Repeat the rsync command—only changes transfer.
   - Use `--dry-run` first: `rsync -aAXH --delete --numeric-ids --dry-run ~/backup_test/ ~/backup_dest/` (preview changes without applying)
   - To see output in dry-run: Add `-v` (verbose): `rsync -avAXH --delete --numeric-ids --dry-run ~/backup_test/ ~/backup_dest/`
6. Remote backup simulation (over SSH):
   - Assume a remote server (e.g., set up a loopback or use a VM). For local test: `ssh-keygen` and `ssh-copy-id localhost` for passwordless SSH.
   - Command: `rsync -aAXH -e ssh --delete --numeric-ids ~/backup_test/ user@localhost:~/remote_backup/`
   - Verify remotely: `ssh user@localhost 'ls -R ~/remote_backup'`
   - Advanced: Use specific key: `rsync -aAXH -e "ssh -i ~/.ssh/id_rsa_backup" --delete --numeric-ids ~/backup_test/ user@localhost:~/remote_backup/`
7. Handling slow or interrupted links:
   - Simulate large transfer: Add more data, e.g., `dd if=/dev/zero of=very_large.img bs=1M count=200` in backup_test.
   - Start transfer: `rsync -aAXH --delete --numeric-ids ~/backup_test/ ~/backup_dest/`
   - Interrupt with Ctrl+C midway.
   - Resume: `rsync -aAXH --delete --numeric-ids --partial --inplace --append-verify ~/backup_test/ ~/backup_dest/`
     - `--partial` (keep partial files), `--inplace` (update in place), `--append-verify` (resume and check integrity)
8. Advanced options:
   - Exclude patterns: `rsync -aAXH --delete --numeric-ids --exclude='*.img' --exclude='sub/' ~/backup_test/ ~/backup_dest/` (skip large files or dirs)
   - Bandwidth limit: `--bwlimit=100` (limit to 100 KB/s for testing slow links)
   - Compression: `-z` for remote transfers to reduce data over wire
   - Logging: `--log-file=rsync.log` to track operations

### Block-Level Backup with ddrescue (Error Handling and Recovery)

1. Create a virtual disk image:
   ```bash
   dd if=/dev/zero of=~/testdisk.img bs=1M count=200
   ```
   (200MB for more space)
2. Format and populate:
   ```bash
   sudo mkfs.ext4 ~/testdisk.img
   sudo mkdir /mnt/testdisk && sudo mount -o loop ~/testdisk.img /mnt/testdisk
   sudo cp -r ~/backup_test/* /mnt/testdisk/
   sudo sync && sudo umount /mnt/testdisk
   ```
   - If space issues: Check `du -sh ~/backup_test/` and ensure < ~180MB usable space. Increase image size if needed (e.g., count=500).
3. Initial clone:
   ```bash
   ddrescue -d -r3 ~/testdisk.img ~/backup.img ~/ddrescue_logfile
   ```
   - Flags: `-d` (direct disk access for speed), `-r3` (retry bad sectors 3 times)
   - Monitor progress: `tail -f ~/ddrescue_logfile`
4. Verify clone:
   ```bash
   sudo mount -o loop ~/backup.img /mnt/testdisk && ls -R /mnt/testdisk
   sudo umount /mnt/testdisk
   ```
5. Simulate disk failure:
   - Corrupt sectors: `dd if=/dev/zero of=~/testdisk.img bs=1M count=1 seek=50 conv=notrunc` (overwrites 1MB at offset 50MB)
   - Add more bad areas: `dd if=/dev/zero of=~/testdisk.img bs=512 count=10 seek=100000 conv=notrunc`
6. Rescue from damaged disk:
   ```bash
   ddrescue -d -r3 -c1KiB ~/testdisk.img ~/rescued.img ~/ddrescue_logfile
   ```
   - `-c1KiB` (smaller cluster size for precise recovery)
   - If interrupted, resume with the same command (uses logfile)
7. Multi-pass strategy:
   - First pass: `ddrescue -n ~/testdisk.img ~/rescued.img ~/ddrescue_logfile` (no splitting, fast copy good areas)
   - Second pass: `ddrescue -d -r3 ~/testdisk.img ~/rescued.img ~/ddrescue_logfile` (retry bad areas)
   - Reverse pass: `ddrescue -d -r3 -R ~/testdisk.img ~/rescued.img ~/ddrescue_logfile` (`-R` for reverse)
8. Advanced: Rescue to/from physical devices (caution: use test USB drives).  
   Example: `sudo ddrescue /dev/sda /dev/sdb rescue.log` (clone sda to sdb)

## Cleanup

```bash
rm -rf ~/backup_test ~/backup_dest *.img *.log
sudo umount /mnt/testdisk
sudo rmdir /mnt/testdisk
```

## Deepen Understanding

- Full system backup: Boot from live USB, create LVM snapshot (`sudo lvcreate -L 5G -s -n rootsnap /dev/mapper/rootvg-root`), then `rsync -aAXH --delete /mnt/snapshot/ /mnt/backup/`.
- Versioning with rsync: Use `--backup --backup-dir=../versions/$(date +%Y%m%d)` for dated backups.
- ddrescue with maps: Edit logfile to mark areas as bad/good manually.
- Integration: Use rsync for user data, ddrescue for full disk images in disaster recovery plans.
- <span style="color: red;">NOTE:Common pitfalls: Forgetting `--delete` leads to accumulation; not using `-d` slows ddrescue; always verify mounts. </span>
- Troubleshooting: If mount fails (e.g., "can't read superblock"), check for corruption with `dmesg`, `file ~/testdisk.img`, or recreate the image.
