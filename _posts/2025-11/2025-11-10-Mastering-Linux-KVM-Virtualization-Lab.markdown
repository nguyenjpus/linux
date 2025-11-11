---
layout: post
title: "Linux Lab Summary: Mastering Linux KVM Virtualization Lab"
date: 2025-11-10 19:11:00 -0700
tags: [Linux+, Linuxlab, 100 Multiple Choices, KVM, QEMU, Virsh, Qcow2, Virtio]
---

This lab provides a hands-on guide to KVM virtualization, focusing on VM management, snapshots, cloning, disk optimization, and networking. It's designed for beginners to data center admins, with step-by-step commands, purposes, expected outputs, and real-world applications. All steps are based on our session—tested on dual-boot Ubuntu with WiFi NAT.

---

## 1. Lab Environment

Our setup used a physical dual-boot machine for bare-metal KVM (Type 1 hypervisor). Here's the key config:

| Component       | Details                                                                 | Notes                                                                            |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Host OS**     | Ubuntu 24.04.3 LTS                                                      | Desktop edition for GUI (virt-manager optional).                                 |
| **Kernel**      | `6.8.0-87-generic`                                                      | Verified with `uname -r`.                                                        |
| **Hypervisor**  | KVM + QEMU (libvirt)                                                    | Installed via `sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system`. |
| **CPU**         | Intel i7 (12 cores total, 4 vCPUs per VM)                               | Nested virt enabled in BIOS.                                                     |
| **RAM**         | 31GiB total (4GiB per VM)                                               | Verified with `free -h`.                                                         |
| **Disk**        | 20GB QCOW2/RAW per VM                                                   | On /dev/sda5 (3TB root FS).                                                      |
| **Network**     | `default` NAT (`virbr0`, 192.168.122.0/24)                              | Host WiFi: `wlx347de44fe1c4` (10.0.0.45).                                        |
| **VMs Created** | `test-server` (main), `clone-vm` (linked), `ubuntu24.04` (virt-manager) | All Ubuntu 24.04.3 Server guests.                                                |

> **Real-World Application**: This mirrors cloud providers like AWS EC2 (KVM under the hood) for isolated servers. Use for dev/test/prod environments in data centers.

---

## 2. Hands-On Steps by Section

Each section includes step-by-step commands, purpose, expected outputs, and troubleshooting tips. Run as root or with sudo where noted.

### **Section 1: VM Lifecycle and Management (Questions 72, 75, 76, 79)**

**Purpose**: Learn CLI management with `virsh` for VM states—essential for automation in data centers.

1. **List VMs** (Question 72):

   - Command: `virsh list --all`
   - Purpose: Show running and shut-off VMs.
   - Expected:
     ```
     Id   Name            State
     ------------------------------
     -    test-server     shut off
     -    clone-vm        shut off
     ```
   - Tip: `--all` includes shut-off; plain `virsh list` shows running only.

2. **Start VM** (Question 75):

   - Command: `virsh start test-server`
   - Purpose: Boot VM from shut-off.
   - Expected: "Domain 'test-server' started"
   - Tip: Monitor with `htop` (qemu process uses 4GiB/4 vCPUs).

3. **Shutdown VM** (Question 76):

   - Command: `virsh shutdown test-server`
   - Purpose: Graceful ACPI shutdown (OS clean exit).
   - Expected: "Domain 'test-server' is being shutdown" (wait 1-2 mins).
   - Tip: Use `virsh destroy` for force if hangs.

4. **Remove VM** (Question 79):
   - Command: `virsh undefine test-server --remove-all-storage`
   - Purpose: Delete config + disks (cleanup).
   - Expected: "Domain 'test-server' undefined" + disk removed.
   - Tip: Use after testing—irreversible.

> **Real-World**: Automate with Ansible: `virsh start {{ vm_name }}` in playbooks for scaling.

### **Section 2: Snapshots & Recovery (Question 77)**

**Purpose**: Create point-in-time states for rollback—key for zero-downtime updates.

1. **Create Snapshot**:

   - Command: `virsh snapshot-create-as test-server clean --description "Fresh install"`
   - Purpose: Capture disk/memory state.
   - Expected: "Domain snapshot clean created"
   - Tip: Offline (shutoff VM) for disk-only; running for memory too.

2. **List Snapshots**:

   - Command: `virsh snapshot-list test-server`
   - Expected:
     ```
     Name        Creation Time             State
     ----------------------------------------------
     clean       2025-11-10 03:00:00       shutoff
     ```

3. **Test Revert**:

   - Commands:
     ```
     virsh start test-server
     ssh qemu@192.168.122.178 "touch ~/proof.txt; ls ~"
     virsh shutdown test-server
     virsh snapshot-revert test-server clean
     virsh start test-server
     ssh qemu@192.168.122.178 "ls ~"  # No proof.txt
     ```
   - Purpose: Restore to point-in-time.
   - Expected: File gone—state restored.

4. **Delete Snapshot**:
   - Command: `virsh snapshot-delete test-server clean`
   - Expected: "Domain snapshot clean deleted"

> **Real-World**: Use for database backups or config changes—revert if deploy fails.

### **Section 3: Linked Cloning (Question 80)**

**Purpose**: Create space-efficient copies using backing files—scales dev environments.

1. **Create Overlay Disk**:

   - Command: `qemu-img create -f qcow2 -b /var/lib/libvirt/images/dev-vm.qcow2 -F qcow2 /var/lib/libvirt/images/clone-vm.qcow2`
   - Purpose: New disk references original (changes separate).
   - Expected: "Formatting ... backing_file=dev-vm.qcow2 backing_fmt=qcow2"

2. **Clone XML**:

   - Commands:
     ```
     virsh dumpxml test-server > /tmp/clone.xml
     nano /tmp/clone.xml  # Edit name, disk source, blank MAC
     virsh define /tmp/clone.xml
     virsh start clone-vm
     ```
   - Purpose: Duplicate config with new disk.
   - Expected: "Domain 'clone-vm' defined"

3. **Test Independence**:

   - Commands:
     ```
     virsh domifaddr clone-vm  # New IP, e.g., 192.168.122.179
     ssh qemu@<clone-ip> "touch ~/clone-only.txt"
     virsh start test-server
     ssh qemu@192.168.122.178 "ls ~"  # No clone-only.txt
     ```
   - Purpose: Verify changes isolated.
   - Expected: Original unchanged.

4. **Space Check**:
   - Command: `du -h /var/lib/libvirt/images/*`
   - Expected: clone-vm.qcow2 ~19M (vs 4.1G base).

> **Real-World**: Deploy 100+ identical dev VMs with <1GB total storage.

### **Section 4: Disk Performance (Question 85)**

**Purpose**: Compare formats for I/O—RAW for high-throughput workloads.

1. **Convert to RAW**:

   - Command: `qemu-img convert -f qcow2 -O raw /var/lib/libvirt/images/dev-vm.qcow2 /var/lib/libvirt/images/dev-vm.raw`
   - Purpose: No snapshots, max perf.
   - Expected: Conversion complete (~minutes).

2. **Pre-allocate**:

   - Command: `fallocate -l 20G /var/lib/libvirt/images/dev-vm.raw`
   - Purpose: Avoid runtime allocation.
   - Expected: File size 20G.

3. **Update XML**:

   - Command: `virsh edit test-server`
   - Purpose: Switch disk type.
   - Edit: `<type='qcow2'/>` → `'raw'`, `<source file='...qcow2'/>` → `'...raw'`.

4. **Benchmark**:
   - Commands:
     ```
     virsh start test-server
     ssh qemu@192.168.122.178
     time sudo dd if=/dev/zero of=/tmp/test bs=1M count=2000 conv=fdatasync
     rm /tmp/test
     ```
   - Expected (RAW): ~11.5s, 218 MB/s  
     (Revert to QCOW2, re-run: ~15.5s, 214 MB/s).

> **Real-World**: RAW for SQL servers (e.g., MySQL on VM)—30% faster queries.

### **Section 5: Virtio Optimization (Question 98)**

**Purpose**: Use paravirtualized drivers for near-native perf.

1. **Verify in XML**:

   - Command: `virsh dumpxml test-server | grep virtio`
   - Expected: `bus='virtio'`, `model='virtio'`.

2. **Guest Check**:
   - Commands:
     ```
     ssh qemu@192.168.122.178
     lsblk -o NAME,TRAN  # vda: virtio
     lspci | grep -i virtio  # Virtio network
     ```
   - Purpose: Confirm optimized devices.

> **Real-World**: Virtio = 50% less CPU for VM I/O—essential for cloud scaling.

### **Section 6: Web App & External Access (Question 78)**

**Purpose**: Deploy app, simulate failure, recover—add external access via DNAT.

1. **Install Nginx**:

   - Commands:
     ```
     ssh qemu@192.168.122.178
     sudo apt update && sudo apt install -y nginx
     sudo systemctl enable --now nginx
     curl localhost  # Welcome page
     ```
   - Expected: Nginx running.

2. **Snapshot Before Failure**:

   - Commands:
     ```
     exit
     virsh shutdown test-server
     virsh snapshot-create-as test-server web-ready --description "Nginx working"
     ```

3. **Simulate Failure**:

   - Commands:
     ```
     virsh start test-server
     ssh qemu@192.168.122.178 "sudo rm /var/www/html/index.nginx-debian.html"
     curl 192.168.122.178  # 403 Forbidden
     ```

4. **Revert & Recover**:

   - Commands:
     ```
     virsh shutdown test-server
     virsh snapshot-revert test-server web-ready
     virsh start test-server
     curl 192.168.122.178  # Welcome page
     ```

5. **External Access (DNAT)**:
   - Commands:
     ```
     sudo sysctl net.ipv4.ip_forward=1  # Enable forwarding
     sudo iptables -t nat -A PREROUTING -i wlx347de44fe1c4 -p tcp -d 10.0.0.45 --dport 8080 -j DNAT --to-destination 192.168.122.178:80
     sudo iptables -I FORWARD -p tcp -d 192.168.122.178 --dport 80 -j ACCEPT
     ```
   - Purpose: Mac curl to host:8080 → VM:80.
   - Expected from Mac: `curl http://10.0.0.45:8080` → Nginx page.

> **Real-World**: DNAT for load-balanced web clusters—external access without public IP.

---

## 3. Questions Addressed

| #   | Question           | Covered In | Key Takeaway               |
| --- | ------------------ | ---------- | -------------------------- |
| 72  | `virsh list --all` | Section 1  | All vs running VMs.        |
| 75  | Start VM           | Section 1  | Boot from shut-off.        |
| 76  | Shutdown VM        | Section 1  | Graceful ACPI.             |
| 77  | Snapshots benefit  | Section 2  | Point-in-time restore.     |
| 78  | Bridged vs NAT     | Section 6  | NAT + DNAT for external.   |
| 80  | Linked cloning     | Section 3  | Backing files for savings. |
| 85  | RAW for perf       | Section 4  | No snapshots, max I/O.     |
| 88  | NAT inbound limit  | Section 6  | No unsolicited external.   |
| 98  | Virtio drivers     | Section 5  | Optimized for virt.        |

---

## 4. Custom Files

### `create_clone.sh` (Automation Script)

```bash
#!/bin/bash
# create_clone.sh — Linked clone helper
ORIGINAL=$1
CLONE=$2

virsh shutdown $ORIGINAL
qemu-img create -f qcow2 -b /var/lib/libvirt/images/${ORIGINAL}.qcow2 -F qcow2 /var/lib/libvirt/images/${CLONE}.qcow2
virsh dumpxml $ORIGINAL > /tmp/${CLONE}.xml
sed -i "s|<name>.*</name>|<name>$CLONE</name>|" /tmp/${CLONE}.xml
sed -i "s|${ORIGINAL}.qcow2|${CLONE}.qcow2|" /tmp/${CLONE}.xml
virsh define /tmp/${CLONE}.xml
virsh start $CLONE
echo "Clone $CLONE created!"
```

**Usage**:

```bash
chmod +x create_clone.sh
./create_clone.sh test-server clone-vm
```

> **Tip**: Run as root; edit for custom paths.

---

## 5. Key Insights from Our Session

- **Snapshot Revert**: Instant recovery—`proof.txt` disappeared after revert; perfect for "what if" testing.
- **Linked Clone IP Reuse**: Same IP (192.168.122.178) due to DHCP lease—reboot clone for new if needed.
- **RAW vs QCOW2**: RAW 11.5s (218 MB/s) vs QCOW2 15.5s (214 MB/s)—30% faster, but QCOW2 for flexibility.
- **DNAT Troubleshooting**: libvirt `<portForward>` stripped; manual iptables + `ip_forward=1` + `-i wlx...` worked. External curl succeeded after interface-specific rule.
- **Nginx Binding**: `listen 80;` (all interfaces) vs `default_server` (loopback)—fixed "Connection refused".
- **Libvirt Recovery**: `systemctl restart libvirtd` restores `virbr0`—never flush FORWARD manually.

> **Observation**: WiFi NAT is reliable for labs; Ethernet for prod bridging.

---

## 6. Bookmarks & Explanations

| Command                                                                                                                              | Purpose         | Tied To     | Explanation                                            |
| ------------------------------------------------------------------------------------------------------------------------------------ | --------------- | ----------- | ------------------------------------------------------ |
| `virsh snapshot-create-as test-server clean --description "Fresh install"`                                                           | Create snapshot | Question 77 | Point-in-time state for revert; offline for disk-only. |
| `qemu-img create -f qcow2 -b dev-vm.qcow2 -F qcow2 clone-vm.qcow2`                                                                   | Overlay disk    | Question 80 | Backing file—clone small (19M vs 4.1G base).           |
| `time sudo dd if=/dev/zero of=/tmp/test bs=1M count=2000 conv=fdatasync`                                                             | I/O benchmark   | Question 85 | RAW faster for databases; clean with `rm /tmp/test`.   |
| `lsblk -o NAME,TRAN`                                                                                                                 | Virtio check    | Question 98 | `vda virtio` = optimized; emulated = slower.           |
| `sudo iptables -t nat -A PREROUTING -i wlx347de44fe1c4 -p tcp -d 10.0.0.45 --dport 8080 -j DNAT --to-destination 192.168.122.178:80` | External DNAT   | Question 78 | Interface-specific for WiFi; + `ip_forward=1`.         |

> **Tip**: Bookmark `man virsh` for all commands.

---

## 7. Next Steps

1. **Cleanup**:

   - Remove clone: `virsh undefine clone-vm --remove-all-storage`
   - Delete RAW: `rm /var/lib/libvirt/images/dev-vm.raw`
   - Flush DNAT: `sudo iptables -t nat -D PREROUTING -i wlx347de44fe1c4 -p tcp -d 10.0.0.45 --dport 8080 -j DNAT --to-destination 192.168.122.178:80`
   - Delete snapshot: `virsh snapshot-delete test-server web-ready`

2. **Further Exploration**:

   - Live migration: `virsh migrate --live test-server qemu+ssh://other-host/system`
   - GPU passthrough: Add `<hostdev>` in XML for NVIDIA.
   - OVA export: `virt-v2v -i libvirt -o local -os /tmp test-server`
   - Automate with Ansible: Playbook for VM spin-up/clone.

3. **Deploy in Data Center**:
   - Scale to Proxmox or OpenStack for 100+ VMs.
   - Use Ceph for shared storage in clones.

---
