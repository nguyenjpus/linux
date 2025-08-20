<xaiArtifact artifact_id="9b6b3ff1-bcc8-4f85-8902-f82520dcdc6f" artifact_version_id="042f680a-c929-420c-8e8b-ef2f2ab46048" title="filesystem_test.md" contentType="text/markdown">

# Testing Filesystem Creation and Management

This guide outlines steps to create, mount, unmount, and check a test filesystem using Linux commands. These steps help build familiarity with filesystem operations.

## Step 1: Create a Test Filesystem

Create a 100MB disk image and format it as an ext4 filesystem.

```bash
sudo dd if=/dev/zero of=test.img bs=1M count=100
sudo mkfs.ext4 test.img
```

**Mnemonic**: "DD: Input Flows Out, Block_Size Count" (for `dd` command: input file, output file, block size, count).

## Step 2: Mount the Filesystem

Create a mount point and mount the disk image.

```bash
sudo mkdir /mnt/test
sudo mount test.img /mnt/test
```

## Step 3: Unmount and Check the Filesystem

Unmount the filesystem and perform a filesystem check.

```bash
sudo umount /mnt/test
sudo fsck -f test.img
```

**Note**: If `sudo fsck -f test.img` fails, use the explicit ext4 check:

```bash
sudo fsck.ext4 -f test.img
```

## Step 4: Repeat and Experiment

Repeat the above steps multiple times to gain familiarity. Experiment with variations, such as:

- Adding timestamps to system logs for debugging:
  ```bash
  dmesg -T
  ```
- Modifying `dd` options (e.g., change `bs` or `count` for different image sizes).
- Exploring other filesystem types (e.g., `mkfs.xfs` or `mkfs.vfat`).

</xaiArtifact>
