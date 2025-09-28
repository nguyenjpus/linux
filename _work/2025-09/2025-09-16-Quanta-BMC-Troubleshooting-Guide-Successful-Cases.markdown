---
layout: post
title: Quanta Server BMC Troubleshooting - Successful cases
date: 2025-09-16
tags: [Quanta, BMC, Ipmitool, Troubleshooting]
---

This guide details the successful process of accessing a Quanta server’s BMC (likely ASPEED) to retrieve the SDR list using `ipmitool`, `nmap`, and `dnsmasq` on a macOS 12 laptop with a USB-to-Ethernet adapter (`en17`, `192.168.0.1/24`). One unit was configured from static IP (`192.168.0.120`) to DHCP (`192.168.0.13`) with `admin/adminpassword`. Another unit was found at `10.138.36.197` (static, expected `10.138.32.197`), pingable after setting `en17` to `10.138.36.1/24`, but credentials failed. This guide includes steps to identify `en17`, set the DHCP pool, troubleshoot tool installations, handle no BMC response, scan subnets systematically (from common small subnets to larger ones), address `ping` failures, and manage credential issues.

## Objective

Connect to the Quanta BMC, assign an IP via DHCP (or detect static IP like `192.168.0.120` or `10.138.36.197`), and retrieve the SDR list:

```
ipmitool -I lanplus -H <BMC_IP> -U admin -P adminpassword sdr list
```

## Tools Used

- **ipmitool (1.8.19)**: For IPMI commands (e.g., SDR list, LAN settings, password reset).
- **nmap (7.98)**: For scanning port 623 (IPMI) and other services.
- **dnsmasq (2.91)**: For serving DHCP IPs (`192.168.0.10–192.168.0.20`) via `en17`.

## Tool Installation and Troubleshooting

### Homebrew Setup

- **Install Homebrew** (if needed):

  ```
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```

- **Update Homebrew**:

  ```
  brew update --force --verbose
  ```

  - **Troubleshooting**: If `Failed to download https://formulae.brew.sh/api/cask.jws.json`:

    ```
    rm -rf "$(brew --cache)"
    cd "$(brew --repo)"
    git fetch
    git reset --hard origin/master
    brew update --force
    ```

### ipmitool Installation

- **Install**:

  ```
  brew install ipmitool
  ```

  - **Troubleshooting**: If fails with `FAILED to download the IANA PEN database`:

    - Manually download IANA file:

      ```
      curl -o /tmp/enterprise-numbers.txt https://www.iana.org/assignments/enterprise-numbers.txt
      ```

      - Verify:

        ```
        ls -l /tmp/enterprise-numbers.txt
        head -5 /tmp/enterprise-numbers.txt
        ```

        - Should start with “PRIVATE ENTERPRISE NUMBERS”.

      - Install file:

        ```
        sudo mkdir -p /usr/local/share/misc
        sudo cp /tmp/enterprise-numbers.txt /usr/local/share/misc/enterprise-numbers
        sudo chmod 644 /usr/local/share/misc/enterprise-numbers
        rm /tmp/enterprise-numbers.txt
        ```

      - Set environment:

        ```
        export IPMITOOL_IANA_DIR=/usr/local/share/misc
        echo 'export IPMITOOL_IANA_DIR=/usr/local/share/misc' >> ~/.zshrc
        ```

      - Reinstall:

        ```
        brew install ipmitool
        ```

- **Verify**:

  ```
  ipmitool -V
  ```

  - Expected: `ipmitool version 1.8.19`.
  - Test: `ipmitool mc info` (returns “No hostname specified!”).

### nmap Installation

- **Install**:

  ```
  brew install nmap
  ```

  - **Troubleshooting**: If permissions error (e.g., `/usr/local/share/man/man8` not writable):

    ```
    sudo chown -R $USER /usr/local/share/man/man8
    chmod u+w /usr/local/share/man/man8
    brew install nmap
    ```

- **Verify**:

  ```
  nmap --version
  ```

  - Expected: `Nmap version 7.98`.
  - Test: `sudo nmap -sU -p 123 8.8.8.8` (`123/udp open|filtered`).

### dnsmasq Installation

- **Install**:

  ```
  brew install dnsmasq
  ```

  - **Troubleshooting**: If ownership warning:

    ```
    sudo chown -R $USER /usr/local/share/man
    chmod u+w /usr/local/share/man
    brew install dnsmasq
    ```

- **Verify**:

  ```
  /usr/local/Cellar/dnsmasq/*/sbin/dnsmasq --version
  ```

  - Expected: `Dnsmasq version 2.91`.

## Setup

### Identify USB-to-Ethernet Adapter (`en17`)

- **List Network Services**:

  ```
  networksetup -listallnetworkservices
  ```

  - Look for entries like `USB 10/100/1000 LAN` (e.g., `USB 10/100/1000 LAN 13`).

- **Check Interfaces**:

  ```
  ifconfig
  ```

  - Identify `en17` (e.g., `ether 5c:85:7e:38:cf:e6`, `status: active`).
  - **How to Confirm**: Unplug/replug the USB adapter; the `enX` that appears/disappears is `en17`.

- **Set IP**:

  ```
  sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
  ```

- **Verify**:

  ```
  ifconfig en17
  ```

  - Expected: `inet 192.168.0.1 netmask 0xffffff00 broadcast 192.168.0.255`.

### Configure dnsmasq DHCP Pool

- **Edit Configuration**:

  ```
  sudo nano /usr/local/etc/dnsmasq.conf
  ```

  - Add or verify:

    ```
    interface=en17
    dhcp-range=192.168.0.10,192.168.0.20,12h
    log-dhcp
    log-facility=/usr/local/var/log/dnsmasq.log
    ```

    - `interface=en17`: Binds to USB-to-Ethernet adapter.
    - `dhcp-range=192.168.0.10,192.168.0.20,12h`: Sets DHCP pool to `192.168.0.10–192.168.0.20`, 12-hour lease.
    - `log-dhcp`: Logs DHCP events.
    - `log-facility`: Specifies log file.

  - Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

- **Create Lease File**:

  ```
  sudo mkdir -p /usr/local/var/lib/misc/dnsmasq
  sudo touch /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
  sudo chmod 666 /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
  ```

- **Start dnsmasq**:

  ```
  sudo /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq --no-daemon -C /usr/local/etc/dnsmasq.conf
  ```

  - Verify DHCP pool:

    ```
    dnsmasq-dhcp: DHCP, IP range 192.168.0.10 -- 192.168.0.20, lease time 12h
    ```

### Hardware Setup

- Quanta server (model unknown, same as unit at `192.168.0.120` or `10.138.36.197`).
- USB-to-Ethernet adapter (`en17`) on macOS 12, set to `192.168.0.1/24` or `10.138.36.1/24`.
- Ethernet cable from BMC port (“IPMI” or “MGMT”) to adapter.
- Server powered off, power supply light on (BMC powered).
- BMC port LEDs: Green (link) and yellow (activity) blinking.

## Troubleshooting Steps

### 1. Fix dnsmasq Conflicts

- **Issue**: “Address already in use” error.
- **Solution**:

  - Check processes:

    ```
    ps aux | grep dnsmasq
    ```

  - Kill conflicts:

    ```
    sudo kill -9 <PID>
    ```

  - Check port 67:

    ```
    sudo lsof -i :67
    ```

  - Restart `dnsmasq`:

    ```
    sudo /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq --no-daemon -C /usr/local/etc/dnsmasq.conf
    ```

  - Clear leases:

    ```
    sudo rm /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    sudo touch /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    sudo chmod 666 /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
    ```

### 2. Monitor DHCP

- Run and monitor:

  ```
  sudo /usr/local/Cellar/dnsmasq/2.91/sbin/dnsmasq --no-daemon -C /usr/local/etc/dnsmasq.conf
  sudo tail -f /usr/local/var/log/dnsmasq.log
  ```

- Success: `DHCPDISCOVER`/`DHCPOFFER`/`DHCPACK` for BMC MAC (e.g., `192.168.0.13 a1:b2:c3:d4:e5:f6`).
- Check leases:

  ```
  cat /usr/local/var/lib/misc/dnsmasq/dnsmasq.leases
  ```

### 3. Scan Subnets Systematically

To locate the BMC, scan subnets starting with commonly used smaller ranges (e.g., `/24`, 256 IPs, ~5–10 seconds) before progressing to less common, larger ranges (e.g., `/16`, 65,536 IPs, ~30–60 minutes). This approach minimizes scan time and resource usage. The BMC at `10.138.36.197` was found by scanning `10.138.0.0/16` after a customer-provided IP (`10.138.32.197`) failed.

- **Step-by-Step Scanning**:

  - **Common Subnets (/24)**: Frequently used for BMCs (e.g., `192.168.0.0/24`, `10.0.0.0/24`, or known subnets like `10.138.32.0/24`).

    - Scan `192.168.0.0/24` (default for your setup):

      ```
      sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.0/24
      ```

    - Scan `10.0.0.0/24` (common for private networks):

      ```
      sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.0.0.1 255.255.255.0
      sudo nmap -sU -p 623 --script=ipmi-version 10.0.0.0/24
      sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
      ```

    - Scan known subnet (e.g., `10.138.32.0/24`, from customer note):

      ```
      sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.32.1 255.255.255.0
      sudo nmap -sU -p 623 --script=ipmi-version 10.138.32.0/24
      sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
      ```

  - **Adjacent Subnets (/24)**: If the known subnet fails, scan nearby subnets (e.g., `10.138.31.0/24`, `10.138.33.0/24`):

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.31.1 255.255.255.0
    sudo nmap -sU -p 623 --script=ipmi-version 10.138.31.0/24
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
    ```

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.33.1 255.255.255.0
    sudo nmap -sU -p 623 --script=ipmi-version 10.138.33.0/24
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
    ```

  - **Larger Subnets (/16)**: If smaller subnets fail, scan broader ranges (e.g., `10.138.0.0/16`, 65,536 IPs, ~30–60 minutes):

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.0.1 255.255.0.0
    sudo nmap -sU -p 623 --script=ipmi-version --min-rate 1000 10.138.0.0/16
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
    ```

  - **Full Range (/8, Last Resort)**: Scan `10.0.0.0/8` (16,777,216 IPs, ~1–2 days) only if necessary:

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.0.0.1 255.0.0.0
    sudo nmap -sU -p 623 --script=ipmi-version --min-rate 1000 10.0.0.0/8
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
    ```

- **How We Found 10.138.36.197**:

  - Customer provided `10.138.32.197` as a potential static IP.
  - Scanned `10.138.32.0/24`:

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.32.1 255.255.255.0
    sudo nmap -sU -p 623 --script=ipmi-version 10.138.32.0/24
    ```

    - Result: Only `10.138.32.1` (`en17`, `623/udp closed`), no BMC.

  - Expanded to `10.138.0.0/16`:

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.0.1 255.255.0.0
    sudo nmap -sU -p 623 --script=ipmi-version --min-rate 1000 10.138.0.0/16
    ```

    - Result: Found BMC at `10.138.36.197` (`623/udp open|filtered`, MAC `f7:e6:d5:c4:b3:a2`, Quanta).
    - Likely cause: Customer mistyped `36` as `32` or BMC was reconfigured.

  - Initial `ping` failed due to `en17` on `192.168.0.1/24` (subnet mismatch).
  - Set `en17` to `10.138.36.1/24`:

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.36.1 255.255.255.0
    ping 10.138.36.197
    ```

    - Result: Ping succeeded, confirming connectivity.

- **Notes**:
  - Always set `en17` to a compatible subnet (e.g., `10.138.36.1/24` for `10.138.36.197`).
  - Use `--min-rate 1000` for larger scans to reduce time, but test on a `/24` first to avoid packet loss.
  - If no BMC is found, proceed to CMOS reset.

### 4. Test BMC Access

- **Static IP (e.g., `10.138.36.197`)**:

  ```
  ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword sdr list
  ```

  - Try alternative credentials:

    ```
    ipmitool -I lanplus -H 10.138.36.197 -U ADMIN -P ADMIN sdr list
    ipmitool -I lanplus -H 10.138.36.197 -U admin -P admin sdr list
    ipmitool -I lanplus -H 10.138.36.197 -U root -P root sdr list
    ipmitool -I lan -H 10.138.36.197 -U admin -P adminpassword sdr list
    ```

  - Check web GUI:

    ```
    open http://10.138.36.197
    open https://10.138.36.197
    ```

- **DHCP IP (e.g., `192.168.0.13`)**:

  ```
  ipmitool -I lanplus -H 192.168.0.13 -U admin -P adminpassword sdr list
  ```

- **If Credentials Fail**:

  - Try password reset:

    ```
    ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword user list
    ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword user set password 2 adminpassword
    ```

  - Perform CMOS reset (see below).

### 5. Configure BMC to DHCP

- Check LAN settings (if BMC accessible):

  ```
  ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword lan print 1
  ```

- Set DHCP:

  ```
  ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword lan set 1 ipsrc dhcp
  ```

- Reset BMC:

  ```
  ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword mc reset cold
  ```

- Monitor DHCP for new IP (e.g., `192.168.0.13`).

### 6. CMOS/BMC Reset (if DHCP or credentials fail)

- Unplug power cables.
- Locate jumper (deep switches) and reset BIOS, BMC settings.
- Reconnect power, wait 3–5 minutes.
- Monitor DHCP and scan:

  ```
  sudo tail -f /usr/local/var/log/dnsmasq.log
  sudo nmap -sU -p 623 --script=ipmi-version 192.168.0.0/24
  ```

### 7. Handle No BMC Response or Ping/Credential Issues

- **Possible Causes**:

  - Static IP outside `192.168.0.0/24` (e.g., `10.138.36.197`).
  - Subnet mismatch (e.g., `en17` on `192.168.0.1/24` can’t reach `10.138.36.197` unless set to `10.138.36.1/24`).
  - BMC ignores ICMP (earlier `ping` failure).
  - Incorrect credentials (custom or corrupted).
  - BMC faulty or in partial state.

- **Next Steps**:

  - Verify BMC port and LEDs (green/yellow).
  - Test cable/adapter with another device.
  - Ensure `en17` in correct subnet:

    ```
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 10.138.36.1 255.255.255.0
    sudo nmap -sU -p 623 --script=ipmi-version 10.138.36.197
    ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword sdr list
    sudo networksetup -setmanual "USB 10/100/1000 LAN 13" 192.168.0.1 255.255.255.0
    ```

  - Check firewall:

    ```
    sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
    sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate off
    ```

  - Contact Quanta support with model if issues persist.

## Key Learnings

1. **Static vs. DHCP**:

   - Some Quanta BMCs default to static IPs (e.g., `192.168.0.120`, `10.138.36.197`); others use DHCP.
   - Use `ipmitool -I lanplus -H 10.138.36.197 -U admin -P adminpassword lan set 1 ipsrc dhcp` to switch to DHCP.

2. **Credentials**:

   - `admin/adminpassword` worked for one unit; others may have custom passwords or require reset.
   - Common: `ADMIN/ADMIN`, `admin/admin`, `root/root`, `admin/Quanta123`.

3. **dnsmasq Setup**:

   - Configure DHCP pool in `/usr/local/etc/dnsmasq.conf` for `192.168.0.10–192.168.0.20`.
   - Resolve conflicts (`ps aux`, `lsof -i :67`) and monitor logs.

4. **nmap Scanning**:

   - Use `-sU -p 623 --script=ipmi-version` for IPMI detection.
   - `en17` must be in a compatible subnet (e.g., `10.138.36.1/24` for `10.138.36.197`).
   - Scan common `/24` subnets first, then larger `/16` or `/8` if needed.
   - BMC may ignore ICMP, causing `ping` to fail unless `en17` is in the correct subnet.

5. **Physical Setup**:

   - Identify `en17` via `networksetup -listallnetworkservices` and `ifconfig`.
   - Verify BMC port (“IPMI”/“MGMT”), LEDs, `en17` (`192.168.0.1/24` or `10.138.36.1/24`).

6. **Tool Troubleshooting**:

   - Handle Homebrew, `ipmitool`, `nmap`, and `dnsmasq` installation issues with manual fixes.

7. **Success Factors**:
   - BMC detected at `192.168.0.120` (static), switched to `192.168.0.13` (DHCP).
   - BMC at `10.138.36.197` (static, expected `10.138.32.197`), found via `10.138.0.0/16` scan, pingable with `en17` on `10.138.36.1/24`, but credentials failed.

## Future Steps

1. Verify setup: BMC port, LEDs, `en17` (`ifconfig en17`).
2. Try alternative credentials and web GUI.
3. Perform CMOS reset to restore default credentials.
4. Switch to DHCP if accessible.
5. Identify model (`dmidecode -t system`) for Quanta support if needed.

## Notes

- **Success**: BMC switched from static (`192.168.0.120`) to DHCP (`192.168.0.13`) with `admin/adminpassword`.
- **Current Case**: BMC at `10.138.36.197` (static, expected `10.138.32.197`), found via `10.138.0.0/16` scan, pingable after setting `en17` to `10.138.36.1/24`, but credentials failed.
- **No BMC Access**: If credentials fail, try CMOS reset or contact Quanta support.
