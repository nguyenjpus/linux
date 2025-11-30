---
layout: post
title: "CHAPTER 3: SERVICES AND USER MANAGEMENT - PROCESS CONTROL, JOB SCHEDULING, AND CONTAINERS OPERATIONS MASTERY"
date: 2025-11-29 16:00:00 -0700
tags:
  [
    Linux+,
    Linuxlab,
    Chapter 3,
    Nice,
    Renice,
    At,
    Cron,
    Pstree,
    Systemd,
    Kill,
    Pkill,
    Curl,
    Systemctl,
    Docker,
    Timer,
    Nginx,
  ]
---

This document summarizes my hands-on lab experience mastering Linux process management, job scheduling, systemd services, and container operations on Ubuntu 24.04. The goal was to deeply understand these core Linux concepts through practical exercises.

## 3.2 Process Management & Scheduling

### 3.2.1 Top-Level Process Tools

```bash
ps -eo pid,ppid,user,stat,pri,ni,cmd --sort=-%mem | head -15
# Shows processes sorted by memory usage. PRI = kernel priority, NI = nice value.

pstree -p
# Visual tree — proves your shell is a child of sshd → systemd.
```

### 3.2.2 Nice & Renice

```bash
sudo apt install stress -y
nice -n 19 stress --cpu 2 --timeout 180 &
LOWPID=$!
# nice 19 = lowest CPU priority. $! = PID of last background job.

sudo renice -n -10 -p $LOWPID
# Changes priority of a running process (only root can increase priority).
```

### 3.2.3 Background Jobs & Signals

```bash
sleep 300 &
jobs -l
fg %1
^C                         # Sends SIGINT (2) → immediate kill
kill -15 <pid>             # Sends SIGTERM → graceful shutdown
pkill -f sleep             # Kills by command name
```

### 3.2.4 Cron, Anacron, and `at`

```bash
crontab -e
# User-specific jobs. Syntax: minute hour dom month dow command

# Example entries we used:
* * * * *           echo "Every minute - $(date)" >> ~/cron-minute.log
*/5 * * * *         echo "Every 5 min - $(date)"  >> ~/cron-5min.log
17 10 * * *         echo "Daily at 10:17 - $(date)" >> ~/cron-daily.log

# One-time job
at now + 2 minutes
  echo "Hello from at - $(date)" >> ~/at-job.log
  Ctrl+D
atq                         # List queued at jobs
```

### 3.2.5–3.2.7 Package Management & Secure Repositories

```bash
dpkg -S $(which nginx)      # Which .deb package owns a file?
dpkg -L nginx               # List every file installed by a package
sudo apt-mark hold nginx    # Prevents accidental upgrades (critical in production)

# Modern (2024+) way to add third-party repo securely
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
# --dearmor converts ASCII-armored key to binary format apt expects

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
# [signed-by=…] = only trust packages signed by this exact key → defense against repo compromise
```

### 3.2.8–3.2.12 Systemd Mastery

```bash
systemctl status ssh nginx
sudo systemctl mask snapd
# mask = creates symlink to /dev/null → unit cannot be started even manually

# Create a real systemd timer (preferred over cron on modern systems)
sudo tee /etc/systemd/system/backup.service <<EOF
[Unit]
Description=Weekly Backup

[Service]
Type=oneshot                    # Job runs → exits immediately
ExecStart=/bin/bash -c 'echo "Backup ran at $(date)" >> /var/log/mybackup.log'
EOF

sudo tee /etc/systemd/system/backup.timer <<EOF
[Timer]
OnCalendar=Fri *-*-* 18:55:00   # Every Friday at 18:55
Persistent=true                 # Runs missed jobs after boot
EOF

sudo systemctl enable --now backup.timer
journalctl -u backup.service    # View only this unit’s logs
```

---

## 3.3 Container Operations (Docker)

### 3.3.1–3.3.2 Image & Container Basics

```bash
docker run -d --name web1 -p 8080:80 nginx:1.26
# -d = detached, -p host:container, --name = friendly name
```

### 3.3.3 Container Networking (User-Defined Network)

```bash
docker network create --subnet=172.20.0.0/16 mynet
# Built-in DNS only works on user-defined networks

docker run -d --name db   --network mynet postgres:16
docker run -d --name web2 --network mynet -p 8081:80 nginx:1.26

# nginx image has no ping → install it (common exam trick)
docker exec web2 sh -c "apt-get update && apt-get install -y iputils-ping"
```

### 3.3.4 Bind Mounts (the part that always trips people up)

```bash
mkdir -p ~/webdata
echo "<h1>VICTORY – $(date)</h1>" > ~/webdata/index.html

# Final working version (explained)
docker run -d --name webvol \
  -p 8082:80 \
  -v ~/webdata:/usr/share/nginx/html:ro \
  --user root \
  nginx:1.26

# Why --user root?
# Official nginx image runs workers as user "nginx" (UID 101).
# UID 101 does not exist on your host → permission denied → 403.
# Running as root bypasses this (perfectly valid for labs/exams).
# Alternative: use :z flag if SELinux is enforced.
```

### 3.3.5 Resource Limits

```bash
docker run -d --name limited \
  --memory=64m --cpus=0.2 \
  -p 8083:80 \
  nginx:1.26

docker stats limited   # Real-time resource usage
```

### Final Cleanup

```bash
docker stop db web2 webvol limited 2>/dev/null || true
docker rm   db web2 webvol limited 2>/dev/null || true
docker network rm mynet 2>/dev/null || true
docker system prune -f
```

---

**Completion Date:** November 29, 2025
