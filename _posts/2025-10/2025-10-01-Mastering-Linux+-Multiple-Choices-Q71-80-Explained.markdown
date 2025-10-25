---
layout: post
title: "Multiple-choices - Questions 71-80 Explained"
date: 2025-10-01 11:15:00 -0700
tags: [Linux+, Multiple-choices]
---

This document explains questions 71-80 from a set of 100 scenario-based multiple-choices questions for CompTIA Linux+ preparation, focusing on virtualization, KVM, VirtualBox, and disk management. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention. Questions 71 and 78 have been updated to clarify that UTM and VMware Workstation are Type 2 hypervisors and to explain NAT security and bridged networking’s relation to UTM setups.

## Question 71: Type of Hypervisor for VirtualBox

**Question**: A developer wants to run a Linux virtual machine on their Windows desktop for testing purposes. They install software like VirtualBox, which runs as an application on top of their existing Windows OS. What type of hypervisor is VirtualBox?  
**Options**:

- Type 1 (Bare-metal)
- Type 2 (Hosted)
- OS-level Virtualization
- Paravirtualization

**Correct Answer**: Type 2 (Hosted)  
**Why Correct**: VirtualBox is a Type 2 (hosted) hypervisor, running as an application on top of a host OS (e.g., Windows or macOS). Similarly, UTM on macOS and VMware Workstation on Windows are also Type 2 hypervisors, as they rely on the host OS to manage hardware resources while running guest VMs.  
**Why Others Wrong**:

- Type 1 (bare-metal) hypervisors (e.g., KVM, ESXi) run directly on hardware without a host OS.
- OS-level virtualization (e.g., containers like Docker) shares the host kernel, not a hypervisor.
- Paravirtualization is a virtualization technique requiring guest OS modifications, not a hypervisor type.  
  **Key Concept**: Type 2 hypervisors depend on a host OS (e.g., Windows, macOS); Type 1 runs on bare metal.  
  **Memory Aid**: “VirtualBox, UTM, VMware Workstation = Type 2, hosted on OS.”

## Question 72: Listing Running VMs with virsh

**Question**: An administrator is using the `virsh` command-line tool to manage KVM virtual machines. To see a list of only the currently running VMs, which command should be used?  
**Options**:

- virsh list --all
- virsh list
- virsh status --running
- virsh net-list

**Correct Answer**: virsh list  
**Why Correct**: `virsh list` displays only currently running VMs by default, showing their ID, name, and state.  
**Why Others Wrong**:

- `virsh list --all` shows all VMs, including shut-down ones.
- `virsh status --running` is invalid; no such option exists.
- `virsh net-list` lists virtual networks, not VMs.  
  **Key Concept**: `virsh list` is for running VMs; `--all` includes all states.  
  **Memory Aid**: “virsh list = Running VMs only.”

## Question 73: Disk Image Format for KVM

**Question**: A new virtual machine needs a virtual disk. The administrator wants to use a disk image format that supports advanced features like snapshots, compression, and copy-on-write. Which disk image format is the native choice for QEMU/KVM?  
**Options**:

- VMDK
- VHD
- RAW
- QCOW2

**Correct Answer**: QCOW2  
**Why Correct**: QCOW2 (QEMU Copy-On-Write version 2) is the native disk image format for QEMU/KVM, supporting snapshots, compression, and copy-on-write for efficient storage.  
**Why Others Wrong**:

- VMDK is VMware’s format, not native to KVM.
- VHD is Microsoft’s format, less optimized for KVM.
- RAW offers no snapshots or compression, just raw disk data.  
  **Key Concept**: QCOW2 is feature-rich for KVM; RAW is simpler but limited.  
  **Memory Aid**: “QCOW2 = QEMU’s advanced disk format.”

## Question 74: Creating a QCOW2 Disk Image

**Question**: An administrator needs to create a new 20GB virtual disk image named `dev-vm.qcow2` for a new KVM virtual machine. Which command correctly creates this file?  
**Options**:

- dd if=/dev/zero of=dev-vm.qcow2 bs=1G count=20
- qemu-img create -f qcow2 dev-vm.qcow2 20G
- truncate -s 20G dev-vm.qcow2
- mkfs.qcow2 dev-vm.qcow2 20G

**Correct Answer**: qemu-img create -f qcow2 dev-vm.qcow2 20G  
**Why Correct**: `qemu-img create -f qcow2 dev-vm.qcow2 20G` creates a 20GB QCOW2 disk image, optimized for KVM with sparse allocation (only uses space as needed).  
**Why Others Wrong**:

- `dd` creates a raw image, not QCOW2, and fully allocates 20GB.
- `truncate -s 20G` creates a sparse file but not a QCOW2 image.
- `mkfs.qcow2` is invalid; `mkfs` formats filesystems, not disk images.  
  **Key Concept**: `qemu-img` is the tool for creating VM disk images; `-f qcow2` specifies the format.  
  **Memory Aid**: “qemu-img = Create QCOW2 for VMs.”

## Question 75: Starting a Shut-Down VM

**Question**: A virtual machine named `test-server` is currently shut down. Which `virsh` command will start this VM?  
**Options**:

- virsh resume test-server
- virsh create test-server.xml
- virsh start test-server
- virsh reboot test-server

**Correct Answer**: virsh start test-server  
**Why Correct**: `virsh start test-server` boots a shut-down VM named `test-server`.  
**Why Others Wrong**:

- `virsh resume` resumes a paused VM, not a shut-down one.
- `virsh create test-server.xml` defines a new VM from an XML file, not starts it.
- `virsh reboot` restarts a running VM, not a shut-down one.  
  **Key Concept**: `virsh start` for shut-down VMs; `resume` for paused.  
  **Memory Aid**: “virsh start = Boot shut-down VM.”

## Question 76: Graceful Shutdown of a VM

**Question**: An administrator needs to perform a graceful shutdown of a running VM named `web-app-vm` from the hypervisor’s command line. Which `virsh` command sends an ACPI shutdown signal to the guest OS?  
**Options**:

- virsh destroy web-app-vm
- virsh shutdown web-app-vm
- virsh stop web-app-vm
- virsh undefine web-app-vm

**Correct Answer**: virsh shutdown web-app-vm  
**Why Correct**: `virsh shutdown web-app-vm` sends an ACPI shutdown signal to the guest OS, allowing a graceful shutdown.  
**Why Others Wrong**:

- `virsh destroy` forcefully terminates the VM, risking data loss.
- `virsh stop` is not a valid command.
- `virsh undefine` removes the VM’s configuration, not shuts it down.  
  **Key Concept**: `shutdown` is graceful; `destroy` is forceful.  
  **Memory Aid**: “virsh shutdown = Graceful guest off.”

## Question 77: Benefit of Snapshots

**Question**: Before making a major change to a virtual machine, an administrator wants to create a snapshot of its current state so they can revert if something goes wrong. What is the primary benefit of using snapshots?  
**Options**:

- They increase the performance of the VM.
- They provide a point-in-time state of the VM’s disk and memory that can be restored later.
- They create a full, independent backup of the VM’s disk image.
- They encrypt the virtual machine’s disk.

**Correct Answer**: They provide a point-in-time state of the VM’s disk and memory that can be restored later.  
**Why Correct**: Snapshots capture the VM’s disk and memory state at a specific moment, allowing reversion to that state if needed.  
**Why Others Wrong**:

- Snapshots don’t improve performance; they may slightly degrade it.
- Snapshots are not full backups; they depend on the base image.
- Snapshots don’t encrypt disks; encryption is separate.  
  **Key Concept**: Snapshots are lightweight, using copy-on-write; full backups are independent.  
  **Memory Aid**: “Snapshot = Point-in-time rollback.”

## Question 78: Bridged Network Adapter

**Question**: A virtual machine is configured with a “bridged” network adapter. How will this VM appear on the physical network?  
**Options**:

- It will be hidden behind the host’s IP address using NAT.
- It will appear as a separate device on the network, with its own MAC and IP address.
- It will only be able to communicate with other VMs on the same host.
- It will not have any network connectivity.

**Correct Answer**: It will appear as a separate device on the network, with its own MAC and IP address.  
**Why Correct**: A bridged network adapter connects the VM directly to the physical network, assigning it a unique MAC and IP address, making it appear as a separate device (e.g., as if it’s another computer on the LAN). This is similar to the bridge setup in UTM on your Mac, where the virtual Ubuntu VM was configured to use a bridged network (e.g., via a virtual bridge like `bridge100`) to communicate directly with your LAN, obtaining its own IP from the router’s DHCP.  
**Why Others Wrong**:

- NAT (Network Address Translation) hides the VM behind the host’s IP address by translating the VM’s traffic through the host’s network interface. This provides some security by isolating the VM from direct external access, reducing exposure to attacks, but it limits direct network visibility. Unlike bridged networking, NAT doesn’t give the VM its own network identity.
- Host-only networking restricts communication to the host and other VMs, not the physical network.
- Bridged VMs have full network connectivity, not none.  
  **Key Concept**: Bridged networking gives VMs direct network access with unique IPs; NAT shares the host’s IP for security and isolation. The bridge in UTM (e.g., `bridge100`) mirrors this by linking the VM to the physical network, as seen in your Ubuntu setup.  
  **Memory Aid**: “Bridged = VM acts like a real device; NAT = Hidden behind host.”

## Question 79: Removing a VM Completely

**Question**: An administrator needs to completely remove a virtual machine named `old-vm`, including its configuration file, from the KVM hypervisor. Which `virsh` command should be used after the VM is shut down?  
**Options**:

- virsh destroy old-vm
- virsh delete old-vm
- virsh undefine old-vm
- virsh remove old-vm

**Correct Answer**: virsh undefine old-vm  
**Why Correct**: `virsh undefine old-vm` removes the VM’s configuration from KVM after it’s shut down, deleting its definition (but not disk images, which must be removed separately).  
**Why Others Wrong**:

- `virsh destroy` forcefully stops a running VM, not removes its config.
- `virsh delete` is not a valid command.
- `virsh remove` is not a valid command; `undefine` is correct.  
  **Key Concept**: `undefine` removes VM config; disk images need manual deletion.  
  **Memory Aid**: “virsh undefine = Erase VM config.”

## Question 80: Using a Golden Image

**Question**: A team needs to deploy dozens of identical development VMs. To save disk space, they want to use a single “golden image” as a read-only base and have each new VM write its changes to a separate, smaller disk image. This technique is known as:  
**Options**:

- Full cloning
- Live migration
- Linked cloning or using backing files
- RAID 1 virtualization

**Correct Answer**: Linked cloning or using backing files  
**Why Correct**: Linked cloning (or backing files in QCOW2) uses a read-only golden image as a base, with each VM writing changes to a separate copy-on-write image, saving disk space.  
**Why Others Wrong**:

- Full cloning creates independent copies, using more space.
- Live migration moves running VMs, not related to cloning.
- RAID 1 is for disk redundancy, not virtualization.  
  **Key Concept**: Linked cloning uses a base image with delta disks; QCOW2 supports backing files.  
  **Memory Aid**: “Linked cloning = Shared golden image.”

## Retention Tips for Questions 71-80

- **Themes**: Virtualization (VirtualBox, UTM, VMware, KVM), `virsh` commands, disk images (QCOW2), snapshots, networking (bridged, NAT), and efficient VM deployment (linked cloning).
- **Mnemonic for Virtualization**: “VirtualBox, UTM, VMware hosted; KVM bare; QCOW2 for disks” (V-U-K-Q).
- **Practice**: In a Linux VM with KVM:
  - Create a QCOW2 image with `qemu-img`, start/stop VMs with `virsh start/shutdown`.
  - List running VMs with `virsh list`, create a snapshot, and test bridged vs. NAT networking in UTM or VirtualBox.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “Command to start a VM?” → “virsh start.” “NAT vs. bridged?” → “NAT hides; bridged exposes.”
- **Quiz Yourself**: Are UTM and VMware Workstation Type 2? Why does NAT hide the VM? How does bridged networking work in UTM?
