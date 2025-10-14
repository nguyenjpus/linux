---
layout: post
title: "Linux Lab Summary: Mastering Linux Networking"
date: 2025-10-13 11:10:00 -0700
tags:
  [
    Linux+,
    Linuxlab,
    100 Multiple Choices,
    DNS,
    routing,
    ssh,
    nginx,
    troubleshooting,
    systemd,
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

## Section 1: Network Interface Management

### Questions Addressed: 36, 39, 43

**Purpose**: Master interface control, IP assignment, and state monitoring.

- Step 1: Interface Discovery & Control

```bash
# List all interfaces
ip link show
# Output shows enp0s1 (UP), enp0s2 (DOWN)

# Bring interface down/up
ip link set enp0s2 down
ip link set enp0s2 up
ip link show enp0s2  # Verify state change
```

````

**Expected Output**:

```
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
3: enp0s2: <BROADCAST,MULTICAST> ... state DOWN
```

**Key Insight**: `LOWER_UP` flag indicates physical link detection. Virtio interfaces in VMs may not log link state changes to dmesg.

- Step 2: IP Address Management

```bash
# View current IP configuration
ip addr show enp0s1
# Shows: inet 10.0.0.20/24 ... dynamic enp0s1

# Manual IP assignment (temporary)
ip addr add 192.168.1.100/24 dev enp0s2
ip addr show enp0s2
```

**Question 39 Answer**: Use `ip addr add` for temporary static IPs during maintenance.

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

**Expected Output**:

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

**Expected Output**:

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

**SSH Socket Deep Dive**:

- **Socket Activation**: systemd listens on port 22, starts sshd on first connection
- **Benefits**: Resource efficiency, faster startup, security (service stops when idle)
- **Production Tip**: Check both `ssh.socket` and `ssh.service` status

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
# Create test file with checksum
echo "This is a test file for checksum verification" > /tmp/testfile.txt
sha256sum /tmp/testfile.txt
# Output: 856e66ee9fdbdbbaf6b228640b24a84c8e996f376671c616b3eb228c84da23bc

# Serve via Nginx
cp /tmp/testfile.txt /var/www/html/
chown www-data:www-data /var/www/html/testfile.txt

# Download and verify
cd /tmp
wget http://localhost/testfile.txt
sha256sum testfile.txt  # Must match original
```

**Question 44 Answer**: `wget URL && sha256sum file` for secure transfers.

**Real-world**: Verify RPM/DEB packages, firmware updates in data centers.

## Section 6: Persistent Configuration

### Question Addressed: 41

**Purpose**: Make network changes survive reboots using Netplan.

#### Netplan Configuration

**File**: `/etc/netplan/50-cloud-init.yaml`

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
# Validate and test
sudo netplan try          # 120s rollback if broken
sudo netplan apply        # Apply permanently
resolvectl status         # Verify DNS
sudo reboot               # Test persistence
```

**Question 41 Answer**: Edit Netplan YAML + `netplan apply` for permanent changes.

**Key Insight**: `netplan try` prevents lockouts during config testing.

## Key Insights & Production Takeaways

### 🔧 **Essential Commands**

- **`ss -tlpn`**: Modern port checking (shows PIDs)
- **`resolvectl`**: systemd-resolved DNS management
- **`netplan apply`**: Ubuntu's declarative networking
- **`journalctl -fk`**: Live system monitoring
- **`sha256sum`**: File integrity verification

### 🏢 **Data Center Applications**

1. **Service Monitoring**: `ss -tlpn` for web/SSH availability
2. **DNS Reliability**: Google DNS fallback for critical services
3. **Configuration Management**: Netplan + Git for infrastructure-as-code
4. **Security**: Socket activation reduces attack surface
5. **Troubleshooting**: `ip route`, `traceroute`, `dig` for connectivity issues

### ⚠️ **Common Pitfalls**

- **VM Quirks**: Virtio-net silent link state (use `journalctl -k`)
- **Socket Activation**: `systemctl stop ssh` doesn't close port 22
- **DNS Persistence**: `/etc/resolv.conf` managed by systemd-resolved
- **Netplan Syntax**: YAML indentation critical (`netplan try` validates)

## Bookmarks & References

### **Question Mapping**

- **Q35**: `resolvectl` vs legacy DNS tools
- **Q36**: `ip link set dev up/down`
- **Q37**: Default route analysis (`ip route show`)
- **Q38**: HTTPS port monitoring (`ss -tlpn :443`)
- **Q39**: Temporary IP assignment (`ip addr add`)
- **Q40**: Path tracing (`traceroute`, `mtr`)
- **Q41**: Persistent config (Netplan)
- **Q42**: DNS debugging (`dig`, `resolvectl query`)
- **Q43**: Interface state verification
- **Q44**: Secure downloads (`wget + sha256sum`)
- **Q45**: systemd-resolved architecture
- **Q46**: Routing table interpretation
- **Q82**: SSH troubleshooting (service vs socket)
- **Q95**: Kernel monitoring (`dmesg -w`, `journalctl -fk`)

### **Production Hardening**

```bash
# Enable UFW firewall
ufw enable
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp

# Monitor services
systemctl enable --now nginx
systemctl enable ssh.socket  # Keep socket activation

# Backup Netplan
cp -r /etc/netplan /etc/netplan.backup
```

## Next Steps & Cleanup

### 🚀 **Further Exploration**

1. **Firewall Configuration**: UFW + nftables integration
2. **Multi-NIC Bonding**: LACP/LAG for redundancy
3. **VPN Setup**: WireGuard for secure management
4. **Container Networking**: Docker bridge networks
5. **Monitoring**: Prometheus + Node Exporter

### 🧹 **Lab Cleanup**

```bash
# Remove test files
rm -f /var/www/html/testfile.txt
rm -f /tmp/testfile.txt*

# Restore original Netplan (if needed)
cp /etc/netplan/backup.yaml /etc/netplan/50-cloud-init.yaml
netplan apply

# Stop services
systemctl stop nginx
```
````
