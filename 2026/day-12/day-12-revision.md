
---
#  I am learning DevOps & Cloud Engineering because: 
-   The industry demand is high and growing globally 
-  DevOps engineers work on real production systems, not just theory 
-   It offers strong career growth, flexibility, and remote opportunities 
-   I want to understand how software actually runs in the real world 
-   I want a role where I can combine: o Problem-solving o Automation o Systems thinking My goal is to move beyond basic development or IT knowledge and become someone who can design, deploy, automate, and maintain scalable systems
---

## ps — Process Inspection

### Base command
```bash
ps
Shows processes attached to the current terminal only.
```
---
# Flags (used individually)
- ps a             → show processes for all users with a terminal

- ps u
- u → user-oriented output (USER, %CPU, %MEM, VSZ, RSS)

- ps x
x → include processes without a controlling terminal (daemons)

Common combined usage
```bash
ps aux
```
- a → all users
- u → user-friendly format
- x → include background services

# Sorting examples
```bash
ps -aux --sort=-%cpu
ps -aux --sort=-%mem
Sort processes by CPU or memory usage (descending).
```
# systemctl status — Service Health
Basic status check
```bash
systemctl status ssh
```
# Displays:
---
- Loaded state
- Active state
- Main PID
- Recent logs


# Basic service logs
```bash
journalctl -u ssh
```

```bash
journalctl -f   →  
```
```
journalctl --since today
```
```
journalctl --since "2026-02-08 10:00"

```
# Output Size & Navigation
`journalctl -n 50`

# Priority / Severity Filtering
```bash 
journalctl -p err
journalctl -p warning
journalctl -p 3
```
# 1. Which 3 commands save you the most time right now, and why?
---
- ls -l — instantly shows permissions, ownership, and file type; catches mistakes fast.
- systemctl status <service> — quickest way to see if a service is running, failed, or misconfigured.

- journalctl -u <service> -n 50 — gets straight to recent, relevant logs without noise.

# 2. How do you check if a service is healthy?

First commands I run:

- systemctl status <service>
- journalctl -u <service> -n 50 --no-pager
- ps aux | grep <service>


Confirms service state

Checks for recent errors

Verifies the process is actually running

# 3. How do you safely change ownership and permissions without breaking access?

```I change ownership first, then permissions, and always verify.```

Example:
```bash
sudo chown appuser:appgroup /var/app/data
sudo chmod 750 /var/app/data
ls -l /var/app/data
```

# 4. What will you focus on improving in the next 3 days?
---
- Faster interpretation of journalctl logs
- More confidence with permissions (chmod numeric modes)
- Building a consistent “incident command flow” without hesitation
---