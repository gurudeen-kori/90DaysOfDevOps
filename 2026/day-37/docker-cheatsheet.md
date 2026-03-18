# 🐳 Docker Cheat Sheet

## 📦 Container Commands
- `docker run -d -p 8080:80 nginx` — Run container in detached mode with port mapping
- `docker ps` — List running containers
- `docker ps -a` — List all containers
- `docker stop <container>` — Stop a running container
- `docker rm <container>` — Remove a container
- `docker exec -it <container> bash` — Access container shell
- `docker logs <container>` — View container logs

---

## 🖼️ Image Commands
- `docker build -t myapp:latest .` — Build image from Dockerfile
- `docker pull nginx` — Download image from Docker Hub
- `docker push myapp:latest` — Push image to Docker Hub
- `docker tag myapp:latest username/myapp:latest` — Tag image for registry
- `docker images` — List images
- `docker rmi <image>` — Remove an image

---

## 💾 Volume Commands
- `docker volume create myvolume` — Create a volume
- `docker volume ls` — List volumes
- `docker volume inspect myvolume` — Show volume details
- `docker volume rm myvolume` — Remove a volume

---

## 🌐 Network Commands
- `docker network create mynetwork` — Create a network
- `docker network ls` — List networks
- `docker network inspect mynetwork` — Show network details
- `docker network connect mynetwork <container>` — Connect container to network

---

## ⚙️ Docker Compose Commands
- `docker compose up -d` — Start services in background
- `docker compose down` — Stop and remove services
- `docker compose ps` — List running services
- `docker compose logs` — View service logs
- `docker compose build` — Build services

---

## 🧹 Cleanup Commands
- `docker system prune -a` — Remove unused data (containers, images, cache)
- `docker volume prune` — Remove unused volumes
- `docker network prune` — Remove unused networks
- `docker system df` — Show Docker disk usage

---

## 📝 Dockerfile Instructions
- `FROM` — Set base image
- `RUN` — Execute commands during build
- `COPY` — Copy files into image
- `WORKDIR` — Set working directory
- `EXPOSE` — Define container port
- `CMD` — Default command to run container
- `ENTRYPOINT` — Define fixed executable