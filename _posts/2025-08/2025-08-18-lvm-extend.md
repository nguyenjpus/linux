---
layout: post
title: "LVM Extend"
date: 2025-08-18
tags: [Basic, LVM]
---

Extending Logical Volumes with lvextend for Linux+ Prep

As part of my **CompTIA Linux+** studies, I’m learning about Logical Volume Management (LVM). Today, I explored how to extend a logical volume (LV) using `lvextend`, specifically the command `lvextend -L +50G /dev/vgData/lvHome -r`. Here’s what I learned.

## What Does `lvextend -L +50G /dev/vgData/lvHome -r` Do?

- **Command Breakdown**:
  - `lvextend`: Extends the size of a logical volume.
  - `-L +50G`: Adds 50 GB to the LV’s current size.
  - `/dev/vgData/lvHome`: Targets the LV `lvHome` in the volume group `vgData`.
  - `-r`: Automatically resizes the filesystem (e.g., Ext4 with `resize2fs`, XFS with `xfs_growfs`).
- **Purpose**: Increases `lvHome`’s capacity by 50 GB and makes the new space usable by the filesystem.

## Where Does the 50 GB Come From?

The extra 50 GB comes from **unallocated space in the volume group (VG)** `vgData`. LVM works like this:

- **Physical Volumes (PVs)**: Disks or partitions (e.g., `/dev/sda1`).
- **Volume Group (VG)**: A pool of space from PVs (e.g., `vgData`).
- **Logical Volume (LV)**: A virtual partition from the VG (e.g., `lvHome`).
- The VG must have at least 50 GB free (check with `vgdisplay vgData`, look for `Free PE / Size`).

If no space is available:

- Add a new disk: `pvcreate /dev/sdc; vgextend vgData /dev/sdc`.
- Or reduce another LV: `lvreduce -L -10G /dev/vgData/lvOther`.

## Steps to Extend Safely

1.  Check VG space: `vgdisplay vgData`.
2.  Extend LV: `lvextend -L +50G /dev/vgData/lvHome -r`.
3.  Verify: `df -h /home` (assuming `lvHome` is mounted at `/home`).

## Reflections

Understanding LVM is crucial for Linux+ and system administration. Practicing `lvextend` on a VM clarified how VGs allocate space. Next, I’ll simulate adding a disk to a VG.

Stay tuned for more Linux+ notes!
