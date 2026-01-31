# 90DaysOfDevOps   day-04
# Process and Job Control Commands (Linux)

The following commands demonstrate process inspection, parent/child relationships,
and background job management in a Linux shell.

```bash
ps -C bash
echo $$ $PPID
ps fx
ps fx | grep bash
ps -ef
pstree
pstree -p
jobs
sleep 200 &
jobs
```
` ps fx | grep bash `
```
   3504 pts/0    Ss     0:00  |   \_ bash
   3885 pts/0    S+     0:00  |      \_ grep --color=auto bash 
   
   ```
 # Service Management Command (Linux)

The following command is used to check the status of a system service using `systemctl`.

## Command Explanation

```bash
systemctl status sshd
systemctl enable sshd --now 

```

# Lab Manual: Rescue Mode to Default Target Troubleshooting

## Objective
To understand how to boot a Linux system into rescue mode, perform basic
troubleshooting, and return the system to the default target.

## Requirements
- Linux system using `systemd`
- Root or sudo privileges

## Theory
Rescue mode provides a minimal single-user environment used to repair system
issues such as failed services, filesystem errors, or boot problems. After
troubleshooting, the system can be returned to its default target.


### Step 1: Check the Default Target
```bash
systemctl get-default ` for view current target
systemctl set-default < target name >

```
# Hands-On journalctl and tail

## Objective
To analyze system and service logs on an Ubuntu system using `journalctl`
and `tail` commands for troubleshooting purposes.

## Requirements
- Ubuntu Linux ( 22.04 Lts )
- Root or sudo privileges

## Theory
Ubuntu uses `systemd` for service management and logging. System logs are
primarily handled by the systemd journal and stored in `/var/log/journal`.
Traditional log files are stored under `/var/log`, with `/var/log/syslog`
being the main system log file.

## Commands and Working

### 1. View Logs for a Specific Service (Ubuntu)
```bash
journalctl -u <service-name>


