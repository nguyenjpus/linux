---
layout: post
title: "Multiple-choices - Questions 41-50 Explained"
date: 2025-10-01 07:55:00 -0700
tags: [Linux+, Multiple-choices]
---

This document explains questions 41-50 from a set of 100 scenario-based multiple-choices questions for CompTIA Linux+ preparation, focusing on hostname management, DNS, networking, file downloads, and shell environment. Each question includes the correct answer, why it’s correct, why other options are incorrect, key concepts, and memory aids for retention.

## Question 41: Setting Hostname Persistently

**Question**: A new Linux server's hostname needs to be set to `db-server-01`. Which command, available on most modern systems with systemd, can be used to set the hostname persistently?  
**Options**:

- hostname db-server-01
- echo "db-server-01" > /etc/hostname
- hostnamectl set-hostname db-server-01
- sysctl kernel.hostname=db-server-01

**Correct Answer**: hostnamectl set-hostname db-server-01  
**Why Correct**: On systemd-based systems (e.g., Ubuntu, Fedora), `hostnamectl set-hostname` sets the hostname persistently by updating `/etc/hostname` and applying it immediately to the running system, ensuring consistency across reboots.  
**Why Others Wrong**:

- `hostname db-server-01` sets the hostname temporarily (lost on reboot).
- `echo "db-server-01" > /etc/hostname` updates the file but requires a reboot or `hostname` command to apply, not immediate.
- `sysctl kernel.hostname` sets the kernel hostname temporarily, not persistently, and is less reliable.  
  **Key Concept**: `hostnamectl` is the systemd tool for managing hostnames; it updates `/etc/hostname` and `/etc/hosts` if needed.  
  **Memory Aid**: “hostnamectl = Control hostname permanently.”

## Question 42: Querying DNS MX Records

**Question**: An administrator wants to query a specific DNS server (1.1.1.1) to find the MX (Mail Exchange) records for the domain `example.com`. Which `dig` command would accomplish this?  
**Options**:

- dig example.com MX @1.1.1.1
- dig MX example.com 1.1.1.1
- nslookup -query=mx example.com 1.1.1.1
- host -t MX example.com 1.1.1.1

**Correct Answer**: dig example.com MX @1.1.1.1  
**Why Correct**: `dig` queries DNS servers, and the syntax `dig <domain> <type> @<server>` (e.g., `dig example.com MX @1.1.1.1`) requests MX records for `example.com` from the DNS server 1.1.1.1.  
**Why Others Wrong**:

- `dig MX example.com 1.1.1.1` has incorrect syntax (type before domain, no `@`).
- `nslookup -query=mx` works but isn’t a `dig` command as asked.
- `host -t MX` is valid but uses `host`, not `dig`.  
  **Key Concept**: `dig` provides detailed DNS output; `@<server>` specifies the DNS server to query.  
  **Memory Aid**: “dig @server = Direct query to DNS server.”

## Question 43: Bringing Up a Network Interface

**Question**: A network interface `eth1` is currently down. Which command would bring the interface up?  
**Options**:

- ip link set eth1 up
- ifup eth1
- nmcli device connect eth1
- All of the above are valid on different systems.

**Correct Answer**: All of the above are valid on different systems.  
**Why Correct**:

- `ip link set eth1 up` directly enables the interface using `iproute2`, universal across Linux.
- `ifup eth1` works on systems using `ifupdown` (e.g., Debian, with `/etc/network/interfaces`).
- `nmcli device connect eth1` works on NetworkManager-based systems (e.g., Ubuntu, Fedora).  
  Each command is valid depending on the system’s network management tool.  
  **Why Others Wrong**: None are incorrect; all are context-dependent.  
  **Key Concept**: Interface management varies by distro; check `ip link show` for status.  
  **Memory Aid**: “ip, ifup, nmcli = Interface up, system decides.”

## Question 44: Resumable File Download

**Question**: An administrator needs to download a large file from a web server via the command line and wants the download to be able to resume if it gets interrupted. Which command-line utility is well-suited for this task?  
**Options**:

- scp
- curl -O
- wget -c
- ftp

**Correct Answer**: wget -c  
**Why Correct**: `wget -c` (continue) resumes interrupted downloads by appending to partially downloaded files, ideal for large files over HTTP/HTTPS/FTP.  
**Why Others Wrong**:

- `scp` is for secure file transfers (SSH), not web downloads.
- `curl -O` downloads files but doesn’t resume automatically.
- `ftp` is interactive and less reliable for resuming.  
  **Key Concept**: `wget --continue` checks file size and resumes; use `-O` for output file naming.  
  **Memory Aid**: “wget -c = Continue web get.”

## Question 45: Local Hostname Mapping

**Question**: To quickly map a hostname `test-server` to the IP address 192.168.1.150 for local resolution only, without involving DNS, which file should be edited?  
**Options**:

- /etc/resolv.conf
- /etc/hosts
- /etc/nsswitch.conf
- /etc/hostname

**Correct Answer**: /etc/hosts  
**Why Correct**: Adding `192.168.1.150 test-server` to `/etc/hosts` maps the hostname to the IP locally, bypassing DNS for resolution.  
**Why Others Wrong**:

- `/etc/resolv.conf` specifies DNS servers.
- `/etc/nsswitch.conf` defines name resolution order, not mappings.
- `/etc/hostname` sets the system’s hostname.  
  **Key Concept**: `/etc/hosts` is checked before DNS if `nsswitch.conf` lists `files` first.  
  **Memory Aid**: “hosts = Hostname-to-IP mapping.”

## Question 46: Viewing Active Network Connections

**Question**: An administrator is troubleshooting a firewall and needs to see all active network connections, including TCP and UDP, and the processes associated with them. Which `ss` command provides this detailed view?  
**Options**:

- ss -a
- ss -l
- ss -tulpn
- ss -s

**Correct Answer**: ss -tulpn  
**Why Correct**: `ss -tulpn` shows TCP/UDP (`-t`, `-u`), listening and non-listening sockets (`-l` for listening, but `-tup` includes established), with process names/PIDs (`-p`) and numeric ports (`-n`).  
**Why Others Wrong**:

- `ss -a` shows all sockets but lacks process details.
- `ss -l` shows only listening sockets.
- `ss -s` shows socket statistics, not connections.  
  **Key Concept**: `ss` replaces `netstat`; `-tulpn` is ideal for firewall troubleshooting.  
  **Memory Aid**: “ss -tulpn = TCP/UDP, listening/processes, numeric.”

## Question 47: Counting Lines in Log Files

**Question**: An administrator wants to find all log files in the `/var/log` directory and its subdirectories, and then count the total number of lines in all of those files combined. Which command pipeline correctly achieves this?  
**Options**:

- find /var/log -name "\* log" | wc -l
- find /var/log -name "\*.log" -exec cat {} + | wc -l
- ls -R /var/log/\*log | wc -l
- grep " log" /var/log | count

**Correct Answer**: find /var/log -name "_.log" -exec cat {} + | wc -l  
**Why Correct**: `find /var/log -name "_.log"`locates files ending in`.log`, `-exec cat {} +`concatenates their contents, and`| wc -l` counts total lines.  
**Why Others Wrong**:

- `find ... | wc -l` counts file names, not lines in files.
- `ls -R` isn’t reliable for recursive file listing and counts lines incorrectly.
- `grep " log" /var/log | count` is invalid (`count` isn’t a command).  
  **Key Concept**: Use `find -exec` to process multiple files; `wc -l` counts lines.  
  **Memory Aid**: “find -exec cat | wc = Find, concatenate, count.”

## Question 48: Bash Cursor Shortcut

**Question**: A user is typing a long command and makes a mistake at the beginning of the line. Which keyboard shortcut will move the cursor directly to the start of the line in a Bash shell?  
**Options**:

- Ctrl + E
- Ctrl + A
- Home
- Both B and C are correct.

**Correct Answer**: Both B and C are correct.  
**Why Correct**: In Bash, `Ctrl + A` and `Home` both move the cursor to the start of the line (beginning of command).  
**Why Others Wrong**:

- `Ctrl + E` moves to the end of the line.  
  **Key Concept**: Bash uses readline shortcuts; `Ctrl + A`/`Home` and `Ctrl + E`/`End` are standard navigation keys.  
  **Memory Aid**: “Ctrl + A or Home = Anchor to start.”

## Question 49: Filtering Script Output

**Question**: A script is generating a lot of diagnostic output, but the administrator only cares about lines that contain the word "ERROR". They also want to redirect these error lines to a file named `errors.txt`. Which command is correct?  
**Options**:

- ./script.sh | grep "ERROR" > errors.txt
- ./script.sh > errors.txt | grep "ERROR"
- ./script.sh < errors.txt grep "ERROR"
- grep "ERROR" ./script.sh > errors.txt

**Correct Answer**: ./script.sh | grep "ERROR" > errors.txt  
**Why Correct**: `./script.sh` runs the script, `| grep "ERROR"` filters lines containing “ERROR”, and `> errors.txt` redirects them to `errors.txt`.  
**Why Others Wrong**:

- `./script.sh > errors.txt | grep "ERROR"` redirects all output to `errors.txt`, so `grep` gets nothing.
- `./script.sh < errors.txt grep "ERROR"` uses input redirection incorrectly.
- `grep "ERROR" ./script.sh` treats `script.sh` as a file, not output.  
  **Key Concept**: Pipe (`|`) sends output to `grep`; `>` redirects to a file.  
  **Memory Aid**: “Pipe to grep, redirect to file = Filter then save.”

## Question 50: Setting Environment Variable

**Question**: An administrator needs to set an environment variable `JAVA_HOME` to `/opt/jdk11` so that it is available to all subsequently executed commands in the current shell and any child shells. Which command should be used?  
**Options**:

- JAVA_HOME=/opt/jdk11
- set JAVA_HOME=/opt/jdk11
- export JAVA_HOME=/opt/jdk11
- env JAVA_HOME=/opt/jdk11

**Correct Answer**: export JAVA_HOME=/opt/jdk11  
**Why Correct**: `export JAVA_HOME=/opt/jdk11` sets the variable and exports it to the current shell and all child shells, making it available to subsequent commands.  
**Why Others Wrong**:

- `JAVA_HOME=/opt/jdk11` sets the variable but doesn’t export it to child shells.
- `set JAVA_HOME=/opt/jdk11` is for local shell variables, not environment variables.
- `env JAVA_HOME=/opt/jdk11` sets it for one command, not persistently in the shell.  
  **Key Concept**: `export` makes variables available to child processes; add to `~/.bashrc` for persistence.  
  **Memory Aid**: “export = Export env to children.”

## Retention Tips for Questions 41-50

- **Themes**: Hostname management (`hostnamectl`), DNS queries (`dig`), network interfaces (`ip`, `nmcli`, `ifup`), file downloads (`wget`), local resolution (`/etc/hosts`), network connections (`ss`), log file processing (`find`), Bash shortcuts, output filtering (`grep`), and environment variables (`export`).
- **Mnemonic for Networking/Shell**: “Host for names, Dig for DNS, Export for env” (H-D-E).
- **Practice**: In a Linux VM:
  - Set hostname with `hostnamectl`, query MX records with `dig @8.8.8.8`.
  - Edit `/etc/hosts`, download a file with `wget -c`, and filter script output.
  - Set `JAVA_HOME` and verify with `echo $JAVA_HOME` in a child shell.
- **Spaced Repetition**: Review in 24 hours, then 3 days. Flashcards: “Command for MX records?” → “dig domain MX @server.”
- **Quiz Yourself**: How to resume a download? File for local hostname mapping?
