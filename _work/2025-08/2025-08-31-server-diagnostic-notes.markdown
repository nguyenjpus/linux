---
layout: post
title: Server Diagnostic Notes And Some Last Hopes Cases
date: 2025-08-31
tags: [Ipmitool, Dmidecode, Troubleshooting, Ipnmi]
---

This guide summarizes key procedures and tools for diagnosing server hardware issues, based on practical experience. It’s designed to clarify troubleshooting steps and avoid common pitfalls.

## 1. Check the POST Screen

- **Why**: The Power-On Self-Test (POST) screen (e.g., American Megatrends BIOS) appears before the GRUB bootloader and shows critical system info (e.g., error codes, hardware status).
- **Action**: Always review for messages about CPU, memory, or disk issues. Never skip, as it’s the first diagnostic clue.
- **Tip**: Write down any error codes or warnings for reference.

## 2. Core Diagnostic Tools

### ipmitool

- **Purpose**: Communicates with the Baseboard Management Controller (BMC) for hardware health, sensor data, logs, and power controls. Works even if the OS is down.
- **Why Essential**: Ideal for remote or on-site diagnostics, covering hardware and firmware issues.
- **Command Example**:
  ```bash
  ipmitool sel list -v
  ```
  - Search for keywords: `failure`, `ierr`, `frb2`, `thermal trip`, `heat`.
- **Tip**: Use when the system is unresponsive but the BMC is powered.

### dmidecode

- **Purpose**: Retrieves static hardware details (e.g., serial numbers, manufacturer, specs).
- **Command Examples**:
  - Memory details:
    ```bash
    sudo dmidecode -t memory | egrep -i "Locator|Serial Number|Part Number|Size"
    ```
  - Processor details:
    ```bash
    sudo dmidecode -t processor | egrep -i "Socket Designation|Version|Serial Number|ID"
    ```
- **Tip**: Use for inventory or to verify hardware during troubleshooting.

## 3. Troubleshooting Steps

- **When to Use**: Apply these when initial diagnostics (POST, `ipmitool`, `dmidecode`) fail.
- **Steps**:
  - **Clear CMOS**: Remove the CMOS battery for 5 minutes to reset BIOS settings.
  - **Reseat Components**: Remove and reinsert DIMMs (memory) and CPU to fix connection issues.
- **Note**: These are last-resort steps. Build confidence through experience fixing units.

## 4. BMC Firmware Upgrade Challenges

- **Issue Example**: A BMC firmware update failed with “Command not supported in present state.”
- **Resolution Steps**:
  - Perform cold resets and check standby power to rule out other issues.
  - Use vendor-specific tools (e.g., `ipnmi`) with original firmware for recovery.
  - If needed, downgrade firmware, then re-update.
  - Clear CMOS (5-minute battery removal) or replace the BMC chip as a last resort.
- **Key Lesson**: Follow vendor-specific procedures. Persistence and methodical testing are critical.

## 5. PSU Replacement Safety

- **Step**: Always run a High Potential (Hipot) test before replacing a Power Supply Unit (PSU).
- **Why**: Ensures electrical safety by checking for faults.
- **Tip**: Never skip this test, even if pressed for time.
