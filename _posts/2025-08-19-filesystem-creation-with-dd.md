Testing Filesystem Creation and Management

This guide outlines steps to create, mount, unmount, and check a test filesystem using Linux commands. These steps help build familiarity with filesystem operations.

Step 1: Create a Test Filesystem

Create a 100MB disk image and format it as an ext4 filesystem.

sudo dd if=/dev/zero of=test.img bs=1M count=100
sudo mkfs.ext4 test.img

Mnemonic: "DD: Input Flows Out, Block_Size Count" (for dd command: input file, output file, block size, count).

Step 2: Mount the Filesystem

Create a mount point and mount the disk image.

sudo mkdir /mnt/test
sudo mount test.img /mnt/test

Step 3: Unmount and Check the Filesystem

Unmount the filesystem and perform a filesystem check.

sudo umount /mnt/test
sudo fsck -f test.img

Note: If sudo fsck -f test.img fails, use the explicit ext4 check:

sudo fsck.ext4 -f test.img

Step 4: Repeat and Experiment

Repeat the above steps multiple times to gain familiarity. Experiment with variations, such as:

Adding timestamps to system logs for debugging:

dmesg -T

Modifying dd options (e.g., change bs or count for different image sizes).

Exploring other filesystem types (e.g., mkfs.xfs or mkfs.vfat).
