# 📘 Day - CI/CD Basics Tasks

---

## 🧠 Task 1: The Problem

### ❌ What can go wrong?

* Developers overwrite each other’s code
* Wrong code gets deployed
* Someone forgets files
* Bugs go to production
* App breaks after deployment
* No proper testing

---

### ⚠️ What does "it works on my machine" mean?

👉 Code works on one developer’s system but fails on others.

### ❓ Why is it a problem?

* Different systems have different settings
* Different software versions
* Causes bugs in production
* Wastes time debugging

---

### ⏱️ Manual deployment frequency

👉 Around **1 time per day (safe)**

### ❓ Why?

* Manual process is slow
* High chance of mistakes
* Needs checking every time

---

### 🔥 Conclusion

* Manual = Slow + Risky
* Automation = Fast + Safe

---

## 🧠 Task 2: CI vs CD

### 🔄 Continuous Integration (CI)

Developers push code frequently.
Code is automatically built and tested.
It catches bugs early.

👉 Example: Tests run automatically on every GitHub push

---

### 📦 Continuous Delivery (CD)

Code is always ready to deploy.
Deployment is done manually.

👉 Example: Click button to deploy after testing

---

### 🚀 Continuous Deployment

Code is automatically deployed after tests pass.
No manual step required.

👉 Example: Code goes live automatically after push

---

### 🔥 Difference

* CI → Test code
* Delivery → Ready to release
* Deployment → Automatically live

---

## 🧠 Task 3: Pipeline Anatomy

### 🔔 Trigger

Starts the pipeline
👉 Example: Code push

---

### 🧱 Stage

Logical phase (build, test, deploy)

---

### ⚙️ Job

Work inside a stage

---

### 🔹 Step

Single command in a job
👉 Example: run tests

---

### 💻 Runner

Machine that runs jobs

---

### 📦 Artifact

Output of a job
👉 Example: Docker image

---

## 🧠 Task 4: CI/CD Pipeline Diagram

```
[ Developer Push Code ]
            ↓
        (Trigger)
            ↓
    ┌───────────────┐
    │   Stage 1     │
    │     Test      │
    │ Run tests     │
    └───────────────┘
            ↓
    ┌───────────────┐
    │   Stage 2     │
    │     Build     │
    │ Build Docker  │
    │   Image       │
    └───────────────┘
            ↓
    ┌───────────────┐
    │   Stage 3     │
    │    Deploy     │
    │ Deploy to     │
    │   Staging     │
    └───────────────┘
            ↓
   ✅ App Running on Staging
```

---

## 🧠 Task 5: Explore in the Wild (Based on YAML)
**repo = https://github.com/LondheShubham153/trivy-action/tree/master**
**file= test.yaml**
### 🔔 What triggers it?

* push
* pull_request
* workflow_dispatch (manual run)

👉 Runs automatically and manually

---

### ⚙️ Jobs

👉 **2 jobs:**

1. lint
2. test

---

### 🔍 What does it do?

#### 1️⃣ Lint Job

* Checks code quality
* Uses tool to find issues

---

#### 2️⃣ Test Job

* Downloads code
* Sets up testing tools (Bats)
* Installs Trivy (security scanner)
* Runs tests

---

### 🔥 Summary

* Trigger → push, PR, manual
* Jobs → 2
* Purpose → check code, run tests, security scan

---

## 📌 What I Learned (3 Key Points)

1. Manual deployment is risky and slow
2. CI/CD helps automate testing and deployment
3. Pipelines use stages, jobs, and steps to organize work
