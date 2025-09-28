---
layout: post
title: "Q2: Rebuilding Initramfs for RAID Driver with Dracut and GRUB Testing"
date: 2025-09-15
tags: [Linux+, GRUB, Dracut, initramfs]
---

This complete lab simulates a CompTIA Linux+ Q2 scenario: a server fails to boot because the initramfs lacks a RAID driver, preventing the RAID array (`/raiddata`) from assembling. We’ll use `dracut` to rebuild the initramfs, include the RAID driver (`md_mod`), simulate a failure by omitting it, verify contents, and test the failure by editing GRUB. Based on real output from an Ubuntu 24.04 VM (aarch64, UTM), it addresses why simulations may not always "crash" (e.g., kernel fallbacks) and provides ways to force errors. Uses a software RAID5 (`md1`) as a hardware HBA stand-in.

**Mnemonic**: Initramfs is a "suitcase" packed with boot tools (drivers). Dracut packs it, GRUB chooses it—omit tools, and the "trip" (boot) fails.

## Objective

- **Learn**: Initramfs/GRUB role in loading RAID drivers early.
- **Do**: Rebuild with/without drivers, verify, test failure via GRUB edit (logs, unmounted RAID).
- **Linux+ Skill**: Fix boot failures (e.g., "no RAID devices") by rebuilding initramfs and GRUB tweaks.

## System Context

- **VM**: Ubuntu 24.04 (noble), aarch64, UTM on Mac.
- **Storage**: `/boot` on `vda2`, root on LVM (`/dev/mapper/ubuntu--vg-ubuntu--lv`), RAID5 `md1` on `vdb`/`vdc`/`vdd` at `/raiddata`.
- **Kernel**: 6.8.0-79-generic.
- **User**: `ron@usv` (sudo).

## Prerequisites

- **Snapshot**: UTM snapshot "Pre-Dracut-Practice" for revert.
- **RAID Check**:
  ```bash
  cat /proc/mdstat  # md1 active RAID5 [UUU]
  df -h /raiddata  # Mounted 9.8G
  ```
- **Dracut**: Installed (`dracut 060+5-1ubuntu3.3`).

## Practice Steps (With Rationale)

Each step has **what/rationale**, **commands**, **breakdown** (for key ones), **expected output**, and **learn**.

### Step 1: Verify Current Initramfs (Baseline)

**What/Rationale**: Check existing initramfs for RAID support; confirm system healthy. Baseline before changes.  
**Commands**:

```bash
uname -r
ls /boot/initramfs*
sudo lsinitrd -k $(uname -r) | grep -iE 'md|raid'
cat /proc/mdstat
df -h /raiddata
```

**Expected Output**:

- `uname -r`: `6.8.0-79-generic`.
- `ls`: `/boot/initramfs-6.8.0-79-generic.img` (default).
- `lsinitrd | grep`: `mdraid`, `drivers/md/raid456.ko.zst`, etc.
- `/proc/mdstat`: `md1 : active raid5 ... [UUU]`.
- `df`: `/dev/md1 9.8G ... /raiddata`.
  **Learn**: Default "suitcase" has RAID tools (`mdraid` module).

### Step 2: Rebuild with RAID Drivers (Simulate Fix)

**What/Rationale**: Rebuild initramfs including drivers to "fix" missing RAID support.  
**Commands**:

```bash
sudo dracut -f --add-drivers "md_mod raid456" /boot/initramfs-full-raid.img $(uname -r)
sudo lsinitrd /boot/initramfs-full-raid.img | grep -iE 'md|raid'
```

**Breakdown** (dracut command):

- `sudo dracut`: Root-run packer.
- `-f`: Overwrite.
- `--add-drivers "md_mod raid456"`: Pack core MD (`md_mod`) and RAID5 (`raid456`).
- `/boot/initramfs-full-raid.img`: Test output.
- `$(uname -r)`: Kernel version sub.
  **Expected Output**:
- Dracut: ~58M image, "Including module: mdraid".
- `lsinitrd | grep`: `mdraid`, `raid456.ko.zst`, `30-parse-md.sh`.
  **Learn**: Dracut "packs" drivers for RAID assembly.

### Step 3: Create "Broken" Initramfs (Simulate Failure)

**What/Rationale**: Build omitting RAID to mimic Q2's issue (no driver = no RAID).
**Commands**:

```bash
sudo dracut -f --omit-drivers "mdraid md_mod raid456 dm-raid" /boot/initramfs-no-raid.img $(uname -r)
sudo lsinitrd /boot/initramfs-no-raid.img | grep -iE 'md|raid'
```

**Breakdown** (omit-drivers): Excludes RAID stack for "light suitcase."
**Expected Output**:

- Dracut: ~50M image.
- `lsinitrd | grep`: Minimal/no RAID (e.g., no `raid456.ko.zst`).
  **Learn**: Omissions leave suitcase "light"—RAID fails to assemble.

### Step 4: Verify Omissions Without Reboot (Safe Check)

**What/Rationale**: Unpack no-raid initramfs to visually confirm missing drivers.
**Commands**:

```bash
cd /tmp
sudo rm -rf initramfs-test
sudo mkdir initramfs-test
cd initramfs-test
sudo lsinitrd --unpack /boot/initramfs-no-raid.img .
ls usr/lib/modules/$(uname -r)/kernel/drivers/md/
sudo rm -rf /tmp/initramfs-test
```

**Expected Output**:

- `ls`: Empty/minimal `md/` (no `.ko.zst`).
  **Learn**: Unpack shows "broken suitcase"—drivers missing.

### Step 5: Test Failure with GRUB (Simulate Boot)

**What/Rationale**: Edit GRUB to load no-raid initramfs, observe failure (logs show no RAID).
**Commands**:

1. `sudo reboot`.
2. GRUB menu (`Esc`/`Shift`):
   - Select "Ubuntu Server," `e`.
3. Edit:
   - `initrd` line: `/initramfs-no-raid.img`.
   - `linux` line: Append `verbose debug` after `ro`.
   - See preivous post "Mastering Linux+ Q1 Grub Rescue Practice Summary" for detailed how to set this grub>.
4. `Ctrl+X` boot.
5. Post-boot:
   ```bash
   sudo dmesg | grep -iE 'md|raid|error|fail' # cannot dupplicate in my VM
   journalctl -b -u mdadm | grep -iE 'error|fail|no devices' # cannot dupplicate in my VM
   df -h /raiddata  # Unmounted? # cannot dupplicate in my VM
   ```
   **Expected Output**:

- Logs: "md: no devices found" or "raid: personality registration failed."
- `df`: No `/raiddata`.
  **Learn**: GRUB's `initrd` picks suitcase—broken one causes RAID failure.

**Alternative for Clearer Failure** (If No Errors):

- Broaden omission (rerun Step 3): `--omit-drivers "mdraid md_mod raid456 dm-raid async_tx"`.
- Add GRUB param: In `linux` line, append `md=0` (disables MD assembly).
- Rerun Step 5: Expect "md: disabled" in logs, `/raiddata` unmounted.
- As I tested this, the system froze.

### Step 6: Recover with Full Initramfs

**What/Rationale**: Restore working initramfs via GRUB or permanent update.
**Commands**:

1. GRUB edit (temporary):
   - Reboot, `e`.
   - `initrd /initramfs-full-raid.img`.
   - `Ctrl+X`.
2. Permanent:
   ```bash
   sudo mv /boot/initramfs-full-raid.img /boot/initrd.img-6.8.0-79-generic
   sudo update-grub
   sudo reboot
   ```
3. Verify:
   ```bash
   sudo lsinitrd -k $(uname -r) | grep -iE 'md|raid'
   cat /proc/mdstat
   df -h /raiddata
   ```
   **Expected Output**:

- `lsinitrd | grep`: RAID modules present.
- `/proc/mdstat`: `md1 [UUU]`.
- `df`: `/raiddata` mounted.
  **Learn**: GRUB edit or dracut fix restores boot.

### Step 7: Clean Up

**What/Rationale**: Remove tests, ensure default config.
**Commands**:

```bash
sudo rm /boot/initramfs-no-raid.img /boot/initramfs-full-raid.img
sudo update-grub
```

**Expected Output**: Clean `/boot/`, GRUB uses default.
**Learn**: Prevents accidental failures.

## Key Learnings

- Initramfs: Boot suitcase for drivers; omissions break RAID.
- Dracut: Packs with `--add-drivers`/`--omit-drivers`.
- GRUB: Edits `initrd` to test suitcases.
- No Crash? VM fallbacks—use broader omissions/params for drama.

## Troubleshooting

- No Errors: Broaden `--omit-drivers "mdraid md_mod raid456 dm-raid async_tx"`; add `md=0` to GRUB linux.
- GRUB Path: `/initramfs-no-raid.img`.
- Revert: UTM snapshot.
