---
layout: post
title: "Q14, Q21, Q23: Iptables, Rsync, and LVM"
date: 2025-09-20
tags: [Linux+, iptables, rsync, LVM]
---

Fix a data center node (Ubuntu 24.04 VM, user ron@usv) after a config error and data loss. Tasks:

1. **Q14**: Secure SSH to prevent 5-minute session drops using iptables.
2. **Q21**: Mirror /etc to /backup/etc with rsync, deleting obsolete files and skipping devices.
3. **Q23**: Restore an accidentally removed LVM logical volume (LV) from /etc/lvm/archive.

This mimics a real-world ops task: secure access, back up configs, recover storage. Analogy: Like fixing a leaky roof—secure the ladder (SSH), pack valuables (backup), patch the hole (LVM). VM setup: enp0s1 interface, networkd renderer (post-Q13 revert), LVM root, kernel 6.8.0-79-generic.

## Initial Setup

- Install iptables-persistent: `sudo apt update && sudo apt install -y iptables-persistent`.
  - Output: `iptables-persistent is already the newest version (1.0.20)`.
- Create test LVM (vgdata, testlv):
  - `sudo dd if=/dev/zero of=/tmp/vgdata.img bs=1M count=100` → `104857600 bytes (105 MB, 100 MiB) copied`.
  - `sudo losetup /dev/loop0 /tmp/vgdata.img`.
  - `sudo pvcreate /dev/loop0` → `Physical volume "/dev/loop0" successfully created`.
  - `sudo vgcreate vgdata /dev/loop0` → `Volume group "vgdata" successfully created`.
  - `sudo lvcreate -L 50M -n testlv vgdata` → `Logical volume "testlv" created`.
  - `sudo lvremove -f vgdata/testlv` → `Logical volume "testlv" successfully removed`.
- Verify: `sudo vgdisplay vgdata`:
  ```
  VG Name               vgdata
  ...
  Cur LV                0
  ...
  VG Size               96.00 MiB
  ...
  ```
- Backups: `sudo ls /etc/lvm/archive` → `vgdata_00000-1445260724.vg  vgdata_00001-158413221.vg`.

## Q14: iptables for SSH

**Question**: An SSH session drops after five minutes. Which iptables rule allows established connections to remain open?  
A. `iptables -A INPUT -p tcp --dport 22 -j ACCEPT`  
B. `iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`  
C. `iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT`  
D. `iptables -A INPUT -p tcp -m conntrack --ctstate NEW -j ACCEPT`

- **Correct**: B. Allows ongoing/related packets, preventing drops. A only allows new SSH; C is for output; D only allows new connections.

**Steps**:

1. **Setup**: Check rules: `sudo iptables -L -v -n` (empty chains). Flush: `sudo iptables -F`.
2. **Execution**: `sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`.
   - Subcommands: `-A INPUT` (append to input), `-m state --state ESTABLISHED,RELATED` (match ongoing/related), `-j ACCEPT` (allow).
   - Save: `sudo netfilter-persistent save` → `run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables save`.
3. **Outputs**:
   - `sudo iptables -L INPUT`:
     ```
     Chain INPUT (policy ACCEPT)
     target     prot opt source               destination
     ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
     ```
   - SSH session stayed active past 5 minutes.
4. **Verification**: Start SSH, wait 5+ mins (no drop).
5. **Cleanup**: `sudo iptables -D INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT && sudo netfilter-persistent save`.

**Analogy**: Like a bouncer letting club regulars stay without re-checking IDs every 5 minutes.

**Data Center Context**: Ensures persistent SSH during maintenance (e.g., AWS EC2), preventing lockouts like Q13’s SSH loss.

**Debugging**:

- Error: “No chain/target/match” → Fix: `sudo modprobe ipt_conntrack`.
- ufw interference: Disabled (per requirement: `sudo ufw status` → inactive).
- Ties to Q13: Prevents SSH drops during Netplan tweaks.

**FAQs**:

- Why not A? Only allows new SSH, drops established.
- Firewalld instead? CompTIA focuses on iptables.
- Persistent? Use netfilter-persistent.
- SSH still drops? Verify rule with `iptables -L`.

## Q21: rsync /etc Backup

**Question**: Mirror /etc to a backup location while deleting files removed from the source, skipping device files. Which rsync command accomplishes this?  
A. `rsync -av --delete --devices /etc /backup/etc`  
B. `rsync -aAXH --delete /etc/ /backup/etc`  
C. `rsync -av --delete --exclude='dev/*' /etc/ /backup/etc`  
D. `rsync -ar /etc /backup/etc`

- **Correct**: B. `-aAXH` preserves attributes (archive, ACLs, xattrs, hard links); `--delete` removes obsolete files; skips devices by default. A includes devices; C’s exclude is irrelevant; D lacks delete/attrs.

**Steps**:

1. **Setup**: Create dir: `sudo mkdir -p /backup/etc` → `drwxr-xr-x 2 root root 4096 ...`. Test file: `sudo touch /etc/testfile.conf`. Remove stale: `sudo rm -f /backup/etc/oldfile`.
2. **Execution**: `sudo rsync -aAXH --delete /etc/ /backup/etc`.
   - Subcommands: `-a` (archive), `-A` (ACLs), `-X` (xattrs), `-H` (hard links), `--delete` (remove dest files not in src), `/etc/` (contents only).
3. **Outputs**:
   - `sudo ls /backup/etc`: Lists `testfile.conf`, `hostname`, `hosts`, etc.
   - `sudo diff -r /etc /backup/etc`: Errors on symlinks (mtab, resolv.conf, os-release—normal, as dynamic).
   - Deletion test:
     ```
     sudo touch /etc/tempfile
     sudo rsync -aAXH --delete /etc/ /backup/etc
     ls /backup/etc/tempfile  # Exists
     sudo rm /etc/tempfile
     sudo rsync -aAXH --delete /etc/ /backup/etc
     ls /backup/etc/tempfile  # Gone
     ```
4. **Verification**: `diff -r` confirms mirror; deletion test proves `--delete`.
5. **Cleanup**: `sudo rm -rf /backup/etc/*`.

**Analogy**: Like syncing phone photos to a USB—keeps structure, deletes old files, skips hardware (devices).

**Data Center Context**: Backs up configs to NAS before ops (e.g., Q13’s Netplan revert, Q23’s LVM restore), like Ansible-driven backups in a Kubernetes cluster.

**Debugging**:

- Error: “change_dir /etc/backup” → Fix: Correct path (`/etc/ /backup/etc`).
- Missing `/backup/etc`: Fixed with `mkdir -p`.
- Symlinks skipped: Normal for mtab, resolv.conf (Q11’s DNS context).
- Ties to Q12: Backs up /etc/hosts (NSS overrides).

**FAQs**:

- Why not C? /etc has no /dev to exclude; misses ACLs/xattrs.
- Dry run? `rsync -aAXH --delete -n /etc/ /backup/etc`.
- Remote backup? `rsync -aAXH --delete -e ssh /etc/ user@remote:/backup/etc` (ties to Q14).
- Disk full? Check: `df -h /backup`.

## Q23: LVM Restore

**Question**: Restore an LVM metadata backup from /etc/lvm/archive to recover an accidentally removed LV. Which command initiates this?  
A. `lvconvert --repair vgdata`  
B. `vgchange --restore vgdata`  
C. `vgrestore vgdata`  
D. `vgcfgrestore -f /etc/lvm/archive vgdata`

- **Correct**: D. Restores VG metadata from archive. A repairs mirrors; B/C are invalid commands.

**Steps**:

1. **Setup**: Verify backups: `sudo ls /etc/lvm/archive` → `vgdata_00000-1445260724.vg  vgdata_00001-158413221.vg`. Check: `sudo vgdisplay vgdata` (`Cur LV 0`).
2. **Execution**:
   ```bash
   sudo vgcfgrestore -f /etc/lvm/archive/vgdata_00001-158413221.vg vgdata
   sudo lvchange -ay /dev/vgdata/testlv
   ```
   - Subcommands: `-f <file>` (specific .vg file), `vgdata` (VG name), `lvchange -ay` (activates LV).
3. **Outputs**:
   - vgcfgrestore: `Restored volume group vgdata`.
   - `sudo lvdisplay vgdata`:
     ```
     --- Logical volume ---
     LV Path                /dev/vgdata/testlv
     LV Name                testlv
     VG Name                vgdata
     ...
     LV Size                52.00 MiB
     ...
     ```
   - `sudo lvs`: `testlv vgdata -wi-a----- 52.00m`.
   - `sudo vgs`: `vgdata 1 1 0 wz--n- 96.00m 44.00m`.
4. **Verification**: `sudo lvs` confirms `testlv`. Mount if filesystem existed (skipped, no filesystem).
5. **Cleanup**:
   ```bash
   sudo lvremove -f vgdata/testlv
   sudo vgremove vgdata
   sudo losetup -d /dev/loop0
   sudo rm /tmp/vgdata.img
   ```
   - Outputs: `Logical volume "testlv" successfully removed`, `Volume group "vgdata" successfully removed`.

**Analogy**: Like recovering a deleted photo from a camera’s metadata backup—rebuilds structure, but data may need reformatting.

**Data Center Context**: Restores deleted LVs (e.g., database disk on OpenStack) without rebuilding storage, critical after Q21’s backup.

**Debugging**:

- Error: “/etc/lvm/archive is not a regular file” → Fix: Use `-f /etc/lvm/archive/vgdata_00001-158413221.vg`.
- Wrong LV name: Fixed `testlv_try_differentname` → `testlv`.
- LV not active: `sudo vgchange -ay vgdata`.
- Ties to Q21: /etc/lvm/archive backed up.
- Ties to Q11: Flush DNS if restored LV holds network configs.

**FAQs**:

- Why not A? `lvconvert` is for mirrors, not restores.
- Pick .vg file? `vgcfgrestore -l vgdata` lists backups.
- Data lost? Recreate filesystem with `mkfs`.
- Safe for root? Loop0-based `vgdata` is isolated (like Q9’s fstab).

## Practice Questions

1. **Q14 Variant**: Block new SSH, allow existing:  
   A. `iptables -A INPUT -p tcp --dport 22 -j DROP`  
   B. `iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport 22 -j DROP`  
   C. `iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT`  
   D. `iptables -P INPUT DROP`  
   _Answer_: B (targets NEW state, preserves established).

2. **Q21 Variant**: rsync /var/log excluding .gz files with delete:  
   A. `rsync -av --delete --exclude='*.gz' /var/log/ /backup/log`  
   B. `rsync -aAXH --delete /var/log /backup/log`  
   C. `rsync -ar --devices /var/log/ /backup/log`  
   D. `rsync -av /var/log /backup/log`  
   _Answer_: A (specific exclude, includes --delete).

3. **Q23 Variant**: List LVM backups:  
   A. `vgcfgrestore -l vgdata`  
   B. `lvconvert -l vgdata`  
   C. `vgchange -l vgdata`  
   D. `vgrestore -l vgdata`  
   _Answer_: A (lists archive files).

4. **Q14+Q21**: Backup /etc over SSH post-iptables:  
   A. `-e ssh`  
   B. `--remote`  
   C. `-r`  
   D. `-a`  
   _Answer_: A (enables SSH for rsync).

5. **Q21+Q23**: LVM restore fails, backup missing: Why?  
   A. No /etc/lvm/archive after lvremove  
   B. rsync --delete removed it  
   C. LVM doesn’t auto-backup  
   D. vgcfgrestore needs -f  
   _Answer_: C (manual if disabled, but Ubuntu auto-backs).

6. **All**: Debug failure after SSH, backup, restore:  
   A. Check iptables  
   B. Verify rsync /etc/lvm/archive  
   C. Run vgdisplay  
   D. All above  
   _Answer_: D (holistic debug, like Q13).
