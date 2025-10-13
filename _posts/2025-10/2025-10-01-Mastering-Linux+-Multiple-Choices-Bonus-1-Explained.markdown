---
layout: post
title: "Multiple-choices - Bonus Questions 1-10 Explained"
date: 2025-10-01 05:15:00 -0700
tags: [Linux+, multiple-choices]
---

Some bonus multiple-choices quetions.

## Question 1: Persistent Network Configuration

**Question**: To configure a static IP address (192.168.1.100/24, gateway 192.168.1.1) persistently for interface `eth0` on a modern Linux distro using NetworkManager, which command is used?  
**Options**:

- ip addr add 192.168.1.100/24 dev eth0; ip route add default via 192.168.1.1
- nmcli con mod "System eth0" ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.method manual
- echo "192.168.1.100/24" > /etc/network/interfaces
- ifconfig eth0 192.168.1.100 netmask 255.255.255.0; route add default gw 192.168.1.1

**Correct Answer**: nmcli con mod "System eth0" ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.method manual  
**Why Correct**: `nmcli` modifies NetworkManager configurations persistently. This command sets the IP, gateway, and manual method for the `eth0` connection, saved in `/etc/NetworkManager/system-connections/`. Apply with `nmcli con up "System eth0"`.  
**Why Others Wrong**:

- `ip addr add` is temporary (lost on reboot).
- `/etc/network/interfaces` is used by Debian’s `ifupdown`, not NetworkManager.
- `ifconfig` and `route` are deprecated and temporary.  
  **Key Concept**: NetworkManager is common in modern distros (e.g., Ubuntu, Fedora); use `nmcli` or edit connection files.  
  **Memory Aid**: “nmcli = NetworkManager CLI for persistent IPs.”

## Question 2: Testing DNS Resolution

**Question**: Which command tests DNS resolution for the domain `example.com` and returns its IP address?  
**Options**:

- ping example.com
- dig example.com
- netstat -r
- ip route get example.com

**Correct Answer**: dig example.com  
**Why Correct**: `dig` queries DNS servers (per `/etc/resolv.conf`) and returns detailed DNS records, including the IP address for `example.com`.  
**Why Others Wrong**:

- `ping` tests reachability but may fail if ICMP is blocked.
- `netstat -r` shows routing tables, not DNS.
- `ip route get` shows routing paths, not DNS resolution.  
  **Key Concept**: Use `dig +short example.com` for just the IP; `nslookup` is an alternative.  
  **Memory Aid**: “dig = Dig into DNS records.”

## Question 3: Setting Hostname Persistently

**Question**: To set the system hostname to `server1.local` persistently, which command is used on a modern Linux system?  
**Options**:

- echo "server1.local" > /etc/hostname
- hostname server1.local
- nmcli general hostname server1.local
- echo "HOSTNAME=server1.local" > /etc/sysconfig/network

**Correct Answer**: nmcli general hostname server1.local  
**Why Correct**: On modern distros (e.g., those using systemd), `nmcli general hostname` sets the hostname persistently, updating `/etc/hostname` and applying it immediately.  
**Why Others Wrong**:

- Editing `/etc/hostname` directly works but requires a manual `hostnamectl set-hostname` or reboot.
- `hostname server1.local` is temporary (lost on reboot).
- `/etc/sysconfig/network` is legacy (used in older Red Hat systems).  
  **Key Concept**: `hostnamectl` is the systemd tool; `nmcli general hostname` integrates with NetworkManager.  
  **Memory Aid**: “nmcli hostname = Name machine permanently.”

## Question 4: Checking Filesystem Type

**Question**: To determine the filesystem type of a mounted partition `/dev/sda1` (e.g., Ext4, XFS), which command is used?  
**Options**:

- fdisk -l /dev/sda1
- df -T /dev/sda1
- lsblk -f /dev/sda1
- parted -l /dev/sda1

**Correct Answer**: lsblk -f /dev/sda1  
**Why Correct**: `lsblk -f` lists block devices with filesystem types, showing `/dev/sda1`’s type (e.g., Ext4, XFS) and mount point.  
**Why Others Wrong**:

- `fdisk -l` shows partition tables, not filesystems.
- `df -T` shows mounted filesystems but requires the mount point, not device.
- `parted -l` shows partition details, not filesystem type.  
  **Key Concept**: Alternative: `blkid /dev/sda1` or `file -s /dev/sda1`.  
  **Memory Aid**: “lsblk -f = Filesystem facts for blocks.”

## Question 5: Finding Files Consuming Disk Space

**Question**: The `/var` directory is nearly full. Which command identifies the largest files or directories consuming space?  
**Options**:

- `find /var -type f -exec du -h {} \;`
- `du -h /var | sort -hr | head`
- `ls -lh /var`
- `df -h /var`

**Correct Answer**: du -h /var | sort -hr | head  
**Why Correct**: `du -h` calculates disk usage for `/var` and its subdirectories in human-readable format; `sort -hr` sorts by size (highest first), and `head` shows the top consumers.  
**Why Others Wrong**:

- `find ... -exec du` lists file sizes but isn’t sorted or summarized.
- `ls -lh` lists files in one directory, not recursive usage.
- `df -h` shows total usage, not individual files/directories.  
  **Key Concept**: Use `ncdu` for an interactive disk usage tool.  
  **Memory Aid**: “du | sort -hr = Disk usage, highest ranked.”

## Question 6: Checking Network Interface Status

**Question**: Which command shows whether the `enp0s3` network interface is up or down?  
**Options**:

- ip link show enp0s3
- ifconfig enp0s3
- nmcli device status enp0s3
- ethtool enp0s3

**Correct Answer**: ip link show enp0s3  
**Why Correct**: `ip link show enp0s3` displays the interface’s state (e.g., `UP` or `DOWN`) along with details like MAC address and MTU.  
**Why Others Wrong**:

- `ifconfig` is deprecated but works.
- `nmcli device status` shows NetworkManager status, which may differ.
- `ethtool` shows driver details, not always clear state.  
  **Key Concept**: Look for `state UP` or `state DOWN` in `ip link` output.  
  **Memory Aid**: “ip link = Interface link status.”

## Question 7: NFS Export Configuration

**Question**: To share the `/data/share` directory via NFS to the 192.168.2.0/24 network with read-write access, which file should be edited?  
**Options**:

- /etc/fstab
- /etc/exports
- /etc/nfs.conf
- /etc/mtab

**Correct Answer**: /etc/exports  
**Why Correct**: `/etc/exports` defines NFS shares on the server. An entry like `/data/share 192.168.2.0/24(rw,sync)` allows read-write access. Run `exportfs -ra` to apply.  
**Why Others Wrong**:

- `/etc/fstab` is for client mounts.
- `/etc/nfs.conf` configures NFS daemon settings, not exports.
- `/etc/mtab` is a runtime mount table.  
  **Key Concept**: NFS server vs. client roles; `/etc/exports` is server-side.  
  **Memory Aid**: “exports = Export shares to network.”

## Question 8: User Unable to Run sudo

**Question**: A user reports they cannot run `sudo` commands, getting a "not in the sudoers file" error. Which file should be edited to grant them sudo access?  
**Options**:

- /etc/passwd
- /etc/shadow
- /etc/sudoers
- /etc/group

**Correct Answer**: /etc/sudoers  
**Why Correct**: Adding the user to `/etc/sudoers` (e.g., `username ALL=(ALL) ALL`) or a sudoers group (e.g., `sudo`) grants sudo privileges. Use `visudo` for safe editing.  
**Why Others Wrong**:

- `/etc/passwd` defines users, not privileges.
- `/etc/shadow` stores passwords.
- `/etc/group` defines groups but not sudo rules directly.  
  **Key Concept**: Use `usermod -aG sudo username` for group-based sudo access.  
  **Memory Aid**: “sudoers = Sudo permissions file.”

## Question 9: User Environment Variable Issue

**Question**: A user reports that their `$PATH` environment variable lacks `/usr/local/bin` when logging in via SSH, but it’s correct in local sessions. Where should you check for misconfiguration?  
**Options**:

- /etc/profile
- ~/.bashrc
- /etc/ssh/sshd_config
- ~/.bash_profile

**Correct Answer**: ~/.bash_profile  
**Why Correct**: `~/.bash_profile` is sourced for login shells (e.g., SSH sessions) and typically sets `$PATH`. A missing `/usr/local/bin` entry here causes the issue.  
**Why Others Wrong**:

- `/etc/profile` affects all users but isn’t SSH-specific.
- `~/.bashrc` is for non-login shells (e.g., terminal opened after login).
- `/etc/ssh/sshd_config` configures SSH daemon, not user environment.  
  **Key Concept**: Login shells (SSH) source `~/.bash_profile` or `~/.profile`; non-login shells source `~/.bashrc`.  
  **Memory Aid**: “bash_profile = Path for login sessions.”

## Question 10: Checking Open Ports Remotely

**Question**: To check if a remote server at 192.168.1.10 is listening on port 22 (SSH), which command is used from a local Linux machine?  
**Options**:

- nc -zv 192.168.1.10 22
- ping 192.168.1.10:22
- ss -tn 192.168.1.10:22
- traceroute 192.168.1.10:22

**Correct Answer**: nc -zv 192.168.1.10 22  
**Why Correct**: `nc -zv` (netcat) tests TCP connections with zero I/O, checking if port 22 is open on 192.168.1.10. `-z` scans without sending data, `-v` is verbose.  
**Why Others Wrong**:

- `ping` tests reachability, not ports.
- `ss -tn` shows local sockets, not remote.
- `traceroute` traces paths, not ports.  
  **Key Concept**: Alternative: `telnet 192.168.1.10 22` or `nmap`.  
  **Memory Aid**: “nc -zv = Netcat checks if port’s alive.”

## Retention Tips for Bonus Questions 1-10

- **Themes**: Persistent networking (`nmcli`), DNS (`dig`), hostname (`nmcli general`), filesystems (`lsblk -f`), disk usage (`du`), NFS (`/etc/exports`), user privileges (`sudoers`), and environment variables (`~/.bash_profile`).
- **Mnemonic for Networking/DNS**: “NMCli for IPs, DIG for DNS, NC for ports” (N-D-N).
- **Practice**: In a Linux VM:
  - Set a static IP with `nmcli`, test DNS with `dig google.com`.
  - Edit `~/.bash_profile` to add `/usr/local/bin` to `$PATH`, test via SSH.
  - Share a directory via NFS and test with `nc -zv`.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “File for NFS shares?” → “/etc/exports.”
- **Quiz Yourself**: How to grant sudo access? Why does `$PATH` differ in SSH?
