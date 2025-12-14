---
layout: post
title: "CompTIA Linux+ Study Guide: Comprehensive Review - System Management - 1st Batch"
date: 2025-09-13
tags: [Linux+, Modprobe]
---

This study guide organizes key concepts from 25 CompTIA Linux+ practice questions, covering System Management, Networking, Security, Scripting, and Virtualization domains. It serves as a centralized reference for retention and connection of knowledge, with mnemonics, practice tasks, and a review quiz to reinforce learning. The guide addresses discrepancies in questions 15, 20, and 21 to clarify potential exam nuances. Quiz answers are provided in a separate section for self-assessment, with answer options randomized to avoid predictability.

## 1. GRUB 2 Rescue Prompt

**Objective**: Troubleshoot boot issues using GRUB 2 rescue mode.  
**Scenario**: System fails to boot, drops to GRUB rescue prompt.  
**Correct Command Sequence**:

```
set root=(hd0,gpt1)
linux (hd0,gpt1)/vmlinuz root=/dev/sda1 ro
initrd (hd0,gpt1)/initrd.img
boot
```

**Key Points**:

- `set root=(hd0,gpt1)`: Sets root partition (first GPT partition on first disk).
- `linux`: Loads kernel with root device and read-only (`ro`).
- `initrd`: Loads initial ramdisk.
- `boot`: Initiates boot process.  
  **Mnemonic**: GRUB as a recipe: **set the table** (`set root`), **cook the main dish** (`linux`), **add the side** (`initrd`), **serve** (`boot`).  
  **Example Command**: As above.  
  **Practice**: In a VM, edit GRUB config to simulate failure, boot from rescue prompt.

## 2. Rebuilding initramfs for RAID HBA Driver

**Objective**: Configure initramfs to include drivers for booting.  
**Scenario**: RISC-V server fails POST due to missing RAID driver.  
**Correct Utility**: `dracut`  
**Key Points**:

- `dracut`: Modern tool to rebuild initramfs, includes drivers via `--add-drivers`.
- `mkinitrd`: Legacy, less common.
- `depmod`, `modprobe`: Handle modules, not initramfs creation.  
  **Mnemonic**: `dracut` packs a “suitcase” (initramfs) for the boot journey.  
  **Example Command**: `dracut -f /boot/initramfs-$(uname -r).img $(uname -r)`  
  **Practice**: Rebuild initramfs in a VM, add a driver, verify with `lsinitrd`.

## 3. Kernel Boot Parameters

**Objective**: View kernel runtime parameters.  
**Scenario**: Display parameters for current custom kernel session.  
**Correct File**: `/proc/cmdline`  
**Key Points**:

- `/proc/cmdline`: Shows kernel parameters (e.g., `root=/dev/sda1 ro`).
- `/proc/modules`: Lists loaded modules.
- `/etc/default/grub`: Configures future boots, not current.  
  **Mnemonic**: `/proc/cmdline` is the kernel’s “flight plan.”  
  **Example Command**: `cat /proc/cmdline`  
  **Practice**: Boot a VM with custom parameters (e.g., `quiet`), verify with `cat /proc/cmdline`.

## 4. Unloading a USB Storage Module

**Objective**: Manage kernel modules, troubleshoot in-use modules.  
**Scenario**: Unload misbehaving `usb_storage` module in use.  
**Correct Sequence**: `umount /media/usb; modprobe -r usb_storage`  
**Key Points**:

- `umount`: Releases the module by unmounting filesystems.
- `modprobe -r`: Safely removes module and dependencies.
- `rmmod`: Fails if module is in use.  
  **Mnemonic**: Unload luggage: **clear space** (`umount`), **remove bag** (`modprobe -r`).  
  **Example Commands**:

```
lsblk
umount /media/usb
modprobe -r usb_storage
lsmod | grep usb_storage
```

**Practice**: Mount a USB, load `usb_storage`, unmount, and remove module.

## 5. Partitioning

**Objective**: Manage disk partitions.  
**Scenario**: List partitions on `/dev/sdb` for storage configuration.  
**Correct Command**: `fdisk -l /dev/sdb`  
**Key Points**:

- `fdisk -l`: Lists partition table for `/dev/sdb`.
- `parted`, `lsblk`: Alternative tools, less specific.  
  **Mnemonic**: `fdisk -l` lists “sdb’s layout.”  
  **Example Command**: `sudo fdisk -l /dev/sdb`  
  **Practice**: In a VM, attach a disk, run `fdisk -l /dev/sdb`, create a partition.

## 6. Display Volume Group Details

**Objective**: Administer LVM volume groups.  
**Scenario**: View details of `vgdata` volume group.  
**Correct Command**: `vgdisplay vgdata`  
**Key Points**:

- `vgdisplay`: Shows VG details (size, extents, PVs).
- `vgs`: Summary, less detail.
- `pvdisplay`: For physical volumes.  
  **Mnemonic**: `vgdisplay` views “vgdata’s volume details.”  
  **Example Command**: `vgdisplay vgdata`  
  **Practice**: Create a VG (`vgcreate vgdata /dev/sdb1`), run `vgdisplay`.

  **Edit - 12/14/2025**:

  - Note 1: Why VG and not LV or PV? The system notifies you about the "VG" (vgdata) being extended, and not the "PV" (/dev/sdd2) or a specific "LV" (Logical Volume), because the Volume Group is the central, organizational resource pool in LVM.
  - Note 2: After vgextend, the LV still needs to be resized with `lvresize` or `lvextend` to utilize the new space, and then the filesystem on that LV must be resized (e.g., with `resize2fs` for ext4: `resize2fs /dev/vgdata/lvdata`; with `xfs_growfs` for XFS: `xfs_growfs /mnt/lvdata`. Note that resize2fs requires device path, while xfs_growfs requires mount path).
  - Note 3: Some newer LVM versions may automatically resize the LV and filesystem when the VG is extended, but it's best practice to manually verify and perform these steps to ensure data integrity. We can also use "lvextend -L +100M /dev/vgdata/lvdata --resizefs" to extend and resize in one command for ext3, ext4, and even xfs filesystems.

## 7. List Logical Volumes

**Objective**: Manage logical volumes in LVM.  
**Scenario**: List logical volumes in `vgdata`.  
**Correct Command**: `lvs vgdata`  
**Key Points**:

- `lvs`: Lists LVs (name, size, attributes).
- `lvdisplay`: More verbose, less concise.  
  **Mnemonic**: `lvs` lists “vgdata’s logical volumes.”  
  **Example Command**: `lvs vgdata`  
  **Practice**: Create an LV (`lvcreate -L 1G -n lvhome vgdata`), verify with `lvs`.

## 8. Re-Add Drive to RAID

**Objective**: Manage RAID arrays.  
**Scenario**: Re-add `/dev/sdc1` to degraded `/dev/md0`.  
**Correct Command**: `mdadm --add /dev/md0 /dev/sdc1`  
**Key Points**:

- `mdadm --add`: Adds a disk to an existing RAID array.
- `--assemble`, `--create`: For different RAID tasks.  
  **Mnemonic**: `mdadm --add` adds a drive to the “RAID team.”  
  **Example Command**: `sudo mdadm --add /dev/md0 /dev/sdc1`  
  **Practice**: Create a RAID1 array, fail a disk, re-add with `mdadm`.

## 9. NFS Mount Options

**Objective**: Configure NFS mounts for reliability.  
**Scenario**: Mount NFS without delaying boot if NAS is offline.  
**Correct Options**: `nofail,x-systemd.automount`  
**Key Points**:

- `nofail`: Boots even if mount fails.
- `x-systemd.automount`: Mounts on demand.  
  **Mnemonic**: “No-stress, patient NFS waiter.”  
  **Example Command**: Edit `/etc/fstab`:

```
/nas:/share /mnt/nfs nfs nofail,x-systemd.automount 0 0
```

**Practice**: Add NFS mount to `/etc/fstab`, test with NAS offline.

## 10. Inode Usage

**Objective**: Monitor filesystem inode usage.  
**Scenario**: Display inode usage for `/`.  
**Correct Command**: `df -iH /`  
**Key Points**:

- `df -i`: Shows inode usage.
- `-H`: Human-readable format.
- `du --inodes`: Counts inodes in directories, not filesystems.  
  **Mnemonic**: `df -i` inquires about “inodes.”  
  **Example Command**: `df -iH /`  
  **Practice**: Run `df -iH /`, create files, recheck inode changes.

## 11. Flush DNS Cache

**Objective**: Manage DNS resolution.  
**Scenario**: Clear `systemd-resolved` cache.  
**Correct Command**: `resolvectl flush-caches`  
**Key Points**:

- `resolvectl flush-caches`: Clears DNS cache for `systemd-resolved`.
- `rndc flush`: For BIND, not `systemd-resolved`.  
  **Mnemonic**: `resolvectl flush-caches` flushes “old DNS mail.”  
  **Example Command**: `sudo resolvectl flush-caches`  
  **Practice**: Resolve a domain, flush cache, re-resolve to verify.

## 12. Hosts File Entry

**Objective**: Configure static DNS mappings.  
**Scenario**: Add `test.lab` mapping to 10.10.5.20 in `/etc/hosts`.  
**Correct Entry**: `10.10.5.20 test.lab`  
**Key Points**:

- Format: `IP hostname [aliases]`.
- Place at file end to avoid overrides.  
  **Mnemonic**: `/etc/hosts` is a “phonebook for test.lab.”  
  **Example Command**: `echo "10.10.5.20 test.lab" | sudo tee -a /etc/hosts`  
  **Practice**: Add entry, ping `test.lab`, verify resolution.

## 13. Test Netplan Configuration

**Objective**: Manage network configurations with Netplan.  
**Scenario**: Test network changes with 120-second rollback.  
**Correct Command**: `netplan try`  
**Key Points**:

- `netplan try`: Tests config, reverts if not confirmed.
- `netplan apply`: Applies without rollback.  
  **Mnemonic**: `netplan try` tries on a “network outfit.”  
  **Example Command**: `sudo netplan try`  
  **Practice**: Edit `/etc/netplan/*.yaml`, test with `netplan try`.

## 14. Allow Established SSH Connections

**Objective**: Configure firewall rules with iptables.  
**Scenario**: Prevent SSH session drops after five minutes.  
**Correct Rule**: `iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`  
**Key Points**:

- `-m state --state ESTABLISHED,RELATED`: Allows ongoing and related connections.
- Requires separate rule for new SSH connections (`-p tcp --dport 22`).  
  **Mnemonic**: “VIP pass for SSH.”  
  **Example Command**: `sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`  
  **Practice**: Set up iptables, test SSH session persistence.

## 15. Diagnose DHCP Failure

**Objective**: Troubleshoot network configuration.  
**Scenario**: Check DHCP leases managed by `systemd-networkd` with iproute2.  
**Correct Command**: `ip addr show` _(Question constraint)_  
**Key Points**:

- `ip addr show`: Shows assigned IPs, indicating DHCP success.
- Book’s answer: `networkctl status eth0` (not iproute2, but shows lease details).  
  **Mnemonic**: `ip addr show` shows “DHCP’s address ticket.”  
  **Example Command**: `ip addr show eth0`  
  **Practice**: Configure `systemd-networkd` for DHCP, check with `ip addr show`.

## 16. Replace Text In-Place

**Objective**: Perform text manipulation with sed.  
**Scenario**: Replace “dev” with “prod” in `file.txt`, log errors.  
**Correct Command**: `sed -i 's/dev/prod/g' file.txt 2> err.log`  
**Key Points**:

- `-i`: In-place editing.
- `s/dev/prod/g`: Global substitution.
- `2> err.log`: Redirects stderr.  
  **Mnemonic**: `sed -i` “instantly swaps dev for prod.”  
  **Example Command**: `sed -i 's/dev/prod/g' file.txt 2> err.log`  
  **Practice**: Create `file.txt`, run `sed`, verify changes and errors.

## 17. Count Unique Columns

**Objective**: Process text with command-line tools.  
**Scenario**: Count unique second-column values in `/etc/passwd`.  
**Correct Command**: `cut -d: -f2 /etc/passwd | sort | uniq -c`  
**Key Points**:

- `cut -d: -f2`: Extracts second field.
- `sort | uniq -c`: Sorts and counts unique lines.  
  **Mnemonic**: “Cut, sort, count passwd slices.”  
  **Example Command**: `cut -d: -f2 /etc/passwd | sort | uniq -c`  
  **Practice**: Run command, verify password field counts.

## 18. Append to MOTD with Heredoc

**Objective**: Use shell scripting for file manipulation.  
**Scenario**: Append three lines to `/etc/motd` using a heredoc.  
**Correct Command**: `tee -a /etc/motd <<'EOF' Line1 Line2 Line3 EOF` (with Line1, Line2, Line3, and EOF are on 4 different lines)  
**Key Points**:

- `tee -a`: Appends to file.
- `<<'EOF'`: Heredoc for multi-line input.  
  **Mnemonic**: `tee -a <<'EOF'` “tees off a message to motd.”  
  **Example Command**: `sudo tee -a /etc/motd <<'EOF' Line1 Line2 Line3 EOF`  
  **Practice**: Append lines, verify with `cat /etc/motd`.

## 19. Find Sticky Bit Files

**Objective**: Manage file permissions.  
**Scenario**: Find files in `/tmp` with sticky bit recursively.  
**Correct Command**: `find /tmp -type f -perm -1000`  
**Key Points**:

- `-type f`: Regular files only.
- `-perm -1000`: Matches sticky bit (octal 1000).  
  **Mnemonic**: `find -perm -1000` finds “sticky files in tmp.”  
  **Example Command**: `find /tmp -type f -perm -1000`  
  **Practice**: Create a sticky bit file, run `find`, verify with `ls -l`.

## 20. Compress Logs with Permissions

**Objective**: Perform backups with tar.  
**Scenario**: Compress `/var/log/*` to `/backup/logs.tgz`, preserving permissions.  
**Correct Command**: `tar -czpf /backup/logs.tgz /var/log`  
**Key Points**:

- `-c`: Create archive.
- `-z`: Gzip compression (default level).
- `-p`: Preserve permissions.
- Discrepancy: Question asked for “maximum” compression, but no option used `GZIP=-9`.  
  **Mnemonic**: `tar -czpf` “zips logs tightly.”  
  **Example Command**: `sudo tar -czpf /backup/logs.tgz /var/log`  
  **Practice**: Create `/backup`, run `tar`, verify with `tar -tzf`.

## 21. Mirror /etc with Rsync

**Objective**: Synchronize files with rsync.  
**Scenario**: Mirror `/etc` to `/backup/etc`, delete removed files, skip device files.  
**Correct Command**: `rsync -av --delete /etc/ /backup/etc` _(Adjusted per book)_  
**Key Points**:

- `-a`: Archive mode (preserves attributes).
- `--delete`: Removes destination files not in source.
- Rsync skips device files by default (no `--devices`).
- Discrepancy: Book’s `--devices` includes device files, contradicting requirement.  
  **Mnemonic**: `rsync -av --delete` “mirrors a room, skips gadgets.”  
  **Example Command**: `sudo rsync -av --delete /etc/ /backup/etc`  
  **Practice**: Mirror `/etc`, verify with `ls -l /backup/etc`.

## 22. Repair ext Filesystem

**Objective**: Maintain filesystems with fsck.  
**Scenario**: Repair corrupted ext filesystem on `/dev/vgdata/lvhome`.  
**Correct Command**: `fsck -f -y /dev/vgdata/lvhome`  
**Key Points**:

- `-f`: Forces full check.
- `-y`: Automatic repairs.
- `e2fsck -p`: Repairs only if dirty, not forced.  
  **Mnemonic**: `fsck -f -y` “forces fixes for ext.”  
  **Example Command**: `sudo fsck -f -y /dev/vgdata/lvhome`  
  **Practice**: Corrupt an ext4 LV, repair with `fsck`, mount to verify.

## 23. Restore LVM Metadata

**Objective**: Recover LVM configurations.  
**Scenario**: Restore accidentally removed LV from `/etc/lvm/archive`.  
**Correct Command**: `vgcfgrestore -f vgdata`  
**Key Points**:

- `vgcfgrestore -f`: Restores VG metadata from backup file.
- Assumes specific file (e.g., `/etc/lvm/archive/vgdata_XXXX.vg`).  
  **Mnemonic**: `vgcfgrestore` “restores the VG blueprint.”  
  **Example Command**: `sudo vgcfgrestore -f /etc/lvm/archive/vgdata_00001.vg vgdata`  
  **Practice**: Delete an LV, restore with `vgcfgrestore`, verify with `lvs`.

## 24. Offline Disk Backup

**Objective**: Back up disks with ddrescue.  
**Scenario**: Perform offline backup of `/dev/sdb` with a log file.  
**Correct Command**: `ddrescue /dev/sdb /backup/disk.img /backup/disk.log`  
**Key Points**:

- `ddrescue`: Copies disk, logs progress/errors.
- Syntax: `[source] [destination] [logfile]`.  
  **Mnemonic**: `ddrescue` is a “disk rescuer with logbook.”  
  **Example Command**: `sudo ddrescue /dev/sdb /backup/disk.img /backup/disk.log`  
  **Practice**: Unmount `/dev/sdb`, run `ddrescue`, verify image and log.

## 25. Create qcow2 Disk

**Objective**: Manage virtual machine storage.  
**Scenario**: Create a 20 GiB qcow2 disk with metadata preallocation.  
**Correct Command**: `qemu-img create -f qcow2 -o preallocation=metadata disk.qcow2 20G`  
**Key Points**:

- `-f qcow2`: Specifies qcow2 format.
- `-o preallocation=metadata`: Preallocates metadata for efficiency.  
  **Mnemonic**: `qemu-img create` “preps a qcow2 canvas.”  
  **Example Command**: `qemu-img create -f qcow2 -o preallocation=metadata disk.qcow2 20G`  
  **Practice**: Create disk, verify with `qemu-img info`, attach to KVM VM.

## Plan for Future Questions

- **Batch Processing**: This guide covers the first 25-question batch. Future questions will create new artifacts or update this one with a new UUID, ensuring continuity.
- **Connections**: Questions link across domains:
  - Storage (Q5–Q8, Q22–Q24) supports filesystems for `/etc` (Q12, Q21), `/var/log` (Q20), and VMs (Q25).
  - Networking (Q9, Q11–Q13, Q15) enables NFS and SSH (Q14).
  - Scripting (Q16–Q17) edits configs (Q12, Q13, Q18).
  - Security (Q14, Q19) protects network and filesystem operations.
- **Retention Strategy**: Use mnemonics daily, practice labs weekly, and review quiz answers to reinforce connections.

## Discrepancy Notes

1. **Q15 (DHCP)**: Question required an `iproute2` command (`ip addr show`), but the book’s answer (`networkctl status eth0`) provides detailed DHCP lease info, not part of `iproute2`. Exam may test tool specificity.
2. **Q20 (Tar)**: Question asked for “maximum” compression, but no option used `GZIP=-9`. Book’s `tar -czpf` uses default compression, suggesting the question accepts level 6.
3. **Q21 (Rsync)**: Book’s `rsync -av -delete --devices` includes device files, contradicting the requirement to skip them. Correct command omits `--devices`, as `rsync` skips device files by default.
