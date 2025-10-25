---
layout: post
title: "Linux Lab Summary: Mastering Linux Networking"
date: 2025-10-13 11:10:00 -0700
tags:
  [
    Linux+,
    Linuxlab,
    "100 Multiple Choices",
    DNS,
    Routing,
    SSH,
    Nginx,
    Troubleshooting,
    Systemd,
  ]
---

A quick summary of my Linux networking lab, covering key concepts like interface management, IP addressing, DNS configuration, routing, SSH troubleshooting, and web server setup. This lab addresses specific questions (35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 82, 95) from the CompTIA Linux+ exam objectives. Each section includes commands executed, expected outputs, explanations for answers chosen based on Linux fundamentals and lab experience, and practical insights for real-world applications. The lab was conducted on Ubuntu 24.04.3 LTS (ARM64) in UTM on an M1 MacBook with two network interfaces: `enp0s1` (virtio_net) for primary connectivity and `enp0s2` (e100) for secondary testing.

**Operating System**: Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-79-generic aarch64)  
**Virtualization**: QEMU/UTM on Apple Silicon Mac (ARM64 architecture)  
**Kernel Features**: PREEMPT_DYNAMIC, AppArmor, systemd-resolved, Netplan

**Network Setup**:

- **enp0s1**: Primary interface (DHCP, 10.0.0.20/24, gateway 10.0.0.1)
- **enp0s2**: Secondary interface (disabled for testing)
- **DNS**: Initially Comcast (75.75.75.75), modified to Google DNS (8.8.8.8)
- **Services**: Nginx web server, SSH with socket activation

**Real-world Application**: This lab simulates data center server management where administrators configure network interfaces, troubleshoot connectivity, and ensure service availability across reboots.

---

## Section 1: Network Interface Management

### Questions Addressed: 36, 39, 43

**Purpose**: Master interface control, IP assignment, and state monitoring.

#### Step 1: Interface Discovery & Control

```bash
# List all interfaces
ip link show
# Output shows enp0s1 (UP), enp0s2 (DOWN)

# Bring interface down/up
ip link set enp0s2 down
ip link set enp0s2 up
ip link show enp0s2  # Verify state change
```

**Expected Output:**

```
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
3: enp0s2: <BROADCAST,MULTICAST> ... state DOWN
```

**Key Insight**: `LOWER_UP` flag indicates physical link detection. Virtio interfaces in VMs may not log link state changes to dmesg.

#### Step 2: IP Address Management

```bash
# View current IP configuration
ip addr show enp0s1
# Shows: inet 10.0.0.20/24 ... dynamic enp0s1

# Manual IP assignment (temporary)
ip addr add 192.168.1.100/24 dev enp0s2
ip addr show enp0s2
```

**Question 39 Answer**: Use `ip addr add` for temporary static IPs during maintenance.

---

## Section 2: DNS Configuration & Troubleshooting

### Questions Addressed: 35, 42, 45

**Purpose**: Configure DNS resolvers and understand systemd-resolved integration.

#### Step 1: DNS Status & Temporary Changes

```bash
# Check current DNS configuration
resolvectl status
cat /etc/resolv.conf  # Symlink to systemd-resolved stub

# Temporary DNS override
resolvectl dns enp0s1 8.8.8.8 8.8.4.4
resolvectl status  # Verify Google DNS applied
nslookup google.com  # Test resolution
```

**Expected Output:**

```
Link 2 (enp0s1)
  Current Scopes: DNS
  DNS Servers: 8.8.8.8 8.8.4.4
```

**Key Observation**: `/etc/resolv.conf` is managed by `systemd-resolved` (stub resolver at 127.0.0.53). Direct edits are overwritten.

#### Step 2: DNS Debugging

```bash
# DNS query debugging
dig google.com @8.8.8.8
resolvectl query google.com
```

**Question 35 Answer**: Use `resolvectl` for modern DNS management over legacy `resolvconf`.

---

## Section 3: Routing & Connectivity Diagnostics

### Questions Addressed: 37, 40, 46, 95

**Purpose**: Analyze routing tables, trace paths, and monitor kernel events.

#### Step 1: Routing Table Analysis

```bash
ip route show
# Output:
# default via 10.0.0.1 dev enp0s1
# 10.0.0.0/24 dev enp0s1 proto kernel scope link src 10.0.0.20
```

**Question 37 Answer**: Default route points to gateway (10.0.0.1) for external traffic.

#### Step 2: Path Tracing

```bash
traceroute 8.8.8.8
mtr 8.8.8.8  # Combined ping/traceroute
```

#### Step 3: Kernel Event Monitoring (Q95)

```bash
# Watch kernel logs live
dmesg -w
journalctl -fk  # Alternative with filtering

# Test link flapping (may be silent in VMs)
ip link set enp0s2 down; sleep 2; ip link set enp0s2 up
```

**Key Insight**: Virtio-net interfaces don't always log link state to dmesg. Use `journalctl -k` for kernel logs in production.

---

## Section 4: Service Troubleshooting

### Questions Addressed: 38, 82

**Purpose**: Diagnose web server and SSH connectivity issues.

#### Step 1: Port Monitoring

```bash
# Check listening services
ss -tlpn | grep :80    # Nginx HTTP
ss -tlpn | grep :443   # HTTPS (empty by default)
ss -tlpn | grep :22    # SSH
```

**Expected Output:**

```
LISTEN 0 511 0.0.0.0:80 0.0.0.0:* users:(("nginx",pid=3237,fd=5),...)
```

**Question 38 Answer**: `ss -tlpn | grep :443` reveals missing HTTPS listener.

#### Step 2: SSH Socket Activation

```bash
systemctl stop ssh        # Stops service, socket remains
ss -tlpn | grep :22       # Still shows systemd listening
systemctl status ssh.socket  # Always active
systemctl status ssh.service # On-demand
```

**Question 82 Diagnosis**: Empty port 22 + firewall disabled + network OK = SSH daemon not running.

---

## Section 5: File Transfer & Integrity

### Question Addressed: 44

**Purpose**: Secure file downloads with checksum verification.

#### Step 1: Web Server Setup

```bash
apt install nginx -y
systemctl start nginx
curl http://localhost  # Verify welcome page
```

#### Step 2: File Serving & Download

```bash
echo "This is a test file for checksum verification" > /tmp/testfile.txt
sha256sum /tmp/testfile.txt

cp /tmp/testfile.txt /var/www/html/
chown www-data:www-data /var/www/html/testfile.txt

cd /tmp
wget http://localhost/testfile.txt
sha256sum testfile.txt  # Must match original
```

**Question 44 Answer**: `wget URL && sha256sum file` for secure transfers.

---

## Section 6: Persistent Configuration

### Question Addressed: 41

#### Netplan Configuration

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s1:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    enp0s2:
      dhcp4: false
```

#### Apply & Test

```bash
sudo netplan try
sudo netplan apply
resolvectl status
sudo reboot
```

**Question 41 Answer**: Edit Netplan YAML + `netplan apply` for permanent changes.

---

## Key Insights & Production Takeaways

- `ss -tlpn`: Port checking
- `resolvectl`: DNS management
- `netplan apply`: Networking config
- `journalctl -fk`: System monitoring
- `sha256sum`: Integrity checks

---

## Cleanup

```bash
rm -f /var/www/html/testfile.txt
rm -f /tmp/testfile.txt*
cp /etc/netplan/backup.yaml /etc/netplan/50-cloud-init.yaml
netplan apply
systemctl stop nginx
```
