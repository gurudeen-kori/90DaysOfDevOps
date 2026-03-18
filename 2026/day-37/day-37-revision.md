## Self-Assessment Checklist
Mark yourself honestly — **can do**, **shaky**, or **haven't done**:

- [x] Run a container from Docker Hub (interactive + detached)
- [x] List, stop, remove containers and images
- [x] Explain image layers and how caching works
- [x] Write a Dockerfile from scratch with FROM, RUN, COPY, WORKDIR, CMD
- [ ] Explain CMD vs ENTRYPOINT
- [x] Build and tag a custom image
- [x] Create and use named volumes
- [x] Use bind mounts
- [x] Create custom networks and connect containers
- [x] Write a docker-compose.yml for a multi-container app
- [x] Use environment variables and .env files in Compose
- [x] Write a multi-stage Dockerfile
- [x] Push an image to Docker Hub
- [x] Use healthchecks and depends_on
- 
# 🚀 Docker Quick-Fire Answers

## 1. Difference between an Image and a Container
- An image is a read-only template that contains the application code, dependencies, and environment setup.  
* A container is a running instance of that image.

---

## 2. What happens to data inside a container when you remove it?
- Data inside a container is ephemeral and will be lost when the container is removed.  
- To persist data, volumes or bind mounts should be used.

---

## 3. How do two containers on the same custom network communicate?
- Containers on the same custom network can communicate using their container names.  
- Docker provides built-in DNS resolution for this.

---

## 4. Difference between `docker compose down` and `docker compose down -v`
- `docker compose down`: Removes containers and networks, but keeps volumes.
- `docker compose down -v`: Removes containers, networks, and volumes (data will be lost).

---

## 5. Why are multi-stage builds useful?
Multi-stage builds help reduce the final image size by excluding unnecessary build dependencies.  
They also improve security and efficiency.

---

## 6. Difference between COPY and ADD
- `COPY`: Copies files from local system to container (recommended for most cases).
- `ADD`: Can also download files from URLs and automatically extract compressed files.

---

## 7. What does `-p 8080:80` mean?
This maps port 8080 on the host machine to port 80 inside the container.  
Accessing `localhost:8080` will route traffic to the container.

---

## 8. How do you check how much disk space Docker is using?
Use the following command:

```bash
docker system df
```

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