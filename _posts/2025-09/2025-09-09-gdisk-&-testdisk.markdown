# GPT/EFI Partition Table Recovery with TestDisk and gdisk Comparison - Complete Lesson

## Objective

This scenario, part of CompTIA Linux+ certification practice, focuses on creating a disk image with a GPT partition table, formatting partitions with ext4 and XFS, adding test data, corrupting the GPT (primary and backup tables), and recovering using TestDisk and gdisk. The experiment demonstrates GPT management, recovery techniques, and troubleshooting, highlighting differences between TestDisk and gdisk.

## Complete Process Overview

The practice was conducted in two parts:

- **Part 1**: Initial setup, corruption, and recovery with TestDisk.
- **Part 2**: Re-corruption and recovery attempt with gdisk, followed by a TestDisk retry with specific attention to partition type adjustments.

## Steps and Commands (Combined from Both Experiments)

### 1. Create a Disk Image

Create a 1000 MiB disk image.

```bash
dd if=/dev/zero of=~/gpt.img bs=1M count=1000
```

- **Output**: Created `~/gpt.img`.

### 2. Partition the Disk Image with GPT

Use `gdisk` to create GPT partitions.

```bash
sudo gdisk ~/gpt.img
```

- Commands:
  - `o`: Create new GPT.
  - `n > 1 > Enter > +100M > 8300`: Partition 1 (100 MiB, Linux).
  - `n > 2 > Enter > +200M > 8300`: Partition 2 (200 MiB, Linux).
  - `n > 3 > Enter > +300M > 8300`: Partition 3 (300 MiB, Linux).
  - `n > 4 > Enter > +320M > 8300`: Partition 4 (320 MiB, Linux; adjusted from initial 398 MiB to fix sizing error).
  - `p`: Verify:
    ```
    Number  Start (sector)    End (sector)  Size       Code  Name
       1            2048          206847   100.0 MiB   8300  Linux filesystem
       2          206848          616447   200.0 MiB   8300  Linux filesystem
       3          616448         1230847   300.0 MiB   8300  Linux filesystem
       4         1230848         1886207   320.0 MiB   8300  Linux filesystem
    ```
  - `w`: Write changes.
- **Troubleshooting**: Initial partition 4 oversized; deleted (`d > 4`) and recreated with `+320M`.

### 3. Set Up Loop Device and Format Partitions

Attach and format.

```bash
sudo losetup -P /dev/loop0 ~/gpt.img
lsblk /dev/loop0
```

- **Output**: Showed partitions p1–p4.
- Format:
  ```bash
  sudo mkfs.ext4 /dev/loop0p1
  sudo mkfs.ext4 /dev/loop0p2  # Used ext4 after XFS failed on 200 MiB
  sudo mkfs.xfs /dev/loop0p3
  sudo mkfs.xfs /dev/loop0p4
  ```
- **Issue**: XFS on p2 failed ("Filesystem must be larger than 300MB"); used ext4.

### 4. Add Test Data

Mount and write files.

```bash
sudo mkdir -p /mnt/p1 /mnt/p2 /mnt/p3 /mnt/p4
sudo mount /dev/loop0p1 /mnt/p1
sudo mount /dev/loop0p2 /mnt/p2
sudo mount /dev/loop0p3 /mnt/p3
sudo mount /dev/loop0p4 /mnt/p4
echo "Data on partition 1" | sudo tee /mnt/p1/file1.txt
echo "Data on partition 2" | sudo tee /mnt/p2/file2.txt
echo "Data on partition 3" | sudo tee /mnt/p3/file3.txt
echo "Data on partition 4" | sudo tee /mnt/p4/file4.txt
cat /mnt/p1/file1.txt  # And others
```

- **Output**: Files created and verified.

### 5. Unmount and Detach

```bash
sudo umount /mnt/p1 /mnt/p2 /mnt/p3 /mnt/p4
sudo losetup -d /dev/loop0
```

- **Issue**: Mounts sometimes persisted; ensured unmount before detach.

### 6. Corrupt the GPT

Corrupt primary and backup GPT.

```bash
dd if=/dev/zero of=~/gpt.img bs=512 count=34 conv=notrunc
dd if=/dev/zero of=~/gpt.img bs=512 count=33 seek=$(($(stat -c %s ~/gpt.img)/512 - 33)) conv=notrunc
```

- **Output**: GPT corrupted; `lsblk` after attach showed no partitions.

### 7. Recovery with TestDisk (Part 1)

```bash
sudo testdisk ~/gpt.img
```

- Steps:
  - `[Create]` (log for debugging).
  - `[EFI GPT] > [Analyse] > [Quick Search]`.
  - Detected partitions; `[Write]` to restore.
- **Output**: Rebooted, reattached, mounted, and verified files.

### 8. Re-Corruption and Recovery with gdisk (Part 2)

Re-corrupt GPT (same `dd` commands).

- Recovery with gdisk:
  ```bash
  sudo gdisk ~/gpt.img
  ```
  - `r`: Enter recovery mode.
  - `b`: Use backup header.
  - `w`: Write changes.
- **Output**: Warned about kernel using old table; ran `sudo partprobe` (had warnings), tried `kpartx`, then rebooted.
- After reboot:
  ```bash
  sudo losetup -P /dev/loop0 ~/gpt.img
  lsblk  # Showed no partitions initially (recovery incomplete)
  ```

### 9. Retry with TestDisk (Part 2 with Detailed Steps)

Re-corrupt GPT and retry recovery with TestDisk.

```bash
sudo testdisk ~/gpt.img
```

- Steps:
  - `[Create]` (log for debugging).
  - `[EFI GPT] > [Analyse] > [Quick Search]`.
  - TestDisk output showed partitions marked as deleted:
    ```
    TestDisk 7.1, Data Recovery Utility, July 2019
    Christophe GRENIER <grenier@cgsecurity.org>
    https://www.cgsecurity.org
    Disk /home/ron/gpt.img - 1048 MB / 1000 MiB - CHS 128 255 63
         Partition               Start        End    Size in sectors
    >D Linux                    0  32 33    12 223 19     204800
     D Linux                   12 223 20    38  94 56     409600
     D Linux                   38  94 57    76 157 17     614400
     D Linux                   76 157 18   117 104 51     655360
    ```
  - Changed all `D` (deleted) to `P` (primary) to match original GPT structure:
    ```
    Disk /home/ron/gpt.img - 1048 MB / 1000 MiB - CHS 128 255 63
         Partition               Start        End    Size in sectors
     P Linux                    0  32 33    12 223 19     204800
     P Linux                   12 223 20    38  94 56     409600
     P Linux                   38  94 57    76 157 17     614400
    >P Linux                   76 157 18   117 104 51     655360
    ```
  - `[Write]`, confirmed with `Y`, and rebooted.
- **Output**: After reboot, reattached and verified:
  ```bash
  sudo losetup -P /dev/loop0 ~/gpt.img
  lsblk  # Showed p1–p4
  sudo mount /dev/loop0p1 /mnt/p1  # And others
  cat /mnt/p1/file1.txt  # And others
  ```
  - Files confirmed: `Data on partition 1`, `Data on partition 2`, `Data on partition 3`, `Data on partition 4`.

### 10. Verify Final Recovery

```bash
sudo losetup -P /dev/loop0 ~/gpt.img
lsblk  # Confirmed partitions
sudo mount /dev/loop0p1 /mnt/p1  # And others
cat /mnt/p1/file1.txt  # And others
```

- **Output**: All files recovered.

### 11. Cleanup

```bash
sudo umount /mnt/p1 /mnt/p2 /mnt/p3 /mnt/p4
sudo losetup -d /dev/loop0
rm ~/gpt.img
sudo rmdir /mnt/p1 /mnt/p2 /mnt/p3 /mnt/p4
```

## Issues Encountered and Resolutions

1. **Partition Sizing in gdisk**:

   - **Issue**: Partition 4 exceeded disk boundaries.
   - **Resolution**: Deleted and recreated with `+320M`.

2. **XFS Formatting Failure**:

   - **Issue**: p2 (200 MiB) too small for XFS.
   - **Resolution**: Used ext4 for p2.

3. **Kernel Partition Table Sync**:

   - **Issue**: `lsblk` showed no partitions after gdisk/TestDisk; `partprobe` warnings.
   - **Resolution**: Rebooted; ignored `/dev/sr0` warnings.

4. **Loop Device Detach/Mount Persistence**:

   - **Issue**: Mounts persisted after `losetup -d`; multiple `losetup -d` failures.
   - **Resolution**: Unmounted first; used correct `losetup -d /dev/loop0`.

5. **gdisk Recovery Incompleteness**:

   - **Issue**: gdisk (`r > b`) didn’t sync partitions; `lsblk` showed no subdevices.
   - **Resolution**: TestDisk recovered reliably; gdisk needed reboot/sync.

6. **TestDisk Deleted Partitions**:
   - **Issue**: Partitions marked `D` (deleted) in TestDisk.
   - **Resolution**: Manually changed to `P` (primary) to match GPT structure before writing.

## Lessons Learned

- **GPT Structure**: GPT’s primary and backup tables enhance redundancy; corrupting both requires signature-based recovery.
- **TestDisk for GPT**: Automatically detects partitions but may mark them `D` (deleted); verify and set to `P` for GPT primary partitions.
- **gdisk Usage**: Effective for manual recovery (`r > b`) but requires kernel sync (`partprobe`/reboot).
- **gdisk vs. TestDisk**: TestDisk is more automated and reliable for GPT recovery; gdisk offers precise control but may need additional steps.
- **Filesystem Choices**: XFS requires ≥300 MiB; ext4 suits smaller partitions.
- **Loop Devices and Kernel**: Unmount before detaching; use `partprobe` or reboot for table updates.
- **CompTIA Linux+ Skills**: Mastered GPT partitioning, recovery with TestDisk/gdisk, and troubleshooting kernel sync issues.

## Conclusion

Scenario 3 demonstrated successful GPT recovery using TestDisk and gdisk. TestDisk’s automated recovery, with careful adjustment of partition types (`D` to `P`), proved effective, while gdisk required additional sync steps. The experiment reinforced critical Linux disk management skills for the CompTIA Linux+ certification.
