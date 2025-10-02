---
layout: post
title: "Multiple-choice - Questions 31-40 Explained"
date: 2025-10-01 07:15:00 -0700
tags: [Linux+, multiple-choice]
---

This document explains questions 31-40 from a set of 100 scenario-based multiple-choice questions on Linux system management, focusing on block devices, filesystems, LVM management, disk troubleshooting, and networking basics. Each question includes the correct answer, why it's correct, why other options are incorrect, key concepts, and memory aids for retention.

## Question 31: Hierarchical View of Block Devices

**Question**: To see a hierarchical view of all block devices, including disks, partitions, LVM logical volumes, and RAID arrays, which command provides the most intuitive output?  
**Options**:

- fdisk -l
- df -h
- lsblk
- pvs

**Correct Answer**: lsblk  
**Why Correct**: `lsblk` displays a tree-like hierarchy of block devices (e.g., disks like `/dev/sda`, partitions like `/dev/sda1`, LVM volumes like `/dev/vg/lv`, and RAID like `/dev/md0`), showing mount points and sizes for easy navigation.  
**Why Others Wrong**:

- `fdisk -l` lists partitions but lacks hierarchy for LVM/RAID.
- `df -h` shows mounted filesystems and usage, not device structure.
- `pvs` lists physical volumes in LVM but misses non-LVM devices.  
  **Key Concept**: `lsblk` reads from `/proc/partitions` and is great for visualizing storage topology.  
  **Memory Aid**: “lsblk = List block devices like a tree.”

## Question 32: Mounting an XFS Filesystem Temporarily

**Question**: What is the correct command to mount a new XFS filesystem on `/dev/sdb1` temporarily to `/srv/web`?  
**Options**:

- mount -t xfs /dev/sdb1 /srv/web
- attach -t xfs /dev/sdb1 /srv/web
- fstab-update /dev/sdb1 /srv/web
- xfs_mount /dev/sdb1 /srv/web

**Correct Answer**: mount -t xfs /dev/sdb1 /srv/web  
**Why Correct**: The `mount` command with `-t xfs` specifies the filesystem type (XFS) and mounts `/dev/sdb1` to `/srv/web` temporarily (until reboot).  
**Why Others Wrong**:

- `attach` is not a standard mount command.
- `fstab-update` doesn't exist; `/etc/fstab` is edited manually.
- `xfs_mount` is invalid; use `mount -t xfs`.  
  **Key Concept**: Use `mount -a` to mount all `/etc/fstab` entries; for temporary mounts, specify options explicitly.  
  **Memory Aid**: “mount -t = Mount type (filesystem type).”

## Question 33: Removing a Volume Group

**Question**: Before removing a volume group `vg_archive`, what must be done to all logical volumes within it?  
**Options**:

- They must be resized to their minimum size.
- They must be unmounted and then removed using lvremove.
- They must be backed up using dd.
- They must be deactivated using vgchange -an.

**Correct Answer**: They must be unmounted and then removed using lvremove.  
**Why Correct**: To remove a VG, first unmount any mounted LVs, then use `lvremove` to delete the LVs. Only then can `vgremove vg_archive` be used. This ensures no data loss or conflicts.  
**Why Others Wrong**:

- Resizing isn't required for removal.
- `dd` backups are good practice but not mandatory for removal.
- `vgchange -an` deactivates the VG but doesn't remove LVs.  
  **Key Concept**: LVM removal order: LVs → VG. Always backup data first.  
  **Memory Aid**: “Unmount, lvremove, vgremove = Clean up from bottom up.”

## Question 34: Disk Space Not Freed After Deleting a File

**Question**: After deleting a large log file `/large.log` with `rm`, `df -h` still shows the root filesystem as full. What is a likely explanation?  
**Options**:

- The rm command failed silently.
- The filesystem is corrupted and needs a fsck.
- A running process still has the deleted file open, preventing the space from being freed.
- The LVM snapshot has not been updated.

**Correct Answer**: A running process still has the deleted file open, preventing the space from being freed.  
**Why Correct**: When a file is deleted but still open by a process (e.g., a logging daemon), the kernel holds the file descriptor open, so the space isn't released until the process closes it or is restarted. Use `lsof` or `fuser` to identify and kill the process.  
**Why Others Wrong**:

- `rm` failure would show an error.
- Corruption doesn't typically cause this; `fsck` checks errors.
- LVM snapshots aren't implied here.  
  **Key Concept**: This is a common issue with log files; restart the process (e.g., `systemctl restart rsyslog`) to free space.  
  **Memory Aid**: “Deleted but open = Space held hostage by process.”

## Question 35: DNS Resolution Failure

**Question**: A Linux server can't resolve external domain names (e.g., www.comptia.org) but can ping IPs like 8.8.8.8. Which file is most likely misconfigured?  
**Options**:

- /etc/hosts
- /etc/hostname
- /etc/resolv.conf
- /etc/nsswitch.conf

**Correct Answer**: /etc/resolv.conf  
**Why Correct**: This file contains DNS server IPs (e.g., `nameserver 8.8.8.8`) and search domains. If misconfigured or empty, DNS lookups fail, but IP pings work.  
**Why Others Wrong**:

- `/etc/hosts` maps local hostnames to IPs, not external DNS.
- `/etc/hostname` sets the system's hostname.
- `/etc/nsswitch.conf` defines lookup order (e.g., files then DNS), but issues here are rarer.  
  **Key Concept**: Use `nslookup` or `dig` to test DNS; systemd-resolved may manage this file dynamically.  
  **Memory Aid**: “resolv.conf = Resolve names via DNS servers.”

## Question 36: Assigning Static IP Temporarily

**Question**: Which set of ip commands assigns a static IP 10.0.1.20/24 and gateway 10.0.1.1 to interface `eth` temporarily?  
**Options**:

- ip addr add 10.0.1.20/24 dev eth; ip route add default via 10.0.1.1
- ifconfig eth0 10.0.1.20 netmask 255.255.255.0; route add default gw 10.0.1.1
- nmcli con add iname eth0 ip4 10.0.1.20/24 gw4 10.0.1.1
- ip link set eth0 address 10.0.1.20; ip gateway add 10.0.1.1

**Correct Answer**: ip addr add 10.0.1.20/24 dev eth; ip route add default via 10.0.1.1  
**Why Correct**: These `ip` commands add the IP to the interface and set the default route temporarily (lost on reboot). Note: Interface is `eth`, but modern naming is `eth0`—the syntax works similarly.  
**Why Others Wrong**:

- `ifconfig` and `route` are deprecated; they work but aren't modern.
- `nmcli` is for persistent NetworkManager configs, not temporary.
- `ip link set ... address` sets MAC, not IP; `ip gateway add` is invalid.  
  **Key Concept**: `ip` from iproute2 is the modern tool; use for temporary changes.  
  **Memory Aid**: “ip addr add = IP address addition; route add = Routing path.”

## Question 37: Viewing Network Routing Table

**Question**: To view the current network routing table on a modern Linux system, which command is preferred?  
**Options**:

- netstat -r
- route -n
- ip route show
- ifconfig -a

**Correct Answer**: ip route show  
**Why Correct**: `ip route show` displays the routing table clearly, showing destinations, gateways, and interfaces. It's part of iproute2 and preferred over legacy tools.  
**Why Others Wrong**:

- `netstat -r` is deprecated (but works).
- `route -n` is also legacy.
- `ifconfig -a` shows interfaces, not routes.  
  **Key Concept**: Routing directs traffic; check with `ip route get <IP>` for specific paths.  
  **Memory Aid**: “ip route show = IP routing showcase.”

## Question 38: Checking Listening Ports

**Question**: To test if a web server is listening on port 443 (HTTPS) locally, which command checks for listening ports?  
**Options**:

- ping localhost:443
- ss -tlpn | grep :443
- traceroute localhost:443
- arp -a

**Correct Answer**: ss -tlpn | grep :443  
**Why Correct**: `ss -tlpn` shows TCP/UDP listening ports (`-t` TCP, `-l` listening, `-p` processes, `-n` numeric), and `grep :443` filters for port 443.  
**Why Others Wrong**:

- `ping` tests reachability, not ports.
- `traceroute` traces paths, not local ports.
- `arp -a` shows ARP cache.  
  **Key Concept**: `ss` is faster than `netstat`; use `ss -tulpn` for all listening ports.  
  **Memory Aid**: “ss -tlpn = Socket stats: TCP listening, process names.”

## Question 39: Finding MAC Address

**Question**: Which command displays the MAC address of the `enp1s0` network interface?  
**Options**:

- ip addr show enp1s0
- ethtool enp1s0
- arp -a
- hostname -I

**Correct Answer**: ip addr show enp1s0  
**Why Correct**: `ip addr show enp1s0` outputs interface details, including the link/ether line with the MAC address (e.g., `link/ether aa:bb:cc:dd:ee:ff`).  
**Why Others Wrong**:

- `ethtool enp1s0` shows settings but not always MAC clearly.
- `arp -a` shows ARP table, not interface MAC.
- `hostname -I` shows IP addresses only.  
  **Key Concept**: Modern interfaces use predictable names like `enp1s0` (instead of `eth0`).  
  **Memory Aid**: “ip addr = IP and address (MAC too).”

## Question 40: Tracing Network Path

**Question**: To trace the path packets take to a remote server at 198.51.100.10, which utility shows each network "hop"?  
**Options**:

- dig
- ping
- traceroute
- netstat

**Correct Answer**: traceroute  
**Why Correct**: `traceroute 198.51.100.10` sends packets with increasing TTL to reveal each router (hop) along the path, helping diagnose connectivity issues.  
**Why Others Wrong**:

- `dig` queries DNS.
- `ping` tests reachability but not path.
- `netstat` shows local connections/stats.  
  **Key Concept**: Use `traceroute -I` for ICMP or `mtr` for combined ping/traceroute.  
  **Memory Aid**: “traceroute = Trace route hops.”

## Retention Tips for Questions 31-40

- **Themes**: Block device visualization (`lsblk`), filesystem mounting (`mount`), LVM cleanup (`lvremove`), disk troubleshooting (open file handles), and networking basics (`ip`, `ss`, `traceroute`).
- **Mnemonic for Networking Tools**: “IP for addresses/routes, SS for sockets, Traceroute for paths” (I-S-T).
- **Practice**: In a Linux VM:
  - Run `lsblk` and `mount -t ext4 /dev/sdb1 /mnt/test`.
  - Delete a file while a process (e.g., `tail -f`) holds it open, then check `df -h`.
  - Assign a temp IP with `ip addr add`, view routes with `ip route show`, and trace to google.com.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “What shows MAC?” → “ip addr show.”
- **Quiz Yourself**: Why does deleted file space not free? Command for listening on port 443?
