---
layout: post
title: "Multi-Partition MBR Setup and Recovery (a complicated case for testdisk utility)"
date: 2025-09-08
tags: [Basic, fdisk, MRB]
---

# Multi-Partition MBR Setup and Recovery (a complicated case for "testdisk" utility)

## Objective

This scenario, part of CompTIA Linux+ certification practice, aims to create a disk image with multiple partitions (primary and logical), format them with ext4 and XFS, add test data, corrupt the MBR, and recover the partitions using TestDisk. The goal is to master disk partitioning, filesystem management, and partition table recovery.

## Steps and Commands

### 1. Create a Disk Image

Create a 1 GiB disk image to accommodate partitions, including a 350 MiB partition for XFS (minimum 300 MiB required).

```bash
dd if=/dev/zero of=~/multi_mbr.img bs=1M count=1024
```

- **Output**: Created a 1 GiB file (`~/multi_mbr.img`).

### 2. Partition the Disk Image

Use `fdisk` to create two primary partitions, one extended partition, and one logical partition inside the extended.

```bash
sudo fdisk ~/multi_mbr.img
```

- Commands in `fdisk`:
  - `o`: Create a new MBR partition table.
  - `n > p > 1 > Enter > +100M`: Create primary partition 1 (100 MiB, ext4).
  - `n > p > 2 > Enter > +350M`: Create primary partition 2 (350 MiB, XFS).
  - `n > e > 3 > Enter > Enter`: Create extended partition 3 (~573 MiB).
  - `n > Enter > Enter > +100M`: Create logical partition 5 (100 MiB, ext4).
  - `p`: Verify partition table:
    ```
    Device                   Start     End Sectors  Size Id Type
    /home/ron/multi_mbr.img1  2048  206847  204800  100M 83 Linux
    /home/ron/multi_mbr.img2 206848  923647  716800  350M 83 Linux
    /home/ron/multi_mbr.img3 923648 2097151 1173504  573M  5 Extended
    /home/ron/multi_mbr.img5 925696 1130495  204800  100M 83 Linux
    ```
  - `w`: Write and exit.

### 3. Set Up Loop Device and Format Partitions

Attach the disk image to a loop device with partition scanning and format the partitions.

```bash
sudo losetup -P /dev/loop0 ~/multi_mbr.img
lsblk /dev/loop0
```

- **Output**:
  ```
  loop0
  ├─loop0p1  100M  ext4
  ├─loop0p2  350M  XFS
  ├─loop0p3    1K  extended
  └─loop0p5  100M  ext4
  ```
- Format:
  ```bash
  sudo mkfs.ext4 /dev/loop0p1
  sudo mkfs.xfs /dev/loop0p2
  sudo mkfs.ext4 /dev/loop0p5
  ```

### 4. Add Test Data

Mount the partitions, write test files, and verify.

```bash
sudo mkdir /mnt/p1 /mnt/p2 /mnt/p5
sudo mount /dev/loop0p1 /mnt/p1
sudo mount /dev/loop0p2 /mnt/p2
sudo mount /dev/loop0p5 /mnt/p5
echo "Data on partition 1" | sudo tee /mnt/p1/file1.txt
echo "Data on partition 2" | sudo tee /mnt/p2/file2.txt
echo "Data on partition 5" | sudo tee /mnt/p5/file5.txt
ls /mnt/p1 /mnt/p2 /mnt/p5
cat /mnt/p1/file1.txt
cat /mnt/p2/file2.txt
cat /mnt/p5/file5.txt
sudo umount /mnt/p1 /mnt/p2 /mnt/p5
sudo losetup -d /dev/loop0
```

- **Output**: Confirmed `file1.txt`, `file2.txt`, `file5.txt` with correct contents.

### 5. Corrupt the MBR

Simulate MBR corruption by overwriting the first 512 bytes.

```bash
dd if=/dev/zero of=~/multi_mbr.img bs=512 count=1 conv=notrunc
```

- **Output**: 512 bytes overwritten.

### 6. Recover with TestDisk

Use TestDisk to recover the partition table.

```bash
sudo testdisk ~/multi_mbr.img
```

- Steps in TestDisk:

  - `[Create]` (log for debugging).
  - `[Intel/PC partition]`.
  - `[Analyse] > [Quick Search]`.
  - Quick Search output (initially incorrect):

    ```
    * Linux                    0  32 33    12 223 19     204800
    P Linux                   12 223 20    57 126  5     716800
    P Linux                   57 158 38    70  94 24     204800
    ```

  - Select `[Deeper Search]` or manually adjust:
    - <span style="color: red;">Change third partition from `P` to `L` (logical), then Enter </span>
  - Final structure:
    ```
    1 * Linux                    0  32 33    12 223 19     204800
    2 P Linux                   12 223 20    57 126  5     716800
    3 E extended                57 126  6    70  94 24     206848
    5 L Linux                   57 158 38    70  94 24     204800
    ```
  - `[List]` to verify files (`file1.txt`, `file2.txt`, `file5.txt`).
  - `[Write]` to restore, confirm with `Y`, quit.

### 7. Verify Recovery

Reattach, mount, and check files.

```bash
sudo losetup -P /dev/loop0 ~/multi_mbr.img
sudo partprobe /dev/loop0
lsblk /dev/loop0
sudo mount /dev/loop0p1 /mnt/p1
sudo mount /dev/loop0p2 /mnt/p2
sudo mount /dev/loop0p5 /mnt/p5
cat /mnt/p1/file1.txt
cat /mnt/p2/file2.txt
cat /mnt/p5/file5.txt
```

- **Output**: Successfully recovered all files.

### 8. Cleanup

```bash
sudo umount /mnt/p1 /mnt/p2 /mnt/p5
sudo losetup -d /dev/loop0
rm ~/multi_mbr.img
sudo rmdir /mnt/p1 /mnt/p2 /mnt/p5
```

## Issues Encountered and Resolutions

1. **XFS Size Requirement**:

   - **Issue**: Initial 100 MiB partition for XFS failed (`Filesystem must be larger than 300MB`).
   - **Resolution**: Increased p2 to 350 MiB, which satisfied XFS’s minimum size.

2. **Data Written to Local Filesystem**:

   - **Issue**: Ran `tee` without mounting partitions, writing to `/mnt/pX` locally.
   - **Resolution**: Ensured partitions were mounted before writing data.

3. **Logical Partition Not Detected**:

   - **Issue**: TestDisk Quick Search misidentified p5 as primary, missing extended p3.
   - **Resolution**: Used Deeper Search or manually set p5 to `L` and added `E` for p3.

4. **Partition Not Recognized Post-Recovery**:
   - **Issue**: `/dev/loop0p5` missing after TestDisk (`special device does not exist`).
   - **Resolution**: Used `sudo partprobe /dev/loop0` or `kpartx` to refresh kernel partition table.

## Lessons Learned

- **Partitioning**: MBR supports primary, extended, and logical partitions. Logical partitions (e.g., p5) require an extended container (p3).
- **Filesystems**: ext4 works on small partitions; XFS requires ≥300 MiB.
- **Recovery**: TestDisk’s Quick Search may miss extended/logical partitions; Deeper Search or manual adjustments are needed.
- **Loop Devices**: `losetup -P` enables partition detection; `partprobe` or `kpartx` resolves kernel sync issues.
- **Mounting**: Always mount before writing to ensure data goes to the disk image, not the local filesystem.
- **CompTIA Linux+ Skills**: This scenario reinforced disk management, filesystem creation, MBR recovery, and troubleshooting—key for the certification.

## Conclusion

This scenario successfully demonstrated creating a multi-partition disk image, corrupting the MBR, and recovering it with TestDisk, verifying data integrity. Despite challenges (XFS size, data writing errors, partition detection), all were resolved through careful troubleshooting, making this a valuable learning experience for Linux system administration.
