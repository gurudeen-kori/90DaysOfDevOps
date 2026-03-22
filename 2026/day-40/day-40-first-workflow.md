# 📘 GitHub Actions Practice (All Tasks in One File)

## 🚀 Task 1: Set Up

### Steps:

1. Create a new public GitHub repository:

   ```
   github-actions-practice
   ```

2. Clone it locally:

   ```bash
   git clone https://github.com/<your-username>/github-actions-practice.git
   cd github-actions-practice
   ```

3. Create folder structure:

   ```bash
   mkdir -p .github/workflows
   ```

---

## 🚀 Task 2: Hello Workflow

### Create file:

```
.github/workflows/hello.yml
```

### Code:

```yaml
name: Hello Workflow

on: push

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print Hello
        run: echo "Hello from GitHub Actions!"
```

### Run Commands:

```bash
git add .
git commit -m "Added hello workflow"
git push
```

👉 Go to GitHub → Actions tab → Check if it's ✅ green

---

## 🚀 Task 3: Understand the Anatomy

### Key Concepts:

* **on:**
  Defines when the workflow runs (e.g., push, pull_request)

* **jobs:**
  A workflow contains one or more jobs

* **runs-on:**
  Defines the runner machine (e.g., ubuntu-latest)

* **steps:**
  Sequence of tasks inside a job

* **uses:**
  Used to call pre-built actions (e.g., actions/checkout)

* **run:**
  Executes shell commands

* **name (step):**
  Human-readable name for each step

---

## 🚀 Task 4: Add More Steps

### Updated `hello.yml`:

```yaml
name: Hello Workflow

on: push

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print Hello
        run: echo "Hello from GitHub Actions!"

      - name: Print Date & Time
        run: date

      - name: Print Branch Name
        run: echo "Branch: ${{ github.ref_name }}"

      - name: List Files
        run: ls -la

      - name: Print OS
        run: echo "Runner OS: $RUNNER_OS"
```

👉 Push again and check new run in Actions tab

---

## 🚀 Task 5: Break It On Purpose

### Add failing step:

```yaml
      - name: Fail Step
        run: exit 1
```

### What happens:

* Workflow becomes ❌ failed
* Steps after failure do NOT run

---

### Fix It:

Remove or correct the failing step:

```yaml
# remove or comment the failing step
```

Push again → should turn ✅ green

---

## 🧠 What I Learned

1. GitHub Actions automates build, test, and deployment
2. YAML indentation is very important
3. One failed step stops the whole pipeline
4. Logs help debug errors
5. Pre-built actions save time

---

## ❗ Understanding Failed Pipeline

* Failed pipeline shows ❌ red status
* Click the job → find the failed step
* Read logs to identify error

### Example Error:

```
Process completed with exit code 1
```

---

## 🎯 Final Result

✅ Created your first workflow
✅ Learned YAML structure
✅ Added multiple steps
✅ Understood debugging in CI/CD

---

🔥 Next Step: Build full CI/CD pipeline (Build + Test + Deploy)
