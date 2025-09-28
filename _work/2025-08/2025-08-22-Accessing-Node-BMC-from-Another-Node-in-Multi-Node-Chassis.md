---
layout: post
title: Accessing a Node's BMC from Another Node in a Multi-Node Chassis
date: 2025-08-22
tags: [Node, Ipmitool, BMC, Troubleshooting]
---

In multi-node systems, such as blade servers or rackmount chassis with multiple nodes, you can use one node's Baseboard Management Controller (BMC) to issue `ipmitool` commands to another node's BMC, even if the target node's operating system fails to boot. This is particularly useful for troubleshooting, accessing the BMC's GUI, or performing management tasks when a node is down, as long as its BMC remains operational. This post explains how to use the `ipmitool` command with the `-t` option to target a specific node's BMC via the Intelligent Platform Management Bus (IPMB) within the same chassis.

## Prerequisites

- **Chassis Support**: The chassis must support IPMI over IPMB, allowing communication between nodes via a shared management bus. Check your hardware documentation to confirm compatibility.
- **Operational BMC**: The target node's BMC must be powered and functional.
- **ipmitool Installed**: The issuing node must have `ipmitool` installed and configured.
- **Node Addresses**: You need the IPMB addresses for the nodes in the chassis. For this example, the addresses are:
  - Node 1: `0x82`
  - Node 2: `0x84`
  - Node 3: `0x86`
  - Node 4: `0x88`

## Steps to Access a Node's BMC

### 1. Verify Node Addresses

Each node in the chassis has a unique IPMB address. To confirm the addresses for your system, run the following command from a functional node:

```bash
ipmitool sdr list
```

This lists all nodes and their IPMB addresses in the chassis. For this example, we assume the addresses listed above (e.g., Node 1 at `0x82`).

### 2. Run Commands on the Target Node

Use the `-t` option with `ipmitool` to target the BMC of the desired node. For example, to retrieve the LAN configuration (e.g., IP address) of Node 1 (IPMB address `0x82`) from another node (e.g., Node 2):

```bash
sudo ipmitool -t 0x82 lan print
```

This command returns the LAN settings for Node 1’s BMC, which can be used to access its GUI remotely, even if Node 1’s operating system is not booting.

### 3. Example: Run `hpm check` on Node 1

To perform an HPM (Hardware Platform Management) check on Node 1’s BMC from another node:

```bash
sudo ipmitool -t 0x82 hpm check
```

This executes the `hpm check` command on Node 1’s BMC, allowing you to verify its firmware or hardware status.

### 4. Verify BMC Status

To ensure the target node’s BMC is operational before issuing commands, check its status:

```bash
ipmitool -t 0x82 mc info
```

This displays information about Node 1’s BMC, confirming it is responsive.

## Additional Examples

- **Check Sensor Data**: To view sensor data (e.g., temperature, fan speed) for Node 1:
  ```bash
  sudo ipmitool -t 0x82 sdr
  ```
- **Power Cycle Node**: To power cycle Node 1:
  ```bash
  sudo ipmitool -t 0x82 power cycle
  ```

## Notes

- **Authentication**: If the target node’s BMC requires authentication, include credentials with the `-t` option:
  ```bash
  sudo ipmitool -t 0x82 -U admin -P <password> lan print
  ```
  Replace `<password>` with the BMC’s admin password.
- **Permissions**: The issuing node’s BMC must have sufficient privileges to communicate with the target node’s BMC. If you encounter an `Insufficient privilege level` error, ensure the user account on the issuing node has `ADMINISTRATOR` privileges and IPMI messaging enabled.
- **Chassis-Specific Addresses**: IPMB addresses (`0x82`, `0x84`, etc.) are specific to your system. Always verify addresses using `ipmitool sdr list` or consult your chassis documentation.
- **Security**: Restrict IPMB access to trusted nodes or users, as BMC communication can expose sensitive system controls. Use strong passwords for BMC accounts.
- **Limitations**: This method requires a functional chassis management module or IPMB bus. If the target node’s BMC is offline or the chassis does not support IPMI over IPMB, this approach will not work.

## Troubleshooting

- **Command Fails**: If commands fail with errors like `Unable to establish IPMI v2 / RMCP+ session`, verify the target BMC is powered and the IPMB address is correct.
- **No Response**: If the target node’s BMC does not respond, check its power state with `ipmitool -t <address> chassis status`.
- **Permission Errors**: If you encounter `0xd4 Insufficient privilege level`, configure the issuing node’s BMC user privileges:
  ```bash
  ipmitool user enable <user_id>
  ipmitool channel setaccess 1 <user_id> link=on ipmi=on callin=on privilege=4
  ```

This approach enables system administrators to manage and troubleshoot nodes in a multi-node chassis efficiently, even when a node’s operating system is down, by leveraging the power of IPMI and the `ipmitool` command.
