---
layout: post
title: "Q11, Q12, Q13: Mastering DNS, Hosts, and Netplan for CompTIA Linux+"
date: 2025-09-19
tags: [Linux+, DNS, Hosts, netplan]
---

This guide tackles three CompTIA Linux+ practice questions (Q11-13/50) on DNS cache flushing, static hosts overrides, and safe Netplan configuration. Tested on an Ubuntu 24.04 VM (UTM, LVM root, `uname -r 6.8.0-79-generic`), it’s beginner-friendly, simulating a server fixing connectivity issues. The scenario: a node clears stale DNS cache (Q11), overrides a service IP (Q12), and updates DNS safely (Q13), like resolving a Kubernetes pod’s endpoint post-migration. Includes recovery from SSH loss via console, mimicking data center IPMI/KVM.

## CompTIA Linux+ Questions

### Q11: DNS Cache Flush

**Question**: The DNS resolver intermittently returns old addresses. Which utility flushes and rebuilds the `systemd-resolved` cache without rebooting?

- A. `resolvectl flush-caches`
- B. `systemetl restart dns.service`
- C. `nmcli dns reload`
- D. `rndc flush`

**Answer**: **A. `resolvectl flush-caches`**

- **Why**: Clears `systemd-resolved` cache, forcing fresh queries. Tested with `dig example.com` (new `Query time: 27 msec`).
- **Why Not**: B (typo, disruptive), C (invalid), D (BIND-specific).

### Q12: Static Hosts Override

**Question**: You need to add an A record override for `test.lab` into the static hosts file. Which line?

- A. `10.10.5.20 test.lab.local test.lab`
- B. `test.lab 10.10.5.20`
- C. `10.10.5.20 test.lab`
- D. `10.10.5.20 test.lab\0`

**Answer**: **C. `10.10.5.20 test.lab`**

- **Why**: Correct `/etc/hosts` format (`IP hostname`). Verified with `getent hosts test.lab`.
- **Why Not**: A (extra alias), B (wrong order), D (invalid null).

### Q13: Netplan with Rollback

**Question**: A Debian system uses Netplan with NetworkManager renderer. Apply a new YAML config without rebooting and roll back automatically if connectivity is lost. Which command?

- A. `netplan apply`
- B. `netplan try`
- C. `nmcli reload`
- D. `systemetl restart networking`

**Answer**: **B. `netplan try`**

- **Why**: Applies YAML, waits ~120s for confirmation, reverts on failure. Tested with DNS `8.8.8.8`, confirmed via `ping`.
- **Why Not**: A (no rollback), C (invalid), D (typo, legacy).

## Prerequisites

- **System**: Ubuntu 24.04 VM (LVM root, `enp0s1`).
- **Snapshot**: Optional (UTM, low-risk).
- **Packages**:
  ```bash
  sudo apt install -y dnsutils network-manager
  ```

## Scenario

Your VM is a server failing to resolve `test.lab`. Steps:

1. Flush DNS cache (Q11).
2. Override `test.lab` to `10.10.5.20` (Q12).
3. Set DNS to `8.8.8.8` with Netplan (Q13).

## Step 1: Q11 - Flush DNS Cache

1. **Verify `systemd-resolved`**:

   ```bash
   systemctl status systemd-resolved
   cat /etc/resolv.conf
   ```

   - **Output**: `Active: active`, `nameserver 127.0.0.53`.

2. **Populate Cache**:

   ```bash
   dig example.com
   sudo resolvectl show-cache | grep example.com
   ```

   - **Output**: IPs like `23.192.228.80`, `Query time: 29 msec`.

3. **Flush Cache**:

   ```bash
   sudo resolvectl flush-caches
   sudo resolvectl show-cache | grep example.com
   ```

   - **Output**: Empty (cache cleared).

4. **Verify**:
   ```bash
   dig example.com
   sudo journalctl -u systemd-resolved --since "10 minutes ago" | grep -iE "(cache|example.com)"
   ```
   - **Output**: New `Query time`, log: `Flushed all caches`.

## Step 2: Q12 - Add Hosts Override

1. **Backup**:

   ```bash
   sudo cp /etc/hosts /etc/hosts.bak-$(date +%Y%m%d)
   ```

2. **Edit**:

   ```bash
   sudo nano /etc/hosts
   # Add: 10.10.5.20 test.lab
   cat /etc/hosts | grep test.lab
   ```

3. **Test**:

   ```bash
   getent hosts test.lab
   dig test.lab
   ```

   - **Output**: `10.10.5.20 test.lab`, `Query time: 0 msec`.

4. **Logs**:
   ```bash
   sudo journalctl --since "10 minutes ago" | grep -iE "(hosts|test.lab)"
   ```
   - **Output**: Sparse (sudo commands only).

## Step 3: Q13 - Apply Netplan

1. **Backup**:

   ```bash
   sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak-$(date +%Y%m%d)
   ```

2. **Edit**:

   ```bash
   sudo nano /etc/netplan/50-cloud-init.yaml
   # Set:
   network:
     version: 2
     ethernets:
       enp0s1:
         dhcp4: true
         nameservers:
           addresses: [8.8.8.8]
   ```

3. **Apply**:

   ```bash
   sudo netplan try
   ping 8.8.8.8  # Another terminal
   ```

   - **Output**: Prompts for Enter; pings succeed (`time=28.8 ms`).

4. **Recover SSH Loss (If Occurs)**:

   - Use UTM console (Ctrl+Alt+F3 for TTY):
     ```bash
     sudo cp /etc/netplan/50-cloud-init.yaml.bak-20250920 /etc/netplan/50-cloud-init.yaml
     sudo netplan apply
     ping 8.8.8.8
     ```

5. **Verify**:
   ```bash
   resolvectl status | grep -i "DNS Servers"
   cat /etc/resolv.conf
   dig example.com
   ```

## Cleanup

1. **Revert Hosts**:

   ```bash
   sudo cp /etc/hosts.bak-20250920 /etc/hosts
   getent hosts test.lab
   ```

2. **Revert Netplan**:

   ```bash
   sudo cp /etc/netplan/50-cloud-init.yaml.bak-20250920 /etc/netplan/50-cloud-init.yaml
   sudo netplan apply
   ```

3. **Verify**:
   ```bash
   ping 8.8.8.8
   cat /etc/resolv.conf
   resolvectl status | grep -i "DNS Servers"
   sudo journalctl -b -p err | grep -iE "(dns|hosts|netplan|network)"
   ```

## FAQ

- **Q: How do `dig`, `systemd-resolved`, NSS, and DNS relate, and what happens with `127.0.0.1 google.com` in `/etc/hosts`?**
  - **A**: DNS maps domains to IPs. `systemd-resolved` (127.0.0.53) caches DNS results, respecting NSS (`hosts: files dns`), which prioritizes `/etc/hosts`. `dig` queries `systemd-resolved`, showing NSS results. Adding `127.0.0.1 google.com` makes `dig google.com` return `127.0.0.1` (0ms), while `dig @8.8.8.8` gets real IPs. Q11 flushed cache; Q12 used `/etc/hosts`; Q13 set DNS.
- **Q: Why “Permission denied” on `resolvectl show-cache` or `cat /etc/netplan/*`?**
  - **A**: Requires `sudo` (systemd/root permissions).
- **Q: Why sparse logs for `/etc/hosts` lookups?**
  - **A**: NSS bypasses `systemd-resolved` for `/etc/hosts`, logging only sudo actions.
- **Q: Why “Broken pipe” on `netplan try`?**
  - **A**: SSH dropped due to renderer switch (`networkd` to NetworkManager). UTM console reverted via backup.
- **Q: How to recover SSH without snapshot?**
  - **A**: Use UTM console (TTY via Ctrl+Alt+F3), revert Netplan (`cp backup.yaml`), apply (`netplan apply`).

## Practice Questions

1. **Q: What does `resolvectl flush-caches` do?**
   - **A**: Clears `systemd-resolved`’s DNS cache.
2. **Q: Why does `dig test.lab` show `/etc/hosts` results?**
   - **A**: NSS prioritizes `/etc/hosts` over DNS.
3. **Q: What’s the correct `/etc/hosts` format?**
   - **A**: `IP hostname` (e.g., `10.10.5.20 test.lab`).
4. **Q: Why use `netplan try`?**
   - **A**: Reverts on failure, ensuring connectivity.
5. **Q: How to fix SSH loss after Netplan?**
   - **A**: Use console to revert YAML and apply.

## Why This Matters

This mirrors a data center task: fix a node’s DNS (Q11), override a service IP (Q12), and update DNS safely (Q13). Recovery via console preps you for real-world “lost SSH” tickets, using IPMI/KVM-like access.
