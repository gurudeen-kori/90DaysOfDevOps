
# 🚀 Day 36 - Docker Project

## 📝 Application Choice

I built a **Todo Application** using Flask, PostgreSQL, and Docker.

### ❓ Why this app?
- Simple but practical use-case (task management)
- Covers full-stack concepts (Frontend + Backend + Database)
- Easy to test API + UI
- Perfect for learning Docker multi-container setup

---

## 🐳 Dockerfile (with Explanation)

```dockerfile
FROM python:3.11-alpine
# Use lightweight Python base image (small size)

RUN apk add --no-cache postgresql-client
# Install PostgreSQL client for DB connectivity

RUN addgroup -g 1000 -S appgroup && \
    adduser -S appuser -u 1000 -G appgroup
# Create non-root user for security

WORKDIR /app
# Set working directory inside container

COPY app/ .
# Copy application code into container

RUN pip install --no-cache-dir -r requirements.txt
# Install Python dependencies

USER appuser
# Switch to non-root user

EXPOSE 5000
# Expose application port

CMD ["python", "app.py"]
```
# Start Flask applicatio
Task 1: Pick Your App
todo-app/
│── Dockerfile
│── docker-compose.yml
│── app/
│   ├── app.py
│   ├── models.py
│   ├── requirements.txt
│   └── templates/
---
## CODE 
Link : https://github.com/gurudeen-kori/todo-app/tree/main

# ports 
- 5432 postgres
- 5000 app 
---
## image 
- python:3.13-slim
- python:3.14-alpine
---
## dependencies 

Flask==2.3.3
Flask-SQLAlchemy==3.0.5
python-dotenv==1.0.0
psycopg2-binary==2.9.9
---

# ⚔️ Challenges Faced & Solutions
## ❌ Issue 1: requirements.txt not found

- Problem: Docker build failed with "No such file or directory"
- Cause: Wrong COPY path
- Solution:
 `Used COPY app/ . to match project structure`

## ❌ Issue 2: App not opening on localhost

- Problem: Browser showed nothing
- Cause: Flask binding issue
- Solution: Used:
     `app.run(host='0.0.0.0', port=5000)`
## ❌ Issue 3: Database connection issue

- Problem: App couldn't connect to PostgreSQL
- Cause: Wrong hostname (localhost)
- Solution: Used Docker service name:
`db`

# 📦 Final Image Size

Image Name: todo-app

Size: ~108 MB

Based on python:3.11-alpine (lightweight)

## 🌐 Docker Hub Link

👉 https://hub.docker.com/r/gurudeenkori/todo-app

## 🎯 Conclusion

**This project helped me understand:**

- Dockerfile creation
- Multi-container apps using Docker Compose
- Environment variables handling
- Debugging container issues
- Pushing images to Docker Hub

## 👨‍💻 Author

***Gurudeen Kori***
