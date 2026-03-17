services:
  app:
    build: .
    container_name: todo-app
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://todo_user:todo_pass@db:5432/todo_db
    depends_on:
      db:
        condition: service_healthy
    networks:
      - todo-network
    restart: always
  db:
    image: postgres:15-alpine
    container_name: todo-db
    environment:
      POSTGRES_DB: todo_db
      POSTGRES_USER: todo_user
      POSTGRES_PASSWORD: todo_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - todo-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todo_user -d todo_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
volumes:
  postgres_data:

networks:
  todo-network:
    driver: bridge
