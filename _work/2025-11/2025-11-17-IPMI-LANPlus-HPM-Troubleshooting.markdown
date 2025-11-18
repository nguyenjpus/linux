---
layout: post
title: IPMI LANPlus & HPM Check Troubleshooting
date: 2025-11-17
tags: [Networking, Ipmitool, Troubleshooting, Hpm]
---

I encountered an issue while trying to run `ipmitool` commands over LANPlus to check the HPM (Hardware Platform Management) status on a server's BMC (Baseboard Management Controller). The local commands worked fine, but remote commands failed to establish a session. Here’s a detailed breakdown of the troubleshooting steps I took, the root cause, and the resolution.

## Problem Description

- Attempting to run `ipmitool -I lanplus -H <BMC_IP> -U admin -P admin hpm check` failed with:
  ```
  Error: Unable to establish IPMI v2 / RMCP+ session
  ```
- However, running `ipmitool hpm check` locally worked fine.

## Investigation Steps

1. Verified BMC GUI accessibility using `admin/admin`.
2. Confirmed IPMI Over LAN (IOL) was enabled in BMC GUI.
3. Checked user privileges via:
   ```bash
   ipmitool channel getaccess 1
   ```
   - `admin` had ADMINISTRATOR privilege and IPMI messaging enabled.
4. Tested IPMI LANPlus with different cipher suites:
   ```bash
   ipmitool -I lanplus -H <BMC_IP> -U admin -P admin -C 12 mc info
   ipmitool -I lanplus -H <BMC_IP> -U admin -P admin -C 17 mc info
   ```
   - All failed with RMCP+ session error.
5. Attempted verbose mode:
   ```bash
   ipmitool -I lanplus -H <BMC_IP> -U admin -P admin -v
   ```
   - Showed `Get Auth Capabilities error`.
6. Suspected network/firewall issue. Tried checking UDP 623 connectivity.
   - Found `nc` missing, so checked firewall rules manually.

## Root Cause

- Server firewall was blocking **UDP port 623**, which is required for IPMI over LAN (RMCP+).

## Resolution

- Removed firewall rule blocking UDP 623. Could be done via BMC GUI (Check firewall settings or server firewall).
- Retested:
  ```bash
  ipmitool -I lanplus -H <BMC_IP> -U admin -P admin hpm check
  ```
  - Success! HPM check now works remotely.

## Key Learnings

- IPMI LANPlus uses **UDP 623** for RMCP+ handshake.
- If `lanplus` fails but local IPMI works, check:
  - Firewall rules (UDP 623 must be open).
  - Cipher suite configuration.
  - Channel privileges.
- SSH being disabled does **not** affect IPMI LAN.

## Commands Cheat Sheet

```bash
# Check channel access
ipmitool channel getaccess 1

# Test IPMI LANPlus
ipmitool -I lanplus -H <BMC_IP> -U admin -P admin chassis status

# Check firewall rules
iptables -L -n | grep 623
firewall-cmd --list-all

# Remove firewall block (example)
iptables -D INPUT -p udp --dport 623 -j DROP
```

---

**Final Outcome:** After allowing UDP 623, IPMI LANPlus and HPM check worked successfully.
