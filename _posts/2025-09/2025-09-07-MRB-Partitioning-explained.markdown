# Understanding MBR Partitioning: Primary, Extended, and Logical Partitions

This document explains the structure of the Master Boot Record (MBR) partitioning scheme, focusing on the roles of primary, extended, and logical partitions.

## Given the following output:

```bash
ron@usv:~$ dd if=/dev/zero of=~/multi_mbr.img bs=1M count=400
400+0 records in
400+0 records out
419430400 bytes (419 MB, 400 MiB) copied, 0.145743 s, 2.9 GB/s
ron@usv:~$ sudo fdisk ~/multi_mbr.img

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x40d48b17.

Command (m for help): o
Created a new DOS (MBR) disklabel with disk identifier 0x7b0d7ff3.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): #selected default here.
First sector (2048-819199, default 2048): #selected default here.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-819199, default 819199): +100M

Created a new partition 1 of type 'Linux' and of size 100 MiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): #selected default here.
First sector (206848-819199, default 206848): #selected default here.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-819199, default 819199): +100M

Created a new partition 2 of type 'Linux' and of size 100 MiB.

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): e
Partition number (3,4, default 3): #selected default here.
First sector (411648-819199, default 411648): #selected default here.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (411648-819199, default 819199): #selected default here.

Created a new partition 3 of type 'Extended' and of size 199 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 5
First sector (413696-819199, default 413696): #selected default here.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (413696-819199, default 819199): +100M

Created a new partition 5 of type 'Linux' and of size 100 MiB.

Command (m for help): p
Disk /home/ron/multi_mbr.img: 400 MiB, 419430400 bytes, 819200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x7b0d7ff3

Device                   Boot  Start    End Sectors  Size Id Type
/home/ron/multi_mbr.img1        2048 206847  204800  100M 83 Linux
/home/ron/multi_mbr.img2      206848 411647  204800  100M 83 Linux
/home/ron/multi_mbr.img3      411648 819199  407552  199M  5 Extended
/home/ron/multi_mbr.img5      413696 618495  204800  100M 83 Linux

Command (m for help): w
The partition table has been altered.
Syncing disks.

```

## The MBR Partitioning Scheme 🧱

The Master Boot Record (MBR) is an older method for partitioning hard drives. It has a primary limitation: it can only define a maximum of **four primary partitions** on a disk. This was often insufficient for users who needed more partitions. To overcome this, the concept of an **extended partition** was introduced.

## Primary Partitions 🥇

- **Analogy**: Think of primary partitions as the main, directly accessible compartments in a toolbox.
- **Function**: These are the main divisions of a hard drive that can be formatted with a filesystem and used to store data directly.
- **Limit**: An MBR disk can have a maximum of **four** primary partitions.
- **Example**: In your `fdisk` output, `/home/ron/multi_mbr.img1` and `/home/ron/multi_mbr.img2` are examples of primary partitions.

## Extended Partitions 📦

- **Analogy**: An extended partition is like a **special container box** or a divider within your main toolbox. Instead of holding data directly, it's designed to hold _other_ partitions.
- **Function**: When you've used up your four primary partition slots and need more, you can designate one of the primary partition slots as an **extended partition**. This extended partition doesn't hold files itself; it acts as a wrapper.
- **Purpose**: Its sole purpose is to contain **logical partitions**.
- **Limit**: You can have only **one** extended partition on an MBR disk.
- **Example**: In your `fdisk` output, `/home/ron/multi_mbr.img3` is an extended partition.

## Logical Partitions 🧩

- **Analogy**: Logical partitions are the compartments _inside_ the special container box (the extended partition).
- **Function**: These are the partitions that are actually formatted with filesystems and used to store data. They reside _within_ the space allocated to the extended partition.
- **Numbering**: Logical partitions are numbered starting from **5**. This is because the MBR table reserves numbers 1 through 4 for primary partitions (and the extended partition also occupies one of these slots). When the system needs to create a logical partition, it starts numbering from 5 onwards to differentiate them from primary partitions.
- **Example**: In your `fdisk` output, `/home/ron/multi_mbr.img5` is a logical partition created within the extended partition.

---

## Why Partition Number 5? 🤔

You asked: "Why do we have partition 5 here? I mean, where is partition 4?"

The MBR partition table has slots for 4 entries. These slots can be used for:

1.  **Four Primary Partitions**: If you create four primary partitions, you cannot create any more partitions, nor can you create an extended partition.
2.  **Three Primary Partitions + One Extended Partition**: This is a common scenario. The extended partition itself takes up one of the four primary slots (usually slot 3 or 4 in `fdisk`'s progression).
3.  **Two Primary Partitions + One Extended Partition**: Similar to the above.
4.  **One Primary Partition + One Extended Partition**: Similar to the above.

In your case:

- You created partition 1 (`/home/ron/multi_mbr.img1`) - Type: Primary.
- You created partition 2 (`/home/ron/multi_mbr.img2`) - Type: Primary.
- You then chose to create an **extended partition** (Type 'e') as partition 3 (`/home/ron/multi_mbr.img3`). This used up the third primary slot.
- Now, you want to create more partitions that can hold data. Since you've used primary slots 1, 2, and 3 (for the extended partition), you can only create logical partitions within the extended partition. The MBR scheme reserves numbers 1-4 for primary/extended partitions. When you create a logical partition, the system starts numbering them from **5**. This is why you see partition 5, and you don't see a partition 4 as a distinct entry in the primary/extended partition list. Partition 3 _is_ the extended partition, and partition 5 is the _first logical partition_ _inside_ it.

## The Point of Extended and Logical Partitions 💡

- **Analogy**: Imagine you have a tool drawer (your hard drive). It only has 4 large slots for tools (primary partitions).

  - If you only have small tools (like screwdrivers, wrenches), you can fit them into these 4 slots.
  - But if you have many small tools, you might want to put them in a smaller, specialized box that fits inside one of the main slots. This specialized box is the **extended partition**.
  - The compartments _inside_ that specialized box are your **logical partitions**.

- **Why do it?**
  - **More Partitions**: The primary reason is to bypass the MBR's limit of four primary partitions. By using an extended partition, you can create many logical partitions within it, effectively having more than four "drives" or divisions on your disk.
  - **Organization**: It allows for better organization, separating different types of data or systems into distinct partitions, even if you need many of them.

## Nested Extended Partitions? 😵

You asked: "Can we keep doing extended logical boxes like this inside these boxes to have, technically, many extended partitions and way more than 4 primary partitions?"

- **No, you cannot nest extended partitions.** The MBR scheme allows for only **one extended partition**. This single extended partition then acts as the container for all subsequent logical partitions. You can't put an extended partition inside another extended partition.

## Partition Numbering and Limits Recap 📊

Let's clarify the slot analogy:

Your main toolbox (the MBR disk) has 4 slots.

- **Scenario 1: Only Primary Partitions**

  - Slot 1: Primary Partition 1
  - Slot 2: Primary Partition 2
  - Slot 3: Primary Partition 3
  - Slot 4: Primary Partition 4
  - You've reached the limit of 4 primary partitions.

- **Scenario 2: Using an Extended Partition**

  - Slot 1: Primary Partition 1
  - Slot 2: Primary Partition 2
  - Slot 3: **Extended Partition** (This slot is now a container)
  - Slot 4: **Free** (This slot can either be another Primary Partition or remain free)

  Now, within the "Extended Partition" (Slot 3), you can create logical partitions:

  - **Logical Partition 5**: The first logical partition inside the extended partition.
  - **Logical Partition 6**: The second logical partition inside the extended partition.
  - **Logical Partition 7**: And so on...

  You can have many logical partitions inside the extended partition, but they will all start numbering from 5. The MBR only defines 4 "slots" at the top level. Slot 3, when designated as extended, points to a structure that allows for a list of logical partitions.

**So, to directly answer your question:**

> "My main toolbox has slots for:
> Slot 1: Primary Partition 1
> Slot 2: Primary Partition 2
> Slot 3: Primary Partition 3
> Slot 4: Extended Partition (Container)
> Slot 5: WILL BE RESERVED for free??
> Slot 6: Available for another Primary Partition 5
> Slot 7: Available for another Primary Partition 6
> Slot 8: Available for another Primary Partition 7
> Slot 9: no more extended partition nested inside an extended partition.
> Is this correct?"

**No, this is not quite correct.**

Here's the corrected version:

- The MBR disk has **4 primary partition table entries**.
- Each entry can be either a **primary partition** or an **extended partition**.
- You can have a **maximum of 4 primary partitions** OR **3 primary partitions + 1 extended partition**.
- The **extended partition** acts as a container for **logical partitions**.
- Logical partitions are numbered starting from **5**.
- You **cannot nest extended partitions**.

**Corrected Analogy:**

Imagine your toolbox has **4 main slots**.

- You can put 4 toolboxes (primary partitions) in these slots.
- OR, you can put 3 toolboxes (primary partitions) and **1 special large container** (extended partition) in these slots.
- The special large container (extended partition) can then hold **many smaller compartments** (logical partitions), which will be numbered 5, 6, 7, and so on.

> "So, either I have 1 or 3 primary partitions, the first one inside the extended partition always picks number 5?"

Yes, that's essentially correct. If you create an extended partition (which takes up one of the 4 primary slots), the _first logical partition you create inside it_ will be numbered **5**. If you have other primary partitions (1, 2, 3, or 4), they will be numbered as such.

> "And, also, how many logical partitions inside extended partition can I have?"

There isn't a strict theoretical limit imposed by the MBR scheme itself on the _number_ of logical partitions. However, practical limits exist due to:

1.  **Disk Size**: The total size of the disk.
2.  **Filesystem Overhead**: Each partition needs some metadata.
3.  **OS and Tool Limitations**: The operating system or partitioning tools (`fdisk`, `parted`) might have their own practical limits for managing a very large number of partitions, but it's generally very high. For most practical purposes, you can create dozens or even hundreds of logical partitions.

---

## `losetup -P` for Partition Detection 🔍

You asked: "To access partitions on a loop device, you need to enable partition detection using the -P (or --partscan) option with losetup"

- **Analogy**: Imagine you have a large box of pre-packaged food items (your disk image). Each item is already wrapped individually (a partition).

  - When you just connect the box to your kitchen counter (mount the loop device without `-P`), your kitchen only sees the _entire box_. It doesn't know what's inside each individual package.
  - When you use the `-P` option with `losetup`, it's like giving your kitchen a special scanner that can **look inside the box** and identify each individual pre-packaged item. The scanner then makes those individual items available as separate things you can work with.

- **Explanation**:
  - A disk image file (like `~/multi_mbr.img`) can contain a partition table (like MBR or GPT). This table defines how the disk space is divided into partitions.
  - When you use `sudo losetup /dev/loop0 ~/multi_mbr.img`, the system creates a **block device** (`/dev/loop0`) that represents the _entire disk image_. However, it doesn't automatically know about the partitions _within_ that image.
  - The `-P` or `--partscan` option tells `losetup` to **read the partition table** on the associated disk image.
  - Once the partition table is read, `losetup` will automatically create **additional device nodes** for each detected partition. For example, if your image has partitions, `losetup -P /dev/loop0 ~/multi_mbr.img` might create devices like `/dev/loop0p1`, `/dev/loop0p2`, etc., which correspond to the partitions defined in the image's partition table.
  - You can then mount these partition-specific loop devices (e.g., `sudo mount /dev/loop0p1 /mnt/partition1`).

Without `-P`, you can only mount the _entire disk image_ as a single volume if it's not partitioned, or if you intend to treat it as a raw disk. If it _is_ partitioned and you want to access those partitions individually, you need `-P`.
