# **Task 1: Git Merge — Hands-On**
# 1. Create a new branch feature-login from main, add a couple of commits to it
1️⃣ Make sure you're on the branch
```bash
git checkout main
git checkout -b feature-login
```
2️⃣ Create and switch to the new branch
```bash
git checkout -b feature-login
```


3️⃣ Make your first change and commit
```bash
Example:

# Create a login controller file
touch login.js

git add login.js
git commit -m "this my commit for testing"
```


# 2.  Merge `feature-login` into `main`

This guide explains how to switch to the `main` branch and merge the `feature-login` branch into it.

---

## 1️⃣ Switch to `main`

```bash
git checkout main

Or (for newer Git versions):

git switch main
```
2️⃣ Update main (Recommended)
```bash
Make sure your local main branch is up to date:
git pull origin main
```
3️⃣ Merge feature-login into main
```
git merge feature-login
````
4️⃣ Resolve Merge Conflicts (If Any)

- If Git reports conflicts:
- Open the conflicting files.
- Resolve the conflicts manually.
- Stage the resolved files:
```bash
git add .
```

5️⃣ Push Updated main Branch
```
git push origin main
```

# 3. Now create another branch feature-signup, add commits to it — but also add a commit to main before merging



## Objective
- Create a new branch `feature-signup`
- Add commits to it
- Add a separate commit to `main`
- Merge `feature-signup` into `main`
- Observe the merge behavior

---

## 1️⃣ Create and Switch to `feature-signup`

```bash
git checkout -b feature-signup

Or (newer Git versions):

git switch -c feature-signup
```

2️⃣ Add Commits to feature-signup
```
Make changes to your project files, then:

git add .
git commit -m "Add signup form UI"

Add another commit (optional example):

git add .
git commit -m "Add signup validation logic"
```
3️⃣ Switch Back to main
```
git checkout main
or
git switch main
```
4️⃣ Add a Commit to main Before Merging
```
Make a change directly on main, then:

git add .
git commit -m "Update homepage content"

⚠️ Now both main and feature-signup have new commits.
This will prevent a fast-forward merge.
```
5️⃣ Merge feature-signup into main
```
git merge feature-signup
```
---
# 4. Answer in your notes
***1. What is a fast-forward merge?***


A **fast-forward merge** in Git occurs when the branch you are merging into (e.g., `main`) has **no new commits** since the branch you are merging from (e.g., `feature-login`) was created.

In this case, Git can simply **move the pointer of the target branch forward** to the latest commit of the feature branch.  

No new merge commit is created, and the commit history remains a **straight line**.

---

## 2.  When Does Git Create a Merge Commit?

Git creates a **merge commit** when:

- Both branches have **new commits** since they diverged  
- The histories of the branches are **not linear**  
- Git cannot perform a fast-forward merge

In this case, Git combines the two histories into a **new commit** with **two parent commits**. This preserves the branch structure in the commit history.

---

## Example Scenario

1. Create a feature branch and add commits:

```bash 
id="m1p8sj"
git checkout -b feature-signup
# Make changes
git add .
git commit -m "Add signup form"
```
2. Switch to main and add a commit:
git checkout main
 ***Make a change***
git add .
git commit -m "Update homepage content"

3. Merge feature-signup into main:
git merge feature-signup

# Key Points

- Created when both branches have new commits
- Keeps a branching history
- Needed when a fast-forward merge is not possible
- Merge commit has two parent commits

# 3. What is a merge conflict?
A **merge conflict** occurs in Git when:

- Two branches modify the **same line of the same file**, or  
- Changes overlap in a way Git cannot automatically reconcile  

Git cannot decide which change to keep, so it **pauses the merge** and requires **manual resolution**.

---

## How Git Shows a Merge Conflict

When a conflict occurs during a merge:

```bash 
id="b2h7qt"
git merge feature-branch
```

Task 2: Git Rebase — Hands-On
# 1. write answer in .md file formate What does rebase actually do to your commits?
# Git Notes: Rebase



`git rebase` **moves or re-applies commits from one branch onto another**.  
It **rewrites the commit history** so that your branch appears as if it was created from the tip of the target branch.

---

## How It Works

Suppose you have the following history:

``` id="v2l5ct"
main:       A---B---C
feature:        D---E

```
If you run:

- git checkout feature
- git rebase main

Git will:

- Temporarily remove commits D and E from feature
- Move the feature branch pointer to the tip of main (commit C)
- Re-apply commits D and E on top of C

Key Points

- Rebase rewrites history; commits are reapplied on top of another branch
- Keeps a linear history without merge commits
- Can cause conflicts that must be resolved manually
- Use carefully for public branches — rewriting shared history can cause problems
- 
# 2. How is the history different from a merge?
# Git Notes: History Differences — Merge vs Rebase

## 1️⃣ Merge History

When you merge a feature branch into `main`:

```bash
git checkout main
git merge feature
```
- Git creates a merge commit if both branches have new commits
- The history preserves the true branching structure
- Shows exactly where the branch diverged and rejoined

main:    A---B---E---M
                \   /
feature:         C---D 

- M is the merge commit
- History shows the branch split and merge

# 2️⃣ Rebase History

When you rebase a feature branch onto main:
``` 
git checkout feature
git rebase main
```
- Git reapplies commits from the feature branch on top of main
- History becomes linear
- No merge commit is created (unless conflicts are resolved along the way)

Example
main:    A---B---C
feature:           D'---E'

D' and E' are the rebased commits (new hashes)

# Git Rebase Notes

| Branch   | Commits      | Notes                                         |
|----------|--------------|-----------------------------------------------|
| main     | A            | Initial commit                                |
| main     | B            | Development on main                           |
| main     | C            | Latest commit before rebase                   |
| feature  | D'           | Rebasing feature branch onto main, new hash  |
| feature  | E'           | Rebasing feature branch onto main, new hash  |

# Why should you never rebase commits that have been pushed and shared with others
When you rebase commits that have already been **pushed to a shared branch** (e.g., `main` or `develop`), Git **rewrites the commit history**.  

This changes the commit hashes for all rebased commits, which can cause serious problems for anyone else working on the same branch.

---

## Problems Caused by Rebasing Shared Commits

1. **History Mismatch**  
   - Your local branch now has new commit hashes  
   - Other developers’ copies still point to the old commits  

2. **Push Conflicts**  
   - If you try to push after rebasing, Git will reject it:  

   ```bash
   ! [rejected]        main -> main (non-fast-forward)
   
   ```
# Git Notes: When to Use Rebase vs Merge

## 1️⃣ When to Use Merge

Use **merge** when:

- You want to **preserve the complete branching history**  
- You are working on a **shared branch** (e.g., `main` or `develop`)  
- You want to **avoid rewriting commit history**  
- You want to clearly show **where features were developed**  

### Example Workflow

```bash 
git checkout main
git merge feature-login
```

# 2️⃣ When to Use Rebase

- Use rebase when:

- You want a clean, linear history
- You are working on a local or private branch that hasn’t been shared
- You want to update your feature branch with the latest changes from main
- You want to organize or squash commits before merging

Example Workflow
```
git checkout feature-login
git rebase main
```
- Reapplies commits on top of main
- No merge commit is created
- History becomes linear and easier to read
- Key Points:
- Makes logs cleaner
- Avoid rebasing branches that others are working on
- Can cause conflicts that need manual resolution
 ---

Task 3: Squash Commit vs Merge Commit
# Git Notes: Squash Merge

## 1️⃣ What Does Squash Merging Do?

A **squash merge** combines all commits from a feature branch into **a single commit** when merging into the target branch (`main`).  

- Keeps the history **linear and clean**  
- The individual commits from the feature branch do **not appear** on the main branch  
- Useful for condensing multiple small or WIP commits into one logical change

### Example

Suppose your feature branch has:

``` id="x1a9kj"
D - E - F (feature)
2️⃣ When to Use Squash Merge vs Regular Merge
Squash Merge ✅

When you want a clean main branch history

The feature branch contains many small or WIP commits

Before merging a short-lived feature branch that doesn’t need detailed history

Regular Merge ✅

When you want to preserve detailed history of the feature branch

When the feature branch is long-lived or collaborative

When seeing individual commits in main is important for auditing or review
```
3️⃣ Trade-Offs of Squashing

***Advantages:***

- Simplifies the main branch history
- Reduces clutter from trivial or incremental commits
-  Easier to read git log

***Disadvantages:***

- Loses the individual commit history of the feature branch
- Can make debugging or tracing changes harder
- Merge commit info may not show the true branching structure



