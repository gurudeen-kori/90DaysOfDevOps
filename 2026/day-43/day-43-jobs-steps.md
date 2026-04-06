# Day 43 – Jobs, Steps, Env Vars & Conditionals

## 🔹 Key Concepts

### needs:
- Used to create dependency between jobs
- Ensures jobs run in order

Example:
test runs after build

### outputs:
- Used to pass data from one job to another
- Helpful for dynamic pipelines

---

## 🔹 Why pass outputs between jobs?

- Share dynamic values (like date, version, build ID)
- Avoid recomputing values
- Helps in real CI/CD (e.g., passing Docker image tag)

---

## 🔹 Environment Variables

Levels:
- Workflow level → global
- Job level → specific job
- Step level → specific step

---

## 🔹 Conditionals

- Run only on main branch
- Run on failure
- Run only on push events
- continue-on-error: true → step fails but workflow continues

---

## 🔹 Learnings

- Built multi-job pipeline
- Understood job dependencies
- Used environment variables
- Passed data between jobs
- Applied condition-based execution

---

## 🔹 Cron Expression

Every Monday at 9 AM:
0 9 * * 1