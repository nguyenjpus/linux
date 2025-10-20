---
layout: post
title: Debugging a PCIe Error on My Server
date: 2025-10-17
tags: [Troubleshooting, dmesg, PCIe, networking]
---

I hit a weird PCIe Receiver Error on my server and spent some time tracking it down. Here's a quick summary of what I learned, how I figured out what’s in Slot 13, and what might’ve caused the issue. Hope this helps anyone debugging similar problems!

## What Went Wrong

- **Server**: 2U rackmount server with a single Intel Xeon Silver 4310 CPU, 64GB DDR4 in two slots, and an older BIOS (version from 07/05/2022).
- **Issue**: Got a PCIe error in `dmesg` on October 17, 2025, at 08:49:23:
  ```
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 0
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: event severity: corrected
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: section_type: PCIe error
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: device_id: 0000:c2:02.0
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: slot: 13
  [Fri Oct 17 08:49:23 2025] {1}[Hardware Error]: secondary_bus: 0xc3
  ```
  - Also saw some Correctable ECC memory errors in earlier logs, but they didn’t seem directly related.
- **Goal**: Find out what hardware is in Slot 13, why the error popped up, and if bad SFP cables were involved.

## What I Figured Out

1. **The PCIe Error**:

   - It’s a “corrected” PCIe Receiver Error at root port `0000:c2:02.0` (Intel PCIe controller) in Slot 13, affecting devices on bus `c3`.
   - Happened around the same time as link issues with network interfaces `ens13f0` and `ens13f2` (08:49:49), hinting at a network card problem.

2. **What’s in Slot 13?**:

   - Ran `lspci -tv`:
     ```
     +-[0000:c2]-+-02.0-[c3]--+-00.0  Intel Corporation Ethernet Controller X710 for 10GbE SFP+
      |           |            +-00.1  Intel Corporation Ethernet Controller X710 for 10GbE SFP+
      |           |            +-00.2  Intel Corporation Ethernet Controller X710 for 10GbE SFP+
      |           |            +-00.3  Intel Corporation Ethernet Controller X710 for 10GbE SFP+
     ```
     - Shows an **Intel X710 10GbE SFP+ NIC** (4 ports) on bus `c3`, tied to root port `c2:02.0`. The error log says “slot: 13” for `c2:02.0`, so the X710 NIC is in Slot 13.
   - Tried `dmidecode -t 9` to check slots, but got nothing—my BIOS doesn’t list PCIe slots.

3. **How Bus `c3` Ties to `i40e`**:

   - dmesg showed the `i40e` driver handling devices on bus `c3`:
     ```
     [Fri Oct 17 08:05:24 2025] i40e 0000:c3:00.0 ens13f0: NIC Link is Up
     [Fri Oct 17 08:49:49 2025] i40e 0000:c3:00.0 ens13f0: NIC Link is Down
     ```
     - `i40e` is Intel’s driver for X710 NICs, managing `c3:00.0–c3:00.3`, which are the interfaces `ens13f0–ens13f3`.
     - So, bus `c3` = X710 NIC = `i40e` driver.

4. **SFP Cable Trouble**:

   - The NIC’s LED wasn’t lighting up, so I replaced the faulty SFP loopback cables. This fixed the link issue but didn’t directly cause the PCIe error.
   - The bad cables probably made the NIC flaky, stressing the PCIe link and causing the error.

5. **ECC Errors**:
   - Earlier logs had system issues (POST failures, resets), but no clear ECC details.
   - Could be bad RAM or power/heat problems, not directly tied to the PCIe issue.

## What I Did

- **Checked Logs**: Looked at `dmesg` for the PCIe error and `i40e` activity, and System Event Log (`ipmitool sel elist`) for other issues.
- **Mapped PCIe**: Used `lspci -tv` to confirm the X710 NIC on bus `c3` in Slot 13.
- **Tried `dmidecode`**: No dice with `dmidecode -t 9` because of BIOS limits.
- **Checked NIC**: Noticed `ens13f0–ens13f3` missing in `ip link show`—they were likely down. Suggested checking `ip link show all` and `ethtool -i ens13f0`.
- **Swapped SFP Cables**: Fixed the NIC’s link problem.
- **Planned NIC Replacement**: Decided to swap the X710 NIC to rule out a bad card.

## Lessons Learned

1. **PCIe Debugging**:

   - Use `lspci -tv` to see how PCIe devices connect. The `secondary_bus` in `dmesg` (e.g., `0xc3`) tells you what’s affected.
   - Example: `c2:02.0-[c3]` means bus `c3` devices are in Slot 13.

2. **Linking Drivers**:

   - dmesg logs like `i40e 0000:c3:00.0` show which driver handles a PCIe bus. Check `ethtool -i <interface>` for `bus-info` to match it up.

3. **BIOS Quirks**:

   - Some BIOSes skip PCIe slot info in `dmidecode`. Stick to `lspci` and `dmesg`.

4. **SFP and NIC Issues**:

   - Faulty SFP cables can mess with NICs and cause PCIe errors. Use `ethtool -m <interface>` to check cables.

5. **ECC Errors**:

   - Check `dmesg | grep EDAC` or run Memtest86+ to test RAM.

6. **Keep an Eye Out**:
   - Monitor errors with `dmesg -w | grep pcie` and `lspci -s <device> -vvv | grep AER`.
   - Test NICs with `iperf3 -c localhost`.

## What’s Next

- **After Replacing the NIC**:
  - Check the new NIC:
    ```bash
    lspci -tv
    ip link show all
    ethtool -i <interface>
    ```
  - Watch for errors:
    ```bash
    dmesg -w | grep pcie
    lspci -s c2:02.0 -vvv | grep AER
    ```
  - Check SFP cables:
    ```bash
    ethtool -m <interface>
    ```
- **ECC Checks**:
  - Run `dmesg | grep EDAC` or Memtest86+ for RAM issues.
  - Check temps: `ipmitool sensor | grep temp`.
- **Updates**:
  - Update NIC firmware (from Intel’s site) and BIOS (check support site).
  - Try turning off PCIe ASPM:
    ```bash
    echo "pcie_aspm=off" >> /etc/default/grub
    grub2-mkconfig -o /boot/grub2/grub.cfg
    reboot
    ```

## Wrap-Up

The PCIe error was linked to an Intel X710 NIC in Slot 13, on bus `c3`, using the `i40e` driver. Bad SFP cables likely caused NIC issues, triggering the error. I’m swapping the NIC to confirm it’s fixed. This taught me to use `lspci`, `dmesg`, and `ethtool` to debug PCIe issues and watch for BIOS and ECC quirks.
