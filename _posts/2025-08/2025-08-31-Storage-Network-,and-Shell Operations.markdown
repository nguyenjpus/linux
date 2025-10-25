---
layout: post
title: "Storage, Network, and Shell Operations"
date: 2025-08-31
tags: [Basic, Storage]
---

This document summarizes hands-on practice for Linux+ exam Topics 5 (Troubleshooting Storage), 6 (Network Configuration Essentials), and 7 (Advanced Shell Operations) on an Ubuntu 24.04.3 LTS (arm64, QEMU/UTM) system, with some practices tested on a physical Linux PC. It includes corrected commands, fixes for issues encountered, and lessons learned. Some exercises (SMART data and GRUB-based passphrase failure) are noted as infeasible in UTM due to virtualization limitations.

## System Overview

- **OS**: Ubuntu 24.04.3 LTS (arm64, QEMU; some practices on physical PC)
- **Disks** (from `lsblk`):
  - `/dev/vda`: 32GB (partitions: `/boot/efi`, `/boot`, LVM for `/`)
  - `/dev/vdb`, `/dev/vdc`, `/dev/vdd`: 5GB each, forming RAID5 (`/dev/md1`, mounted at `/raiddata`)
  - `/dev/vde`: 5GB, used for LUKS practice
- **Physical PC Disks** (from `lsscsi`):
  - `/dev/sda`: TOSHIBA MG09ACA1 HDD
  - `/dev/nvme0n1`: Samsung PM9A1 NVMe SSD (512GB)
- **Network**: Managed by Netplan
- **Shell**: Bash

## Topic 5: Troubleshooting Storage

Focuses on diagnosing and fixing storage issues, including stale signatures, filesystem errors, SMART data, and LUKS encryption.

### Practice Exercise 1: Handling Stale Device-Mapper Signatures

**Goal**: Clear old metadata (e.g., LVM/RAID) from a disk.
**Steps**:

1. Create a test loopback device:
   ```bash
   sudo dd if=/dev/zero of=/tmp/testdisk.img bs=1M count=100
   sudo losetup -fP /tmp/testdisk.img
   losetup -a  # Confirm /dev/loop0
   ```
2. Simulate a partition table:
   ```bash
   sudo fdisk /dev/loop0
   ```
   - Use `o` (new DOS table), `n` (new partition), `p` (primary), `1`, accept defaults, `w` (write).
3. Check signatures:
   ```bash
   sudo wipefs -n /dev/loop0  # Lists metadata (e.g., "dos")
   ```
4. Clear signatures:
   ```bash
   sudo wipefs --all /dev/loop0
   sudo wipefs -n /dev/loop0  # Should be empty
   ```
5. Clean up:
   ```bash
   sudo losetup -d /dev/loop0
   rm /tmp/testdisk.img
   ```
   **Lessons**:

- Use `wipefs` to diagnose and remove stale LVM/RAID metadata.
- Always verify with `wipefs -n` before wiping.
- Exam tip: Interpret `wipefs -n` output for “LVM2_member” or “linux_raid_member”.

### Practice Exercise 2: Fixing Filesystem Errors (Ext4 and XFS)

**Goal**: Repair ext4 and XFS filesystems.
**Steps**:

1. **Ext4**:
   ```bash
   sudo dd if=/dev/zero of=/tmp/ext4test.img bs=1M count=50
   sudo losetup -fP /tmp/ext4test.img  # e.g., /dev/loop1
   sudo mkfs.ext4 /dev/loop1
   sudo mkdir /tmp/mnt
   sudo mount /dev/loop1 /tmp/mnt
   echo "Test data" | sudo tee /tmp/mnt/testfile.txt
   sudo umount /tmp/mnt
   sudo fsck.ext4 -n /dev/loop1  # Dry-run
   sudo fsck.ext4 -f /dev/loop1  # Force check (answer 'y' if prompted)
   ```
2. **XFS** (install: `sudo apt install xfsprogs`):
   ```bash
   sudo dd if=/dev/zero of=/tmp/xfstest.img bs=1M count=50
   sudo losetup -fP /tmp/xfstest.img  # e.g., /dev/loop2
   sudo mkfs.xfs /dev/loop2
   sudo mount /dev/loop2 /tmp/mnt
   echo "XFS test" | sudo tee /tmp/mnt/xfsfile.txt
   sudo umount /tmp/mnt
   sudo xfs_repair -n /dev/loop2  # Dry-run
   sudo xfs_repair -L /dev/loop2  # Force repair
   ```
3. Clean up:
   ```bash
   sudo losetup -d /dev/loop1 /dev/loop2
   rm /tmp/ext4test.img /tmp/xfstest.img
   sudo rmdir /tmp/mnt
   ```
   **Lessons**:

- Run `fsck.ext4` or `xfs_repair` on unmounted filesystems only.
- Check `dmesg | grep EXT4-fs` for errors like bad inodes.
- Exam tip: Use rescue mode (`rd.break`) for real fixes (infeasible in UTM due to incomplete GRUB).

### Practice Exercise 3: Checking SMART Data

**Note**: Infeasible in UTM/QEMU (virtual disks lack SMART support). Tested on physical PC.
**Steps** (on physical system):

1. Install: `sudo apt install smartmontools`
2. Check HDD (`/dev/sda`):
   ```bash
   sudo smartctl -a -d sat /dev/sda
   sudo smartctl -t short -d sat /dev/sda
   sudo smartctl -l selftest -d sat /dev/sda
   ```
3. Check NVMe (`/dev/nvme0n1`):
   ```bash
   sudo smartctl -a -d nvme /dev/nvme0n1
   ```
   **Lessons**:

- Use `-d sat` for SATA HDDs, `-d nvme` for NVMe SSDs.
- Interpret: Non-zero `Reallocated_Sector_Ct` (HDD) or high `Percentage Used` (NVMe) indicates failure risk.
- Exam tip: Note “SMART not supported” in VMs; focus on command syntax and output interpretation.

### Practice Exercise 4: LUKS Encryption Issues (Using `/dev/vde`)

**Goal**: Set up and troubleshoot a LUKS-encrypted `/dev/vde` (5GB).
**Steps**:

1. Install: `sudo apt install cryptsetup`
2. Create LUKS container:
   ```bash
   sudo cryptsetup luksFormat /dev/vde  # Passphrase: test1234
   sudo cryptsetup luksOpen /dev/vde cryptvde
   sudo mkfs.ext4 /dev/mapper/cryptvde
   sudo mkdir /mnt/cryptvde
   sudo mount /dev/mapper/cryptvde /mnt/cryptvde
   echo "Encrypted data" | sudo tee /mnt/cryptvde/secret.txt
   ```
3. Make persistent:
   ```bash
   sudo blkid /dev/vde  # e.g., UUID="0d9342f9-c94a-405d-ab78-48313a0e16fe"
   sudo nano /etc/crypttab
   # Add: cryptvde UUID="0d9342f9-c94a-405d-ab78-48313a0e16fe" none luks
   sudo blkid /dev/mapper/cryptvde  # e.g., UUID="d2e0fadb-c8a7-4377-8360-545676b9794d"
   sudo nano /etc/fstab
   # Add: UUID="d2e0fadb-c8a7-4377-8360-545676b9794d" /mnt/cryptvde ext4 defaults 0 2
   sudo umount /mnt/cryptvde
   sudo cryptsetup luksClose cryptvde
   sudo systemctl daemon-reload
   sudo systemctl restart systemd-cryptsetup@cryptvde
   sudo mount -a
   ls /mnt/cryptvde
   ```
4. **Fix Persistent Mounting Issue**:
   - Issue: `mount -a` failed with “can’t find UUID” because `/dev/mapper/cryptvde` was closed.
   - Fix:
     ```bash
     sudo cryptsetup luksOpen /dev/vde cryptvde
     sudo mount /mnt/cryptvde
     ```
5. **Scenario 1: Passphrase Failure**:
   - **Note**: Disclaimer: I haven't test this screnario yet, still getting to emergency boot is good to know.
   - **Note**: Infeasible in UTM due to incomplete GRUB menu. On physical system:
     ```bash
     # Boot, press e at GRUB, add rd.break, Ctrl+x
     # In initramfs:
     cryptsetup luksOpen /dev/vde cryptvde  # Try wrong passphrase, then test1234
     exit
     ```
6. **Scenario 2: Header Corruption**:
   ```bash
   sudo cryptsetup luksHeaderBackup /dev/vde --header-backup-file /tmp/vde-header.bak
   sudo dd if=/dev/zero of=/tmp/lukstest2.img bs=1M count=50
   sudo losetup -fP /tmp/lukstest2.img  # e.g., /dev/loop0
   sudo cryptsetup luksFormat /dev/loop0  # Passphrase: test4321
   sudo cryptsetup luksHeaderBackup /dev/loop0 --header-backup-file /tmp/test2-header.bak
   sudo dd if=/dev/zero of=/dev/loop0 bs=1M count=1
   sudo cryptsetup luksOpen /dev/loop0 testcrypt2  # Fails
   sudo cryptsetup luksHeaderRestore /dev/loop0 --header-backup-file /tmp/test2-header.bak
   sudo cryptsetup luksOpen /dev/loop0 testcrypt2
   sudo mkfs.ext4 /dev/mapper/testcrypt2
   sudo mkdir /mnt/testcrypt2
   sudo mount /dev/mapper/testcrypt2 /mnt/testcrypt2
   sudo umount /mnt/testcrypt2
   sudo cryptsetup luksClose testcrypt2
   sudo losetup -d /dev/loop0
   rm /tmp/lukstest2.img /tmp/test2-header.bak
   ```
7. **Scenario 3: Filesystem on LUKS**:
   ```bash
   sudo cryptsetup luksOpen /dev/vde cryptvde
   sudo umount /mnt/cryptvde
   sudo fsck.ext4 -f /dev/mapper/cryptvde
   sudo mount /mnt/cryptvde
   ```
8. **Clean Up**:
   ```bash
   sudo umount /mnt/cryptvde
   sudo cryptsetup luksClose cryptvde
   sudo rmdir /mnt/cryptvde
   sudo nano /etc/crypttab  # Remove cryptvde
   sudo nano /etc/fstab  # Remove /mnt/cryptvde
   ```
   **Lessons**:

- Always back up LUKS headers (`luksHeaderBackup`).
- Use UUIDs in `/etc/crypttab` and `/etc/fstab`.
- Exam tip: Practice `rd.break` for initramfs fixes (requires full GRUB on physical systems).

## Topic 6: Network Configuration Essentials

Covers configuring interfaces, DNS, and hostnames.

### Practice Exercise 1: Using Network Configuration Tools

**Goal**: Configure a static IP with Netplan.
**Steps**:

1. Check interfaces:
   ```bash
   nmcli device show | grep -E "GENERAL.DEVICE|GENERAL.TYPE|GENERAL.STATE|IP4.ADDRESS"
   ```
   - Choose Ethernet interface (e.g., `eth0`, not `lo`) with `connected` state.
2. Backup and edit Netplan:
   ```bash
   sudo cp /etc/netplan/*.yaml /tmp/
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
   Add (replace `eth0` with your interface, e.g., from `nmcli`):
   ```yaml
   network:
     version: 2
     ethernets:
       eth0:
         dhcp4: no
         addresses: [192.168.1.100/24]
         gateway4: 192.168.1.1
         nameservers:
           addresses: [8.8.8.8]
   ```
3. Apply and verify:
   ```bash
   sudo netplan apply
   ip addr show eth0
   ping 8.8.8.8
   ```
4. Universal commands:
   ```bash
   ip link show
   ip addr show
   sudo ip route add 10.0.0.0/24 via 192.168.1.1
   ip route
   ss -tulpn
   ```
   **Lessons**:

- Use `nmcli` or `ip` to identify active interfaces.
- Fix: Ensure correct interface name in Netplan.
- Exam tip: Troubleshoot “no route to host” with `ip route`.

### Practice Exercise 2: DNS Resolution

**Steps**:

1. Check local resolution:
   ```bash
   cat /etc/hosts
   echo "127.0.0.1 test.local" | sudo tee -a /etc/hosts
   cat /etc/nsswitch.conf  # Verify hosts: files dns
   ```
2. Check systemd-resolved:
   ```bash
   resolvectl status
   ```
3. Troubleshoot:
   ```bash
   dig example.com @1.1.1.1 +dnssec
   getent hosts example.com
   ```
   **Lessons**:

- `/etc/hosts` takes precedence over DNS.
- Exam tip: Fix “unknown host” by editing `/etc/hosts` or checking `resolvectl`.

### Practice Exercise 3: Hostnames

**Steps**:

1. Set hostname:
   ```bash
   sudo hostnamectl set-hostname mynewhost.example.com
   hostnamectl
   ```
2. Automate (optional):
   ```bash
   sudo apt install cloud-init
   sudo nano /etc/cloud/cloud.cfg  # Add: fqdn: myhost.example.com
   sudo cloud-init clean
   sudo cloud-init init
   ```
   **Lessons**:

- `hostnamectl` updates `/etc/hostname`.
- Exam tip: Correct “FODNs” to “FQDNs” for cloud-init.

## Topic 7: Advanced Shell Operations

Covers Bash navigation, shortcuts, redirection, and variables.

### Practice Exercise 1: Navigation Shortcuts

**Steps**:

1. Basic:
   ```bash
   cd /etc
   cd -
   ```
2. Stack:
   ```bash
   pushd /var/log
   pushd /boot
   dirs -v
   cd ~1
   popd
   ```
   **Lessons**:

- `cd -`, `pushd`, `popd` improve navigation.
- Exam tip: Use `dirs -v` for stack-based navigation.

### Practice Exercise 2: Keyboard Shortcuts

**Steps**:

- Type: `echo "This is a test command with args"`
- `Ctrl-a`: Move to start.
- `Ctrl-e`: Move to end.
- Run: `ls /etc`
- `Alt-.`: Insert `/etc`.
- `Ctrl-r`: Search history (type `echo`).
  **Lessons**:
- Shortcuts speed up command editing.
- Exam tip: Practice `Ctrl-r` for history search.

### Practice Exercise 3: Input/Output Redirection

**Steps**:

1. Basic:
   ```bash
   echo "Overwrite" > /tmp/test.txt
   echo "Append" >> /tmp/test.txt
   ls /nonexistent 2>&1 | tee /tmp/error.log
   ```
2. Fix process substitution issue:
   - Incorrect: `tar -czf /tmp/home.tgz -C / <(grep -E "^root" /etc/passwd | cut -d: -f6)`
   - Correct:
     ```bash
     tar -czf /tmp/home.tgz -C / $(grep -E "^root" /etc/passwd | cut -d: -f6)
     ```
     Or:
     ```bash
     grep -E "^root" /etc/passwd | cut -d: -f6 | xargs -I {} tar -czf /tmp/home.tgz -C / {}
     ```
3. Verify:
   ```bash
   tar -tzf /tmp/home.tgz  # Should show root/.bashrc, etc.
   ```
4. Multiple users:
   ```bash
   grep -E "^(root|ubuntu)" /etc/passwd | cut -d: -f6 | xargs -I {} tar -czf /tmp/users.tgz -C / {}
   tar -tzf /tmp/users.tgz
   ```
   **Lessons**:

- Process substitution (`<(...)`) creates file descriptors, unsuitable for `tar` arguments.
- Use command substitution (`$(...)`) or `xargs` instead.
- Exam tip: Build complex pipelines like the `tar` example.

### Practice Exercise 4: Environment Variables

**Steps**:

1. Set and check:
   ```bash
   export API_KEY=12345
   printenv | grep API_KEY
   ```
2. Customize:
   ```bash
   export PS1="\u@\h:\w\$ "
   export PATH=/usr/local/bin:$PATH
   ```
3. Persistent:
   ```bash
   echo 'export MYVAR=test' >> ~/.bashrc
   source ~/.bashrc
   sudo echo 'export GLOBALVAR=global' > /etc/profile.d/custom.sh
   source /etc/profile.d/custom.sh
   ```
   **Lessons**:

- Use `export` for environment variables; `~/.bashrc` for user, `/etc/profile.d/` for system-wide.
- Exam tip: Modify `PATH` for command lookup.

## Key Takeaways

- **Storage**: Master `wipefs`, `fsck.ext4`, `xfs_repair`, `cryptsetup`. SMART is VM-limited; focus on syntax and output interpretation.
- **Network**: Identify interfaces with `nmcli`, configure with Netplan/`ip`, troubleshoot DNS with `dig`.
- **Shell**: Use `$(...)` for `tar`, not `<(...)`. Practice shortcuts and variables.
- **Fixes**:
  - LUKS: Open device before mounting (`cryptsetup luksOpen`).
  - `tar`: Avoid process substitution for arguments.
- **UTM Limitations**: No SMART data; incomplete GRUB menu prevents `rd.break`.
