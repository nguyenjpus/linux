---
layout: post
title: Junior System Administrator at TechCorp
date: 2025-08-21
---

# Troubleshooting IPMI Insufficient Privilege Level Error

## Issue

When attempting to run the `ipmitool hpm check` command over the LAN interface, the following error occurred:

```
Get HPM.x Capabilities request failed, compcode = d4
Get Device ID command failed: 0xd4 Insufficient privilege level
```

The `ipmitool user list 1` output showed that the `admin` user (ID 2) had `ADMINISTRATOR` privilege but `IPMI Msg` was disabled (`false`), indicating insufficient permissions for IPMI commands on the LAN channel.

```
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    false   false      false      NO ACCESS
2   admin            false   true       false      ADMINISTRATOR
```

## Root Cause

The error (`0xd4 Insufficient privilege level`) was due to the `admin` user lacking enabled IPMI messaging and proper channel access settings for the LAN interface (likely channel 1 or 14, as indicated by `ipmitool sol info 1` showing `Payload Channel: 14`).

## Quick Fix

To resolve the issue, the following commands were used to enable the `admin` user and configure channel access:

```bash
ipmitool user enable 2
ipmitool channel setaccess 1 2 link=on ipmi=on callin=on privilege=4
```

- `user enable 2`: Activates the `admin` user (ID 2).
- `channel setaccess`: Enables link authentication, IPMI messaging, callin access, and sets the privilege level to `ADMINISTRATOR` (4) on channel 1.

After applying these commands, the `hpm check` command should execute successfully:

```bash
ipmitool -I lanplus -H 10.230.1.55 -U admin -P admin hpm check
```

## Bonus: Verify Channel Access

To confirm the channel access settings for the `admin` user on channel 1:

```bash
ipmitool channel getaccess 1 2
```

This displays the current settings, ensuring `link=on`, `ipmi=on`, and `privilege=4`.

## Bonus 2: Reset Admin Password

If the `admin` user’s password needs to be reset (e.g., due to being unknown or for security reasons), use the following commands:

```bash
ipmitool user disable 2
ipmitool user set name 2 admin
ipmitool user set password 2 <new_password>
ipmitool user enable 2
```

- `user disable 2`: Temporarily disables the `admin` user.
- `user set name 2 admin`: Confirms or sets the username to `admin`.
- `user set password 2 <new_password>`: Sets a new password (replace `<new_password>` with a secure password).
- `user enable 2`: Re-enables the `admin` user.

## Diagnostic Step: Verify LAN Interface

If the `hpm check` command still fails after applying the quick fix, the LAN interface for channel 1 may be disabled. Check the LAN configuration:

```bash
ipmitool lan print 1
```

If `Access Mode` is `disabled` or `pre-boot only`, enable the LAN interface:

```bash
ipmitool lan set 1 access on
```

Verify the change with `ipmitool lan print 1` and retry the `hpm check` command. If channel 1 is not the LAN channel, check channel 14 (based on `Payload Channel: 14` from `sol info`):

```bash
ipmitool lan print 14
ipmitool lan set 14 access on
```

## Additional Notes

- **Channel Verification**: The `ipmitool sol info 1` output indicated the LAN payload channel might be 14 (`Payload Channel: 14`). If the fix doesn’t work on channel 1, try configuring channel 14:
  ```bash
  ipmitool channel setaccess 14 2 link=on ipmi=on callin=on privilege=4
  ipmitool channel getaccess 14 2
  ```
- **Security**: Use a strong password for the `admin` user, as IPMI over LAN is accessible remotely. Restrict access to trusted IP addresses if possible.
- **Testing**: After applying fixes or resetting the password, verify with the `hpm check` command using the updated credentials.

This procedure resolved the privilege error and provided methods to manage the `admin` user’s password and verify LAN interface settings, ensuring full IPMI functionality.
