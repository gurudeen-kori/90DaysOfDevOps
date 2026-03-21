# Day 38 - YAML Practice (DevOps Learning)

---

## 📁 Task 1: Key-Value Pairs

### person.yaml

```yaml
name: Gurudeen Kori
role: DevOps Engineer (Learning Phase)
experience_years: 0
learning: true
```

### Verification

```bash
cat person.yaml
```

---

## 📁 Task 2: Lists

### Updated person.yaml

```yaml
name: Gurudeen Kori
role: DevOps Engineer (Learning Phase)
experience_years: 0
learning: true

tools:
  - Docker
  - Kubernetes
  - GitHub Actions
  - Jenkins
  - Terraform

hobbies: [coding, learning, watching-tech-videos]
```

### Notes

* Block format → uses `-`
* Inline format → uses `[ ]`

---

## 📁 Task 3: Nested Objects

### server.yaml

```yaml
server:
  name: my-server
  ip: 192.168.1.10
  port: 8080

database:
  host: localhost
  name: mydb
  credentials:
    user: admin
    password: secret123
```

---

## 📁 Task 4: Multi-line Strings

### Updated server.yaml

```yaml
startup_script_literal: |
  #!/bin/bash
  echo "Starting server..."
  sudo systemctl start nginx
  echo "Server started"

startup_script_folded: >
  #!/bin/bash
  echo "Starting server..."
  sudo systemctl start nginx
  echo "Server started"
```

### Notes

* `|` → preserves new lines (best for scripts)
* `>` → converts into single line (best for long text)

---

## 📁 Task 6: Spot the Difference

### Correct YAML

```yaml
name: devops
tools:
  - docker
  - kubernetes
```

### Broken YAML

```yaml
name: devops
tools:
- docker
  - kubernetes
```

### Issue

* Indentation mismatch in list items
* YAML structure breaks due to inconsistent spacing

---

## 📚 What I Learned (Key Points)

1. YAML is **indentation-based** — spaces define structure, not brackets
2. Lists can be written in **two ways**: block (`-`) and inline (`[ ]`)
3. Multi-line strings use `|` (preserve format) and `>` (fold into one line)

---

## 🚀 Conclusion

YAML is simple but very sensitive to formatting.
It is widely used in **DevOps tools like Docker, Kubernetes, and CI/CD pipelines**, so mastering it is important.

---

🔥 Day 38 Completed
