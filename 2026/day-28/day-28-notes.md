# Day 28 – Revision Day: Everything from Day 1 to Day 27

## Task

You've covered a lot of ground in 27 days — DevOps fundamentals, Linux deep dives, Shell scripting, Python basics, Git & GitHub, and even your developer branding. Today, **stop and revise**. No new concepts. Just solidify what you've learned.

The goal is to identify gaps, revisit topics you struggled with, and make sure you can confidently explain and use everything covered so far.

---

## What You've Covered So Far

| Days | Topic | Key Concepts |
|------|-------|-------------|
| 1 | DevOps & Cloud Intro | What is DevOps, SDLC, Cloud basics |
| 2–7 | Linux Fundamentals | Architecture, commands, processes, systemd, file system hierarchy, troubleshooting, text files |
| 8 | Cloud Server Setup | Docker, Nginx, web deployment |
| 9–11 | Users, Permissions & Ownership | User/group management, file permissions, chown/chgrp |
| 12 | Revision Day 1 | Days 1–11 recap |
| 13 | Volume Management | LVM — physical volumes, volume groups, logical volumes |
| 14–15 | Networking | Fundamentals, DNS, IP, subnets, ports, hands-on checks |
| 16–18 | Shell Scripting | Basics, loops, arguments, error handling, functions |
| 19–20 | Shell Scripting Projects | Log rotation, backup, crontab, log analyzer |
| 21 | Shell Scripting Cheat Sheet | Personal reference guide |
| 22–25 | Git & GitHub | Init, branching, merge, rebase, stash, cherry pick, reset, revert, branching strategies |
| 26 | GitHub CLI | Managing GitHub from the terminal |
| 27 | GitHub Profile | Profile README, repo organization, developer branding |

---

## Challenge Tasks

### Task 1: Self-Assessment Checklist
Go through the checklist below. For each item, mark yourself honestly:
- **Can do confidently**
- **Need to revisit**
- **Haven't done yet**

#### Linux
- [ ] Navigate the file system, create/move/delete files and directories
- [ ] Manage processes — list, kill, background/foreground
- [ ] Work with systemd — start, stop, enable, check status of services
- [ ] Read and edit text files using vi/vim or nano
- [ ] Troubleshoot CPU, memory, and disk issues using top, free, df, du
- [ ] Explain the Linux file system hierarchy (/, /etc, /var, /home, /tmp, etc.)
- [ ] Create users and groups, manage passwords
- [ ] Set file permissions using chmod (numeric and symbolic)
- [ ] Change file ownership with chown and chgrp
- [ ] Create and manage LVM volumes
- [ ] Check network connectivity — ping, curl, netstat, ss, dig, nslookup
- [ ] Explain DNS resolution, IP addressing, subnets, and common ports

#### Shell Scripting
- [ ] Write a script with variables, arguments, and user input
- [ ] Use if/elif/else and case statements
- [ ] Write for, while, and until loops
- [ ] Define and call functions with arguments and return values
- [ ] Use grep, awk, sed, sort, uniq for text processing
- [ ] Handle errors with set -e, set -u, set -o pipefail, trap
- [ ] Schedule scripts with crontab

#### Git & GitHub
- [ ] Initialize a repo, stage, commit, and view history
- [ ] Create and switch branches
- [ ] Push to and pull from GitHub
- [ ] Explain clone vs fork
- [ ] Merge branches — understand fast-forward vs merge commit
- [ ] Rebase a branch and explain when to use it vs merge
- [ ] Use git stash and git stash pop
- [ ] Cherry-pick a commit from another branch
- [ ] Explain squash merge vs regular merge
- [ ] Use git reset (soft, mixed, hard) and git revert
- [ ] Explain GitFlow, GitHub Flow, and Trunk-Based Development
- [ ] Use GitHub CLI to create repos, PRs, and issues

---

### Task 2: Revisit Your Weak Spots
1. Pick **3 topics** from the checklist where you marked "Need to revisit"
- Use grep, awk, sed, sort, uniq for text processing
- Handle errors with set -e, set -u, set -o pipefail, trap
- Use git stash and git stash po
2. Go back to that day's challenge and redo the hands-on tasks
3. Document what you re-learned in `day-28-notes.md`

---

### Task 3: Quick-Fire Questions
Answer these from memory (no Googling). Then verify your answers:

1. What does `chmod 755 script.sh` do?
**Answer** 
- `chmod` - it is command for change permission 
- `755` - rwxr-xr-x 
- `script.sh` it is a file  
2. What is the difference between a process and a service?
- **Process** - Process is a running program. At a particular instant of time, it can be either running, sleeping, or zombie (completed process, but waiting for it's parent process to pick up the return value).
- **Daemons** They are the processes which run in the background and are not interactive. They have no controlling terminal



3. How do you find which process is using port 8080?
**Using netstat**
```bash 
netstat -tulpn | grep 8080
```
4. What does `set -euo pipefail` do in a shell script?
`set -euo pipefail` is a best practice used in Bash scripts to make scripts safer and fail fast when errors occur.
5. What is the difference between `git reset --hard` and `git revert`?
**Ans.**
`git reset --hard` removes commits and resets the working directory to a previous state, rewriting history.
`git revert` creates a new commit that reverses the changes of a previous commit without modifying history, making it safer for shared repositories.

6. What branching strategy would you recommend for a team of 5 developers shipping weekly?
**Ans.**

**1. Main Branch (main)**

- Always contains production-ready code
- Protected branch (no direct commits)
- Only merged via Pull Requests

**2. Develop Branch (develop)**

- Integration branch where developers merge their completed work

- Used for testing and preparing the weekly release
---

***3. Feature Branches (feature/*)***
Each developer creates a branch for a specific feature or task.

**Example:**
```
git checkout -b feature/login-api
```
Create feature branch from develop

Work on the feature
- Push branch
- Create Pull Request to develop
- Code review + merge

7. What does `git stash` do and when would you use it?
8. How do you schedule a script to run every day at 3 AM?
```
crontab -e
```
`0 3 * * * /home/student/Day_03/script.sh`
9. What is the difference between `git fetch` and `git pull`?

This note explains the difference between `git fetch` and `git pull`, which is a common DevOps and Git interview topic.

---

## 1. `git fetch`

- **Definition:** Downloads changes from the remote repository but **does not merge them** into your local branch.  
- **Purpose:** To check for remote changes before integrating them.

### Example

```bash
git fetch origin
```

## 2. `git pull`

Definition: Downloads changes from the remote and immediately merges them into your current branch.

Purpose: To update your local branch in one step.

Example
```bash
git pull origin main
```
10. What is LVM and why would you use it instead of regular partitions?

---

### Task 4: Organize Your Work
1. Make sure all your daily submissions (day-1 through day-27) are committed and pushed
2. Check that your `git-commands.md` is up to date
3. Check that your shell scripting cheat sheet is complete
4. Verify your GitHub profile and repos are clean (from Day 27)

---

### Task 5: Teach It Back
Pick **one topic** you've learned and write a short explanation (5-10 lines) as if you're teaching it to someone who has never heard of it. Add it to your `day-28-notes.md`.

Examples:
- Explain Git branching to a non-developer
- Explain file permissions to a new Linux user
- Explain what a crontab is and why sysadmins use it

Teaching is the best test of understanding.

---



Happy Learning!
**TrainWithShubham**