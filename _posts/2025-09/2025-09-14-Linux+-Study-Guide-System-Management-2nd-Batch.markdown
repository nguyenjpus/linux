---
layout: post
title: "CompTIA Linux+ Study Guide: Comprehensive Review - System Management - 2nd Batch"
date: 2025-09-14
tags: [Linux+, modprobe]
---

This study guide organizes key concepts from CompTIA Linux+ practice questions 26–50, covering **Virtualization**, **Scripting and Automation**, **Networking**, **System Monitoring**, **Storage Management**, and **Security**. It serves as a centralized reference for retention and knowledge connection, with mnemonics, practical tasks, and common errors to reinforce learning. The guide addresses discrepancies in questions (e.g., Q36, Q42, Q44) to clarify exam nuances. Retention tips optimize learning through spaced repetition and scenario-based practice. No review quiz is included per user preference, focusing on practical application and connections across domains.

## Questions 26–50

### 26. Headless Debian 12 VM Setup

**Objective**: Configure a KVM virtual machine for headless operation.  
**Scenario**: Create a Debian 12 VM with 2 CPUs, 4 GiB RAM, 20 GiB disk, headless.  
**Correct Command**: `virt-install --name vm1 --vcpus 2 --memory 4096 --disk size=20 --os-variant debian12 --cdrom /isos/debian.iso --graphics none`  
**Key Points**:

- `virt-install`: Creates KVM VMs with `libvirt`.
- `--vcpus 2`, `--memory 4096`: Sets CPU and RAM.
- `--disk size=20`: Allocates 20 GiB qcow2 disk.
- `--os-variant debian12`: Optimizes for Debian 12.
- `--graphics none`: Enables headless mode (console access).  
  **Mnemonic**: `virt-install --graphics none` “installs a headless Debian villa.”  
  **Example Command**: As above.  
  **Practice**: Install `libvirt`, create a headless Debian VM, connect via `virsh console`.  
  **Common Errors**:
- Omitting `--graphics none`, causing VNC/SPICE to start.
- Incorrect `--os-variant` (e.g., `debian11`), leading to suboptimal VM settings.
- Missing `libvirt-bin` package, resulting in `virt-install` command not found.

### 27. Display Physical CPUs

**Objective**: Monitor physical CPU resources.  
**Scenario**: Display physical CPU cores on a system.  
**Correct Command**: `lscpu`  
**Key Points**:

- `lscpu`: Shows CPU details (cores, sockets, threads).
- Fields like “CPU(s)” and “Core(s) per socket” indicate physical CPUs.  
  **Mnemonic**: `lscpu` “lists system CPU cores.”  
  **Example Command**: `lscpu`  
  **Practice**: Run `lscpu`, note physical cores, compare with `htop`.  
  **Common Errors**:
- Using `cat /proc/cpuinfo`, which is less readable and harder to parse.
- Misinterpreting “CPU(s)” as physical cores (includes threads).
- Not installing `util-linux` package, causing `lscpu` to be unavailable.

### 28. Convert Raw Disk to qcow2

**Objective**: Convert disk images for virtualization.  
**Scenario**: Convert `rawdisk.img` to compressed qcow2 format as `disk.qcow2`.  
**Correct Command**: `qemu-img convert -O qcow2 -c rawdisk.img disk.qcow2`  
**Key Points**:

- `qemu-img convert`: Converts disk image formats.
- `-O qcow2`: Specifies qcow2 output format.
- `-c`: Enables compression for smaller file size.  
  **Mnemonic**: `qemu-img convert -c` “compresses raw to qcow2.”  
  **Example Command**: As above.  
  **Practice**: Create a raw disk image, convert to qcow2, verify with `qemu-img info disk.qcow2`.  
  **Common Errors**:
- Omitting `-c`, resulting in an uncompressed, larger qcow2 file.
- Using `-f raw` instead of `-O qcow2`, causing incorrect output format.
- Specifying non-existent `rawdisk.img`, leading to “No such file” error.

### 29. Manage KVM VM Snapshot Space Consumption

**Objective**: Monitor and manage VM snapshot storage usage.  
**Scenario**: Merge an active snapshot (or snapshot chain) for disk vda of VM vm1 into the base image (--active) and switch to the base image (--pivot), reducing disk usage.  
**Correct Command**: `virsh blockcommit vm1 vda --active --pivot`  
**Key Points**:

- `virsh blockcommit`: Merges a snapshot (or chain) into the base disk image for a running VM.
- `--active`: Commits the active snapshot layer.
- `--pivot`: Switches the VM to the base image after merging.
- `vda`: Refers to the VM’s disk (e.g., `/var/lib/libvirt/images/vm1.qcow2`).  
  **Mnemonic**: `virsh blockcommit` “commits and pivots snapshots back to base.”  
  **Example Command**: `virsh blockcommit vm1 vda --active --pivot`  
  **Practice**: Create a VM (vm1) with a qcow2 disk, take a snapshot (`virsh snapshot-create-as vm1 snap1`), run blockcommit, verify with `virsh snapshot-list vm1` and `qemu-img info`.  
  **Common Errors**:
- Omitting `--pivot`, leaving the VM using the snapshot layer.
- Running `blockcommit` on a stopped VM, causing “VM not running” error.
- Incorrect disk identifier (e.g., `vdb` instead of `vda`), leading to failure.

### 30. Parse /etc/passwd with awk

**Objective**: Extract data with `awk`.  
**Scenario**: Print usernames from `/etc/passwd` using `awk`.  
**Correct Command**: `awk -F: '{print $1}' /etc/passwd`  
**Key Points**:

- `-F:`: Sets colon as delimiter.
- `print $1`: Outputs first field (username).  
  **Mnemonic**: `awk -F:` “fishes usernames from passwd.”  
  **Example Command**: As above.  
  **Practice**: Run command, pipe to `wc -l` to count users.  
  **Common Errors**:
- Using wrong delimiter (e.g., `-F,`), outputting incorrect fields.
- Omitting single quotes around `{print $1}`, causing syntax errors.
- Misspelling `/etc/passwd` as `/etc/password`, resulting in “No such file” error.

### 31. Create Shell Alias

**Objective**: Simplify commands with aliases.  
**Scenario**: Create `cls` alias to clear screen and print “Ready”.  
**Correct Command**: `alias cls='clear && echo Ready'`  
**Key Points**:

- Single quotes preserve literal commands.
- `&&` ensures `echo` runs if `clear` succeeds.  
  **Mnemonic**: `cls` “clears and signals ready.”  
  **Example Command**: As above.  
  **Practice**: Add alias to `~/.bashrc`, test with `cls`.  
  **Common Errors**:
- Using double quotes, causing variable expansion issues.
- Forgetting to add to `~/.bashrc`, making alias non-persistent.
- Missing `&&`, causing `echo Ready` to run independently.

### 32. Script Error Handling

**Objective**: Control script execution on errors.  
**Scenario**: Exit script on non-zero return, continue non-critical loop commands.  
**Correct Command**: `set -e`  
**Key Points**:

- `set -e`: Exits on any non-zero return.
- `|| true` allows loop commands to continue.  
  **Mnemonic**: `set -e` “stops script errors, loops live on.”  
  **Example Command**: `echo 'set -e; for i in 1 2; do false || true; done' > test.sh; bash test.sh`  
  **Practice**: Write a script with `set -e`, test loop continuation.  
  **Common Errors**:
- Omitting `|| true` in loops, causing premature script exit.
- Placing `set -e` inside a function, limiting its scope.
- Using `set -x` instead, enabling debugging instead of error handling.

### 33. Modify PATH in zsh

**Objective**: Customize shell environment.  
**Scenario**: Append `/usr/local/bin` to PATH in zsh if not present.  
**Correct Command**: `[[ ":$PATH:" != *":/usr/local/bin:"* ]] && PATH+=:/usr/local/bin`  
**Key Points**:

- `[[ ... ]]`: zsh test for PATH check.
- `+=`: Appends to PATH safely.  
  **Mnemonic**: `PATH+=` “pluses /usr/local/bin to zsh path.”  
  **Example Command**: As above.  
  **Practice**: Add to `~/.zshrc`, verify with `echo $PATH`.  
  **Common Errors**:
- Using `PATH=$PATH:/usr/local/bin`, risking duplicates.
- Adding to `~/.bashrc` instead of `~/.zshrc`, causing zsh to ignore it.
- Omitting `[[ ... ]]`, appending even if path exists.

### 34. Backup Directory with Bash Function

**Objective**: Create reusable backup scripts.  
**Scenario**: Create a function to back up a directory to a date-stamped `tar.xz` file.  
**Correct Command**: `bk() { tar -cJf /backup/"$1"_$(date +%F).tar.xz "$1"; }`  
**Key Points**:

- `-cJf`: Creates `tar.xz` archive.
- `"$1"`: Handles directory names with spaces.  
  **Mnemonic**: `bk` “backs up with tar.xz and date.”  
  **Example Command**: `bk mydir`  
  **Practice**: Define function, archive a directory, verify with `tar -tf`.  
  **Common Errors**:
- Omitting quotes around `"$1"`, failing with spaces in directory names.
- Using `-z` instead of `-J`, creating `tar.gz` instead of `tar.xz`.
- Incorrect path `/backup`, causing “Permission denied” if not writable.

### 35. Create Swap File

**Objective**: Manage swap space.  
**Scenario**: Create a 512 MiB swap file with correct permissions.  
**Correct Command**: `dd if=/dev/zero of=/swapfile bs=1M count=512; chmod 600 /swapfile; mkswap /swapfile; swapon /swapfile`  
**Key Points**:

- `dd`: Creates file with zeros.
- `chmod 600`: Secures permissions.
- `mkswap`, `swapon`: Initialize and enable swap.  
  **Mnemonic**: `dd` “drops zeros for swap space.”  
  **Example Command**: As above.  
  **Practice**: Create swap file, enable, verify with `swapon --show`.  
  **Common Errors**:
- Skipping `chmod 600`, risking security vulnerabilities.
- Using incorrect `bs` or `count`, creating wrong swap size.
- Forgetting to add `/swapfile` to `/etc/fstab`, making swap non-persistent.

### 36. Configure tmpfs Filesystem

**Objective**: Set up temporary RAM-backed storage.  
**Scenario**: Create a 64 MiB tmpfs at `/run/temp` with `noexec`.  
**Correct Entry**: `d /run/temp 0755 root root - -`  
**Key Points**:

- `d`: Creates tmpfs in `systemd-tmpfiles`.
- `noexec`, `size=64M`: Set via separate mount command.  
  **Mnemonic**: `tmpfiles` “drops tmpfs directories.”  
  **Example Command**: `echo 'd /run/temp 0755 root root - -' | sudo tee /etc/tmpfiles.d/temp.conf; sudo mount -o remount,size=64M,noexec /run/temp`  
  **Practice**: Create tmpfs, verify with `df -h /run/temp`.  
  **Common Errors**:
- Omitting `mount` command, leaving tmpfs without `noexec` or size limits.
- Using `/tmp` instead of `/run/temp`, risking conflicts with system tmpfs.
- Incorrect permissions (e.g., `0777`), allowing unauthorized access.

### 37. Detect Hardware Sensors

**Objective**: Monitor CPU temperatures with `lm_sensors`.  
**Scenario**: Identify hardware sensors for monitoring.  
**Correct Command**: `sensors-detect`  
**Key Points**:

- Probes hardware, loads kernel modules (e.g., `coretemp`).
- Run `sensors` afterward to view temperatures.  
  **Mnemonic**: `sensors-detect` “detects temperature sensors.”  
  **Example Command**: `sudo sensors-detect`  
  **Practice**: Install `lm_sensors`, run `sensors-detect`, verify with `sensors`.  
  **Common Errors**:
- Running without `sudo`, causing insufficient permissions to probe hardware.
- Not installing `lm-sensors` package, resulting in command not found.
- Ignoring `sensors-detect` prompts, missing critical kernel modules.

### 38. Load Kernel Module at Boot

**Objective**: Persist kernel module loading.  
**Scenario**: Load `ipmi_si` module at boot.  
**Correct File**: `/etc/modules-load.d/ipmi.conf`  
**Key Points**:

- Lists modules for `systemd-modules-load.service`.
- Add `ipmi_si` to file.  
  **Mnemonic**: `modules-load.d` “loads modules like ipmi_si.”  
  **Example Command**: `echo "ipmi_si" | sudo tee /etc/modules-load.d/ipmi.conf`  
  **Practice**: Add module, verify with `lsmod | grep ipmi_si` after reboot.  
  **Common Errors**:
- Using `/etc/modules` (older systems), not compatible with systemd.
- Misspelling `ipmi_si`, causing module load failure.
- Forgetting `sudo`, resulting in “Permission denied” when writing to file.

### 39. Blacklist Kernel Module

**Objective**: Prevent unwanted module loading.  
**Scenario**: Blacklist `floppy` module permanently.  
**Correct Entry**: `blacklist floppy` in `/etc/modprobe.d/blacklist.conf`  
**Key Points**:

- `blacklist`: Prevents module loading by `modprobe`.
- Use `/etc/modprobe.d/` for persistence.  
  **Mnemonic**: `blacklist` “blocks floppy forever.”  
  **Example Command**: `echo "blacklist floppy" | sudo tee /etc/modprobe.d/blacklist.conf`  
  **Practice**: Blacklist `floppy`, test with `modprobe floppy`.  
  **Common Errors**:
- Placing `blacklist` in `/etc/modprobe.conf`, which is deprecated.
- Not using `sudo`, causing file write failure.
- Blacklisting a module still loaded, requiring `modprobe -r` first.

### 40. Monitor Network Bandwidth

**Objective**: Display real-time network usage.  
**Scenario**: Show bandwidth per interface in terminal.  
**Correct Command**: `iftop`  
**Key Points**:

- Displays live bandwidth for connections/interfaces.
- Requires root (`sudo iftop`).  
  **Mnemonic**: `iftop` “tops interface bandwidth.”  
  **Example Command**: `sudo iftop`  
  **Practice**: Install `iftop`, monitor traffic, generate load with `wget`.  
  **Common Errors**:
- Running without `sudo`, resulting in “Permission denied” for interface access.
- Not installing `iftop`, causing command not found.
- Specifying wrong interface (e.g., `-i eth0` when using `enp0s3`), showing no data.

### 41. Capture DNS Packets

**Objective**: Perform network packet analysis.  
**Scenario**: Capture DNS packets on `enp0s3` to `dns.pcap`.  
**Correct Command**: `tcpdump -i enp0s3 port 53 -w dns.pcap`  
**Key Points**:

- `port 53`: Filters DNS (UDP/TCP).
- `-w`: Saves to PCAP format.  
  **Mnemonic**: `tcpdump port 53` “dumps DNS packets.”  
  **Example Command**: `sudo tcpdump -i enp0s3 port 53 -w dns.pcap`  
  **Practice**: Run `tcpdump`, generate DNS traffic (`dig`), verify with Wireshark.  
  **Common Errors**:
- Omitting `sudo`, causing “Permission denied” for packet capture.
- Using wrong interface (e.g., `eth0` instead of `enp0s3`), capturing no packets.
- Forgetting `-w`, outputting to terminal instead of file.

### 42. Test HTTPS Endpoint

**Objective**: Check HTTP server status.  
**Scenario**: Return only HTTP status code for HTTPS endpoint.  
**Correct Command**: `curl -sI -o /dev/null -w "%{http_code}\n" https://example.com`  
**Key Points**:

- `-s`: Silent mode.
- `-I`: Fetch headers only
- `-o /dev/null`: Discards body.
- `-w "%{http_code}\n"`: Outputs status (e.g., `200`).  
  **Mnemonic**: `curl -w` “writes HTTP code quietly.”  
  **Example Command**: As above.  
  **Practice**: Test `https://example.com`, verify status code (e.g., `200`).  
  **Common Errors**:
- Using `-I` instead of `-w`, fetching headers instead of just status code.
- Omitting `-o /dev/null`, printing body to terminal.
- Incorrect URL (e.g., `http://` instead of `https://`), causing connection failure.

### 43. Redirect Script Output

**Objective**: Manage script output streams.  
**Scenario**: Redirect stdout/stderr to separate logs, show stdout on console.  
**Correct Command**: `myscript.sh > >(tee out.log) 2>err.log`  
**Key Points**:

- `> >(tee out.log)`: Sends stdout to console and `out.log`.
- `2>err.log`: Redirects stderr to `err.log`.  
  **Mnemonic**: `tee` “tees stdout to console and log.”  
  **Example Command**: `echo 'echo "Hello"; echo "Error" >&2' > myscript.sh; bash myscript.sh > >(tee out.log) 2>err.log`  
  **Practice**: Create script, test redirection, verify logs.  
  **Common Errors**:
- Using `>` instead of `> >(tee)`, losing console output.
- Mixing `2>` and `>`, redirecting both streams to same file.
- Forgetting to make `myscript.sh` executable (`chmod +x`), causing “Permission denied”.

### 44. Schedule Monthly Backup

**Objective**: Automate tasks with cron.  
**Scenario**: Run `/usr/local/bin/backup.sh` at 23:30 on the last day of the month.  
**Correct Command**: `30 23 28-31 * * ["$(date +%m -d tomorrow)" != "$(date +%m)" ] && /usr/local/bin/backup.sh`  
**Key Points**:

- `28-31`: Targets possible last days.
- Date logic checks for last day.  
  **Mnemonic**: Cron “checks tomorrow for last-day backup.”  
  **Example Command**: `echo '30 23 28-31 * * ["$(date +%m -d tomorrow)" != "$(date +%m)" ] && /usr/local/bin/backup.sh' | crontab -`  
  **Practice**: Add to crontab, test logic on last day.  
  **Common Errors**:
- Using `L` instead of date logic, which is non-standard in Linux cron.
- Incorrect syntax in `[ ... ]`, causing cron job failure.
- Missing executable permissions on `backup.sh`, preventing execution.

### 45. Ensure Single Script Instance

**Objective**: Prevent concurrent script execution.  
**Scenario**: Use `flock` for single instance of `cleanup.sh` in systemd timer.  
**Correct Command**: `ExecStart=/usr/bin/flock -n /tmp/cleanup.lock /usr/local/bin/cleanup.sh`  
**Key Points**:

- `flock -n`: Non-blocking lock, prevents overlaps.
- `ExecStart`: Runs in systemd service.  
  **Mnemonic**: `flock -n` “locks cleanup to one run.”  
  **Example Command**: `echo '[Service]\nExecStart=/usr/bin/flock -n /tmp/cleanup.lock /usr/local/bin/cleanup.sh' | sudo tee /etc/systemd/system/cleanup.service`  
  **Practice**: Create service/timer, verify single instance with `ps aux`.  
  **Common Errors**:
- Omitting `-n`, causing script to wait indefinitely for lock.
- Incorrect lock file path, leading to multiple instances running.
- Forgetting `sudo` when creating systemd service, causing file write failure.

### 46. Recover LVM Logical Volume

**Objective**: Restore deleted LVM volumes.  
**Scenario**: Activate partial VGs for recovery.  
**Correct Command**: `vgchange -ay --partial`  
**Key Points**:

- `--partial`: Activates incomplete VGs.
- Use after `vgscan` for recovery.  
  **Mnemonic**: `vgchange --partial` “partially revives VGs.”  
  **Example Command**: `sudo vgchange -ay --partial`  
  **Practice**: Simulate VG failure, activate, verify with `lvs`.  
  **Common Errors**:
- Skipping `vgscan`, causing missing VG detection.
- Running without `sudo`, resulting in “Permission denied”.
- Omitting `--partial`, failing to activate incomplete VGs.

### 47. Analyze Boot Performance

**Objective**: Identify slow systemd services.  
**Scenario**: View historical boot times and slowest services.  
**Correct Command**: `systemd-analyze blame`  
**Key Points**:

- Lists service startup times, sorted by duration.
- Historical data from journal.  
  **Mnemonic**: `systemd-analyze blame` “blames slow booters.”  
  **Example Command**: `systemd-analyze blame`  
  **Practice**: Run command, identify top three slowest services.  
  **Common Errors**:
- Using `systemd-analyze` without `blame`, showing only total boot time.
- Running on a system with no journal, causing missing data.
- Misinterpreting service times as total boot time.

### 48. Mount ISO File

**Objective**: Access ISO contents.  
**Scenario**: Mount `/isos/alma.iso` at `/mnt/iso` read-only with loop device.  
**Correct Command**: `mount -o loop,ro /isos/alma.iso /mnt/iso`  
**Key Points**:

- `-o loop,ro`: Uses loop device, read-only.
- Implies ISO9660 filesystem.  
  **Mnemonic**: `mount -o loop,ro` “loops ISO read-only.”  
  **Example Command**: `sudo mount -o loop,ro /isos/alma.iso /mnt/iso`  
  **Practice**: Mount ISO, verify with `ls /mnt/iso`, test read-only.  
  **Common Errors**:
- Omitting `sudo`, causing “Permission denied” for mount.
- Forgetting `-o ro`, risking accidental writes to ISO.
- Incorrect mount point (e.g., non-existent `/mnt/iso`), causing failure.

### 49. Improve VM Disk Throughput

**Objective**: Optimize QEMU/KVM guest performance.  
**Scenario**: Enable virtio driver for better disk I/O.  
**Correct Driver**: `virtio_blk`  
**Key Points**:

- Paravirtualized block driver for disk performance.
- Enabled in guest kernel and VM config.  
  **Mnemonic**: `virtio_blk` “boosts VM disk blocks.”  
  **Example Command**: `lsmod | grep virtio_blk`  
  **Practice**: Verify `virtio_blk` in guest, test with `dd`.  
  **Common Errors**:
- Not enabling virtio in VM config, falling back to slower IDE driver.
- Missing guest kernel support for `virtio_blk`, causing disk failure.
- Confusing `virtio_blk` with `virtio_scsi`, impacting performance.

### 50. Attach LUKS Device in initramfs

**Objective**: Recover encrypted root in initramfs.  
**Scenario**: Create `/dev/mapper/cryptroot` to resume boot.  
**Correct Command**: `cryptsetup open /dev/sda3 cryptroot; exit`  
**Key Points**:

- `cryptsetup open`: Decrypts LUKS partition.
- `exit`: Resumes boot process.  
  **Mnemonic**: `cryptsetup open` “unlocks cryptroot, exits to boot.”  
  **Example Command**: As above.  
  **Practice**: In a LUKS VM, enter initramfs, run command, verify boot.  
  **Common Errors**:
- Incorrect partition (e.g., `sda2` instead of `sda3`), failing to unlock.
- Omitting `exit`, stalling in initramfs.
- Missing `cryptsetup` in initramfs, causing command not found.

### Plan for Future Questions

- **Batch Processing**: This guide covers questions 26–50. Future batches (e.g., 51–75) will create new artifacts or update this one with a new version ID, maintaining continuity.
- **Connections**:
  - **Virtualization**: Q26, Q29, Q49 (VM setup, snapshot space, virtio) connect to Q28, Q35, Q46, Q48, Q50 (storage tasks).
  - **Scripting/Automation**: Q30–34, Q43–45 cover Bash, cron, and systemd for automation.
  - **Networking**: Q40–42 (bandwidth, packet capture, HTTP) link to Q15 (DHCP) from batch 1–25.
  - **System Monitoring**: Q27, Q37, Q38, Q47 (CPUs, sensors, modules, boot) support VM optimization.
  - **Storage/Security**: Q28, Q35, Q46, Q48, Q50 (raw disk conversion, swap, LVM, ISO, LUKS) tie to Q24, Q20 (backup) from batch 1–25.
- **Retention Strategy**: Use mnemonics daily, practice labs biweekly, and visualize commands (e.g., flowcharts for `tcpdump`, `cryptsetup`).

### Retention Tips

- **Spaced Repetition**: Review questions 26–50 in three sessions (Day 1: Q26–35, Day 3: Q36–45, Day 5: Q46–50). Use flashcards for mnemonics (e.g., “`virsh blockcommit` commits and pivots snapshots back to base”).
- **Scenario-Based Practice**: Set up a VM lab (e.g., KVM on Ubuntu) to test Q26, Q29, Q49 (VM tasks), Q28, Q46, Q50 (raw disk, LVM, LUKS), and Q40–42 (networking). Use a test VM to avoid data loss.
- **Visual Aids**: Create diagrams for:
  - LVM structure (Q46, Q50: PVs → VGs → LVs → LUKS).
  - Network flow (Q40–42: `iftop` → `tcpdump` → `curl`).
  - Systemd workflow (Q45, Q47: timers → services → boot analysis).
  - Virtualization (Q26, Q29, Q49: VM → snapshot space → virtio).
- **Group Study**: Discuss Q30–34, Q43–45 (scripting) with peers to reinforce Bash logic.
- **Command Journal**: Log commands (e.g., `virt-install`, `qemu-img convert`, `virsh blockcommit`) in a notebook, noting outputs for retention.

### Discrepancy Notes

- **Q36 (tmpfs)**: `noexec` and `size=64M` require a separate `mount` command, not specified in the `systemd-tmpfiles` entry. Exam may assume defaults.
- **Q44 (cron)**: `L` is non-standard in Linux cron; date logic is required for last-day scheduling.
