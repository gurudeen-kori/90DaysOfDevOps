# Task 1: Capture Quick System Health Snapshot (Ubuntu)

## Objective
To collect a snapshot of the system's CPU, memory, disk, and network status on an Ubuntu system.

---

## Commands and Output

### 1. CPU Usage
```bash
[student@docker ~]$ top -b -n 1 | head -n 10

top - 21:42:57 up  1:50,  2 users,  load average: 0.08, 0.03, 0.03
Tasks: 310 total,   2 running, 308 sleeping,   0 stopped,   0 zombi
%Cpu(s):  9.5 us,  9.5 sy,  0.0 ni, 78.6 id,  0.0 wa,  0.0 hi,  2.4
MiB Mem :   2755.8 total,    531.9 free,   1595.7 used,    814.8 bu
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   1160.1 av

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM
   4023 student   20   0  226012   4360   3476 R  10.5   0.2
   3486 student   20   0  772996  55136  39620 S   5.3   2.0
      1 root      20   0  107140  18384  10876 S   0.0   0.7

```
# 🔍 Trace Logs for a Linux Service

## 1️⃣ Using `journalctl` (systemd-based systems)

```bash
# View all logs for a specific service
journalctl -u sshd

# Follow logs in real-time
journalctl -u sshd -f

# Show logs for today only
journalctl -u sshd --since today

# Show last 50 log lines
journalctl -u sshd -n 50
```
# What’s a runbook?
A runbook is a short, repeatable checklist you follow during an incident: the exact commands you run, what you observed, and the next actions if the issue persists. Keep it concise so you can reuse it under pressure.
# 📝 Mini Runbook: Tracing and Troubleshooting a Linux Service

**Service:** sshd  
**Date:** 31-Jan-2026  
**Owner:** Gurudeen Kori  

---

## 1️⃣ What I Did

1. **Checked service status**  
```bash
systemctl status sshd
# Ubuntu/Debian
tail -f /var/log/syslog | grep sshd
```



