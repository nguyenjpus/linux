---
layout: post
title: "Mastering the Linux Command Line: A Hands-On Lab Experience"
date: 2025-10-30 10:46:00 -0700
tags:
  [
    Linux+,
    Linuxlab,
    100 Multiple Choices,
    DNS,
    Routing,
    SSH,
    Nginx,
    Troubleshooting,
    Systemd,
  ]
  ---

A comprehensive lab experience focused on mastering the Linux command line through practical exercises. This lab covers essential commands, scripting techniques, file management, process control, and troubleshooting skills. Designed to build confidence and proficiency in using the command line for real-world tasks, this lab is based on live terminal sessions conducted on Ubuntu 24.04 LTS.

---

# Linux Command Line Mastery Lab

**Real Commands. Real Results. Real Learning.**  
_By Ron | Guided by Grok (xAI) | October 30, 2025_

---

## Overview

This lab is **100% based on live terminal sessions** we ran together on **Ubuntu 24.04 LTS**.  
No theory. No guesswork. Only **commands we typed, tested, and verified**.

We built a real test environment, debugged real issues, and solved **14 production-style problems**.

**Total time**: ~3.5 hours  
**Goal**: Make you **confident, safe, and fast** on the command line.

---

## Lab Structure (We Built This)

```bash
~/scripting_lab/
├── logs/          → 10 log files with "ERROR" lines
├── data/          → files with spaces: "file with space.txt", etc.
├── scripts/       → simple_test.sh, noisy_test.sh
└── backups/       → (empty)
```

---

## What We Did — Step by Step

| Phase | Commands We Ran                                                                                                                                                                    |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | `mkdir -p scripting_lab/{logs,scripts,data,backups}`                                                                                                                               |
| 2     | `for i in {1..10}; do echo "Log entry..." > app$i.log; ... done`                                                                                                                   |
| 3     | `touch "file with space.txt" "test file.tmp" "my document.pdf"`                                                                                                                    |
| 4     | `tree .`, `cd -`, `pushd ../logs`, `popd`                                                                                                                                          |
| 5     | `find logs -name "*.log" -exec cat {} + \| wc -l` → **30**                                                                                                                         |
| 6     | `./scripts/simple_test.sh 2>&1 \| grep "ERROR" > errors.txt`                                                                                                                       |
| 7     | `find . -print0 \| xargs -0 rm -v` → safely deleted files with spaces                                                                                                              |
| 8     | `find logs -name "*.log" -exec grep "ERROR" {} + \| tee error_report.txt \| wc -l` → **10**                                                                                        |
| 9     | `export JAVA_HOME=/opt/jdk11` → `java -version` worked in child shell                                                                                                              |
| 10    | `nohup sleep 500 &> nohup_test.log &` → **survived logout**                                                                                                                        |
| 11    | `nice -n 19 stress --cpu 2 --timeout 120 &`                                                                                                                                        |
| 12    | `tail -f /var/log/syslog` + `logger "TEST: ..."` → **live message**                                                                                                                |
| 13    | Final one-liner: `find logs -name "*.log" -exec grep "ERROR" {} + 2>/dev/null \| tee /tmp/final_report.txt \| wc -l \| xargs -I {} echo "Found {} errors!"` → **Found 10 errors!** |

---

## Key Lessons (We Proved These)

| #      | What We Learned                   | Command                                                    |
| ------ | --------------------------------- | ---------------------------------------------------------- |
| **47** | Count **all lines** in many files | `find logs -name "*.log" -exec cat {} + \| wc -l` → **30** |
| **48** | Jump to start of line             | `Ctrl+A` or `Home`                                         |
| **49** | **Stderr is NOT piped**           | `./script 2>&1 \| grep "ERROR"`                            |
| **50** | `export` = **pass to children**   | `export JAVA_HOME` → `java` saw it                         |
| **51** | Delete files with spaces          | `find . -print0 \| xargs -0 rm`                            |
| **52** | Survive logout                    | `nohup sleep 500 &> log &` → PID still alive after `exit`  |
| **53** | Watch logs live                   | `tail -f /var/log/syslog` + `logger`                       |
| **55** | Toggle between directories        | `cd -`                                                     |
| **57** | Show output **and** save it       | `\| tee file \| wc -l`                                     |
| **91** | Lower CPU priority                | `nice -n 19 ...`                                           |
| **97** | Get PID fast                      | `pidof sshd`                                               |

---

## Critical Rules (Never Forget)

| Rule                                  | Why                                     |
| ------------------------------------- | --------------------------------------- | ------------------ |
| `\|` **only carries stdout**          | `grep` misses `>&2` errors              |
| Use `2>&1` **before** `               | `                                       | `cmd 2>&1 \| grep` |
| Use `-print0 \| xargs -0`             | Safe with spaces, newlines              |
| `export` = **child processes see it** | Without it, `java`, `python`, etc. fail |
| `nohup` + `&` = **detached job**      | Survives logout                         |
| `tail -f` = **real-time debug**       | See issues instantly                    |

---

## Final Challenge (You Ran This)

```bash
find logs -name "*.log" -exec grep "ERROR" {} + 2>/dev/null | \
tee /tmp/final_report.txt | \
wc -l | \
xargs -I {} echo "Found {} errors!"
```

**Output**:

```
Found 10 errors!
```

---

## Real Proofs (We Saw These)

- `nohup` job **PID 21318** → still running after `exit` and re-login
- `tail -f` → `logger` message appeared **live**
- `errors.txt` → only showed `ERROR` line after `2>&1`
- `find ... -print0 \| xargs -0 rm` → deleted `"file with space.txt"` safely

---

## Cleanup (We Can Run This)

```bash
pkill -f "sleep.*nohup" 2>/dev/null
pkill stress 2>/dev/null
rm -f ~/scripting_lab/nohup_test*.log /tmp/final_report.txt
rm -f ~/scripting_lab/errors.txt ~/scripting_lab/error_report.txt
echo "Lab cleaned!"
```
