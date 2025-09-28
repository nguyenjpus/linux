---
layout: post
title: "RAID Setup Notes for Linux+"
date: 2025-08-18
tags: [Basic, RAID]
---

As part of my Linux+ prep, here's a quick guide to setting up RAID.

## What is RAID?

- Redundant Array of Independent Disks.
- Levels: RAID 0 (striping), RAID 1 (mirroring), etc.

## Commands

- Install: `sudo apt install mdadm`
- Create: `sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb`
- Check: `cat /proc/mdstat`

Reflections: Simulating RAID on VMs helped understand data redundancy.

More notes coming!
