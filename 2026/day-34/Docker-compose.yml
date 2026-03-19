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
    restart: always
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
