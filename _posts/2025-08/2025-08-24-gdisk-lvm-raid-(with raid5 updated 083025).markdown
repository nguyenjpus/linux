---
layout: post
title: "Linux Hands-On Practice: Disk Management, LVM, RAID, and fstab"
date: 2025-08-24
tags: [Basic, LVM, RAID, Fstab]
---

This document summarizes hands-on exercises for Linux disk management and related tasks, based on practice in an Ubuntu VM on UTM. It includes step-by-step instructions, verifications, pitfalls, and lessons learned.

## 1. Practice Hands-On: Creating a GPT Partition with gdisk

### Prerequisites

- Ensure you have administrative privileges (use `sudo` for commands that require root).
- Install gdisk if not already installed: Run `sudo apt update && sudo apt install gdisk -y`.
- Add a new virtual disk to your VM via UTM settings (e.g., go to UTM > Edit VM > Drives > Add a new drive, set it to 10GB, raw image or whatever UTM supports). After adding, reboot the VM or rescan disks with `sudo partprobe`. In your lsblk output, existing disks are vda (system), vdb (10G with partition), vdc (5G with partition). The new disk might appear as `/dev/vdd` (confirm with lsblk after adding).

### Step-by-Step Instructions

1. Boot into your Ubuntu VM and open a terminal.
2. Identify the new disk: Run `lsblk` or `sudo fdisk -l`. Look for the new 10GB disk (e.g., `/dev/vdd` with no partitions yet). **Critical: Double-check the device name to avoid overwriting your system disk (/dev/vda) or existing ones (/dev/vdb, /dev/vdc). If it's not showing, run `sudo partprobe` or reboot.**
3. Launch gdisk on the new disk: `sudo gdisk /dev/vdd` (replace `/dev/vdd` with your actual device).
4. If the disk isn't already GPT (gdisk will warn if it's MBR or empty), create a new GPT table: Type `o` and press Enter, then type `y` to confirm.
   - **What does the `o` option do?** The `o` command in gdisk creates a new empty GPT (GUID Partition Table) on the disk, erasing any existing partition table (MBR or GPT) and preparing the disk for new partitions.
5. Add a new partition: Type `n` and press Enter.
   - Partition number: Press Enter for default (1).
   - First sector: Press Enter for default.
   - Last sector: Type `+5G` (for a 5GB partition) and press Enter.
   - Hex code (partition type): Type `8300` (for Linux filesystem) and press Enter.
6. (Optional: Add another partition with the remaining space: Repeat step 5, but use `+5G` or press Enter for the rest of the disk, and set hex code to `8300` again.)
7. Verify your changes before writing: Type `p` to print the partition table.
8. Write changes to disk: Type `w` and press Enter, then `y` to confirm. Exit gdisk with `q`.
9. Format the new partition: `sudo mkfs.ext4 /dev/vdd1` (replace with your partition, e.g., `/dev/vdd1`).

### Normal Sequence for `sudo gdisk /dev/vdd`

The typical sequence is: `o` (creates a new empty GPT partition table) => `n` (add new partition) => Specify size (e.g., `+5G` for last sector) => `w` (write changes) => `y` (confirm).

If `/dev/vdd` still has empty space left, you can execute `sudo gdisk /dev/vdd` again. This time, the sequence is: `p` (print current table to verify) => `n` (add new partition) => Press Enter for defaults (partition number, first sector); for last sector, press Enter to use all remaining space (or specify `+5G`) => Set hex code to `8300` => `w` => `y`.

Notice the warning after “y”:  
“OK; writing new GUID partition table (GPT) to /dev/vdd.  
Warning: The kernel is still using the old partition table.  
The new table will be used at the next reboot or after you  
run partprobe(8) or kpartx(8)  
The operation has completed successfully.”

This warning means: The kernel hasn't reloaded the updated partition table yet, so changes (e.g., new partition like `/dev/vdd2`) may not appear immediately in `lsblk`. You have to reboot the system (or run `sudo partprobe` or `sudo kpartx -a /dev/vdd`) to see the new partition created.

### Verification Steps

- Run `sudo gdisk -l /dev/vdd` to list the GPT table.
- Run `lsblk -f` to see the disk, partitions, and filesystem types.
- Mount temporarily to test: `sudo mkdir /mnt/test`, `sudo mount /dev/vdd1 /mnt/test`, `ls /mnt/test` (should be empty, but may show `lost+found` for ext4 filesystem), then `sudo umount /mnt/test`.

### Potential Pitfalls and Extensions

- **Pitfall:** If you select the wrong device (e.g., `/dev/vda`), you could wipe your system—always verify with `lsblk`. If gdisk shows errors like "invalid GPT," it might already be partitioned; use `x` for expert mode to fix.
- **Extension:** Simulate an error by adding a disk with fake data (create a file on it first: `sudo dd if=/dev/zero of=/mnt/test/fakefile bs=1M count=100`), then try partitioning over it (but snapshot your VM first in UTM to revert). Practice advanced options: In gdisk, after `n`, use `a` to set attributes or `c` to name the partition. For SSD alignment, specify first sector as 2048.

## 2. Practice Hands-On: Setting Up an LVM Volume (pvcreate, vgcreate, lvcreate) and Resizing It

### Prerequisites

- Install LVM tools: `sudo apt update && sudo apt install lvm2 -y`.
- You'll need at least two partitions for PVs. Use the partitions from the previous task (e.g., `/dev/vdd1` and `/dev/vdd2` if you created two), or add two new 5GB virtual disks in UTM (they'll appear as `/dev/vde` and `/dev/vdf`). Partition them first as LVM type: Follow the gdisk steps above, but set hex code to `8E00` for each (e.g., whole disk as one partition). Confirm with `lsblk`.

### Step-by-Step Instructions

1. Identify your partitions: `lsblk`. Assume `/dev/vdd1` and `/dev/vde1` (adjust as needed). **Ensure they have no important data.**
2. Create Physical Volumes (PVs): `sudo pvcreate /dev/vdd1 /dev/vde1`.
3. Create a Volume Group (VG): `sudo vgcreate myvg /dev/vdd1 /dev/vde1`.
4. Create a Logical Volume (LV): `sudo lvcreate -L 8G -n mylv myvg` (this creates an 8GB LV; if your partitions are smaller, use `-L 4G` or check free space with `sudo vgdisplay`).
5. Format the LV: `sudo mkfs.xfs /dev/myvg/mylv` (or `mkfs.ext4` if you prefer ext4).
6. Mount it: `sudo mkdir /data`, `sudo mount /dev/myvg/mylv /data`.
7. Test by creating a file: `sudo touch /data/testfile`, `ls /data`.

### Resizing Steps

1. **Check Available Space in the Volume Group (VG)**:

   - Before resizing, verify free space in the VG `myvg` to ensure you can extend the LV:
     ```bash
     sudo vgdisplay myvg | grep "Free PE"
     ```
     - **Output Example**: `Free PE / Size: 1024 / 4.00 GiB` (indicates 4GB available).
     - If no free space is available, add a new Physical Volume (PV) to the VG (see **Potential Pitfalls and Extensions** below).

2. **Extend the Logical Volume (LV)**:

   - To increase the LV size by 2GB (e.g., from 8GB to 10GB):
     ```bash
     sudo lvextend -L +2G /dev/myvg/mylv
     ```
     - Alternatively, set an absolute size (e.g., to 10GB total):
       ```bash
       sudo lvextend -L 10G /dev/myvg/mylv
       ```
     - Verify the new LV size:
       ```bash
       sudo lvs /dev/myvg/mylv
       ```
       - **Output Example**: `mylv myvg -wi-ao---- 10.00g`.

3. **Resize the Filesystem**:

   - For **XFS** (if `/dev/myvg/mylv` was formatted with `mkfs.xfs`):
     ```bash
     sudo xfs_growfs /data
     ```
     - <span style="color: red;">NOTE: For **XFS**, shrinking is not supported. You’d need to back up data, reformat, and restore.</span>
   - For **ext4** (if formatted with `mkfs.ext4`):
     ```bash
     sudo resize2fs /dev/myvg/mylv
     ```
     - `resize2fs` automatically adjusts to the new LV size.
   - Verify the new filesystem size:
     ```bash
     df -h /data
     ```
     - **Output Example**: Shows `/data` now has ~10GB capacity.

4. **Reduce the Logical Volume (Optional)**:

   - **Warning**: Reducing an LV is destructive if not done carefully. Always back up data and unmount the filesystem first.
   - Unmount the filesystem:
     ```bash
     sudo umount /data
     ```
   - For **ext4**, check the filesystem and resize it to a smaller size (e.g., 6GB):
     ```bash
     sudo e2fsck -f /dev/myvg/mylv
     sudo resize2fs /dev/myvg/mylv 6G
     ```
   - Reduce the LV to match (e.g., to 6GB):
     ```bash
     sudo lvreduce -L 6G /dev/myvg/mylv
     ```
     - Confirm with `y` when prompted (use `-f` to skip the prompt, but be cautious).
   - For **XFS**, shrinking is not supported. You’d need to back up data, reformat, and restore.
   - Remount and verify:
     ```bash
     sudo mount /dev/myvg/mylv /data
     df -h /data
     ```
     - **Output Example**: Shows `/data` now has ~6GB capacity.

5. **Verify the Changes**:
   - Check LV details:
     ```bash
     sudo lvdisplay /dev/myvg/mylv
     ```
   - Check filesystem size:
     ```bash
     df -h /data
     ```
   - Check VG free space:
     ```bash
     sudo vgdisplay myvg
     ```

### Lessons Learned: Wiping Signatures for pvcreate

<span style="color: red;">**IMPORTANT: You must wipe any existing filesystem signatures (e.g., ext4) before running pvcreate, as LVM requires partitions to be initialized as physical volumes (PVs) without conflicting filesystems.**</span>

Yes, you should wipe the ext4 signature on `/dev/vdd1` if you intend to use it for the LVM task, as LVM requires partitions to be initialized as physical volumes (PVs) without conflicting filesystems like ext4. Since `/dev/vdd1` is not mounted and was created for practice (with no important data, just `lost+found`), wiping it is safe and necessary to proceed with `pvcreate`.  
However, before wiping, confirm you don’t need the ext4 filesystem on `/dev/vdd1` for any other purpose (e.g., if you wanted to keep it for testing mounts or other tasks). Since you’re following the LVM hands-on task, wiping is appropriate.

#### Should You Unmount `/dev/vdd2` Before pvcreate?

Yes, you must unmount `/dev/vdd2` before running `pvcreate` on it. A mounted filesystem prevents exclusive access, causing `pvcreate` to fail with an error like "Can't open /dev/vdd2 exclusively. Mounted filesystem?". After unmounting, you’ll also need to wipe the ext4 signature on `/dev/vdd2` (similar to `/dev/vdd1`) because it was formatted as ext4 during the gdisk task.

The wipe commands are: `sudo wipefs -a /dev/vdd1` and then `sudo wipefs -a /dev/vdd2`.

### Lessons Learned: Resizing Filesystem After Extending Logical Volume

**After extending a logical volume with `lvextend`, you must resize the filesystem to make the additional space usable on the mounted `/data` directory.** Simply extending the LV increases the underlying block device size, but the filesystem (e.g., ext4 or XFS) remains unchanged until explicitly expanded. For ext4, use `sudo resize2fs /dev/myvg/mylv` to grow the filesystem to the new LV size. For XFS, use `sudo xfs_growfs /data`. Without this step, `df -h /data` will show the original size, and the extended space will remain inaccessible.

**Shrinking Considerations**: Reducing an LV requires unmounting the filesystem (`sudo umount /data`) and, for ext4, checking and resizing the filesystem first with `sudo e2fsck -f` and `sudo resize2fs` before running `sudo lvreduce`. XFS does not support shrinking, so you must back up data, reformat, and restore if reduction is needed. Always verify free space in the VG with `sudo vgdisplay` before extending, and back up data before shrinking to avoid data loss.

### Verification Steps

- PVs: `sudo pvs` or `sudo pvdisplay`.
- VGs: `sudo vgs` or `sudo vgdisplay`.
- LVs: `sudo lvs` or `sudo lvdisplay`.
- After resize: `df -h` should show increased space on `/data`.

### Potential Pitfalls and Extensions

- **Pitfall:** No free space in VG? Add another PV: Add a new disk/partition, `sudo pvcreate /dev/vdf1`, `sudo vgextend myvg /dev/vdf1`. Always unmount before shrinking: `sudo umount /data`, `sudo lvreduce -L -2G /dev/myvg/mylv`, then resize fs with `e2fsck -f` first for ext4.
- **Extension:** Create a snapshot: `sudo lvcreate -L 1G -s -n mysnap /dev/myvg/mylv`. Merge it back: `sudo lvconvert --merge /dev/myvg/mysnap`. Simulate expansion by adding PVs as above.

## 3. Practice Hands-On: Creating a RAID 1 Array with mdadm and Checking /proc/mdstat

### Prerequisites

- Install mdadm: `sudo apt update && sudo apt install mdadm -y`.
- Your lsblk shows an existing RAID1 /dev/md0 on /dev/vdb1 (10G) and /dev/vdc1 (5G), size 5G—note the size mismatch; effective size is the smaller disk. For practice, add two new identical 5GB virtual disks in UTM (e.g., /dev/vde and /dev/vdf). **Do not use existing md0 to avoid data loss.**

### Step-by-Step Instructions

1. Identify new disks: `lsblk`. Assume `/dev/vde` and `/dev/vdf` (whole disks, no partitions needed for this, but you can partition if preferred).
2. Create RAID1 array: `sudo mdadm --create --verbose /dev/md1 --level=1 --raid-devices=2 /dev/vde /dev/vdf`. Type `yes` to confirm.
3. Wait for sync: Run `cat /proc/mdstat` repeatedly—it shows [UU] when done (may take minutes).
4. Format: `sudo mkfs.ext4 /dev/md1`.
5. Mount: `sudo mkdir /raiddata`, `sudo mount /dev/md1 /raiddata`.
6. Save config for persistence: `sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf` (Ubuntu uses /etc/mdadm/mdadm.conf). Then update initramfs: `sudo update-initramfs -u`. (Explanation of this command as belows)

### To add the RAID array to `/etc/fstab` (without this, the mount will not persist!)

1. Check the filesystem type: `sudo blkid /dev/md0` (e.g., `TYPE="ext4"`).
2. Get the UUID: Note the UUID from `blkid` output (e.g., `UUID="abc123-..."`).
3. Edit `/etc/fstab`: `sudo nano /etc/fstab`, add a line like `UUID=abc123-... /raiddata ext4 defaults 0 0` (replace with actual UUID and filesystem type).
4. Test: Run `sudo mount -a` to verify no errors, then check `df -h /raiddata`.
5. Reboot and confirm: `sudo reboot`, then `lsblk` or `df -h /raiddata` to ensure the mount persists.

### Explanation: Saving Config for Persistence

The goal here is to ensure your newly created software RAID 1 array (e.g., `/dev/md1`) persists across system reboots. Without these steps, the RAID array might not automatically assemble (rebuild and become available) when you restart your Ubuntu VM, potentially leading to data inaccessibility or manual intervention each time.

Software RAID in Linux (managed by `mdadm`) relies on configuration files and boot-time mechanisms to detect and activate arrays early in the boot process. These commands save the RAID metadata to a config file and update the boot environment to include that info.

#### Breakdown of the Commands

1. **`sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf`**:

   - **`mdadm --detail --scan`**: Scans for all active RAID arrays and outputs detailed info (e.g., UUID, RAID level, member devices like `/dev/vde` and `/dev/vdf`) in a format suitable for the config file. Example output:
     ```
     ARRAY /dev/md1 metadata=1.2 name=ron-ubuntu:1 UUID=abc123:def456:ghi789:jkl012
     ```
   - **`| sudo tee -a /etc/mdadm/mdadm.conf`**: Appends this output to `/etc/mdadm/mdadm.conf`, the main configuration file for `mdadm` on Ubuntu. This file tells the system which RAID arrays to assemble automatically during boot. The `-a` flag ensures existing content (e.g., for `/dev/md0`) is preserved.
   - **What this achieves**: Stores array details so the system "remembers" `/dev/md1` on reboot, preventing issues like the array being detected with a different name or not at all.

2. **`sudo update-initramfs -u`**:
   - Updates the initramfs (initial RAM filesystem), a temporary filesystem loaded into memory during early boot, before the full root filesystem is mounted.
   - The `-u` flag updates the initramfs for the current kernel, embedding the latest `mdadm` config so the RAID array is assembled early in the boot process.
   - **What this achieves**: Ensures `/dev/md1` is available for mounting (e.g., at `/raiddata`) during boot, especially critical if the array contains important data or system files.

If these steps are skipped, you’d need to manually run `sudo mdadm --assemble --scan` after each reboot to activate the array, which is error-prone. On Ubuntu, the config file is `/etc/mdadm/mdadm.conf` (note the double "mdadm" directory).

### Lessons Learned: Updating fstab for Automatic RAID Mounting

**To make a RAID array (e.g., `/dev/md0` or `/dev/md1`) mount automatically at boot, you must add an entry to `/etc/fstab`.** Manually mounting with `sudo mount /dev/md0 /raiddata` is temporary and does not persist across reboots. The `/etc/fstab` file specifies which filesystems to mount at startup, including their mount points, filesystem types, and options. Without an `fstab` entry, the RAID array will assemble (if configured in `/etc/mdadm/mdadm.conf`), but it won’t be mounted automatically.

To add the RAID array to `/etc/fstab`:

1. Check the filesystem type: `sudo blkid /dev/md0` (e.g., `TYPE="ext4"`).
2. Get the UUID: Note the UUID from `blkid` output (e.g., `UUID="abc123-..."`).
3. Edit `/etc/fstab`: `sudo nano /etc/fstab`, add a line like `UUID=abc123-... /raiddata ext4 defaults 0 0` (replace with actual UUID and filesystem type).
4. Test: Run `sudo mount -a` to verify no errors, then check `df -h /raiddata`.
5. Reboot and confirm: `sudo reboot`, then `lsblk` or `df -h /raiddata` to ensure the mount persists.

### Verification Steps

- Array status: `cat /proc/mdstat` (shows md1 with [UU]).
- Details: `sudo mdadm --detail /dev/md1`.
- After mount: `df -h /raiddata`.
- Post-reboot: Reboot the VM, then run `cat /proc/mdstat` to confirm `/dev/md1` auto-assembles (shows [UU]). Check `cat /etc/mdadm/mdadm.conf` to verify the `ARRAY` line for `/dev/md1`.
- For `/dev/md0`: After adding to `fstab`, reboot and check `lsblk` to confirm `/raiddata` is mounted.

### Potential Pitfalls and Extensions

- **Pitfall:** Disk sizes must match for full utilization; if mismatched, array uses smaller size (e.g., `/dev/md0` is 5GB due to `/dev/vdc1` being smaller than `/dev/vdb1`). If array doesn't assemble on boot, check `/etc/mdadm/mdadm.conf` and run `sudo mdadm --assemble --scan`. If `/raiddata` doesn't mount, verify `/etc/fstab` for typos or incorrect UUIDs.
- **Extension:** Simulate failure: `sudo mdadm /dev/md1 --fail /dev/vde`, check `cat /proc/mdstat` (shows [U_]). Remove failed: `sudo mdadm /dev/md1 --remove /dev/vde`. Add replacement (add new disk /dev/vdg): `sudo mdadm /dev/md1 --add /dev/vdg`. Watch resync in /proc/mdstat.

### Creating a RAID 5 Array with /dev/vdb, /dev/vdc, and /dev/vdd

This section outlines the process to create a RAID 5 array (with RAID 1 as an alternative) using `/dev/vdb`, `/dev/vdc`, and `/dev/vdd`, repurposing disks previously used for LVM. This leverages three disks for better capacity (RAID 5: 10GB usable) or maximum redundancy (RAID 1: 5GB usable).

#### Prerequisites

- **Install mdadm**: Ensure `mdadm` is installed:
  ```bash
  sudo apt update && sudo apt install mdadm -y
  ```
- **Backup Data**: If `/dev/myvg/mylv` has test files (e.g., `/data/testfile`), back them up:
  ```bash
  sudo mkdir /root/backup
  sudo cp -r /data/* /root/backup
  ```
- **Confirm Disks**: Verify `/dev/vdb`, `/dev/vdc`, and `/dev/vdd` are the correct disks:
  ```bash
  lsblk
  ```
  Ensure no critical data exists on these disks.

#### Step 1: Remove LVM Configurations

Since `/dev/vdb1`, `/dev/vdb2`, and `/dev/vdc2` are part of `myvg`, remove them to use the whole disks for RAID.

1. **Unmount `/dev/myvg/mylv` (if mounted)**:

   ```bash
   sudo umount /data
   ```

   - If `umount` fails (e.g., “device not mounted”), proceed as it’s not mounted.

2. **Deactivate and Remove the Logical Volume**:

   ```bash
   sudo lvremove /dev/myvg/mylv
   ```

   - Confirm with `y` when prompted.

3. **Remove the Volume Group**:

   ```bash
   sudo vgremove myvg
   ```

4. **Remove Physical Volumes**:

   ```bash
   sudo pvremove /dev/vdb1 /dev/vdb2 /dev/vdc2
   ```

5. **Wipe Filesystem and LVM Signatures**:

   - Remove existing partitions and signatures:
     ```bash
     sudo wipefs -a /dev/vdb
     sudo wipefs -a /dev/vdc
     sudo wipefs -a /dev/vdd
     ```

6. **Remove Partitions** (optional):
   - Whole disks are used for RAID (as in the previous RAID section). If you prefer partitions, use `gdisk` to create new ones (type `8300`) and adjust commands to use `/dev/vdb1`, `/dev/vdc1`, `/dev/vdd1`.

#### Step 2: Create the RAID Array

For **RAID 5** (10GB usable):

```bash
sudo mdadm --create --verbose /dev/md1 --level=5 --raid-devices=3 /dev/vdb /dev/vdc /dev/vdd
```

- Type `yes` to confirm if prompted.
- **Expected Size**: `(3-1) * 5GB = 10GB`.

**Alternative RAID 1** (5GB usable):

```bash
sudo mdadm --create --verbose /dev/md1 --level=1 --raid-devices=3 /dev/vdb /dev/vdc /dev/vdd
```

#### Step 3: Wait for Sync

- Monitor synchronization:
  ```bash
  cat /proc/mdstat
  ```
- Look for `/dev/md1` with `[UUU]` (all disks up). Sync may take minutes.

#### Step 4: Format the RAID Array

- Format with ext4:
  ```bash
  sudo mkfs.ext4 /dev/md1
  ```

#### Step 5: Mount the RAID Array

- Create and mount:
  ```bash
  sudo mkdir /raiddata
  sudo mount /dev/md1 /raiddata
  ```
- Verify:
  ```bash
  ls /raiddata  # Shows lost+found
  df -h /raiddata  # Shows ~10GB (RAID 5) or ~5GB (RAID 1)
  ```

#### Step 6: Ensure Persistence

- Save configuration:
  ```bash
  sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
  ```
- Update initramfs:
  ```bash
  sudo update-initramfs -u
  ```

#### Step 7: Add to `/etc/fstab` for Automatic Mounting

1. Get UUID:
   ```bash
   sudo blkid /dev/md1
   ```
   - Example: `UUID="abc123-..." TYPE="ext4"`
2. Backup `/etc/fstab`:
   ```bash
   sudo cp /etc/fstab /etc/fstab.bak
   ```
3. Edit `/etc/fstab`:
   ```bash
   sudo nano /etc/fstab
   ```
   - Add:
     ```
     UUID=abc123-... /raiddata ext4 defaults 0 0
     ```
     (Replace `abc123-...` with actual UUID.)
4. Test:
   ```bash
   sudo mount -a
   df -h /raiddata
   ```

#### Step 8: Verify Post-Reboot

- Reboot:
  ```bash
  sudo reboot
  ```
- Check:
  ```bash
  cat /proc/mdstat  # Shows /dev/md1 with [UUU]
  df -h /raiddata   # Confirms mount
  lsblk             # Shows /dev/md1 at /raiddata
  ```

### Verification Steps

- **Array Status**:
  ```bash
  cat /proc/mdstat
  sudo mdadm --detail /dev/md1
  ```
- **Mount**:
  ```bash
  df -h /raiddata
  findmnt /raiddata
  ```
- **Config**:
  ```bash
  cat /etc/mdadm/mdadm.conf  # Verify ARRAY line for /dev/md1
  ```

### Potential Pitfalls and Extensions

- **Pitfall: LVM Conflicts**: Failing to remove `/dev/vdb1`, `/dev/vdb2`, `/dev/vdc2` from `myvg` causes `mdadm` errors like “device in use.” Always wipe signatures and LVM metadata.
- **Pitfall: Size Mismatch**: `/dev/vdd` (10GB) contributes only 5GB to match `/dev/vdb` and `/dev/vdc`. RAID 5 yields 10GB; RAID 1 yields 5GB.
- **Lesson**: Wipe signatures with `wipefs -a` before repurposing disks for RAID (as done for LVM in Section 2).
- **Lesson**: Adding to `/etc/mdadm/mdadm.conf` and updating `initramfs` ensures array persistence; `/etc/fstab` ensures mounting.
- **Extension**: Test resilience:
  ```bash
  sudo mdadm /dev/md1 --fail /dev/vdb
  cat /proc/mdstat  # Shows [U_U] for RAID 5
  sudo mdadm /dev/md1 --remove /dev/vdb
  sudo mdadm /dev/md1 --add /dev/vdb
  cat /proc/mdstat  # Watch resync
  ```

## 4. Practice Hands-On: Editing /etc/fstab with UUIDs (Use blkid)

### Prerequisites

- Have a device ready (e.g., your LVM LV /dev/myvg/mylv or RAID /dev/md1 from above). Backup fstab first: `sudo cp /etc/fstab /etc/fstab.bak`.

### Step-by-Step Instructions

1. Get the UUID: `sudo blkid /dev/myvg/mylv` (or /dev/md1). Copy the UUID value (e.g., UUID="abc123-...").
2. Edit fstab: `sudo nano /etc/fstab` (or vi if preferred).
3. Add a new line at the end: `UUID=your-uuid-here /data xfs defaults 0 0` (replace UUID, mount point, and fs type—use ext4 if formatted as such).
4. Save and exit (Ctrl+O, Enter, Ctrl+X in nano).
5. Test without reboot: `sudo mount -a` (should show no errors).

### Verification Steps

- Reboot the VM: `sudo reboot`. After boot, run `df -h`—/data should be mounted.
- If issues, boot to recovery (in UTM, edit boot args or use live USB), mount root, fix fstab.

### Potential Pitfalls and Extensions

- **Pitfall:** Typos cause boot failure—test with `mount -a` first. If stuck, in recovery: `mount -o remount,rw /`, edit fstab, reboot.
- **Extension:** Use labels instead: `sudo e2label /dev/md1 myraid` (for ext4), then `blkid -l -t LABEL=myraid`. In f fstab: `LABEL=myraid /raiddata ext4 defaults 0 0`. Add options: `defaults,noatime 0 0`.
