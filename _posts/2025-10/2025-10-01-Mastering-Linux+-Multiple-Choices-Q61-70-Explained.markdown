---
layout: post
title: "Multiple-choices - Questions 61-70 Explained"
date: 2025-10-01 10:45:00 -0700
tags: [Linux+, Multiple-choices]
---

This document explains questions 61-70 from a set of 100 scenario-based multiple-choices questions for CompTIA Linux+ preparation, focusing on compression, backups, file synchronization, filesystems, and virtualization. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention. Question 63 has been updated to clarify the `-e` option in the `rsync` command.

## Question 61: Compressing a Log File with Gzip

**Question**: An administrator has a large, uncompressed log file named `access.log` and wants to compress it to save space, using the `gzip` utility. What will be the name of the resulting file after running `gzip access.log`?  
**Options**:

- access.log (the original file is compressed in place)
- access.gz
- access.log.gz
- access.zip

**Correct Answer**: access.log.gz  
**Why Correct**: The `gzip` command compresses `access.log`, replacing it with a compressed file named `access.log.gz` (appending `.gz`). The original file is not kept unless `gzip -k` (keep) is used.  
**Why Others Wrong**:

- `access.log` (in place) is incorrect; `gzip` renames the file with `.gz`.
- `access.gz` is incorrect; `gzip` retains the original filename base.
- `access.zip` is incorrect; `gzip` produces `.gz`, not `.zip`.  
  **Key Concept**: `gzip` compresses a single file, adding `.gz`; use `gunzip` to decompress.  
  **Memory Aid**: “gzip = Add .gz to filename.”

## Question 62: Creating a Raw Disk Image

**Question**: A critical configuration file was accidentally deleted. Fortunately, a full backup of the filesystem exists as a raw disk image file named `disk.img`. Which utility is commonly used to create a bit-for-bit copy of a block device or file, making it useful for both backup and recovery?  
**Options**:

- cp
- rsync
- dd
- tar

**Correct Answer**: dd  
**Why Correct**: `dd` creates a bit-for-bit copy of a block device or file (e.g., `dd if=/dev/sda of=disk.img`), ideal for raw disk images used in backups and recovery.  
**Why Others Wrong**:

- `cp` copies files but not raw block devices.
- `rsync` synchronizes files, not bit-for-bit copies.
- `tar` archives files, not raw devices.  
  **Key Concept**: `dd` is low-level; use `if=` (input file) and `of=` (output file).  
  **Memory Aid**: “dd = Direct disk copy.”

## Question 63: Synchronizing Directories Over SSH

**Question**: An administrator needs to synchronize the contents of a local directory, `/srv/www`, with a remote directory on a backup server. The goal is to only transfer files that have changed, do it securely over SSH, and preserve permissions. Which command is best suited for this task?  
**Options**:

- scp -r /srv/www user@backup:/srv/www
- rsync -avz -e ssh /srv/www/ user@backup:/srv/www/
- tar -cz /srv/www | ssh user@backup "cd /srv/www && tar -xz"
- ftp -s /srv/www user@backup

**Correct Answer**: rsync -avz -e ssh /srv/www/ user@backup:/srv/www/  
**Why Correct**: The `rsync` command with `-avz -e ssh` efficiently synchronizes `/srv/www` to the remote server, transferring only changed files. Here’s what each option does:

- `-a`: **Archive** mode preserves permissions, timestamps, and symbolic links.
- `-v`: **Verbose** output shows transfer details.
- `-z`: Enables **compression** to reduce data transfer over the network.
- `-e ssh`: Specifies the **remote shell** as `ssh`, ensuring secure transport over SSH.  
  The trailing `/` on `/srv/www/` ensures only the directory’s contents are synced, not the directory itself.  
  **Why Others Wrong**:
- `scp -r` copies all files recursively, not just changed ones, and is slower and less efficient.
- `tar -cz | ssh` archives and transfers files but doesn’t sync incrementally, requiring full transfers each time.
- `ftp -s` is insecure (unencrypted) and not suitable for directory synchronization.  
  **Key Concept**: `rsync` is optimal for incremental transfers; `-a` preserves file attributes, `-e ssh` ensures secure SSH transport, and the trailing `/` syncs directory contents.  
  **Memory Aid**: “rsync -avz -e ssh = Archive, verbose, zipped, secure sync.”

## Question 64: Extracting a Zip Archive

**Question**: A user has a compressed file `archive.zip`. Which command-line utility is used to extract the contents of a zip archive?  
**Options**:

- gunzip
- unrar
- unzip
- tar -xf

**Correct Answer**: unzip  
**Why Correct**: The `unzip` command extracts files from a `.zip` archive (e.g., `unzip archive.zip`).  
**Why Others Wrong**:

- `gunzip` decompresses `.gz` files, not `.zip`.
- `unrar` extracts `.rar` archives, not `.zip`.
- `tar -xf` works for `.tar` archives, not `.zip`.  
  **Key Concept**: Use `unzip` for `.zip` files; `zip` to create them.  
  **Memory Aid**: “unzip = Unpack zip files.”

## Question 65: Listing Contents of a Tar Archive

**Question**: An administrator wants to list the contents of a compressed archive `project.tar.gz` without extracting the files to the disk. Which `tar` command should be used?  
**Options**:

- tar -xvf project.tar.gz
- tar -cvf project.tar.gz
- tar -tvf project.tar.gz
- tar -rf project.tar.gz

**Correct Answer**: tar -tvf project.tar.gz  
**Why Correct**: `tar -tvf` lists (`-t`) the contents of `project.tar.gz` (`-f`) with verbose details (`-v`), handling gzip compression automatically.  
**Why Others Wrong**:

- `tar -xvf` extracts files, not lists them.
- `tar -cvf` creates an archive, not lists it.
- `tar -rf` appends files to an archive, not lists them.  
  **Key Concept**: `tar -t` lists archive contents; `-z` is implied for `.tar.gz`.  
  **Memory Aid**: “tar -tvf = Table of verbose files.”

## Question 66: Checking and Repairing Filesystem

**Question**: After an improper shutdown, an administrator suspects filesystem corruption on an unmounted Ext4 partition `/dev/sdb1`. Which command should be run to check for and interactively repair errors?  
**Options**:

- mount -o remount,ro /dev/sdb1
- fsck /dev/sdb1
- debugfs /dev/sdb1
- mkfs.ext4 -c /dev/sdb1

**Correct Answer**: fsck /dev/sdb1  
**Why Correct**: `fsck /dev/sdb1` checks and interactively repairs errors on the unmounted Ext4 partition `/dev/sdb1`. It prompts for fixes if issues are found.  
**Why Others Wrong**:

- `mount -o remount,ro` mounts read-only, not repairs.
- `debugfs` is for low-level filesystem debugging, not routine checks.
- `mkfs.ext4 -c` formats the partition, destroying data.  
  **Key Concept**: Run `fsck` on unmounted partitions; use `-y` for automatic repairs.  
  **Memory Aid**: “fsck = Filesystem check and fix.”

## Question 67: Highest Compression Ratio

**Question**: To achieve the highest possible compression ratio for archiving old log files, even if it takes more CPU time, which compression utility is generally considered the most effective among common Linux tools?  
**Options**:

- gzip
- bzip2
- xz
- zip

**Correct Answer**: xz  
**Why Correct**: `xz` offers the highest compression ratio among common Linux tools, using the LZMA algorithm, though it’s slower and CPU-intensive.  
**Why Others Wrong**:

- `gzip` is faster but less effective than `xz`.
- `bzip2` compresses better than `gzip` but less than `xz`.
- `zip` is less effective than `xz` and not native for tar archives.  
  **Key Concept**: `xz` for max compression; `tar -J` for `.tar.xz`.  
  **Memory Aid**: “xz = Extreme zip.”

## Question 68: Backing Up Partition Table

**Question**: An administrator needs to back up the partition table of the `/dev/sda` disk as a data preservation measure before making changes. Which command can be used to save the MBR, including the partition table?  
**Options**:

- fdisk -l /dev/sda > partition.txt
- dd if=/dev/sda of=sda_mbr.bak bs=512 count=1
- parted /dev/sda print > partition.txt
- tar -czvf sda.tar.gz /dev/sda

**Correct Answer**: dd if=/dev/sda of=sda_mbr.bak bs=512 count=1  
**Why Correct**: `dd if=/dev/sda of=sda_mbr.bak bs=512 count=1` copies the first 512 bytes of `/dev/sda` (the MBR, including the partition table) to `sda_mbr.bak`.  
**Why Others Wrong**:

- `fdisk -l` lists partitions in text, not a raw backup.
- `parted print` outputs partition info, not a restorable copy.
- `tar -czvf` is for files, not raw device backups.  
  **Key Concept**: MBR is 512 bytes; `dd` with `bs=512 count=1` backs it up.  
  **Memory Aid**: “dd MBR = 512 bytes to safety.”

## Question 69: Running Isolated Linux Environments

**Question**: A company wants to run multiple isolated Linux server environments on a single physical server to maximize hardware utilization. Which technology allows for the creation of multiple virtual machines, each with its own virtualized hardware and operating system?  
**Options**:

- Logical Volume Manager (LVM)
- Software RAID
- Virtualization (using a hypervisor)
- Containerization (using Docker)

**Correct Answer**: Virtualization (using a hypervisor)  
**Why Correct**: Virtualization with a hypervisor (e.g., KVM, VMware) creates virtual machines, each with its own virtualized hardware and OS, ensuring full isolation.  
**Why Others Wrong**:

- LVM manages disk partitions, not VMs.
- Software RAID manages redundant storage, not VMs.
- Containerization (Docker) shares the host OS, not fully isolated.  
  **Key Concept**: Hypervisors emulate hardware; containers share the kernel.  
  **Memory Aid**: “Virtualization = Full VM isolation.”

## Question 70: Type of Hypervisor

**Question**: An administrator is setting up a new server to be a dedicated virtualization host. They install a hypervisor like KVM directly onto the hardware, which then manages the guest VMs. What type of hypervisor is this?  
**Options**:

- Type 1 (Bare-metal)
- Type 2 (Hosted)
- Hybrid Hypervisor
- Container Engine

**Correct Answer**: Type 1 (Bare-metal)  
**Why Correct**: A Type 1 (bare-metal) hypervisor like KVM runs directly on hardware, managing guest VMs without a host OS, ideal for dedicated virtualization hosts.  
**Why Others Wrong**:

- Type 2 (hosted) runs on top of a host OS (e.g., VirtualBox).
- Hybrid hypervisor is not a standard term.
- Container engine (e.g., Docker) is not a hypervisor.  
  **Key Concept**: Type 1 is bare-metal (KVM, Hyper-V); Type 2 is hosted (VirtualBox).  
  **Memory Aid**: “Type 1 = Bare-metal, no host OS.”

## Retention Tips for Questions 61-70

- **Themes**: Compression (`gzip`, `xz`), backups (`dd`), synchronization (`rsync`), archive management (`unzip`, `tar`), filesystem repair (`fsck`), and virtualization (hypervisors).
- **Mnemonic for Backup/Compression**: “Gzip for quick, XZ for max, DD for raw” (G-X-D).
- **Practice**: In a Linux VM:
  - Compress a file with `gzip` and `xz`, list a `tar.gz` with `tar -tvf`.
  - Back up a partition table with `dd`, sync directories with `rsync -avz`.
  - Check an unmounted partition with `fsck`.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “Command to list tar.gz contents?” → “tar -tvf.”
- **Quiz Yourself**: What’s the output file of `gzip access.log`? Best tool for incremental sync over SSH? Role of `-e` in `rsync`? (Answer: Specifies SSH as remote shell.)
