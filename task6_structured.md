
# 🚀 Проект: Развёртывание Kanban-приложения с помощью Docker Compose

---

## 🎯 Цель задания

Развернуть kanban-приложение, состоящее из:
- **Frontend**: TypeScript/HTML/CSS/JavaScript (собирается и запускается через `serve`)
- **Backend**: Spring Boot + PostgreSQL + Liquibase
- **Балансировка и проксирование**: NGINX

---

## Настройка окружения

### 1. Установка Docker и docker-compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

### 2. Добавление текущего пользователя в группу docker

```bash
sudo usermod -aG docker $USER
newgrp docker  # либо перезайдите
```

### 3. Клонирование проектов

```bash
git clone https://gitlab.com/astn-dvps/kanban-frontend.git
git clone https://gitlab.com/astn-dvps/kanban-backend.git
```

---

## 🐳 docker-compose.yml

```yaml
version: '3'

services:
  frontend:
    build: ../kanban-frontend
    expose:
      - "80"
    deploy:
      replicas: 2
    networks:
      - appnet

  backend:
    build: ../kanban-backend
    ports:
      - "8081:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://kanban-postgres:5432/kanban
      SPRING_DATASOURCE_USERNAME: kanban
      SPRING_DATASOURCE_PASSWORD: kanban
    depends_on:
      - kanban-postgres
    networks:
      - appnet

  kanban-postgres:
    image: postgres:14
    container_name: kanban-postgres
    restart: always
    environment:
      POSTGRES_DB: kanban
      POSTGRES_USER: kanban
      POSTGRES_PASSWORD: kanban
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    networks:
      - appnet

volumes:
  pgdata:

networks:
  appnet:
```

---

## 🛠 Dockerfile для frontend (kanban-frontend)

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install --legacy-peer-deps && npm install -g serve
RUN npm run build || true
EXPOSE 80
CMD ["serve", "-s", "build", "-l", "80"]
```

---

## 🛠 Dockerfile для backend (kanban-backend)

```Dockerfile
# Сборка
FROM maven:3.8.7-eclipse-temurin-11 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Запуск
FROM eclipse-temurin:11
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

---

## 📄 application.properties (внутри backend)

```properties
server.port=8080
server.servlet.context-path=/api
spring.datasource.url=jdbc:postgresql://kanban-postgres:5432/kanban
spring.datasource.username=kanban
spring.datasource.password=kanban
spring.liquibase.change-log=classpath:/db/changelog/db.changelog-master.xml
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation=true
```

---

## 🌐 nginx.conf (в task6/)

```nginx
events {}

http {
  upstream frontend_cluster {
    server frontend:80;
    server frontend:80;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://frontend_cluster;
    }
  }
}
```

---

## ▶️ Запуск проекта

```bash
cd task6
sudo docker-compose down -v
sudo docker-compose up --build
```

---

## 🔍 Проверка

| Компонент   | URL                              |
|-------------|-----------------------------------|
| Frontend    | http://localhost:8080            |
| Backend API | http://localhost:8081/api        |

---

