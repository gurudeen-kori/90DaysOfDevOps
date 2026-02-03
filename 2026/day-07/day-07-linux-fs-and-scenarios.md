# Linux File System Hierarchy (FHS)

The Linux filesystem follows the **Filesystem Hierarchy Standard (FHS)**, which defines
where files and directories should be located and what purpose they serve.
This structure helps maintain consistency across Linux distributions.

---

## Root Directory `/`

The root directory is the top-most directory in Linux.

All files, directories, and devices originate from `/`.

Without the root filesystem, Linux cannot function.
```
  /
├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── lib64
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var
```

---

## Directory Descriptions

### `/bin` — Binary Executables
- This directory contains essential command-line programs required by all users.
- Commands like `ls`, `cp`, `mv`, `cat`, and `echo` are stored here.
- These binaries are necessary for basic system operation, even in recovery mode.

---

### `/boot` — Boot Loader Files
- The `/boot` directory contains files required to start the Linux system.
- This includes the Linux kernel, initial RAM disk (`initrd`), and bootloader files.
- Without these files, the system cannot boot.

---

### `/dev` — Device Files
- This directory contains special files that represent hardware devices.
- Hard drives, USB devices, terminals, and printers appear as files here.
- The kernel communicates with hardware through these device files.

---

### `/etc` — Configuration Files
- The `/etc` directory stores system-wide configuration files.
- These files control system behavior, services, networking, and user settings.
- Most files here are plain text and can be edited by administrators.

---

### `/home` — User Home Directories
- Each normal user has a personal directory inside `/home`.
- It contains user files such as documents, downloads, and personal settings.
- Permissions ensure users cannot access each other’s data by default.

---

### `/lib` — Essential Shared Libraries
- This directory contains shared libraries needed by essential system binaries.
- Programs in `/bin` and `/sbin` depend on these libraries to run.
- Without `/lib`, many basic commands would fail to execute.

---

### `/lib64` — 64-bit Libraries
- On 64-bit systems, `/lib64` stores 64-bit shared libraries.
- It separates 64-bit libraries from 32-bit ones for compatibility.
- This ensures proper execution of 64-bit applications.

---

### `/media` — Removable Media
- The `/media` directory is used for mounting removable devices.
- USB drives, CDs, and DVDs are automatically mounted here.
- Each device usually gets its own subdirectory.

---

### `/mnt` — Temporary Mount Point
- This directory is traditionally used for manual or temporary mounts.
- System administrators often mount filesystems here for maintenance.
- Unlike `/media`, it is not usually used for automatic mounting.

---

### `/opt` — Optional Software
- The `/opt` directory contains optional or third-party software.
- Applications installed here are usually self-contained.
- This keeps them separate from system-managed software.

---

### `/proc` — Process Information
- `/proc` is a virtual filesystem created in memory.
- It provides real-time information about running processes and the kernel.
- Files here do not exist on disk but are generated dynamically.

---

### `/root` — Root User Home
- This is the home directory for the root (administrator) user.
- It is separate from `/home` for security and system integrity.
- Only the root user can access this directory.

---

### `/run` — Runtime Data
- The `/run` directory stores temporary runtime data.
- It includes process ID files, sockets, and system state information.
- Data here is cleared on system reboot.

---

### `/sbin` — System Binaries
- This directory contains administrative commands for system management.
- Utilities like `fsck`, `mount`, and `reboot` are stored here.
- These commands are typically used by the root user.

---

### `/srv` — Service Data
- The `/srv` directory holds data for system services.
- For example, web server or FTP server data may be stored here.
- It separates service data from configuration and binaries.

---

### `/sys` — System Information
- `/sys` is a virtual filesystem similar to `/proc`.
- It provides information about devices, drivers, and kernel features.
- System administrators use it to interact with hardware settings.

---

### `/tmp` — Temporary Files
- This directory stores temporary files created by applications.
- Files here are usually deleted automatically on reboot.
- All users can write to `/tmp`, but with restricted permissions.

---

### `/usr` — User Programs and Data
- The `/usr` directory contains user-level applications and utilities.
- It includes binaries, libraries, documentation, and shared resources.
- Most installed software resides under `/usr`.

/usr
├── bin
├── lib
├── local
└── share


---

### `/var` — Variable Data
- The `/var` directory stores data that changes frequently.
- This includes logs, mail queues, caches, and spool files.
- It grows dynamically as the system runs.

/var
├── log
├── mail
├── spool
└── tmp


---

## Summary

- `/bin` and `/sbin` contain essential commands  
- `/etc` controls system configuration  
- `/home` stores user data  
- `/var` holds logs and changing files  
- `/proc` and `/sys` provide virtual system information  

---

## Reference

- Filesystem Hierarchy Standard (FHS)
- Linux manual page: `man hier`
- 

# Scenario 01: A Directory Is Taking Too Much Space

## Problem Statement
- A directory on a Linux system is consuming excessive disk space.
- The goal is to analyze the directory, identify what is using the space,
and determine possible cleanup actions without affecting system stability.

---

## Step 1: Check Overall Disk Usage
First, verify which filesystem is running out of space.

```bash
df -h
This command shows disk usage at the filesystem level.
It helps confirm whether the issue is isolated to a specific partition.
```
# Step 2: Check Total Size of the Directory
Measure how much space the directory is consuming.
```bash
du -sh /path/to/directory
-s shows the total size

-h makes the output human-readable

This confirms whether the directory is truly the source of the issue.
```

# Step 3: Identify Large Subdirectories
Break down disk usage inside the directory.
```bash
du -h --max-depth=1 /path/to/directory
This command lists the size of each subdirectory.
It helps pinpoint which folder is consuming the most space.
```

# Step 4: Find Large Files
Search for large individual files inside the directory.
```bash
find /path/to/directory -type f -size +500M -exec ls -lh {} \;
This identifies files larger than 500MB.
Large files are often logs, backups, or temporary data.
```

# Step 5: Sort Files by Size (Optional)
To quickly see the largest files first:
```bash
du -ah /path/to/directory | sort -hr | head -20
This lists the top 20 largest files or directories.
```


# Step 6: Take Action
Based on analysis, you can:

- Delete unnecessary files
- Compress or archive old data
- Rotate or clean log files
- Move data to another disk or storage
- ⚠️ Always verify file importance before deleting.
# Summary
df -h identifies filesystem-level usage
du helps locate space-consuming directories
find detects large files
Systematic analysis prevents accidental data loss
This approach is safe, efficient, and commonly used by Linux administrators.

# Scenario 02: File Deleted but Still in Use by a Process

## Problem Statement
A file has been deleted using `rm`, but disk space is not freed.
This usually happens when the file is still being used by a running process.
The goal is to identify the process and safely release the disk space.

---

## What Actually Happens
When a file is deleted in Linux:
- The file is removed from the directory structure.
- If a process is still using the file, the inode remains in memory.
- Disk space is not released until the process closes the file.

This is common with log files and long-running services.

---

## Step 1: Confirm Disk Space Issue
Check filesystem usage:

```bash
df -h
If space is still full after deleting files, the file may still be in use.
```
# Step 2: Find Deleted Files Still in Use
Use lsof to identify deleted but open files:
```bash
lsof | grep deleted
This command shows:
Process name
Process ID (PID)
File marked as (deleted)
```

# Step 3: Identify the Responsible Process
Example output:
```bash
java    1234  user  5w  REG  8,1  2G  /var/log/app.log (deleted)
```
Here:
- java is the process
- 1234 is the PID
- The deleted file is still consuming disk space

# Step 4: Release the Disk Space
You have two safe options:
```bash
Option 1: Restart the Service (Recommended)
systemctl restart service_name
Option 2: Kill the Process (If Restart Is Not Possible)
kill -9 <PID>
Once the process stops, the disk space is released immediately.
```
# Step 5: Verify Disk Space
Recheck disk usage:

df -h
The freed space should now be visible.


