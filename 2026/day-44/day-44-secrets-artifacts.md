# Day 44 – Secrets, Artifacts & Tests

## 🔐 Secrets
- Secrets are masked in logs (***)
- Never print secrets → security risk

## 📦 Artifacts
- Used to store files between jobs
- Example: logs, reports, builds

## 🔁 Artifact Use Case
- Passing build output to deploy job

## 🧪 Tests
- CI fails if script exits non-zero
- Helps catch bugs early

## ⚡ Caching
- Speeds up dependency installation
- Stored in GitHub cache system