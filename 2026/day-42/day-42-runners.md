# 🚀 Day 42 – GitHub Runners (Hosted & Self-Hosted)

## 🔹 Task 1: GitHub-Hosted Runners

### Workflow YAML
```yaml
name: Hosted Runners Demo

on: push

jobs:
  ubuntu-job:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "OS: Ubuntu"
          hostname
          whoami

  windows-job:
    runs-on: windows-latest
    steps:
      - run: |
          echo "OS: Windows"
          hostname
          whoami

  mac-job:
    runs-on: macos-latest
    steps:
      - run: |
          echo "OS: macOS"
          hostname
          whoami
```

### 📝 Notes
- GitHub-hosted runner = GitHub ke servers par temporary VM
- GitHub hi manage karta hai (setup, updates, security)

---

## 🔹 Task 2: Pre-installed Tools

### YAML
```yaml
- name: Check tools
  run: |
    docker --version
    python --version
    node --version
    git --version
```

### 📝 Notes
- Runners me tools already installed hote hain
- Time save hota hai (install karne ki zarurat nahi)
- Faster CI/CD pipelines 🚀

---

## 🔹 Task 3: Self-Hosted Runner Setup

### Steps:
1. GitHub Repo → Settings → Actions → Runners
2. Click: **New self-hosted runner**
3. Select: Linux
4. Commands copy karo aur machine me run karo:
   ```bash
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner-linux-x64.tar.gz -L <url>
   tar xzf actions-runner-linux-x64.tar.gz
   ./config.sh --url <repo-url> --token <token>
   ./run.sh
   ```

### ✅ Verification:
- Runner list me **green dot (Idle)** dikhega

---

## 🔹 Task 4: Use Self-Hosted Runner

### YAML
```yaml
name: Self Hosted Test

on: push

jobs:
  test:
    runs-on: self-hosted
    steps:
      - name: Print hostname
        run: hostname

      - name: Print working directory
        run: pwd

      - name: Create file
        run: echo "Hello from self-hosted runner" > test.txt
```



---

## 🔹 Task 5: Labels

### YAML
```yaml
runs-on: [self-hosted, my-linux-runner]
```



---

## 🔹 Task 6: Comparison Table

| Feature              | GitHub-Hosted           | Self-Hosted                |
|---------------------|------------------------|---------------------------|
| Who manages it?     | GitHub                 | You                       |
| Cost                | Limited free / paid    | Your server cost          |
| Pre-installed tools | Yes                    | You install               |
| Good for            | Quick CI/CD            | Custom setups             |
| Security concern    | Less control           | Full control              |

---

## 📸 Screenshots
(Add screenshots here)

1. Self-hosted runner (Idle)
2. Workflow running on self-hosted runner

---

## ✅ Submission
- File path: `2026/day-42/day-42-runners.md`
- Commit & push 🚀