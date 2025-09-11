# GPT/EFI Partition Table Recovery with TestDisk and gdisk Comparison - Complete Lesson with LUKS

## Objective

Continue practicing the previous lessons (posted on 2025/09/09) focusing on LUKS.

### Detailed Step-by-Step Guide to Proceed

Since the original LUKS data was overwritten by `luksFormat`, we’ll:

1. Restart from a clean disk image to simulate the LUKS setup and corruption.
2. Recover the partition table using `gdisk` (since TestDisk is unreliable with LUKS).
3. Access the encrypted data to confirm recovery.
   This guide assumes you want to practice the full process again, including LUKS setup, corruption, and recovery.

#### 0. Reset the Environment

Clean up any existing setup:

```
sudo umount /mnt/encrypted /mnt/p1 /mnt/p2 /mnt/p3 /mnt/p4 2>/dev/null
sudo cryptsetup luksClose encrypted 2>/dev/null
sudo losetup -d /dev/loop0 2>/dev/null
rm ~/gpt.img 2>/dev/null
```

#### 1. Recreate the Disk Image

Create a new 1000 MiB image:

```
dd if=/dev/zero of=~/gpt.img bs=1M count=1000
```

#### 2. Partition the Disk Image

Recreate the GPT partitions:

```
sudo gdisk ~/gpt.img
```

- Commands:
  - `o`: Create new GPT (confirm with `Y`).
  - `n > 1 > 2048 > +100M > 8300`: p1 (100 MiB).
  - `n > 2 > 206848 > +200M > 8300`: p2 (200 MiB).
  - `n > 3 > 616448 > +300M > 8300`: p3 (300 MiB).
  - `n > 4 > 1230848 > +320M > 8300`: p4 (320 MiB).
  - `p`: Verify:
    ```
    Number  Start (sector)    End (sector)  Size       Code  Name
       1            2048          206847   100.0 MiB   8300  Linux filesystem
       2          206848          616447   200.0 MiB   8300  Linux filesystem
       3          616448         1230847   300.0 MiB   8300  Linux filesystem
       4         1230848         1886207   320.0 MiB   8300  Linux filesystem
    ```
  - `w`: Write and exit.

#### 3. Set Up Loop Device and Format Partitions

Attach the image:

```
sudo losetup -P /dev/loop0 ~/gpt.img
lsblk /dev/loop0
```

- **Expected Output**: Shows p1–p4.

Set up LUKS on p1 and format others:

```
sudo cryptsetup luksFormat /dev/loop0p1
```

- Type `YES`, use passphrase "test123".

```
sudo cryptsetup luksOpen /dev/loop0p1 encrypted
```

- Enter "test123".

```
sudo mkfs.ext4 /dev/mapper/encrypted
sudo mkfs.ext4 /dev/loop0p2
sudo mkfs.xfs /dev/loop0p3
sudo mkfs.xfs /dev/loop0p4
```

#### 4. Add Test Data

Mount and write files:

```
sudo mkdir /mnt/encrypted /mnt/p2 /mnt/p3 /mnt/p4
sudo mount /dev/mapper/encrypted /mnt/encrypted
sudo mount /dev/loop0p2 /mnt/p2
sudo mount /dev/loop0p3 /mnt/p3
sudo mount /dev/loop0p4 /mnt/p4
echo "Encrypted test data" | sudo tee /mnt/encrypted/encrypted.txt
echo "Data on partition 2" | sudo tee /mnt/p2/file2.txt
echo "Data on partition 3" | sudo tee /mnt/p3/file3.txt
echo "Data on partition 4" | sudo tee /mnt/p4/file4.txt
cat /mnt/encrypted/encrypted.txt  # And others
```

- **Expected Output**: Verify all files.

Unmount and close:

```
sudo umount /mnt/encrypted /mnt/p2 /mnt/p3 /mnt/p4
sudo cryptsetup luksClose encrypted
sudo losetup -d /dev/loop0
```

#### 5. Corrupt the GPT

Wipe headers:

```
sudo dd if=/dev/zero of=~/gpt.img bs=512 count=34 conv=notrunc
sudo dd if=/dev/zero of=~/gpt.img bs=512 count=33 seek=$(($(stat -c %s ~/gpt.img)/512 - 33)) conv=notrunc
```

#### 6. Recover with gdisk (Preferred)

Since TestDisk failed, use `gdisk`:

```
sudo gdisk ~/gpt.img
```

- Commands:
  - `o`: Create new GPT (`Y`).
  - Recreate partitions exactly as above (step 2).
  - `p`: Verify table.
  - `w`: Write.
- Sync:
  ```
  sudo partprobe
  ```
  - If errors, reboot:
    ```
    sudo reboot
    ```

#### 7. Verify Recovery and Access Data

Reattach:

```
sudo losetup -P /dev/loop0 ~/gpt.img
lsblk /dev/loop0
```

- **Expected Output**: Shows p1–p4.

Access LUKS:

```
sudo cryptsetup luksOpen /dev/loop0p1 encrypted
```

- Enter "test123".

```
sudo e2fsck -n /dev/mapper/encrypted
sudo mkdir /mnt/encrypted
sudo mount /dev/mapper/encrypted /mnt/encrypted
cat /mnt/encrypted/encrypted.txt
```

- **Expected Output**: `Encrypted test data`.

Verify p2–p4:

```
sudo mount /dev/loop0p2 /mnt/p2
sudo mount /dev/loop0p3 /mnt/p3
sudo mount /dev/loop0p4 /mnt/p4
cat /mnt/p2/file2.txt  # And others
```

#### 8. Cleanup

```
sudo umount /mnt/encrypted /mnt/p2 /mnt/p3 /mnt/p4
sudo cryptsetup luksClose encrypted
sudo rmdir /mnt/encrypted /mnt/p2 /mnt/p3 /mnt/p4
sudo losetup -d /dev/loop0
```

#### 9. Optional: Retry TestDisk again as follow but failed to recover the partitions.

If you want to try TestDisk again:

```
sudo testdisk ~/gpt.img
```

- Follow steps from previous response, manually adding partitions if needed:
  - Use sector values: p1 (2048–206847), p2 (206848–616447), p3 (616448–1230847), p4 (1230848–1886207).
  - Set all to `P` (primary).
  - Write and sync.

---

### Troubleshooting Tips

- **LUKS Open Fails**: Verify `/dev/loop0p1` exists. Check LUKS header: `sudo cryptsetup luksDump /dev/loop0p1`.
- **TestDisk Fails**: Switch to `gdisk` for reliable manual recreation.
- **Mount Errors**: Run `sudo e2fsck -fy /dev/mapper/encrypted` for p1, or `xfs_repair` for p3/p4.

---

### Analysis of Your Output

1. **LUKS Setup and Questions**:

   - Your questions:
     - **"To encrypt a partition, we have to cryptsetup it first using luksOpen, this will mount this encrypted partition on /dev/mapper/name-of-the-cryptsetup-partition"**:
       - **Clarification**: This is partially correct. `cryptsetup luksFormat` creates the encrypted container. `cryptsetup luksOpen` unlocks it, mapping it to `/dev/mapper/<name>` (e.g., `/dev/mapper/encrypted`). This mapped device represents the decrypted contents, but it’s not mounted yet—it’s just a block device. You then format (`mkfs.ext4`) and mount (`mount`) the mapped device to a directory (e.g., `/mnt/encrypted`) to access it as a filesystem.
     - **"What is the previous step with /dev/mapper for?"**:
       - **Answer**: `/dev/mapper/encrypted` is the decrypted view of the LUKS container. `luksOpen` decrypts the partition using the passphrase, creating a virtual device in `/dev/mapper` that exposes the plaintext data. You then format or mount this device, not the raw `/dev/loop0p1`, because the latter is encrypted and unreadable without decryption.
   - You correctly closed the LUKS container and detached the loop device.

2. **Why Recovery Failed**:
   - **TestDisk**: As noted previously, LUKS encryption on p1 obscures the ext4 signature, and XFS on p3/p4 is harder for TestDisk to detect after complete GPT corruption.
   - **gdisk**: The backup GPT header was overwritten (`dd ... count=33`), so `r > b` couldn’t recover it. Manual recreation is needed, as you did initially.
   - **Key Issue**: The initial `gdisk` recreation was correct, but re-running `luksFormat` erased the original encrypted data, making recovery of the original `encrypted.txt` impossible. The new LUKS container you created works, but it’s a fresh setup, not a recovery.

---

### Clarification on Your LUKS Understanding

- **Correct Part**: To encrypt a partition, you use `cryptsetup luksFormat` to create the LUKS container, then `cryptsetup luksOpen` to unlock it, creating a `/dev/mapper/<name>` device.
- **Misconception**: `luksOpen` doesn’t mount the partition; it only maps the decrypted contents to `/dev/mapper`. You must explicitly mount `/dev/mapper/<name>` to a directory (e.g., `/mnt/encrypted`) to access the filesystem.
- **Process**:
  1. `luksFormat`: Sets up encryption on the raw partition (e.g., `/dev/loop0p1`).
  2. `luksOpen`: Unlocks the container, creating `/dev/mapper/encrypted`.
  3. `mkfs.ext4`: Formats the decrypted device (if new).
  4. `mount`: Mounts the decrypted device to a directory to access files.
