---
layout: post
title: "Q6: Linux RAID Recovery Lesson"
date: 2025-09-18
tags: [Linux+, LVM]
---

This guide captures a hands-on journey to master Logical Volume Manager (LVM) on an Ubuntu VM (kernel 6.8.0-79, Intel UTM), tackling real-world storage challenges like those faced in data centers. We fixed an over-allocated Logical Volume (LV), extended it precisely with a new Physical Volume (PV), and navigated filesystem checks on a mounted root filesystem. Designed for CompTIA+ Linux+ prep and data center roles, it covers shrinking an LV from 65.94 GiB to 30.47 GiB, extending it by 5 GiB, and understanding why PVs don’t need filesystems. FAQs address critical concepts like skipping `resize2fs`, determining `lvreduce` sizes, and LVM’s abstraction. With analogies (VG as a community fridge, LV as a tenant), this guide is beginner-friendly yet deep enough for your GitHub portfolio to impress future employers!

## Setup

- **VM:** Ubuntu on UTM (Intel Mac), kernel 6.8.0-79.
- **Storage:**
  - `sda`: 64G disk (sda1: 1G /boot/efi, sda2: 2G /boot, sda3: ~60.95G PV in `ubuntu-vg`).
  - `sdb`: 5G disk (new PV, added to `ubuntu-vg` via `pvcreate` and `vgextend`).
  - VG `ubuntu-vg`: 65.94 GiB total (sda3 + sdb), initially 0 free (post `lvextend -l +100%FREE`).
  - LV `ubuntu-lv`: 65.94 GiB (16881 extents, later shrunk to 30.47 GiB), mounted as root (/).
- **Issues:**
  - `sudo lvextend -L +5G /dev/ubuntu-vg/ubuntu-lv` failed: "Insufficient free space" (VG fully allocated).
  - `sudo e2fsck -f /dev/mapper/ubuntu--vg-ubuntu--lv` failed: Root filesystem mounted.
- **Goal:** Shrink LV to 30.47 GiB (original), extend by sdb’s 5 GiB (~35.47 GiB), resize filesystem, and understand LVM’s PV filesystem behavior.
- **Snapshot:** UTM snapshot ensures safe rollback.

**Analogy:** The VG is a community fridge, PVs (`sda3`, `sdb`) are grocery deliveries, and the LV is a tenant’s meal prep. The tenant hogged all 65.94G of groceries, so we returned extras (to 30.47G), added a 5G pizza (sdb), and checked the fridge safely while they’re cooking (FS mounted).

## Step 1: Fixing the Over-Allocation

The LV `ubuntu-lv` consumed all 65.94 GiB of `ubuntu-vg` via `sudo lvextend -l +100%FREE`, leaving no free space (VFree: 0). This caused `sudo lvextend -L +5G` to fail. Since `df -h /` showed ~30G (filesystem unchanged), we safely shrank the LV to its original 30.47 GiB (7801 extents).

### Commands

```bash
# Verify state
sudo vgs  # VFree: 0, VSize: 65.94G
sudo lvs  # LSize: 65.94G (16881 extents)
df -h /   # Size: ~30G, confirms FS didn't grow

# Shrink LV to original 30.47G
sudo lvreduce -l 7801 /dev/ubuntu-vg/ubuntu-lv
# Output: "Size of logical volume ubuntu-vg/ubuntu-lv changed to 30.47 GiB (7801 extents)."
```

### Why It Worked

- **No e2fsck Needed:** The filesystem (~30G) matched the original LV size (30.47 GiB), so `lvreduce` was safe without checking the filesystem, as no data was at risk.
- **Outcome:** Freed ~35.47 GiB in the VG (VFree: ~30.47G from sda3 + 5G from sdb).
- **Analogy:** Returned extra groceries to the fridge, leaving space for the 5G pizza.
- **Data Center Tie-In:** In a data center, you’d fix an over-allocated VM disk (e.g., a Kubernetes pod’s Persistent Volume Claim) like this, logging changes in Jira to maintain SLAs. Cert tip: Know `lvreduce` risks data loss if the filesystem size exceeds the target LV size.

## Step 2: Handling the Mounted Filesystem (`e2fsck` Failure)

Running `sudo e2fsck -f /dev/mapper/ubuntu--vg-ubuntu--lv` failed because the root filesystem (/) was mounted, preventing checks to avoid corruption. Since the filesystem didn’t grow (per `df -h`), we skipped `e2fsck` for `lvreduce`. For future shrinks or data center rigor, use a rescue environment to unmount the filesystem.

### Rescue Mode (Optional Practice)

1. **Boot Live CD:**
   - Attach Ubuntu 24.04 ISO in UTM → Boot → “Try Ubuntu”.
2. **Activate LVM:**
   ```bash
   sudo vgscan
   sudo vgchange -a y ubuntu-vg
   # Output: "Found volume group ubuntu-vg", "1 logical volume(s) active"
   ```
3. **Check Filesystem:**
   ```bash
   sudo e2fsck -f /dev/mapper/ubuntu--vg-ubuntu--lv
   # Output: "Filesystem is clean" or minor fixes
   ```
4. **Reboot:** Eject ISO, boot normally.
   ```bash
   sudo vgchange -a n ubuntu-vg
   sudo reboot
   ```

### Why It Matters

- **Mounted FS Issue:** `e2fsck` requires an unmounted filesystem to prevent corruption, like repairing a road without traffic.
- **DC Tie-In:** Rescue boots are standard for root filesystem maintenance (e.g., fixing a corrupted VM disk on a Proxmox cluster). Cert tip: Know `vgscan` and `vgchange -a y` for accessing LVM in rescue mode.
- **Here:** Skipped `e2fsck` safely (filesystem unchanged), but practicing rescue mode builds job-ready skills.

## Step 3: Extending LV by sdb’s 5 GiB

With ~35.47 GiB free in the VG (post-`lvreduce`), we extended the LV by exactly 5 GiB (sdb’s contribution), from 30.47 GiB to ~35.47 GiB, then resized the filesystem to make the new space usable.

### Commands

```bash
# Extend LV by 5G
sudo lvextend -L +5G /dev/ubuntu-vg/ubuntu-lv
# Output: "Size of logical volume ubuntu-vg/ubuntu-lv changed to 35.47 GiB (9081 extents)."

# Resize filesystem
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
# Output: "Resizing the filesystem to 9106176 (4k) blocks."

# Verify
sudo lvs  # LSize: 35.47g
sudo vgs  # VFree: ~30.47g
df -h /   # Size: ~35G, Avail: ~27G
lsblk     # LV: ~35.5G
```

### Why It Worked

- **Precision:** `-L +5G` added only sdb’s 5 GiB, unlike `-l +100%FREE`, which grabbed all free space.
- **Filesystem Resize:** `resize2fs` expanded the ext4 filesystem to use the LV’s new size, making the space accessible to applications.
- **Analogy:** Gave the tenant a 5G pizza, unpacked it (resize2fs), leaving ~30.47G in the fridge for others.
- **DC Tie-In:** This mirrors boosting a workload (e.g., /var/log for a web server) without hogging the VG. Cert tip: Know `resize2fs` is safe for online ext4 growth.

## Step 4: Understanding sdb’s Role (No Filesystem Needed)

Adding `sdb` (5 GiB) to `ubuntu-vg` and extending `ubuntu-lv` worked without formatting `sdb` with ext4/XFS because LVM uses PVs as raw storage, with the filesystem living on the LV.

### How It Works

- **LVM Abstraction:** `pvcreate /dev/sdb` initialized `sdb` as a PV, adding LVM metadata (no filesystem). `vgextend` added `sdb`’s 5 GiB to the VG’s pool. `lvextend` grew the LV, and `resize2fs` expanded the existing ext4 filesystem on `ubuntu-lv` to use `sdb`’s space.
- **Filesystem on LV:** The system sees only the LV’s block device (`/dev/mapper/ubuntu--vg-ubuntu--lv`) with ext4, not `sdb` directly. LVM maps LV data to PVs (`sda3`, `sdb`) via extents (4 MiB chunks).
- **Verification:**
  ```bash
  sudo lvs -o +seg_pe_ranges  # Shows LV extents on sda3, sdb (e.g., sdb:0-1279 for 5G)
  df -h /  # Shows ~35G, ext4 using new space
  ```
- **Analogy:** PVs are raw ingredients (flour, eggs) in the fridge (VG). The LV’s kitchen (ext4) cooks the meal, not the ingredients themselves.

## Step 5: Clean Up (Optional)

To revert fully (e.g., decommission sdb):

```bash
sudo lvreduce -l 7801 /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
sudo vgreduce ubuntu-vg /dev/sdb
sudo pvremove /dev/sdb
```

- **UTM:** Shut down → Edit → Remove sdb drive.
- **Verify:**
  ```bash
  sudo pvs  # Only sda3
  sudo vgs  # VSize: ~60.95G
  ```
- **Analogy:** Sent back the pizza, cleaned the fridge—lean setup restored.
- **DC Tie-In:** Decommissioning test disks or cleaning up migrations keeps SANs tidy, a common task in storage management.

## FAQ: Key LVM Questions

### 1. What Happens If We Don’t Run `resize2fs` After `vgextend` and `lvextend`?

- **Answer:** The LV grows (e.g., to 35.47 GiB), but the filesystem (ext4) stays at its original size (e.g., ~30G). Applications can’t use the new space, and `df -h` shows the old size. No data is lost, but the extra LV space is unusable until `resize2fs` (or `xfs_growfs` for XFS) expands the filesystem.
- **Verification:**
  ```bash
  sudo lvs  # LV size: 35.47g
  df -h /   # FS size: ~30G
  ```
- **Fix:** Run `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`.
- **DC Tie-In:** Forgetting `resize2fs` triggers “disk full” alerts despite LV growth, a common rookie mistake. Always script `lvextend` + `resize2fs` in DCs. Cert tip: Know filesystem resizing is critical post-LVM changes.

### 2. How Do We Know How Much to `lvreduce`? Is `vgdisplay` Always Needed?

- **Answer:** You need the LV’s original size (GiB or extents) before `lvextend`. Sources:
  1. **`lvs`:** Shows LSize (e.g., 30.47g, 7801 extents). Check logs or prior `lvs` output.
  2. **`lvdisplay`:** Shows “Current LE” (e.g., 7801 extents).
  3. **`df -h`:** If filesystem unchanged, shows original size (~30G, less precise).
  4. **Logs/Backups:** Check automation logs or CMDB for prior size.
- **`vgdisplay`:** Shows VG-level Alloc PE/Free PE (e.g., 16881/0), not LV size. Useful for confirming free space post-`lvreduce`, not for target size.
- **Example:**
  ```bash
  sudo lvs -o name,size,seg_le_ranges
  # Shows: ubuntu-lv, 30.47g, 7801 extents (pre-lvextend)
  ```
- **Best Practice:** Log LV sizes before changes (`lvs > lv_before.txt`). In DCs, use monitoring (e.g., Prometheus) or CMDBs (e.g., NetBox).
- **DC Tie-In:** Tracking LV sizes in configs prevents data loss during shrinks, a must for audits. Cert tip: Verify sizes with `lvs`/`lvdisplay` before `lvreduce`.

### 6. Why Does the System Work When sdb Has No ext4/XFS Filesystem?

- **Answer:** LVM uses PVs (`sda3`, `sdb`) as raw storage in the VG pool, with the filesystem (ext4) on the LV (`ubuntu-lv`). `pvcreate /dev/sdb` adds LVM metadata to `sdb`, not a filesystem. `vgextend` adds `sdb`’s 5 GiB to the VG, and `lvextend` extends the LV, whose ext4 filesystem (resized via `resize2fs`) manages data across all PVs transparently.
- **How It Works:**
  - LVM maps LV data to PVs via extents (4 MiB chunks).
  - The system sees only the LV’s block device (`/dev/mapper/ubuntu--vg-ubuntu--lv`) with ext4, not `sdb` directly.
  - `resize2fs` expands ext4 to use the LV’s new size, including `sdb`’s space.
- **What If sdb Had a Filesystem?** `pvcreate` overwrites `sdb`’s header with LVM metadata, erasing any prior filesystem. LVM treats PVs as raw blocks, not filesystems.
- **Verification:**
  ```bash
  sudo lvs -o +seg_pe_ranges  # Shows LV extents on sda3, sdb (e.g., sdb:0-1279 for 5G)
  df -h /  # Shows ~35G, ext4 using new space
  ```
- **DC Tie-In:** In DCs, raw disks are added to VGs without formatting (e.g., SSDs in a Ceph cluster). The LV’s filesystem handles data, enabling live scaling. Cert tip: Know LVM’s PV → VG → LV → filesystem hierarchy.

## Data Center Relevance

- **Over-Allocation Fixes:** Shrinking LVs recovers space from provisioning errors (e.g., over-extended VM disks), maintaining SLAs.
- **Precision Allocation:** `-L +5G` targets specific workloads (e.g., logs), saving VG space. Skipping `resize2fs` wastes LV space.
- **Rescue Mode Skills:** Handling mounted filesystems via live CD preps you for critical server maintenance (e.g., Kubernetes node recovery).
- **Size Tracking:** Logging LV sizes (`lvs`) prevents data loss during shrinks, essential for DC audits.
- **LVM Abstraction:** PVs like `sdb` don’t need filesystems, enabling flexible scaling (e.g., hot-adding disks to a SAN).
- **Automation:** Scripts like `extend_5g.sh` streamline storage tasks across server fleets, reducing errors in high-pressure shifts.

## Practice Questions

1. **Q:** What happens if we don’t run `resize2fs` after `vgextend` and `lvextend`?  
   **A:** LV grows, but filesystem stays small (e.g., ~30G), leaving new space unusable. Fix: Run `resize2fs`.
2. **Q:** How to know how much to `lvreduce` without `vgdisplay`?  
   **A:** Use `lvs` or `lvdisplay` for LSize/extents, or logs for prior size.
3. **Q:** Why does `e2fsck` fail on a mounted root FS?  
   **A:** Risks corruption on active FS. Fix: Unmount via live CD or single-user mode.
4. **Q:** When is skipping `e2fsck` safe before `lvreduce`?  
   **A:** When filesystem size matches target LV size (e.g., ~30G for 30.47G).
5. **Q:** Why did `lvextend -L +5G` fail initially?  
   **A:** VG had no free extents (VFree: 0). Fix: Shrink LV to free space.
6. **Q:** Why does `sdb` work without ext4/XFS?  
   **A:** LVM uses PVs as raw storage; the filesystem (ext4) on the LV manages data.
