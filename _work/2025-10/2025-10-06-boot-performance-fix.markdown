---
layout: post
title: Slashing Server Boot Time from 23 Minutes to 5.5 Minutes - A Real-World Sysadmin Win
date: 2025-10-06
tags: [Troubleshooting, systemd]
---

Tackling a server at work with a brutal 23-minute boot time, applying skills learned from my CompTIA Linux+ prep ([Linux Lab Summary: Mastering the Boot Process and Recovery](https://nguyenjpus.github.io/linux/2025/10/05/Mastering-the-Boot-Process-and-Recovery.html)). Using `systemd-analyze plot` and `systemd-analyze blame`, I pinpointed bottlenecks and slashed boot time to ~5.5 minutes by updating BIOS/BMC firmware—a 75% improvement! Here’s my journey, with visuals and tips for sysadmins.

## The Problem: A 23-Minute Boot Slog

The server crawled, with `systemd-analyze` revealing:

- **Total time**: 23m 24.514s
- **Firmware**: 19m 4.631s (81%, BIOS/BMC delays)
- **Loader**: 2m 34.677s (GRUB/UEFI)
- **Kernel**: 59.369s
- **Userspace**: 45.835s

The `before_boot.svg` timeline shows the pain:

<img src="/_work/2025-10/assets/before_boot.svg" alt="Boot Timeline: Slow (23 minutes) - Firmware and Network Delays" style="width: 100%; max-width: 1400px;">

**Pro Tip:** This SVG is highly detailed. **Right-click and select 'Open Image in New Tab'** to zoom in on the timeline and see the precise delays.

<!-- Fallback for raw GitHub: <img src="https://raw.githubusercontent.com/nguyenjpus/linux/main/assets/before_boot.svg" alt="Boot Timeline: Slow (23 minutes) - Firmware and Network Delays" style="width: 100%; max-width: 800px;"> -->

**Key bottlenecks** (from `systemd-analyze blame`):

- `NetworkManager-wait-online.service`: 30.030s, waiting for network interfaces (`enp175s0f0`–`enp175s0f3`).
- `rc-local.service`: 13.429s, likely custom scripts for NFS mounts (`mnt-nfs_home.mount`) or hardware setup.
- `tuned.service`: 1.098s, slowed by CPU/disk contention.
- **Note**: `systemd-networkd-wait-online.service` wasn’t used (system uses `NetworkManager`). `fwupd.service` wasn’t active; firmware delays were pre-OS (BIOS/BMC).

**Skills applied**: This mirrors Linux+ Question 86 (networking, `systemd` mounts), Question 11 (hardware-related kernel issues), and Question 5 (initramfs delays). The 19m firmware phase screamed BIOS issues.

## The Fix: Firmware Update FTW

Using IPMI tools, I updated the BIOS/BMC firmware, dropping boot time to 5m 42.628s:

- **Firmware**: 1m 57.661s (down from 19m 4s, 89% faster)
- **Loader**: 1m 56.582s (down from 2m 34s, 25% faster)
- **Kernel**: 1m 2.519s (up slightly from 59.369s)
- **Userspace**: 45.863s (unchanged)

The `after_boot.svg` shows the optimized flow:

<img src="/_work/2025-10/assets/after_boot.svg" alt="Boot Timeline: Fast (5.5 minutes) - Optimized Firmware" style="width: 100%; max-width: 1400px;">

**Pro Tip:** This SVG is highly detailed. **Right-click and select 'Open Image in New Tab'** to zoom in on the timeline and see the precise delays.

<!-- Fallback for raw GitHub: <img src="https://raw.githubusercontent.com/nguyenjpus/linux/main/assets/after_boot.svg" alt="Boot Timeline: Fast (5.5 minutes) - Optimized Firmware" style="width: 100%; max-width: 800px;"> -->

**Post-update bottlenecks** (from `systemd-analyze blame`):

- `NetworkManager-wait-online.service`: 30.036s (unchanged, network delays persist).
- `rc-local.service`: 13.464s (no change, likely NFS or disk scripts).
- `tuned.service`: 1.074s (slightly faster).
- `postfix.service`: 0.411s (slightly faster).

**Improvements**:

- Firmware update saved ~17 minutes by speeding up hardware init (e.g., NVMe drives `nvme0n1`–`nvme5n1`, SAS drives `sda`–`sdm`).
- Loader improved by 38s (faster disk I/O).
- **Skills applied**: systemd-analyze plot/blame. Using SCP to transfer svg files to my desktop for visualization.

## How I Did It

- **Generate timelines**:
  ```bash
  systemd-analyze plot > before_boot.svg
  systemd-analyze plot > after_boot.svg
  ```
- **Identify bottlenecks**:
  ```bash
  systemd-analyze blame
  ```
  Before:
  ```
  30.030s NetworkManager-wait-online.service
  13.429s rc-local.service
   1.098s tuned.service
   0.425s postfix.service
   ...
  ```
  After:
  ```
  30.036s NetworkManager-wait-online.service
  13.464s rc-local.service
   1.074s tuned.service
   0.411s postfix.service
   ...
  ```
- **View SVGs**: Open in a browser (e.g., Chrome) to zoom into timelines.

## Next Steps: Optimizing Userspace

The unchanged userspace (45s) points to:

- **NetworkManager-wait-online.service** (30s): Check `/etc/NetworkManager/NetworkManager.conf` for misconfigured NICs or DNS. If networking isn’t critical at boot, disable it:
  ```bash
  sudo systemctl disable NetworkManager-wait-online.service
  sudo systemctl restart NetworkManager
  ```
- **rc-local.service** (13s): Inspect `/etc/rc.local` or `/etc/rc.d/rc.local` for slow commands (e.g., NFS mounts). Example:
  ```bash
  cat /etc/rc.local
  ```
- Reboot and regenerate SVG:
  ```bash
  systemd-analyze plot > assets/after_boot_optimized.svg
  ```

## Why It Matters

This fix saved ~18 minutes per boot, boosting efficiency and showcasing Linux+ skills like `systemd` targets.
