---
layout: post
title: "Tar"
date: 2025-09-02
tags: [Basic, Tar]
---

This guide walks through creating, extracting, and compressing archives using `tar`, including reproducible builds and different compression algorithms. It addresses common errors and provides practical steps for experimentation.

## Prerequisites

Install required packages:

```bash
sudo apt install -y tar gzip bzip2 xz-utils zstd logrotate rsync gddrescue testdisk extundelete e2fsprogs xfsprogs lvm2
```

## Step-by-Step Instructions

### 1. Create Test Files

Simulate a project directory with sample files:

```bash
mkdir test_dir && cd test_dir
echo "File1 content" > file1.txt
echo "File2 content" > file2.txt
mkdir subdir
echo "Subfile" > subdir/subfile.txt
cd ..
```

### 2. Basic Archiving

Create an uncompressed archive from outside `test_dir` to avoid the error `tar: archive cannot contain itself`:

```bash
tar -cf archive.tar test_dir
```

- **List contents**:

```bash
tar -tf archive.tar
```

Expected output:

```
test_dir/
test_dir/file1.txt
test_dir/file2.txt
test_dir/subdir/
test_dir/subdir/subfile.txt
```

- **Extract**:

```bash
mkdir extract
tar -xf archive.tar -C extract
```

- **Verify**:

```bash
ls -R extract
```

Expected output:

```
extract:
test_dir

extract/test_dir:
file1.txt  file2.txt  subdir

extract/test_dir/subdir:
subfile.txt
```

### 3. Compression Algorithms

Compress the `test_dir` directory using different algorithms:

- **Gzip** (fast):

```bash
tar -czf archive.tar.gz test_dir
```

- **Bzip2** (better ratio, slower):

```bash
tar -cjf archive.tar.bz2 test_dir
```

- **Xz** (best ratio):

```bash
tar -cJf archive.tar.xz test_dir
```

- **Zstd** (modern, balanced; requires `zstd` installed):

```bash
tar --zstd -cf archive.tar.zst test_dir
```

- **Compare sizes**:

```bash
ls -lh archive.tar*
```

Example output (sizes vary):

```
-rw-r--r-- 1 user user 10K Sep  1 09:00 archive.tar
-rw-r--r-- 1 user user 1.2K Sep  1 09:00 archive.tar.gz
-rw-r--r-- 1 user user 1.1K Sep  1 09:00 archive.tar.bz2
-rw-r--r-- 1 user user 1.0K Sep  1 09:00 archive.tar.xz
-rw-r--r-- 1 user user 1.1K Sep  1 09:00 archive.tar.zst
```

Observations:

- Uncompressed `.tar` is largest.
- `xz` typically produces the smallest file, followed by `zstd`, `bzip2`, and `gzip`.
- `gzip` is fastest, `xz` slowest but smallest.

- **Extract with auto-detection**:

```bash
mkdir extract_gz
tar -xf archive.tar.gz -C extract_gz
```

`tar` auto-detects compression (no need for `-z`, `-j`, etc.). Verify with `ls -R extract_gz`.

### 4. Reproducible Builds

Ensure identical archives across runs:

```bash
tar -cf reproducible.tar --owner=0 --group=0 --sort=name test_dir
sha256sum reproducible.tar > hash1.txt
tar -cf reproducible.tar --owner=0 --group=0 --sort=name test_dir
sha256sum reproducible.tar > hash2.txt
diff hash1.txt hash2.txt
```

No output from `diff` confirms identical hashes (reproducible due to `--sort=name` and fixed ownership).

For comparison, create a non-reproducible archive:

```bash
tar -cf non_repro.tar test_dir
sha256sum non_repro.tar > hash3.txt
tar -cf non_repro.tar test_dir
sha256sum non_repro.tar > hash4.txt
diff hash3.txt hash4.txt
```

Differences may appear due to variable file order or metadata.

### 5. Advanced: Long-Term Storage

Use `xz` for archives retained >3 months with reproducible flags:

```bash
tar -cJf reproducible.tar.xz --owner=0 --group=0 --sort=name test_dir
```

Extract and verify:

```bash
mkdir extract_xz
tar -xf reproducible.tar.xz -C extract_xz
ls -R extract_xz
```

### 6. Experiment with Large Files

Test compression ratios and speeds with a 10MB random file:

```bash
mkdir test_dir && cd test_dir
dd if=/dev/urandom of=bigfile bs=1M count=10
cd ..
tar -czf bigfile.tar.gz test_dir/bigfile
tar -cjf bigfile.tar.bz2 test_dir/bigfile
tar -cJf bigfile.tar.xz test_dir/bigfile
tar --zstd -cf bigfile.tar.zst test_dir/bigfile
ls -lh bigfile.tar*
```

- Random data compresses poorly, but `xz` and `zstd` typically yield smaller files than `gzip`.
- Time commands to compare speeds:

```bash
time tar -czf bigfile.tar.gz test_dir/bigfile
time tar -cJf bigfile.tar.xz test_dir/bigfile
```

Example: `gzip` is faster, `xz` is slower but produces a smaller file.

### 7. Cleanup

Remove temporary files:

```bash
rm -rf test_dir extract extract_gz extract_xz archive* reproducible* non_repro* hash*.txt bigfile*
```

## Notes

- **Error Fix**: Creating archives from outside `test_dir` prevents `tar: archive cannot contain itself`.
- **Zstd Clarification**: Use `.tar.zst` for Zstandard-compressed archives, not `.tar.gz`, to avoid confusion.
- **Troubleshooting**: If `--zstd` fails, ensure `zstd` is installed (`sudo apt install zstd`). Check file paths and permissions if archives aren't created.
- **Best Practices**: Use `xz` or `zstd` for long-term storage due to better compression ratios. Use `--owner=0 --group=0 --sort=name` for reproducible archives.
