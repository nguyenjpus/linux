---
layout: post
title: "Q36, Q43, Q45: Temporary Filesystems, Redirection, and Process Locking"
date: 2025-09-23
tags: [Linux+, tmpf, Redirection, flock]
---

This guide summarizes a hands-on practice session for CompTIA Linux+ questions Q36, Q43, and Q45, performed on a UTM Ubuntu system (`uname -r: 6.8.0-79-generic`, LVM on `/dev/vda3`). It covers configuring a temporary RAM-backed filesystem (tmpfs), redirecting script output, and automating tasks with `flock` to prevent overlaps. Errors encountered (permissions, history expansion, and `flock` issues) were resolved, building skills for data center automation and troubleshooting.

## Scenario

As a data center technician, I set up a monitoring system:

1. **Q36**: Create a 64 MiB tmpfs at `/run/temp` with `noexec` to store logs securely.
2. **Q43**: Write a script that logs stdout to console and file, stderr to a separate file.
3. **Q45**: Automate the script with a systemd timer, using `flock` to ensure single instances.

## Q36: Temporary RAM-backed Filesystem

**Question**: Configure a tmpfs at `/run/temp` (64 MiB, `noexec`). Which `systemd-tmpfiles` entry is correct?

- Options: `z /run/temp 0755 root root 64M noexec`, `z /run/temp 0755 root root 0 0`, `d /run/temp 0755 root root - -`, `d /run/temp 0755 root root 64M`
- **Answer**: None are fully correct; `d /run/temp 0755 root root 64M` is closest but flawed, as `systemd-tmpfiles` `d` doesn’t support size/`noexec`.

**What We Did**:

- Tried `/etc/tmpfiles.d/temp.conf` with `d /run/temp 0755 root root 64M noexec`, but got error: `/etc/tmpfiles.d/temp.conf:1: d lines don't take argument fields, ignoring`.
- **Fix**: Used a systemd mount unit (`/etc/systemd/system/run-temp.mount`):
  ```bash
  sudo nano /etc/systemd/system/run-temp.mount
  # Content:
  [Unit]
  Description=Temporary RAM-backed Filesystem at /run/temp
  [Mount]
  What=tmpfs
  Where=/run/temp
  Type=tmpfs
  Options=defaults,noexec,nosuid,size=64M,mode=0755,uid=0,gid=0
  [Install]
  WantedBy=multi-user.target
  ```
- Enabled/started: `sudo systemctl daemon-reload; sudo systemctl enable run-temp.mount; sudo systemctl start run-temp.mount`.
- Verified: `mount | grep /run/temp` showed `tmpfs on /run/temp ... noexec ... size=65536k`.
- Tested `noexec`:
  - Failed to create `test.sh` due to permissions (`/run/temp` root-owned). Fixed with:
    ```bash
    sudo bash -c 'echo "#!/bin/bash" > /run/temp/test.sh'
    sudo chmod +x /run/temp/test.sh
    /run/temp/test.sh  # Failed: "Permission denied" (noexec worked)
    ```

**Why It Matters**: In data centers, tmpfs stores volatile data (e.g., server metrics) to reduce disk I/O. `noexec` prevents malicious code execution. **Analogy**: tmpfs is a hotel safe (temporary, erased on reboot); `noexec` locks out gadgets.

**Connection Clarified**:

- `test.sh`: Tests `noexec` by failing to run in `/run/temp`.
- `tempfiles.d/temp.conf`: Attempted tmpfs setup but limited; doesn’t support `noexec` or size.
- `run-temp.mount`: Correct way to configure tmpfs with custom options, used in practice.

## Q43: Redirecting stdout and stderr

**Question**: Redirect a script’s stdout to console and `out.log`, stderr to `err.log`.

- **Answer**: `myscript.sh > >(tee out.log) 2>err.log`

**What We Did**:

- Created `~/monitor.sh`:
  ```bash
  #!/bin/bash
  echo "Monitoring system at $(date)"
  echo "Error: Disk check failed" >&2
  chmod +x ~/monitor.sh
  ```
- Ran with redirection:
  - First attempt: `./monitor.sh > >(tee /run/temp/out.log) 2> >(sudo tee /run/temp/err.log > /dev/null)` failed ("tee: /run/temp/out.log: Permission denied") because `tee` (for stdout) ran as non-root.
  - Fixed: `./monitor.sh > >(sudo tee /run/temp/out.log) 2> >(sudo tee /run/temp/err.log > /dev/null)`
    - Console showed: `Monitoring system at ...`
    - Logs: `sudo cat /run/temp/out.log` (stdout), `sudo cat /run/temp/err.log` (stderr).
  - Tested variation: `./monitor.sh > >(sudo tee /run/temp/out2.log) 2> err.log` created `err.log` in home dir (non-root).

**Why It Matters**: In data centers, scripts monitor servers (e.g., disk usage). Stdout to console/logs tracks progress; stderr logs isolate issues. **Analogy**: Stdout is a chef’s progress report; stderr is a separate error log for fixes.

## Q45: Using flock with Systemd Timer

**Question**: Ensure one instance of `cleanup.sh` runs via systemd timer with `flock`.

- **Answer**: `ExecStart=/usr/bin/flock -n /tmp/cleanup.lock /usr/local/bin/cleanup.sh`

**What We Did**:

- Moved script: `sudo mv ~/monitor.sh /usr/local/bin/monitor.sh`.
- Created service: `/etc/systemd/system/monitor.service`:
  ```bash
  [Unit]
  Description=System Monitoring Script
  [Service]
  ExecStart=/usr/bin/flock -n /run/temp/monitor.lock /usr/local/bin/monitor.sh > >(tee -a /run/temp/out.log) 2> >(tee -a /run/temp/err.log > /dev/null)
  ```
- Created timer: `/etc/systemd/system/monitor.timer`:
  ```bash
  [Unit]
  Description=Run monitor script every minute
  [Timer]
  OnCalendar=*:0/1
  Persistent=true
  [Install]
  WantedBy=timers.target
  ```
- Enabled/started: `sudo systemctl daemon-reload; sudo systemctl enable monitor.timer; sudo systemctl start monitor.timer`.
- Verified: `systemctl list-timers` showed `monitor.timer` active (first run ~04:14 UTC 2025-09-22).
- Tested `flock`:
  - `/usr/bin/flock /run/temp/monitor.lock sleep 70 &` failed initially ("Permission denied") as non-root couldn’t write to `/run/temp`.
  - Multiple runs succeeded but didn’t lock (failed silently). Timer likely created `monitor.lock` ~04:14 UTC.
  - `/usr/bin/flock -n /run/temp/monitor.lock /usr/local/bin/monitor.sh` sometimes ran due to no lock contention or permission quirks.
  - **Fix**: Pre-create lock file for testing:
    ```bash
    sudo touch /run/temp/monitor.lock
    sudo chmod 666 /run/temp/monitor.lock  # Test-only, insecure
    /usr/bin/flock /run/temp/monitor.lock sleep 70 &
    /usr/bin/flock -n /run/temp/monitor.lock /usr/local/bin/monitor.sh  # Fails: "flock: failed to get lock"
    ```

**Why It Matters**: In data centers, timers automate backups or log cleanups. `flock` prevents overlaps, avoiding conflicts. **Analogy**: `flock` is a single-key meeting room; only one task uses it at a time.

## Errors and Fixes

1. **Q36 tmpfiles.d Error**: `/etc/tmpfiles.d/temp.conf` ignored (`d` doesn’t support `64M noexec`). **Fix**: Used `run-temp.mount` for tmpfs.
2. **Q36 Permission Denied**: Non-root couldn’t write `test.sh` to `/run/temp`. **Fix**: `sudo bash -c 'echo "#!/bin/bash" > /run/temp/test.sh'`.
3. **Q36 History Expansion**: `sudo bash -c "echo '#!/bin/bash' ..."` failed (`!/bin/bash` event not found). **Fix**: Used single quotes or escaped `!`.
4. **Q43 Permission Denied**: `tee /run/temp/out.log` failed (non-root). **Fix**: `sudo tee`.
5. **Q45 flock Permission**: Non-root couldn’t create `/run/temp/monitor.lock`. **Fix**: Pre-created with `sudo touch` (timer did this as root).
6. **Q45 flock Inconsistency**: Non-root `flock` runs sometimes worked. **Cause**: No lock file initially; timer created it later. **Fix**: Ensure lock file exists with write permissions for tests.

## Data Center Relevance

- **Q36**: tmpfs reduces disk wear (SSDs); `noexec` secures servers.
- **Q43**: Separating logs streamlines monitoring and debugging.
- **Q45**: `flock` and timers ensure reliable automation (e.g., backups).
- **Troubleshooting**: Permissions and quoting errors mimic real data center issues, teaching debugging.

## Cleanup

```bash
sudo systemctl stop monitor.timer
sudo systemctl disable monitor.timer
sudo rm /etc/systemd/system/monitor.{service,timer}
sudo rm /etc/systemd/system/run-temp.mount
sudo systemctl daemon-reload
sudo umount /run/temp
sudo rm /usr/local/bin/monitor.sh /run/temp/{out.log,err.log,out2.log,test.sh,monitor.lock}
rm ~/err.log
```

## FAQ

- **Q**: Why did `tempfiles.d` fail for tmpfs?
  - **A**: `d` type doesn’t support size/`noexec`; use mount units or `/etc/fstab`.
- **Q**: Why `test.sh` for `noexec` test?
  - **A**: Verifies `noexec` by failing to run an executable script.
- **Q**: Why permission errors in `/run/temp`?
  - **A**: Root-owned (`mode=0755,uid=0,gid=0`); non-root needs `sudo` to write.
- **Q**: Why history expansion error?
  - **A**: `!` in double quotes triggers it; use single quotes or escape (`\#\!/bin/bash`).
- **Q**: Why did `flock` runs fail or succeed inconsistently?
  - **A**: Non-root couldn’t write lock file; timer created it later, causing variable behavior.
- **Q**: When was `monitor.lock` created?
  - **A**: Likely 04:14 UTC 2025-09-22 (first timer run); check with `sudo stat /run/temp/monitor.lock`.

## Practice Questions

1. **Q**: How to verify tmpfs options?
   - **A**: `mount | grep /run/temp` or `cat /proc/mounts`.
2. **Q**: Why does `noexec` block execution even with `sudo`?
   - **A**: Mount option applies system-wide, ignoring file permissions.
3. **Q**: How to write to root-owned dir as non-root?
   - **A**: Use `sudo tee`, e.g., `echo "data" | sudo tee /run/temp/file`.
4. **Q**: How to test `flock` blocking?
   - **A**: Hold lock (`flock /path/lock sleep 70 &`), then try `flock -n /path/lock script.sh`.
