---
layout: post
title: "CHAPTER 3: SERVICES AND USER MANAGEMENT - FILES & DIRECTORIES MASTERY"
date: 2025-11-14 21:51:00 -0700
tags:
  [Linux+, Linuxlab, Chapter 3, File Permissions, Stickybit, Setgid, Stat, Suid]
---

This document summarizes my hands-on lab experience mastering Linux file permissions, ownership, special bits, links, device files, and user/group management on Ubuntu 24.04. The goal was to deeply understand these core Linux concepts through practical exercises.

> **Goal of This Chapter**  
> Master **Linux file permissions, ownership, special bits, links, device files, and user/group management** through **real interactive labs** on Ubuntu 24.04.
>
> **Target**: Chapter 3 of CompTIA Linux+ : Services and user management - Files & Directories Mastery.  
> **Approach**: Learn by doing — break, fix, explain, repeat.  
> **Outcome**: You _understand_, not just memorize.

---

## Lab Environment Setup

```bash
# VM: Ubuntu 24.04 LTS (UTM/macOS)
# Users: ron (admin), testuser (standard)
# Groups: teamdev, testgroup, newteam (created during labs)
# Working dir: ~/labs
# umask: 0002 (group-writable defaults)
```

```bash
# Key users created
sudo useradd -m testuser
echo "testuser:test123" | sudo chpasswd
```

---

## 3.1.1–3.1.2: File Permissions & Special Bits

### **Core Concepts**

| Permission | Owner | Group | Others | Octal |
| ---------- | ----- | ----- | ------ | ----- |
| Read       | r     | r     | r      | 4     |
| Write      | w     | w     | w      | 2     |
| Execute    | x     | x     | x      | 1     |

```bash
chmod 664 file.txt   # -rw-rw-r--
chmod 2775 dir/      # drwxrwsr-x (setgid)
chmod 1777 /tmp      # drwxrwxrwt (sticky)
```

---

### **Special Bits — The Holy Trinity**

| Bit        | Octal | Meaning                                            | Where         | Effect              |
| ---------- | ----- | -------------------------------------------------- | ------------- | ------------------- |
| **SUID**   | `4`   | Run as **owner**                                   | Files         | `passwd`, `sudo`    |
| **SGID**   | `2`   | Run as **group** (file)<br>**Inherit group** (dir) | Files & Dirs  | `wall`, shared dirs |
| **Sticky** | `1`   | Only owner/dir-owner/root can delete               | **Dirs only** | `/tmp`              |

> **Gem #1: Partial chmod twist**
>
> ```bash
> sudo chgrp root /tmp/gitrepo
> chmod 2775 /tmp/gitrepo   # → 775 only! (s ignored)
> stat -c %a /tmp/gitrepo   # → 775
> ```
>
> **Why?** You’re not in `root` group → **kernel blocks special bit**.

---

### **Setgid Directory = Auto Group Inheritance**

```bash
mkdir mydir
sudo groupadd teamdev
sudo chown :teamdev mydir
sudo chmod 2775 mydir
# → drwxrwsr-x ... teamdev

touch mydir/newfile.txt
# → -rw-rw-r-- ron teamdev
```

> **Gem #2: `newgrp` is for _creation_, not access**
>
> ```bash
> usermod -aG teamdev testuser   # → immediate access
> newgrp teamdev                 # → only for new files
> ```

---

### **Sticky Bit = /tmp Safety (with a catch)**

```bash
chmod 1777 /tmp/rondir
touch /tmp/rondir/ronfile.txt
# testuser: rm → "Operation not permitted"
```

> **Gem #3: Sticky ≠ invincible**
>
> ```bash
> echo "" > /tmp/rondir/ronfile.txt   # Empties it!
> ```
>
> **Fix**: Use `660` files + setgid dir.

---

## 3.1.3: Hard vs Symbolic Links

| Type     | Inode | Survives `rm`? | Cross-FS? | Broken?  |
| -------- | ----- | -------------- | --------- | -------- |
| **Hard** | Same  | Yes            | No        | Never    |
| **Sym**  | Diff  | No             | Yes       | Dangling |

```bash
echo "data" > original.txt
ln original.txt hardlink.txt     # same inode
ln -s original.txt symlink.txt   # -> arrow

rm original.txt
cat hardlink.txt   # → works!
cat symlink.txt    # → broken
```

> **Gem #4: Hard links = atomic backups**

---

## 3.1.4: /dev — Devices as Files

| Type   | `ls -l` | Example                                  | Use                   |
| ------ | ------- | ---------------------------------------- | --------------------- |
| Block  | `b`     | `/dev/vda`                               | `mount`, `dd`         |
| Char   | `c`     | `/dev/tty`                               | streams               |
| Pseudo |         | `/dev/null`, `/dev/zero`, `/dev/urandom` | discard, fill, random |

### **Loop Devices**

```bash
dd if=/dev/zero of=mini.iso bs=1M count=100
sudo mkfs.ext2 mini.iso
sudo losetup -f --show mini.iso   # → /dev/loop0
sudo mount /dev/loop0 mnt
echo "test" > mnt/file.txt
sudo umount mnt; losetup -d /dev/loop0
```

> **Gem #5: Mount ISO without burning**
>
> ```bash
> mount -o loop ubuntu.iso /mnt/iso
> ```

---

## 3.1.5–3.1.6: User & Group Management

### **User Lifecycle**

```bash
# Add
sudo useradd -m -c "Lab User" -s /bin/bash labuser
sudo passwd labuser

# Modify
sudo usermod -aG teamdev labuser
sudo usermod -L labuser     # lock → "Authentication failure"
sudo usermod -U labuser     # unlock

# Expire
sudo passwd --expire labuser  # force change on login

# Delete
sudo userdel -r labuser
```

> **Gem #6: Lock = silent failure**  
> No "account locked" message → security by obscurity.

---

### **Group Strategy**

```bash
sudo groupadd newteam
mkdir shareddir
sudo chgrp newteam shareddir
sudo chmod 2775 shareddir

sudo usermod -aG newteam testuser
```

> **Gem #7: `gpasswd -d` = cleanest removal**
>
> ```bash
> sudo gpasswd -d testuser newteam   # better than usermod -G
> ```

---

### **Traversal ≠ Ownership**

```bash
ls -ld /home/ron
# drwxr-x--- → others: no x → testuser blocked
chmod o+x /home/ron
# → testuser can now ls /home/ron/labs/shareddir
```

> **Gem #8: THE GOLDEN RULE**
>
> ```
> Possession (group) ≠ Access (path)
> ```
>
> **To reach a file, need `x` on every parent directory.**

---

## 3.1.7: UID/GID Ranges (Ubuntu)

| Range     | Type           | Example            |
| --------- | -------------- | ------------------ |
| `0`       | root           | `root:x:0:0`       |
| `1–99`    | static system  | `daemon:x:1:1`     |
| `100–999` | dynamic system | `www-data:x:33:33` |
| `1000+`   | human users    | `ron:x:1000:1000`  |

```bash
# Find free UIDs
seq 1000 1010 | while read u; do
  getent passwd $u >/dev/null || echo "Free: $u"
done
# → Free: 1002, 1003...
```

---

## Final Lab State (Clean)

```bash
ls /home/          → ron  testuser
getent passwd labuser → (gone)
getent group newteam  → (ron only or deleted)
rm -rf ~/labs/shareddir mini.iso loopmnt
```

---

## Exam-Ready Cheat Sheet

```bash
# Permissions
chmod 2775 dir/          # setgid
chmod 1777 /tmp          # sticky
stat -c %a file          # true octal

# Links
ln file hardlink         # hard
ln -s file symlink       # sym

# /dev
dd if=/dev/zero of=file bs=1M count=10
head -c 32 /dev/urandom | base64 > key

# Users
useradd -m -s /bin/bash user
usermod -aG team user
gpasswd -d user team     # clean remove
userdel -r user

# Groups
chmod 2775 shared/       # inherit
newgrp team              # only for new files

# Traversal
chmod o+x /home/ron      # allow others to enter
```

---

## Key Gems (Your Discoveries)

| #   | Gem                                                    |
| --- | ------------------------------------------------------ |
| 1   | `chmod 2775` can **partially fail** → use `stat -c %a` |
| 2   | `newgrp` = **creation only** → supp group = access     |
| 3   | Sticky bit ≠ safe from `> file` → use `660`            |
| 4   | Hard links = **atomic backups**                        |
| 5   | Mount ISO with `loop`                                  |
| 6   | Account lock = **silent failure**                      |
| 7   | `gpasswd -d` > `usermod -G`                            |
| 8   | **Traversal > Ownership**                              |

---

## EDITED 11/15/25: CLARIFIED THE HOLY TRINITY & USER/GROUP MANAGEMENT

**unpack the "Holy Trinity" of special bits** in Linux — **SUID, SGID, and Sticky** — with **crystal-clear explanations**, **real-world examples**, and **why they matter** for the exam and **real systems**.

---

## The Holy Trinity of Special Bits

### `SUID` | `SGID` | `Sticky`

| Bit        | Octal | Meaning                                                | Where It Works   | Effect                                                     |
| ---------- | ----- | ------------------------------------------------------ | ---------------- | ---------------------------------------------------------- |
| **SUID**   | `4`   | **Run as _owner_**                                     | **Files only**   | Program runs with **owner’s privileges**, not the caller’s |
| **SGID**   | `2`   | **Run as _group_** (file)<br>**Inherit _group_** (dir) | **Files & Dirs** | File: runs as group<br>Dir: new files inherit dir’s group  |
| **Sticky** | `1`   | **Only owner/dir-owner/root can delete**               | **Dirs only**    | Prevents deletion by others (even with write perms)        |

---

## 1. SUID — **Run as Owner** (Octal `4xxx`)

### What it does:

When a **file** has SUID set, **anyone who runs it** gets the **privileges of the file’s _owner_** — **not their own**.

### Real Example: `/usr/bin/passwd`

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... passwd
#     ^── s = SUID (4)
```

- You (ron) run `passwd` → need to edit `/etc/shadow` (owned by root)
- But **you’re not root**
- **SUID saves the day**: `passwd` runs **as root** → can write to `/etc/shadow`

> **Without SUID**: You’d get "Permission denied"  
> **With SUID**: You change your password safely

### Other SUID Examples:

```bash
sudo        # -rwsr-xr-x root root
su          # -rwsr-xr-x root root
mount       # -rwsr-xr-x root root
```

> **Security Note**: SUID on user-owned scripts = **huge risk** (privilege escalation). Only root should own SUID binaries.

---

## 2. SGID — **Two Superpowers** (Octal `2xxx`)

### **A. On Files: Run as Group**

```bash
ls -l /usr/bin/wall
# -rwxr-sr-x 1 root tty ... wall
#       ^── s = SGID (2)
```

- `wall` sends messages to all terminals
- Needs access to `/dev/tty*` (owned by `tty` group)
- When you run `wall`, it runs **as group `tty`** → gets access

---

### **B. On Directories: Inherit Group** (The Real Magic)

```bash
mkdir shared
sudo chgrp devteam shared
sudo chmod 2775 shared
# → drwxrwsr-x 2 ron devteam
#         ^── s = SGID
```

Now:

```bash
touch shared/report.txt
# → -rw-rw-r-- 1 ron devteam   ← group = dir's group!
```

> **Why?**  
> Team members create files → all get `devteam` group → **shared access** without manual `chgrp`

> **Gem**: This is **how `/tmp` works** (with sticky bit too):

```bash
ls -ld /tmp
# drwxrwxrwt ... root root
#        ^ t = sticky, s = SGID (often hidden in ls)
```

---

## 3. Sticky Bit — **Delete Protection** (Octal `1xxx`)

### What it does:

In a directory with sticky bit:

> **Only these can delete/rename a file:**
>
> - File owner
> - Directory owner
> - root

Even if **others have write permission**.

### Real Example: `/tmp`

```bash
ls -ld /tmp
# drwxrwxrwt 10 root root ...
#           ^── t = sticky bit
```

```bash
touch /tmp/ron-secret.txt
# You own it

# testuser tries:
rm /tmp/ron-secret.txt
# → "Operation not permitted"
```

But:

```bash
echo "" > /tmp/ron-secret.txt   # ← STILL WORKS! (truncates)
```

> **Gem**: Sticky bit **blocks deletion**, but **not modification**  
> → Always use `chmod 660` on shared files

---

## Visual Summary (ls -l Output)

| Mode         | Meaning           |
| ------------ | ----------------- |
| `-rwsr-xr-x` | **SUID** on file  |
| `-rwxr-sr-x` | **SGID** on file  |
| `drwxrwsr-x` | **SGID** on dir   |
| `drwxrwxrwt` | **Sticky** on dir |

> **Pro Tip**:
>
> - `s` in **owner** = SUID
> - `s` in **group** = SGID
> - `t` in **others** = Sticky

---

## How to Set Them (Octal)

| Goal          | Command                                  |
| ------------- | ---------------------------------------- |
| SUID on file  | `chmod 4755 script.sh`                   |
| SGID on file  | `chmod 2755 script.sh`                   |
| SGID on dir   | `chmod 2775 shared/`                     |
| Sticky on dir | `chmod 1777 /tmp`                        |
| All three     | `chmod 4777 file` (SUID + SGID + Sticky) |

---

## Exam-Ready Cheat Sheet

```bash
# Check special bits
ls -l /usr/bin/passwd   # SUID
ls -l /tmp              # SGID + Sticky
ls -l /usr/bin/wall     # SGID

# Set them
chmod u+s file          # SUID
chmod g+s dir           # SGID
chmod o+t dir           # Sticky

# Symbolic
chmod 4755 file         # u+rwx, g+rx, o+rx, u+s
chmod 2775 dir          # g+rwx, o+rx, g+s
chmod 1777 dir          # o+rwx, o+t
```

---

## Why This Matters (LPIC-1)

| Question                             | Answer                |
| ------------------------------------ | --------------------- |
| Why can users change passwords?      | **SUID on `passwd`**  |
| How do team files stay shared?       | **SGID on directory** |
| Why can't I delete your `/tmp` file? | **Sticky bit**        |
| What does `s` mean in `drwxrwsr-x`?  | **SGID inheritance**  |

---

## Your Lab Proof (from our work)

```bash
# SGID dir
drwxrwsr-x 2 ron newteam ... shareddir/
# → testuser creates file → gets newteam group

# Sticky + SGID
drwxrwsrwt ... /tmp/testdir/
# → testuser edits, but can't rm
```

---

## Final Gem: **The Trinity in One Line**

```bash
chmod 3777 /shared/tmp
# 3 = SUID (4? no) + SGID (2) + Sticky (1)
# → drwxrwsrwt
# → Run as owner, inherit group, only owners delete
```

---

## Final Gem 2: **Command to list SUID files**

```bash
find / -type f -perm /4000 2>/dev/null
or:
sudo find / \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null
```
