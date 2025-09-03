# Hands-On: Log Rotation with `logrotate`

This guide demonstrates log rotation, compression, and management using `logrotate`, with `xz` compression for long-term storage and integration with `systemd` journald using `zstd`. It fixes issues with premature `xz` compression in `logrotate` and persistent journal storage errors in `systemd-journald`.

## Prerequisites
Install required packages:
```bash
sudo apt install -y tar gzip bzip2 xz-utils zstd logrotate rsync gddrescue testdisk extundelete e2fsprogs xfsprogs lvm2
```

## Step-by-Step Instructions

### 1. Setup Test Log
Create a log file and simulate growth:
```bash
sudo mkdir -p /var/log/myapp
sudo touch /var/log/myapp/app.log
sudo chmod 644 /var/log/myapp/app.log
sudo chown root:root /var/log/myapp/app.log
for i in {1..100}; do echo "Log entry $i" | sudo tee -a /var/log/myapp/app.log; done
```

### 2. Configure `logrotate`
Create `/etc/logrotate.d/myapp`:
```bash
sudo nano /etc/logrotate.d/myapp
```
Paste:
```
/var/log/myapp/app.log {
    daily
    rotate 7
    create 0644 root root
    delaycompress
    missingok
    notifempty
    postrotate
        /usr/bin/xz -9 /var/log/myapp/app.log.2 2>/dev/null || true
    endscript
}
```
- `daily`: Rotate daily.
- `rotate 7`: Keep 7 rotated logs.
- `create 0644 root root`: Create a new `app.log` with `0644` permissions and `root:root` ownership.
- `delaycompress`: Delay compression until the next cycle.
- `missingok`: Ignore missing logs.
- `notifempty`: Skip empty logs.
- `postrotate`: Compress `app.log.2` with `xz -9`, producing `app.log.2.xz`.

**Note**: `delaycompress` leaves `app.log.1` uncompressed after the first rotation; it’s compressed to `app.log.2.xz` on the second rotation.

### 3. Test Rotation
Force first rotation:
```bash
sudo logrotate -f /etc/logrotate.d/myapp
```
Verify:
```bash
ls -l /var/log/myapp/
```
Expected output:
```
-rw-r--r-- 1 root root    0 Sep  2 06:49 app.log
-rw-r--r-- 1 root root 1650 Sep  2 06:49 app.log.1
```
Copy and check uncompressed log:
```bash
sudo cp /var/log/myapp/app.log.1 .
cat app.log.1
```
Force second rotation to trigger compression:
```bash
echo "New log" | sudo tee -a /var/log/myapp/app.log
sudo logrotate -f /etc/logrotate.d/myapp
ls -l /var/log/myapp/
```
Expected output:
```
-rw-r--r-- 1 root root    0 Sep  2 06:50 app.log
-rw-r--r-- 1 root root   20 Sep  2 06:50 app.log.1
-rw-r--r-- 1 root root  512 Sep  2 06:50 app.log.2.xz
```
Extract and check:
```bash
sudo cp /var/log/myapp/app.log.2.xz .
xz -d app.log.2.xz
cat app.log.2
```
Simulate multiple rotations:
```bash
for i in {1..3}; do echo "New log" | sudo tee -a /var/log/myapp/app.log; sudo logrotate -f /etc/logrotate.d/myapp; done
ls -l /var/log/myapp/
```

### 4. Integration with Systemd (zstd)
Enable `zstd` compression for journald:
```bash
sudo mkdir -p /var/log/journal
sudo chown root:systemd-journal /var/log/journal
sudo chmod 2755 /var/log/journal
sudo nano /etc/systemd/journald.conf
```
Set:
```
[Journal]
Storage=auto
Compress=yes
```
Restart:
```bash
sudo systemctl restart systemd-journald
```
Generate a test log:
```bash
logger "Test journal entry"
```
Verify journal operation:
```bash
journalctl -u systemd-journald.service
```
Expected output (example):
```
Sep 02 06:49:00 usv systemd-journald[12345]: Journal started
Sep 02 06:49:00 usv systemd-journald[12345]: System Journal (/var/log/journal/fc6176dab13c44a48a589f25835fffca) is 8.0M, max 4.0G, 3.9G free.
```
Verify test log:
```bash
journalctl | grep "Test journal entry"
```
Expected output:
```
Sep 02 06:49:01 usv user[12346]: Test journal entry
```
Check disk usage (indicating compression):
```bash
journalctl --disk-usage
```
Expected output:
```
Archived and active journals take up 8.0M on disk.
```

### 5. Cleanup
Remove test files and revert changes:
```bash
sudo rm -rf /var/log/myapp /etc/logrotate.d/myapp
sudo sed -i 's/Storage=auto/#Storage=auto/' /etc/systemd/journald.conf
sudo sed -i 's/Compress=yes/#Compress=no/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

### 6. Deepen Understanding
Simulate continuous log growth:
```bash
while true; do echo "Log $(date)" | sudo tee -a /var/log/myapp/app.log; sleep 60; done
```
Run in the background and monitor rotations with `ls /var/log/myapp`.

## Notes
- **Error Fixes**:
  - Removed `compress` directive in `logrotate` to prevent `gzip`, ensuring `xz` compression via `postrotate`.
  - Added `/var/log/journal` setup to fix `systemd-journald` error `Failed to create new system journal`.
- **Troubleshooting**:
  - For `logrotate`: Use `logrotate -d` to debug.
  - For journald: Check `/var/log/journal/` permissions and `journalctl --disk-usage`.
  - Ensure `xz-utils` is installed (`sudo apt install xz-utils`).
- **Best Practices**: Use `xz` for long-term log storage (>3 months). Use `create` and `delaycompress` in `logrotate`. Ensure persistent journal storage for `systemd-journald`.