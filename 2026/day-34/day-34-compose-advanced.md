# Task 1: Build Your Own App Stack
```Docker
services:
  flask:
    build: .
    container_name: web-app
    environment:
      - DATABASE_URL=mysql+pymysql://root:test@123@3306/mydb
      - REDIS_URL=redis://cache:6379
    networks:
      - myapp-nw
    ports:
      - "5000:5000"
    depends_on:
      - mysql
      - cache
    restart: unless-stopped
  mysql:
    image: mysql:8
    container_name: mysql
    ports:
      - "3306:3306"
    networks:
      - myapp-nw
    environment:
      MYSQL_ROOT_PASSWORD: test@123
      MYSQL_DATABASE: mydb
    restart: on-failure
    volumes:
      - mysql-database:/var/lib/mysql
  cache:
    image: redis:7
    container_name: cache
    ports:
      - "6379:6379"
    networks:
      - myapp-nw
volumes:
  mysql-database:
networks:
  myapp-nw:
    driver: bridge
```
# Task 2: depends_on & Healthchecks
```docker
services:
  flask:
    build: .
    container_name: web-app
    environment:
      - DATABASE_URL=mysql+pymysql://root:test@123@3306/mydb
      - REDIS_URL=redis://cache:6379
    networks:
      - myapp-nw
    ports:
      - "5000:5000"
    depends_on:
      - mysql
      - cache
    restart: unless-stopped
  mysql:
    image: mysql:8
    container_name: mysql
    ports:
      - "3306:3306"
    networks:
      - myapp-nw
    environment:
      MYSQL_ROOT_PASSWORD: test@123
      MYSQL_DATABASE: mydb
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ptest@123"]
      interval: 5s
      timeout: 3s
      retries: 10
    volumes:
      - mysql-database:/var/lib/mysql
  cache:
    image: redis:7
    container_name: cache
    ports:
      - "6379:6379"
    networks:
      - myapp-nw
volumes:
  mysql-database:
networks:
  myapp-nw:
    driver: bridge
    
```

Task 3: Restart Policies
```
services:
  flask:
    build: .
    container_name: web-app
    environment:
      - DATABASE_URL=mysql+pymysql://root:test@123@3306/mydb
      - REDIS_URL=redis://cache:6379
    networks:
      - myapp-nw
    ports:
      - "5000:5000"
    depends_on:
      - mysql
      - cache
    restart: on-failure
  mysql:
    image: mysql:8
    container_name: mysql
    ports:
      - "3306:3306"
    networks:
      - myapp-nw
    environment:
      MYSQL_ROOT_PASSWORD: test@123
      MYSQL_DATABASE: mydb
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ptest@123"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: on-failure
    volumes:
      - mysql-database:/var/lib/mysql
  cache:
    image: redis:7
    container_name: cache
    ports:
      - "6379:6379"
    networks:
      - myapp-nw
volumes:
  mysql-database:
networks:
  myapp-nw:
    driver: bridge
```
# 🔹 Why restart: always didn’t restart it

Docker interprets exit code 0 as a clean/intentional exit.
- restart: always does not restart containers that exit with code 0 when killed manually using docker stop or docker kill.
- Only if the container crashes with a non-zero exit code, Docker will automatically restart it under restart: on-failure.
- So when you manually “kill” or “stop” a container, Docker assumes you intentionally stopped it — it won’t bring it back automatically even with restart: always.
- 

# Task 4: Custom Dockerfiles in Compose

Rebuild and restart with one command
```
docker compose -f Docker-compose.yml up -d --build
```
