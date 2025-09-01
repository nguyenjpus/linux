# Linux Boot Troubleshooting: GRUB and Systemd Simulations

This guide walks through simulating and resolving common Linux boot issues using GRUB (bootloader) and Systemd (service manager) in a virtual machine (e.g., UTM with Ubuntu 24.04). The goal is to practice creating, diagnosing, and fixing boot problems like misconfigured kernel parameters and masked services. **Use a VM snapshot to revert changes safely**.

## Prerequisites
- Ubuntu VM (e.g., in UTM).
- Root access (`sudo`).
- Text editor (`nano` or `vim`).

## Baseline Measurement: Normal Boot Metrics
Before any experiments, measure the system's normal boot performance for comparison.

1. **Run Boot Analysis**  
   ```bash
   systemd-analyze
   systemd-analyze blame
   ```
   - **Why**: `systemd-analyze` shows total boot time (e.g., firmware + loader + kernel + userspace). `systemd-analyze blame` lists services by time taken, providing a baseline for detecting delays later.

2. **Reboot and Re-Measure (Optional)**  
   ```bash
   sudo reboot
   ```
   - After reboot, re-run the commands above to average multiple boots.  
   - **Why**: Boot times can vary; a baseline helps spot differences from simulations.

## 1. Simulating a GRUB Boot Issue (Invalid Kernel Parameter)
GRUB loads the Linux kernel and passes parameters to it. Adding an invalid parameter mimics real-world issues (e.g., from bad updates) that cause boot delays or errors. The correct GRUB settings (`GRUB_TIMEOUT_STYLE=menu`, `GRUB_TIMEOUT=5`, `#GRUB_HIDDEN_TIMEOUT=0`, `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash console=tty0"`) should display the GRUB menu for 5 seconds, allowing selection of "Advanced options for Ubuntu" and recovery mode, but UTM’s UEFI/QEMU may bypass this, requiring alternative troubleshooting.

### Step-by-Step
1. **Backup GRUB Configuration**  
   Always back up to restore if needed.  
   ```bash
   sudo cp /etc/default/grub /etc/default/grub.bak
   ```

2. **Edit GRUB to Add Invalid Parameter**  
   ```bash
   sudo nano /etc/default/grub
   ```
   - Find `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash console=tty0"`.  
   - Change to: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash console=tty0 invalid=param"`.  
   - Save (Ctrl+O, Enter, Ctrl+X in nano).  
   - **Why**: `quiet` reduces boot messages, `splash` enables a loading screen, `console=tty0` sets the console. `invalid=param` is a fake parameter the kernel won’t recognize, causing a warning or delay.

3. **Update GRUB and Reboot**  
   ```bash
   sudo update-grub
   sudo reboot
   ```
   - **Why**: `update-grub` applies changes to `/boot/grub/grub.cfg`. Reboot tests the effect.  
   - **Expect**: Possible boot delay or kernel error (e.g., "unknown parameter 'invalid'").

4. **Diagnose the Issue**  
   After reboot, check kernel logs and compare to baseline:  
   ```bash
   dmesg | grep invalid
   journalctl -b -1 | grep invalid
   systemd-analyze
   systemd-analyze blame
   ```
   - **Why**: `dmesg` shows kernel messages; `journalctl -b -1` shows previous boot logs. Look for errors like "Unknown kernel command line parameters 'invalid=param'". Compare `systemd-analyze` outputs to baseline for delays.

5. **Fix the Issue**  
   ```bash
   sudo nano /etc/default/grub
   ```
   - Remove `invalid=param` (restore to `quiet splash console=tty0`).  
   - Save and run:
     ```bash
     sudo update-grub
     sudo reboot
     ```
   - **Why**: Reverts the bad parameter, regenerates GRUB config, and restores normal boot.  
   - Verify fix (compare to baseline):
     ```bash
     dmesg | grep invalid
     systemd-analyze
     systemd-analyze blame
     ```
     - Expect no `invalid` errors and boot times similar to baseline.

6. **Recover to Pre-Experiment State**  
   To ensure the system is fully restored:  
   ```bash
   sudo cp /etc/default/grub.bak /etc/default/grub
   sudo update-grub
   sudo reboot
   ```
   - **Why**: Restores the original GRUB configuration from the backup, undoing all changes (e.g., `invalid=param`). Verifies the system boots normally with original settings.

## 2. Simulating a Systemd Service Issue (Masking Services)
Systemd manages services. Masking non-critical services like CUPS (printing) and Bluetooth simulates a boot issue if dependent services wait for them, causing delays. Masking multiple services amplifies the effect for clearer comparison.

### Step-by-Step
1. **Check Service Status**  
   ```bash
   systemctl status cups.service
   systemctl status bluetooth.service
   ```
   - **Why**: Confirms services are running (`active`) before masking.

2. **Mask Services**  
   Mask one or both for a stronger simulation:  
   ```bash
   sudo systemctl mask cups.service
   sudo systemctl mask bluetooth.service
   systemctl status cups.service
   systemctl status bluetooth.service
   ```
   - **Why**: Masking links services to `/dev/null`, preventing startup. Masking multiple (non-essential) services creates a more noticeable boot impact.  
   - Expect: `Loaded: masked`, `Active: inactive` for each.

3. **Reboot and Observe**  
   ```bash
   sudo reboot
   ```
   - **Why**: Tests if masking causes delays or errors (more pronounced with multiple masks).  
   - **Expect**: Possible longer boot time; check logs for issues.

4. **Diagnose the Issue**  
   ```bash
   journalctl -u cups.service
   journalctl -u bluetooth.service
   systemd-analyze
   systemd-analyze blame
   ```
   - **Why**: `journalctl -u <service>` shows logs (expect no "Starting" lines). `systemd-analyze blame` lists delays—compare to baseline for differences (e.g., total time increase).

5. **Fix the Issue**  
   ```bash
   sudo systemctl unmask cups.service
   sudo systemctl unmask bluetooth.service
   sudo systemctl start cups.service
   sudo systemctl start bluetooth.service
   systemctl status cups.service
   systemctl status bluetooth.service
   sudo reboot
   ```
   - **Why**: Unmasks and starts services, restoring normal behavior.  
   - Verify (compare to baseline):
     ```bash
     systemd-analyze
     systemd-analyze blame
     ```
     - Expect boot times closer to baseline.

6. **Recover to Pre-Experiment State**  
   To ensure the system is fully restored:  
   ```bash
   sudo systemctl unmask cups.service
   sudo systemctl unmask bluetooth.service
   sudo systemctl enable cups.service
   sudo systemctl enable bluetooth.service
   sudo systemctl start cups.service
   sudo systemctl start bluetooth.service
   systemctl status cups.service
   systemctl status bluetooth.service
   ```
   - **Why**: Ensures services are unmasked, enabled (start on boot), and running, restoring the original state before masking.

## 3. Alternative Systemd Simulation
For even more impact, mask additional non-critical services (e.g., `avahi-daemon.service` for network discovery). Diagnose/fix/recover as above.

## Key Learnings
- **GRUB**: Bootloader parameters control kernel behavior. Invalid parameters (e.g., `invalid=param`) cause warnings or delays, logged in `dmesg` or `journalctl`. Correct settings (`GRUB_TIMEOUT_STYLE=menu`, `GRUB_TIMEOUT=5`, `#GRUB_HIDDEN_TIMEOUT=0`) should show the GRUB menu for 5 seconds to access "Advanced options for Ubuntu" and recovery mode, but UTM’s UEFI may bypass this.
- **Systemd**: Masking services prevents startup, simulating dependency-related boot delays (amplified by multiple masks). Use `systemd-analyze` and `journalctl` for diagnostics, comparing to baseline metrics.
- **Practice**: Simulating issues in a VM builds skills for real-world scenarios (e.g., bad updates).

## Notes
- **Safety**: Always snapshot your VM before changes. Restore GRUB with `sudo cp /etc/default/grub.bak /etc/default/grub` if needed.
- **UTM**: If GRUB menu doesn’t show (despite correct settings), spam Shift/Esc on boot or use a live ISO to edit `/etc/default/grub`.