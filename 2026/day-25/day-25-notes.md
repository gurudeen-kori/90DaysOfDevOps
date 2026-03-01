## Task 1: Git Reset — Hands-On
**1. Make 3 commits in your practice repo (commit A, B, C)**

This guide walks through creating three commits (A, B, C) in a practice Git repository.

---

## Step 1: Initialize the Repository

If you do not already have a repository:

```bash
mkdir practice-repo
cd practice-repo
git init
```

# Step 2: Create Commit A

Create a new file and commit it.
```bash
echo "This is commit A" > file.txt
git add file.txt
git commit -m "Commit A: Add initial file"
```
# Step 3: Create Commit B

Append new content to the file and commit again.
```bash
echo "This is commit B" >> file.txt
git add file.txt
git commit -m "Commit B: Update file with second line"
```
# Step 4: Create Commit C

Append more content and create the third commit.
```bash
echo "This is commit C" >> file.txt
git add file.txt
git commit -m "Commit C: Update file with third line"
```
# Step 5: Verify the Commits

Run the following command to see your commit history:
```bash
git log --oneline
```
Expected output will look similar to:

![Alt Text](git log.png)

---

**2. Use git reset --soft to go back one commit — what happens to the changes?**

Commit C disappears from the commit history, BUT…

**✅ The changes from commit C are:**
- Still in your working directory
- Still staged (in the index)
- Ready to be committed again

**📦 Think of It Like This**

--soft = “Undo the commit, but keep everything prepared.”

- Deleted the commit
- But kept all the changes staged for a new commit

**Re-commit, then use git reset --mixed to go back one commit — what happens now?**

“Undo the commit and unstage the changes, but don’t delete the work.”

So Git:

- Moves HEAD back
- Clears the staging area
- Keeps your file modifications

**🧪 What You’ll See**

Run:
```bash
git status
```
**Now you’ll see:**

**3. Re-commit, then use git reset --hard to go back one commit — what happens this time**

Changes not staged for commit:❌ What Happens to the Changes from Commit C?

They are:

- ❌ Removed from commit history
- ❌ Removed from the staging area
- ❌ Removed from your working directory

They are gone.

Your project files now look exactly like they did at commit B.

**📦 Think of --hard Like This**

“Make everything exactly like the previous commit — erase everything after it.”

Git resets:

HEAD

- The staging area (index)
- Your working directory files

**🧪 What You’ll See**

If you run:
```bash
git status
```
You’ll see:

- nothing to commit, working tree clean
- And if you open file.txt, the changes from commit C will be missing.

**⚠️ Important Warning**

git reset --hard is destructive.

If commit C was never pushed and you didn’t save it anywhere else, it may be difficult to recover (though sometimes possible using git reflog).
    modified: `file.txt`
    
## 5. Answer in your notes
# Difference Between `git reset --soft`, `--mixed`, and `--hard`

The `git reset` command is used to move the current branch (HEAD) to a specific commit.  
The difference between `--soft`, `--mixed`, and `--hard` determines what happens to:

- The commit history
- The staging area (index)
- The working directory (your files)
- 
## 1️⃣ git reset --soft**
**What it does:**

- Moves HEAD back one commit
- Keeps changes in the staging area
- Keeps changes in your working directory

**Result:**

- Commit C is removed from history
- Changes from C are still staged
- Ready to commit again immediately



## 2️⃣ git reset --mixed 
**What it does:**

- Moves HEAD back one commit
- Unstages the changes
- Keeps changes in your working directory

**Result:**

- Commit C is removed from history
-  Changes from C are NOT staged
- You must run git add before committing again

## 3️⃣ git reset --hard
**What it does:**

- Moves HEAD back one commit
- Removes changes from staging area
- Removes changes from working directory

**Result:**
- Commit C is removed from history
- Changes from C are permanently deleted
- Project files return exactly to commit B state

**⚠️ Warning: This is destructive and can permanently delete work.**
## Which one is destructive and why

## ✅ The Destructive Option: `git reset --hard`

The destructive reset mode is:

```bash
git reset --hard 7e2c8e5
```
🔎 Why Is It Destructive?

--hard resets:

- The commit history (moves HEAD back)
- The staging area (clears it)
- The working directory (overwrites your files)
- This means:
- The last commit is removed 
- All changes from that commit are deleted

Your files are restored exactly to the previous commit state
#  **When would you use each one?**
# Git Reset Modes Summary

| Mode     | Keep Staged? | Keep Working Directory? | Use Case Summary                         |
|----------|--------------|------------------------|-----------------------------------------|
| --soft   | Yes          | Yes                    | Edit/recommit last commit               |
| --mixed  | No           | Yes                    | Unstage changes, reorganize commits    |
| --hard   | No           | No                     | Discard commits and changes, reset to previous state |

# Task 2: Git Revert — Hands-On
**1. How is git revert different from git reset?**

| Feature              | `git reset`                | `git revert`                 |
| -------------------- | -------------------------- | ---------------------------- |
| Alters history       | Yes                        | No                           |
| Safe for shared repo | No (if commits pushed)     | Yes                          |
| Deletes changes      | Can delete (`--hard`)      | No, creates inverse commit   |
| Usage                | Local undo, branch cleanup | Undo changes safely publicly |

**2. Why is revert considered safer than reset for shared branches?**
## 1️⃣ `git reset` Changes History

- `git reset` **moves the branch pointer** (HEAD) backward in history.  
- Commits that existed after the reset are **removed from the branch history**.  
- If the branch has already been pushed to a remote repository:

  - Other team members still have the old commits.
  - Pushing a reset branch may cause **conflicts, lost work, and confusion**.
  - Team members may need to force-pull or rebase to sync.
  - 
  
# 2️⃣ git revert Preserves History

- git revert does not remove commits.
- It creates a new commit that reverses the changes of a specific commit.
- History remains linear and intact, so other collaborators are unaffected.
- 

# Task 3: Reset vs Revert — Summary

| Feature                       | git reset                                           | git revert                                     |
|--------------------------------|---------------------------------------------------|------------------------------------------------|
| What it does                   | Moves HEAD to a previous commit; optionally changes staging area and working directory (`--soft`, `--mixed`, `--hard`) | Creates a new commit that undoes the changes of a specified commit; history remains intact |
| Removes commit from history?    | Yes (can remove one or more commits)             | No (original commit stays; new inverse commit added) |
| Safe for shared/pushed branches?| ❌ No (rewriting history can break other collaborators) | ✅ Yes (history is preserved; safe to push)   |
| When to use                     | Undo local commits, cleanup branch, change last commit, or discard changes (`--hard`) | Undo changes on shared/public branches safely, revert mistakes without rewriting history |

# Task 4: Branching Strategies

# Git Branching Strategies Notes

---

## 1️⃣ Gitflow

### How it works (short description)
Gitflow uses multiple long‑lived branches to separate development, releases, and fixes. The main branches are `main` (production) and `develop` (integration). Feature, release, and hotfix branches branch off and merge back into these. :contentReference[oaicite:0]{index=0}

### Simple diagram / flow (text‑based)
feature/* → develop
release/* → develop & main
hotfix/* → main & develop
main ← develop ← feature

### When/where it's used
Large teams or projects with **scheduled releases**, semantic versioning, and formal release gating. :contentReference[oaicite:1]{index=1}

### Pros
- Clear separation of roles for different work (features, releases, hotfixes). :contentReference[oaicite:2]{index=2}
- Works well for managing multiple parallel versions. :contentReference[oaicite:3]{index=3}
- Keeps production branch stable. :contentReference[oaicite:4]{index=4}

### Cons
- Complex and heavy (many long‑lived branches). :contentReference[oaicite:5]{index=5}
- Merge overhead and possible merge conflicts. :contentReference[oaicite:6]{index=6}
- Not ideal for fast CI/CD workflows. :contentReference[oaicite:7]{index=7}

---

## 2️⃣ GitHub Flow

### How it works (short description)
A simple workflow with one long‑lived branch (`main`) and short‑lived feature branches. Developers branch off `main`, open pull requests, review, and then merge back into `main`. :contentReference[oaicite:8]{index=8}

### Simple diagram / flow (text‑based)
main ← feature/one
main ← feature/two


### When/where it's used
Teams practicing **continuous delivery** and **frequent deployments**, especially web apps or small teams. :contentReference[oaicite:9]{index=9}

### Pros
- Simple and lightweight. :contentReference[oaicite:10]{index=10}
- Encourages frequent, small merges (fast feedback). :contentReference[oaicite:11]{index=11}
- Works well with pull requests and CI/CD. :contentReference[oaicite:12]{index=12}

### Cons
- Harder to manage multiple versions/releases. :contentReference[oaicite:13]{index=13}
- If tests are weak, unstable code can reach `main`. :contentReference[oaicite:14]{index=14}

---

## 3️⃣ GitLab Flow

### How it works (short description)
GitLab Flow mixes ideas from GitHub Flow and Gitflow. It uses environment branches (like `staging` and `production`) and feature branches. Features merge into testing or environment branches, and deploy accordingly. :contentReference[oaicite:15]{index=15}

### Simple diagram / flow (text‑based)
feature/* → staging → production
feature/* → main → preproduction → production


### When/where it's used
Teams oriented toward **continuous integration and deployment** with strong automated testing and feature flags. :contentReference[oaicite:22]{index=22}

### Pros
- Minimizes long‑lived branches → fewer merge conflicts. :contentReference[oaicite:23]{index=23}
- Works well with CI/CD and fast releases. :contentReference[oaicite:24]{index=24}
- Keeps main releasable at all times. :contentReference[oaicite:25]{index=25}

### Cons
- Requires strong testing, discipline, and feature flagging. :contentReference[oaicite:26]{index=26}
- Not always suitable for very large teams without strong coordination. :contentReference[oaicite:27]{index=27}

---

## 5️⃣ Feature Branch Workflow (Basic)

### How it works (short description)
Each new feature or fix is developed in a separate branch off main (or develop). When complete and reviewed, it merges back via pull request. Often used in tandem with other flows. :contentReference[oaicite:28]{index=28}

### Simple diagram / flow (text‑based)

✅ Summary Tips

- Staging: git add prepares files for commit
- Committing: git commit -m "msg" records changes
- Branching: git branch + git switch for parallel development
- Merging vs Rebase: Merge preserves branch history; rebase keeps it linear
- Remote Collaboration: Use push, pull, fetch carefully
- Undo Changes: reset (local), revert (public)
- Temporary Changes: Use stash to save uncommitted work



