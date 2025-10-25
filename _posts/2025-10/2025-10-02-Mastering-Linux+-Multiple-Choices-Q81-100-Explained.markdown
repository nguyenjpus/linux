---
layout: post
title: "Multiple-choices - Questions 81-100 Explained"
date: 2025-10-02 10:30:00 -0700
tags: [Linux+, Multiple-choices]
---

This document explains questions 81-100 from a set of 100 scenario-based multiple-choices questions for CompTIA Linux+ preparation, covering filesystem management, SSH troubleshooting, LVM, scripting, virtualization, RAID, process management, kernel modules, networking, and cloud provisioning. Questions 84, 85, 86, 87, and 94 have been updated to clarify `xargs` options, RAW disk format in UTM, `/etc/fstab` in the boot process, `-mtime -1`, and swap file creation. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention.

## Question 81: Freeing Up Space on a Full Filesystem

**Question**: A web server’s root filesystem (/) is 99% full. After investigation, an administrator finds that `/var/log` is consuming most of the space. The `/var/log` directory is part of the root filesystem. What is the most appropriate immediate action to free up space without causing data loss?  
**Options**:

- Reboot the server to clear temporary files.
- Archive old log files to a compressed tarball, verify the archive, and then delete the original log files.
- Use `lvextend` to increase the size of the root filesystem.
- Delete all files in `/tmp`.

**Correct Answer**: Archive old log files to a compressed tarball, verify the archive, and then delete the original log files.  
**Why Correct**: Archiving old log files (e.g., `tar -czvf logs.tar.gz /var/log/*.log`), verifying the archive (e.g., `tar -tzvf logs.tar.gz`), and deleting the originals safely frees space without data loss.  
**Why Others Wrong**:

- Rebooting may clear some temporary files but doesn’t address `/var/log` size.
- `lvextend` requires LVM and free space in the volume group, not guaranteed here.
- Deleting `/tmp` files may disrupt running processes and doesn’t target `/var/log`.  
  **Key Concept**: Log rotation or archiving preserves data; use `logrotate` for automation.  
  **Memory Aid**: “Tar, verify, delete = Safe space saver.”

## Question 82: SSH Connection Failure

**Question**: A user reports they cannot SSH into a new server at 10.10.5.25. The administrator on the server runs `ss -tlpn` and sees no process listening on port 22. The firewall is disabled. What is the most likely issue?  
**Options**:

- The server’s `/etc/resolv.conf` is missing.
- The SSH daemon (sshd) service is not running.
- The user is using the wrong password.
- The server’s default gateway is not configured.

**Correct Answer**: The SSH daemon (sshd) service is not running.  
**Why Correct**: If `ss -tlpn` shows no process on port 22, `sshd` is not running, preventing SSH connections.  
**Why Others Wrong**:

- A missing `/etc/resolv.conf` affects DNS, not port 22 binding.
- A wrong password would still show `sshd` listening on port 22.
- A misconfigured gateway affects outbound traffic, not incoming SSH.  
  **Key Concept**: Check `systemctl status sshd` or `service sshd start` to fix.  
  **Memory Aid**: “No port 22 = sshd not running.”

## Question 83: LVM Commands for New Disk

**Question**: An administrator adds a new 1TB disk (`/dev/sdd`) to a server to expand the `vg_data` volume group. After creating a partition (`/dev/sdd1`), what is the correct sequence of LVM commands?  
**Options**:

- vgextend vg_data /dev/sdd1 followed by pvcreate /dev/sdd1
- pvcreate /dev/sdd1 followed by vgextend vg_data /dev/sdd1
- lvcreate -L 1T -n new_lv vg_data followed by pvcreate /dev/sdd1
- mkfs.ext4 /dev/sdd1 followed by vgextend vg_data /dev/sdd1

**Correct Answer**: pvcreate /dev/sdd1 followed by vgextend vg_data /dev/sdd1  
**Why Correct**: First, `pvcreate /dev/sdd1` initializes the partition as an LVM physical volume. Then, `vgextend vg_data /dev/sdd1` adds it to the `vg_data` volume group.  
**Why Others Wrong**:

- `vgextend` before `pvcreate` fails; the partition must be a physical volume first.
- `lvcreate` creates a logical volume, not relevant until the volume group is extended.
- `mkfs.ext4` formats the partition, bypassing LVM setup.  
  **Key Concept**: LVM sequence: `pvcreate` → `vgextend` → `lvcreate`.  
  **Memory Aid**: “PV first, then VG expand.”

## Question 84: Processing a Server List with Ping

**Question**: A script needs to process a list of servers from a file named `server_list.txt`. The administrator wants to execute a `ping` command for each server listed in the file. Which command construct correctly reads the file line by line and executes the command?  
**Options**:

- A) `for i in $(cat server_list.txt); do ping -c 1 $i; done`
- B) `cat server_list.txt | xargs ping -c 1`
- C) `ping -c 1 < server_list.txt`
- D) Both A and B are common and effective methods.

**Correct Answer**: D) Both A and B are common and effective methods.  
**Why Correct**:

- Option A uses a `for` loop to read `server_list.txt` line by line, executing `ping -c 1` for each server. It works but may split lines with spaces incorrectly unless quoted properly (e.g., `for i in "$(cat server_list.txt)"` mitigates this).
- Option B uses `xargs` to pass each line to `ping -c 1`. By default, `xargs` splits input on whitespace, but adding `xargs -I{}` or `xargs -d '\n'` ensures better handling of spaces:
  - `xargs -I{}` specifies a placeholder (`{}`) for each line, so `cat server_list.txt | xargs -I{} ping -c 1 {}` treats each line as a single argument, preserving spaces (e.g., “server name” pings as one hostname).
  - `xargs -d '\n'` sets the delimiter to newline, so `cat server_list.txt | xargs -d '\n' ping -c 1` processes each line intact, avoiding splitting on spaces. Both methods make Option B robust for complex hostnames.  
    **Why Others Wrong**:
- Option C is invalid; `ping` doesn’t read input from stdin via `<`.  
  **Key Concept**: `for` loops and `xargs` process files line by line; `xargs -I{}` or `-d '\n'` ensures proper handling of spaces and special characters.  
  **Memory Aid**: “For or xargs = Ping each line; -I{} or -d '\n' saves spaces.”

## Question 85: Disk Format for Maximum I/O Performance

**Question**: An administrator is setting up a KVM virtual machine that will run a database. For maximum disk I/O performance, which virtual disk format is generally recommended, even though it lacks features like snapshots?  
**Options**:

- QCOW2
- VMDK
- VDI
- RAW

**Correct Answer**: RAW  
**Why Correct**: The RAW disk format offers maximum I/O performance for KVM VMs by directly mapping to a disk or file without overhead, ideal for databases requiring high-speed read/write operations. Unlike QCOW2, RAW doesn’t support snapshots, compression, or copy-on-write, as it stores data as a direct, unformatted block image, minimizing processing. In UTM on macOS, when adding a disk (e.g., via the “Drives” section), you can select RAW (or “raw disk image”) alongside QCOW2, VMDK, and VDI. RAW’s lack of metadata makes it faster but less feature-rich, aligning with the question’s focus on performance over features.  
**Why Others Wrong**:

- QCOW2 supports snapshots and compression but adds overhead, reducing I/O performance.
- VMDK (VMware) and VDI (VirtualBox) are not native to KVM and introduce similar overhead, making them less optimal.  
  **Key Concept**: RAW prioritizes speed by avoiding format overhead; QCOW2 offers features at a performance cost. In UTM, RAW is available and ideal for performance-critical VMs like databases.  
  **Memory Aid**: “RAW = Fastest disk I/O, no frills.”

## Question 86: Component Reading /etc/fstab

**Question**: A server fails to boot, dropping into an emergency shell. The administrator suspects a corrupted `/etc/fstab` file. During the boot process, which component is responsible for reading `/etc/fstab` and mounting the filesystems?  
**Options**:

- The GRUB 2 bootloader
- The Linux kernel itself
- The systemd init process
- The mount command in /etc/rc.local

**Correct Answer**: The systemd init process  
**Why Correct**: The `systemd` init process reads `/etc/fstab` during boot to mount filesystems as part of system initialization. Regarding the boot process (UEFI → GRUB2 → kernel → initramfs → systemd), `/etc/fstab` is not typically included in the initramfs, which contains a minimal filesystem and scripts to mount the root filesystem (often specified via kernel parameters like `root=`). Instead, `/etc/fstab` resides on the root filesystem (e.g., `/dev/vda1`). Once the kernel and initramfs mount the root filesystem, `systemd` takes over, parsing `/etc/fstab` to mount additional filesystems (e.g., `/home`, `/var`). A corrupted `/etc/fstab` can cause `systemd` to fail, dropping to an emergency shell, as seen in the question.  
**Why Others Wrong**:

- GRUB 2 loads the kernel and initramfs but doesn’t access `/etc/fstab`.
- The kernel handles low-level mounting but not `/etc/fstab` parsing.
- `/etc/rc.local` is for custom scripts, not standard mounting.  
  **Key Concept**: `systemd` manages mounts via `/etc/fstab` after the root filesystem is mounted; initramfs handles initial mounts.  
  **Memory Aid**: “systemd = Mounts fstab at boot, post-initramfs.”

## Question 87: Archiving Recently Modified Files

**Question**: An administrator needs to find all files in their home directory that have been modified in the last 24 hours and create a compressed archive of them. Which command pipeline would achieve this?  
**Options**:

- `find ~ -mtime -1 -print0 | xargs -0 tar -czvf last_day.tar.gz`
- `ls -lt ~ | head | tar -czvf last_day.tar.gz`
- `tar -czvf last_day.tar.gz $(find ~ -mtime -1)`
- `grep -r "$(date)" ~ | tar -czvf last_day.tar.gz`

**Correct Answer**: `find ~ -mtime -1 -print0 | xargs -0 tar -czvf last_day.tar.gz`  
**Why Correct**: The `find ~ -mtime -1` command locates files in the home directory modified within the last 24 hours; `-mtime -1` specifies files whose modification time is less than 1 day old (measured in 24-hour periods from now). The `-print0` option uses null terminators to handle filenames with spaces or special characters, and `xargs -0` passes these to `tar -czvf` to create a compressed gzip archive (`last_day.tar.gz`).  
**Why Others Wrong**:

- `ls -lt | head` lists files but limits output, missing many files.
- `$(find ~ -mtime -1)` splits on spaces, breaking filenames with spaces.
- `grep -r "$(date)"` searches file contents, not modification times.  
  **Key Concept**: `find -mtime -n` finds files modified less than n days ago; `-print0` and `xargs -0` ensure safe file handling.  
  **Memory Aid**: “find -mtime -1 + xargs -0 = Safe tar for recent files.”

## Question 88: NAT Blocking SSH Connections

**Question**: A virtual machine’s network is configured to use NAT. The VM can access the internet, but a user on the external network cannot initiate an SSH connection to the VM. Why is this happening?  
**Options**:

- The VM’s firewall is blocking the connection.
- NAT, by default, does not allow unsolicited inbound connections from the external network.
- The SSH service is not running on the VM.
- The host machine’s network is down.

**Correct Answer**: NAT, by default, does not allow unsolicited inbound connections from the external network.  
**Why Correct**: NAT hides the VM behind the host’s IP, translating outbound traffic but blocking unsolicited inbound connections (like SSH) unless port forwarding is configured.  
**Why Others Wrong**:

- The VM’s firewall may block connections, but NAT’s behavior is the primary issue.
- If SSH wasn’t running, it wouldn’t explain internet access.
- The host’s network being down would prevent VM internet access.  
  **Key Concept**: NAT requires port forwarding for inbound connections; bridged networking allows direct access.  
  **Memory Aid**: “NAT = No inbound without forwarding.”

## Question 89: Paging Long Output

**Question**: An administrator is running a command that produces a very long output. They want to view the output one page at a time. Which command should they pipe the output to?  
**Options**:

- less or more
- cat
- head
- tail

**Correct Answer**: less or more  
**Why Correct**: Piping output to `less` or `more` allows paging through long output one screen at a time; `less` is more feature-rich (e.g., backward navigation).  
**Why Others Wrong**:

- `cat` dumps output without paging.
- `head` shows only the first lines.
- `tail` shows only the last lines.  
  **Key Concept**: `less` for interactive paging; `more` for simpler paging.  
  **Memory Aid**: “less or more = Page through output.”

## Question 90: RAID 5 vs. RAID 1

**Question**: A server has three disks configured in a RAID 5 array. What is the primary benefit of this configuration compared to RAID 1?  
**Options**:

- It provides better read performance and more usable storage capacity than RAID 1 with the same number of disks.
- It is simpler to configure than RAID 1.
- It does not require a dedicated controller.
- It provides double the redundancy of RAID 1.

**Correct Answer**: It provides better read performance and more usable storage capacity than RAID 1 with the same number of disks.  
**Why Correct**: RAID 5 stripes data with parity across three or more disks, offering better read performance and more usable space (n-1 disks) than RAID 1 (mirroring, n/2 disks).  
**Why Others Wrong**:

- RAID 5 is more complex to configure than RAID 1.
- Both RAID levels can use software or hardware controllers.
- RAID 5 has single-disk failure tolerance, less than RAID 1’s mirroring.  
  **Key Concept**: RAID 5 maximizes storage; RAID 1 maximizes redundancy.  
  **Memory Aid**: “RAID 5 = More space, fast reads.”

## Question 91: Adjusting Process Priority

**Question**: A user is running a script that is consuming 100% of a CPU core. To lower its priority and make it “nicer” to other processes, which command can be used to change the priority of a running process?  
**Options**:

- nice
- renice
- kill -9
- top

**Correct Answer**: renice  
**Why Correct**: `renice` adjusts the priority (nice value) of a running process (e.g., `renice 10 <PID>`), lowering its CPU priority.  
**Why Others Wrong**:

- `nice` sets priority for new processes, not running ones.
- `kill -9` terminates processes, not adjusts priority.
- `top` monitors processes, not changes priority.  
  **Key Concept**: `renice` for running processes; `nice` for new ones.  
  **Memory Aid**: “renice = Reprioritize running process.”

## Question 92: Symbolic Link Indicator

**Question**: An administrator is examining a directory and sees a file entry like `lrwxrwxrwx 1 root root 4 Jul 26 20:00 link -> file`. What does the `l` at the beginning of the permissions string indicate?  
**Options**:

- The file is a log file.
- The file is locked.
- The file is a symbolic link.
- The file has large file support enabled.

**Correct Answer**: The file is a symbolic link.  
**Why Correct**: The `l` in `lrwxrwxrwx` indicates a symbolic link, pointing to another file (`file` in this case).  
**Why Others Wrong**:

- Log files have no special permission indicator.
- Locked files don’t use `l`; they may use attributes (e.g., `chattr`).
- Large file support is not indicated in permissions.  
  **Key Concept**: `l` in `ls -l` denotes a symbolic link; use `ln -s` to create.  
  **Memory Aid**: “l = Link, symbolic.”

## Question 93: Persistent Kernel Module Parameters

**Question**: A new kernel module for a specific device needs a custom parameter to be set every time it is loaded. Where should the administrator define this option persistently?  
**Options**:

- In a conf file within `/etc/modprobe.d/` using the syntax `options [module_name] [param]=[value]`.
- In the `/etc/modules` file.
- As a kernel boot parameter in `/etc/default/grub`.
- In the user’s `~/.bashrc` file.

**Correct Answer**: In a conf file within `/etc/modprobe.d/` using the syntax `options [module_name] [param]=[value]`.  
**Why Correct**: A file in `/etc/modprobe.d/` (e.g., `mymodule.conf`) with `options module_name param=value` sets persistent module parameters.  
**Why Others Wrong**:

- `/etc/modules` lists modules to load, not parameters.
- GRUB parameters affect the kernel, not specific modules.
- `~/.bashrc` is for user shell settings, not modules.  
  **Key Concept**: `/etc/modprobe.d/` for module options; syntax is `options`.  
  **Memory Aid**: “modprobe.d = Module options persist.”

## Question 94: Creating a Swap File

**Question**: An administrator needs to create a 1GB file filled with zeros, to be used as a swap file. Which command is most efficient for creating a file of a specific size without writing all the data immediately?  
**Options**:

- `dd if=/dev/zero of=swapfile bs=1G count=1`
- `truncate -s 1G swapfile`
- `fallocate -l 1G swapfile`
- Both B and C are efficient methods for this.

**Correct Answer**: Both B and C are efficient methods for this.  
**Why Correct**:

- `truncate -s 1G swapfile` creates a 1GB sparse file on the hard drive (not RAM), reserving space without immediately writing zeros. “Lazily” means the file appears as 1GB but only consumes disk space as data is written, optimizing creation speed.
- `fallocate -l 1G swapfile` preallocates 1GB on the hard drive efficiently, reserving space without writing zeros, faster than `dd`. Both create the file on the filesystem (e.g., `/swapfile` on `/dev/vda1`), not in RAM (unlike tmpfs).  
  **Why Others Wrong**:
- `dd` writes 1GB of zeros to the disk, consuming time and I/O unnecessarily.  
  **Key Concept**: `truncate` and `fallocate` create files quickly on disk; “lazily” refers to sparse allocation; swap files reside on the hard drive.  
  **Memory Aid**: “truncate or fallocate = Fast swap file on disk.”

## Question 95: Real-Time Kernel Messages

**Question**: A server’s network connection is flapping. The administrator wants to watch the kernel’s messages in real-time to see if any network-related errors are being logged. Which command would they use?  
**Options**:

- tail -f /var/log/syslog
- dmesg -w or journalctl -fk
- watch "ip addr show"
- netstat -i 1

**Correct Answer**: dmesg -w or journalctl -fk  
**Why Correct**: `dmesg -w` (wait) or `journalctl -fk` (follow kernel) displays kernel messages in real-time, including network errors.  
**Why Others Wrong**:

- `tail -f /var/log/syslog` shows system logs, not always kernel-specific.
- `watch "ip addr show"` monitors interfaces, not kernel logs.
- `netstat -i 1` is invalid; `netstat` doesn’t support real-time logging.  
  **Key Concept**: `dmesg -w` for kernel logs; `journalctl -fk` for systemd journals.  
  **Memory Aid**: “dmesg -w or journalctl -fk = Live kernel logs.”

## Question 96: Extending XFS Filesystem

**Question**: A logical volume `/dev/vg01/data` is formatted with the XFS filesystem. The administrator extends the LV by 20GB. Which command must be run to make the new space available to the filesystem?  
**Options**:

- resize2fs /dev/vg01/data
- xfs_growfs /mount/point/of/data
- xfs_repair /dev/vg01/data
- mount -o remount,resize /mount/point/of/data

**Correct Answer**: xfs_growfs /mount/point/of/data  
**Why Correct**: `xfs_growfs` extends an XFS filesystem to use new space on a grown logical volume, run on the mount point.  
**Why Others Wrong**:

- `resize2fs` is for ext filesystems, not XFS.
- `xfs_repair` fixes errors, not resizes.
- `mount -o remount,resize` is invalid; no such option.  
  **Key Concept**: `xfs_growfs` for XFS; `resize2fs` for ext.  
  **Memory Aid**: “xfs_growfs = Grow XFS on mount.”

## Question 97: Finding SSHD PID

**Question**: An administrator wants to find the process ID (PID) of the `sshd` service. Which command would quickly provide just the PID?  
**Options**:

- ps aux | grep sshd
- pidof sshd
- top -b -n 1 | grep sshd
- systemctl status sshd

**Correct Answer**: pidof sshd  
**Why Correct**: `pidof sshd` returns only the PID(s) of the `sshd` process, simple and direct.  
**Why Others Wrong**:

- `ps aux | grep sshd` shows detailed output, not just the PID.
- `top -b -n 1 | grep sshd` is verbose and includes other data.
- `systemctl status sshd` shows service status, not just the PID.  
  **Key Concept**: `pidof` for quick PID lookup; `pgrep` is similar.  
  **Memory Aid**: “pidof = PID only.”

## Question 98: Paravirtualized vs. Emulated Drivers

**Question**: When creating a virtual machine, the administrator is given a choice between paravirtualized (virtio) and emulated (e.g., e1000) drivers for network and disk devices. For best performance, which type should be chosen?  
**Options**:

- Emulated drivers, because they mimic real hardware perfectly.
- Paravirtualized (virtio) drivers, because they are optimized for virtualization.
- It does not matter, as the hypervisor handles performance.
- IDE drivers for disk and Realtek drivers for network.

**Correct Answer**: Paravirtualized (virtio) drivers, because they are optimized for virtualization.  
**Why Correct**: Virtio drivers are designed for virtualization, offering better performance by reducing emulation overhead.  
**Why Others Wrong**:

- Emulated drivers (e.g., e1000) mimic hardware, adding overhead.
- The hypervisor doesn’t fully handle performance; drivers matter.
- IDE/Realtek drivers are outdated or emulated, not optimal.  
  **Key Concept**: Virtio for performance; requires guest OS support.  
  **Memory Aid**: “virtio = Virtual optimized speed.”

## Question 99: Hiding Error Messages

**Question**: A command is expected to produce an error. The administrator wants to run the command but hide any error messages from the terminal. Which redirection should be used?  
**Options**:

- command > /dev/null
- command 2> /dev/null
- command | grep -v "error"
- command &> /dev/null

**Correct Answer**: command 2> /dev/null  
**Why Correct**: `2> /dev/null` redirects stderr (file descriptor 2) to `/dev/null`, hiding error messages while showing stdout.  
**Why Others Wrong**:

- `command > /dev/null` redirects stdout, not stderr.
- `command | grep -v "error"` filters output but may miss some errors.
- `command &> /dev/null` hides both stdout and stderr.  
  **Key Concept**: `2>` for stderr; `&>` for both stdout and stderr.  
  **Memory Aid**: “2> /dev/null = Silence errors only.”

## Question 100: Cloud Filesystem Resizing

**Question**: A Linux server is being provisioned in the cloud. The cloud provider’s documentation states that the server’s root filesystem can be resized. What underlying technology most likely enables this flexibility?  
**Options**:

- Standard MBR partitions
- A software RAID 0 array
- The use of Logical Volume Manager (LVM)
- A Btrfs filesystem with subvolumes

**Correct Answer**: The use of Logical Volume Manager (LVM)  
**Why Correct**: LVM allows dynamic resizing of logical volumes, enabling flexible filesystem expansion in cloud environments (e.g., with `lvextend` and `resize2fs` or `xfs_growfs`).  
**Why Others Wrong**:

- MBR partitions are fixed and not easily resized.
- RAID 0 stripes data but doesn’t support dynamic resizing.
- Btrfs supports resizing but is less common than LVM in cloud setups.  
  **Key Concept**: LVM for dynamic volume resizing; common in cloud.  
  **Memory Aid**: “LVM = Cloud’s flexible filesystem.”

## Retention Tips for Questions 81-100

- **Themes**: Filesystem management (`/var/log`, LVM, XFS), SSH troubleshooting, scripting (`find`, `xargs`), virtualization (RAW, virtio), RAID, process management (`renice`, `pidof`), kernel modules (`modprobe.d`), and cloud technologies (LVM).
- **Mnemonic for Key Tools**: “LVM resizes, virtio speeds, pidof finds, tar archives” (L-V-P-T).
- **Practice**: In a Linux VM (e.g., UTM Ubuntu):
  - Archive logs with `tar`, troubleshoot SSH with `ss -tlpn`, extend LVM with `pvcreate`/`vgextend`.
  - Create a swap file with `fallocate`, monitor kernel logs with `dmesg -w`, use `virtio` in KVM/UTM.
  - Test `find -mtime -1 | xargs` and `xfs_growfs` on an XFS volume.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “Fix full filesystem?” → “Archive logs.” “Hide errors?” → “2> /dev/null.” “RAW vs. QCOW2?” → “RAW for speed.”
- **Quiz Yourself**: How does `xargs -I{}` work? Where is `/etc/fstab` read? What does `-mtime -1` mean?
