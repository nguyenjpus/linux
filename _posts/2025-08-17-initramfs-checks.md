 ---
 layout: post
 title: Understanding Initramfs Checks for Linux+ Prep
 date: 2025-08-17
 ---
 # Understanding Initramfs Checks for Linux+ Prep

 As part of my **CompTIA Linux+** studies, I'm diving into the Linux boot process, focusing on the initramfs stage. Below is a summary of key checks to perform before and after initramfs to troubleshoot boot issues, based on my recent learning.

 ## Before Initramfs (Pre-Boot Preparation)
 These checks ensure the system is ready to load the initramfs:
 - **Kernel Image and Modules**: Verify the kernel file (`/boot/vmlinuz`) exists and isn't corrupted. Check `/lib/modules/$(uname -r)/` for required modules using `lsmod`.
 - **Boot Loader Configuration**: Inspect GRUB config (`/boot/grub/grub.cfg` or `/etc/default/grub`) for correct parameters like `root=UUID=...` or `initrd=/initramfs.img`.
 - **Hardware Detection**: Confirm BIOS/UEFI settings allow booting from the correct device. Update firmware if hardware isn't recognized.
 - **File System Integrity**: Run `fsck` on the root partition to fix errors before booting.
 - **Dependencies**: Ensure drivers for storage controllers are included in the kernel or initramfs.

 ## After Initramfs (Post-Handover to Root FS)
 These checks verify the system transitions correctly to the root filesystem:
 - **Root File System Mount**: Check if the root FS is mounted (`mount | grep root`). If not, drop to a shell and mount manually.
 - **Logs Review**: Use `dmesg | grep error` or check `/var/log/boot.log` for initramfs errors (e.g., missing modules).
 - **Process Handover**: Confirm `systemd` (or init) started with `ps aux | grep init` or `journalctl`.
 - **Module Loading**: Verify modules loaded with `lsmod`; reload with `modprobe` if needed.
 - **Emergency Mode**: If in an emergency shell, check `/proc/mounts` and fix issues like incorrect UUID in `/etc/fstab`.

 ## Reflections
 These checks are critical for debugging boot failures, a key skill for Linux+. Practicing commands like `fsck` and `dmesg` on my Ubuntu VM has been super helpful. Next, I’ll explore RAID setups and simulate boot failures for deeper understanding.

 Stay tuned for more Linux+ notes!
