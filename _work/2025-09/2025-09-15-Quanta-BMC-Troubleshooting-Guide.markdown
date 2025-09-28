---
layout: post
title: Quanta Server BMC Troubleshooting - A note for future reference
date: 2025-09-15
tags: [Quanta, BMC, Ipmitool, Troubleshooting]
---

# Quanta Server BMC Troubleshooting Guide

This guide summarizes the process of troubleshooting a Quanta server’s BMC (likely ASPEED) to retrieve the SDR list using `ipmitool`, `nmap`, and `dnsmasq` on a macOS 12 laptop with a USB-to-Ethernet adapter (`en17`, `192.168.0.1/24`). Despite multiple attempts, the BMC didn’t respond, likely due to a hardware fault, incorrect port, or configuration issue. Use this guide to retry when conditions (e.g., correct BMC port, functional hardware) are met.

## Objective

Connect to the Quanta server’s BMC via the dedicated BMC port, assign it an IP address via DHCP (or detect a static IP), and retrieve the SDR list using:

```
ipmitool -I lanplus -H <BMC_IP> -U ADMIN -P ADMIN sdr list
```

## Tools Used

- **ipmitool (1.8.19)**: For IPMI commands to interact with the BMC (e.g., SDR list).
- **nmap (7.98)**: For scanning the network to detect the BMC on port 623 (IPMI).
- **dnsmasq (2.91)**: For serving DHCP IPs (`192.168.0.10–192.168.0.20`) to the BMC via `en17`.

## Setup

- **Hardware**:
  - Quanta server.
  - USB-to-Ethernet adapter (`en17`) on macOS 12 laptop, configured as `192.168.0.1/24`.
  - Ethernet cable connected from the BMC port (labeled “IPMI” or “MGMT”) to the USB-to-Ethernet adapter.
  - Server powered off, but power supply light on (BMC powered).
  - BMC port LEDs: Green (link) and yellow (activity) blinking, confirming connection.
- **Network**:
  - `en17`: `192.168.0.1 netmask 255.255.255.0 broadcast 192.168.0.255`.
  - `dnsmasq` configured to serve IPs in `192.168.0.10–192.168.0.20` with a 12-hour lease.

## Troubleshooting Steps

### 1. Verify Tool Installation

- **ipmitool**:
  ```
  ipmitool -V
  ```
  - Output: `ipmitool version 1.8.19`.
  - Test: `ipmitool mc info` (returns “No hostname specified!” on macOS, expected).
  - IANA file: `/usr/local/share/misc/enterprise-numbers` (starts with “PRIVATE ENTERPRISE NUMBERS”).
- **nmap**:
  ```
  nmap --version
  ```
  - Output: `Nmap version 7.98`.
  - Test: `sudo nmap -sU -p 123 8.8.8.8` (shows `123/udp open|filtered`, confirming UDP scanning).
- **dnsmasq**:
  ```
  dnsmasq --version
  ```
  - Output: `Dnsmasq version 2.91`.
  - Config: `/usr/local/etc/dnsmasq.conf`:
    ```
    interface=en17
    dhcp-range=192.168.0.10,192.168.0.20,12h
    log-dhcp
    log-facility=/usr/local/var/log/dnsmasq.log
    ```
  - Lease file: `/usr/local/var/lib/misc/dnsmasq/dnsmasq.leases` (permissions `666`).
  - Log file: `/usr/local/var/log/dnsmasq.log`.

### 2. Fix dnsmasq Issues

- **Issue**: `dnsmasq` failed with “address already in use” due to multiple instances.
- **Solution**:
  - Check running processes:
    ```
    ps aux | grep dnsmasq
    ```
    - Example output:
      ```
      root 42224 0.0 0.0 34140688 840 ?? S 12:07PM 0:00.01 /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq ...
      ```
  - Kill conflicting processes:
    ```
    sudo kill -9 <PID>
    ```
    - Replace `<PID>` with process ID (e.g., `42224`).
  - Check port 67 (DHCP):
    ```
    sudo lsof -i :67
    ```
  - Restart `dnsmasq`:
    ```
    sudo /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq --no-daemon -C /usr/local/etc/dnsmasq.conf
    ```
    - Expected output:
      ```
      dnsmasq: started, version 2.91 cachesize 150
      dnsmasq-dhcp: DHCP, IP range 192.168.0.10 -- 192.168.0.20, lease time 12h
      ...
      ```
  - Clear lease file (optional):
    ```
    sudo rm /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    sudo touch /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    sudo chmod 666 /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    ```

### 3. Monitor DHCP Activity

- Run `dnsmasq` and watch logs:
  ```
  sudo tail -f /usr/local/var/log/dnsmasq.log
  ```
- Look for `DHCPDISCOVER` and `DHCPOFFER` (e.g., BMC assigned `192.168.0.10`).
- **Observation**: No DHCP activity from the BMC, even after reset.

### 4. Scan for BMC

- **nmap** scan:
  ```
  sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.0/24
  ```
  - Output: Only `192.168.0.1` (Mac) with `623/udp closed`.
  - Note: `nmap` automatically uses `en17` for `192.168.0.0/24`.
- Broader scan (ports 22, 80, 443, 623, 161, 162):
  ```
  sudo nmap -sU -p 22,80,443,623,161,162 192.168.0.0/24
  ```
  - Purpose: Check for BMC web interface, SSH, or SNMP.
- **Observation**: No BMC detected.

### 5. Test Static IP

- Previous unit used static IP `192.168.0.120` (outside DHCP range, not assigned by `dnsmasq`).
- Test:
  ```
  ping 192.168.0.120
  sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.120
  ipmitool -I lanplus -H 192.168.0.120 -U ADMIN -P ADMIN sdr list
  ```
  - Try credentials: `admin/admin`, `root/root`.
- **Observation**: No response at `192.168.0.120`.

### 6. Perform CMOS/BMC Reset

- **Purpose**: Reset BMC to default settings (DHCP, `ADMIN/ADMIN`).
- **Steps**:
  1. Unplug server power cables.
  2. Locate jumper (“CLR_CMOS”, “JCMOS1”, or “BMC_RST”).
  3. Reset:
  4. Reconnect power, wait 3–5 minutes.
  5. Monitor DHCP:
     ```
     sudo tail -f /usr/local/var/log/dnsmasq.log
     ```
  6. Scan:
     ```
     sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.0/24
     ```
- **Observation**: No DHCP activity or BMC response post-reset.

### 7. Test BMC

- If an IP is assigned (e.g., `192.168.0.10`):
  ```
  ipmitool -I lanplus -H 192.168.0.10 -U ADMIN -P ADMIN sdr list
  ```
- **Observation**: No IP assigned, no response.

## Key Learnings

1. **Tool Setup**:
   - `ipmitool`, `nmap`, and `dnsmasq` were correctly installed and verified on macOS.
   - `dnsmasq` requires careful management to avoid “address already in use” errors (check `ps aux | grep dnsmasq` and `lsof -i :67`).
2. **DHCP vs. Static IP**:
   - Most Quanta BMCs use DHCP, but some use static IPs like `192.168.0.120`.
   - Check static IPs outside the DHCP range (`192.168.0.10–192.168.0.20`).
3. **Physical Setup**:
   - Confirm BMC port (labeled “IPMI” or “MGMT”) with green/yellow LEDs.
   - Verify USB-to-Ethernet adapter (`en17`, `192.168.0.1/24`) and cable.
   - BMC is powered even when the server is off (power supply light on).
4. **CMOS Reset**:
   - Resetting the BMC (via jumper) should enable DHCP, but may require longer duration (15–30 seconds).
   - No response suggests incomplete reset or hardware issue.
5. **Network Scanning**:
   - `nmap` UDP scans (`-sU -p 623`) are effective for detecting IPMI devices.
   - Broader scans (ports 22, 80, 443, etc.) can detect BMC web interfaces or other services.
6. **Challenges**:
   - Unknown server model limited specific guidance (e.g., jumper location).
   - No DHCP activity or IPMI response suggests BMC fault, incorrect port, or firmware issue.

## Future Retry Steps

To retry when conditions are met (e.g., confirmed BMC port, functional hardware, known server model):

1. **Verify Physical Setup**:
   - Connect Ethernet cable to BMC port (labeled “IPMI” or “MGMT”).
   - Check green/yellow LEDs (link/activity).
   - Use a known-working cable and USB-to-Ethernet adapter.
   - Confirm `en17`: `ifconfig en17` (should show `192.168.0.1/24`).
2. **Start dnsmasq**:
   ```
   sudo /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq --no-daemon -C /usr/local/etc/dnsmasq.conf
   sudo tail -f /usr/local/var/log/dnsmasq.log
   ```
3. **Perform CMOS Reset**:
   - Unplug power, short jumper (15–30 seconds), reconnect power, wait 3–5 minutes.
4. **Scan for BMC**:
   ```
   sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.0/24
   sudo nmap -sU -p 22,80,443,623,161,162 192.168.0.0/24
   ```
5. **Test Static IP**:
   ```
   ping 192.168.0.120
   ipmitool -I lanplus -H 192.168.0.120 -U ADMIN -P ADMIN sdr list
   ```
6. **Test Assigned IP** (if DHCP works):
   ```
   ipmitool -I lanplus -H <BMC_IP> -U ADMIN -P ADMIN sdr list
   ```
7. **Identify Server Model**:
   - Check chassis label or boot to BIOS/OS for `dmidecode -t system`.
   - Use model to find exact jumper locations or Quanta support resources.

## Notes

- **Why It Failed**: Likely causes include incorrect BMC port, faulty BMC, or incomplete reset. The previous unit’s static IP (`192.168.0.120`) worked by chance, but this unit didn’t respond via DHCP or static IP.
- **Next Steps**: If retrying fails, identify the server model and contact Quanta support for BMC diagnostics or firmware updates.
- **Tools**: Stick to `ipmitool`, `nmap`, and `dnsmasq` for consistency, as they’re sufficient for most BMC access tasks.
