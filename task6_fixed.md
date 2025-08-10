# 🚀 Проект: Развёртывание Kanban‑приложения с Docker Compose (исправленная версия)

> Основано на вашем исходном описании, но с исправлениями под старый Angular/`node-sass`, корректным проксированием `/api` и CORS на бэкенде.  

---

## 🎯 Цель задания
Развернуть kanban‑приложение из двух сервисов и БД:
- **Frontend** — Angular 7, отдаётся Nginx.
- **Backend** — Spring Boot + Liquibase.
- **DB** — PostgreSQL.
- Фронт доступен на **http://localhost:8080**, API — на **http://localhost:8081**.  
- (Опционально) поднять **2 экземпляра фронта** и балансировщик Nginx перед ними.

---

## 🧰 Подготовка окружения
1) Установите Docker + Compose v2 (`docker compose version`).  
2) Разложите проект так:
```
task6/
 ├─ kanban-backend
 ├─ kanban-frontend
 ├─ docker-compose.yml
 ├─ docker-compose.lb.yml        # опционально
 └─ lb.conf                      # опционально
```
3) Если в Linux запускаете Docker без sudo:
```bash
sudo usermod -aG docker $USER && newgrp docker
```

---

## 🐳 Dockerfile’ы (исправленные)

### Backend — `kanban-backend/Dockerfile`
```dockerfile
# ------------ Build stage ------------
FROM maven:3.8.8-eclipse-temurin-8 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

# ------------ Runtime stage ------------
FROM openjdk:8-jre-slim
WORKDIR /app
COPY --from=build /app/target/*.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### Frontend — `kanban-frontend/Dockerfile`
> Главная правка: используем **Node 12** и **`npm install`**, а не `npm ci` (иначе падает сборка `node-sass`). Плюс ставим `python3 make g++` для node‑gyp.
```dockerfile
# ------------ Build stage ------------
FROM node:12-bullseye AS build
WORKDIR /app
RUN apt-get update && apt-get install -y python3 make g++ && rm -rf /var/lib/apt/lists/*

COPY package*.json ./
RUN npm install --legacy-peer-deps --unsafe-perm --no-audit --no-fund
COPY . .
RUN npm run build

# ------------ Runtime stage ------------
FROM nginx:1.25-alpine
COPY --from=build /app/dist/kanban-ui /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`.dockerignore` рядом:
```
node_modules
dist
.git
```

---

## 🌐 Nginx фронта — `kanban-frontend/default.conf`
> Весь `/api/` проксируем на бэкенд **без добавления хвоста `/api`** — у бэка уже `server.servlet.context-path=/api`.
```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        proxy_pass http://kanban-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # Альтернатива правке CORS в бэкенде (если нужно):
        # proxy_set_header Origin "";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## ⚙️ Angular окружения
Убедитесь, что **в обоих** env базовый URL = `/api`:
```ts
// src/environments/environment.ts
export const environment = { production: false, kanbanAppUrl: '/api' };
// src/environments/environment.prod.ts
export const environment = { production: true,  kanbanAppUrl: '/api' };
```

---

## 🔒 CORS на бэкенде (обязательно)
В контроллерах `KanbanController` и `TaskController` замените аннотацию:
```java
// было:
// @CrossOrigin(origins = "http://localhost:4200")

// стало (любой из вариантов):
@CrossOrigin(origins = "*")
// или явно:
@CrossOrigin(origins = {"http://localhost:8080", "http://127.0.0.1:8080", "http://192.168.1.31:8080"})
```
> Без этой правки браузер будет отдавать **403** на `POST/PUT` из UI.

> Альтернатива: оставить Java как есть и добавить в Nginx `proxy_set_header Origin ""` в блоке `location /api/` — тогда фильтр CORS в Spring не сработает.

---

## 🧩 Compose (база) — `task6/docker-compose.yml`
```yaml
version: "3.9"

services:
  kanban-postgres:
    image: postgres:13-alpine
    container_name: kanban-postgres
    environment:
      POSTGRES_DB: kanban
      POSTGRES_USER: kanban
      POSTGRES_PASSWORD: kanban
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kanban -d kanban"]
      interval: 10s
      timeout: 5s
      retries: 5

  kanban-app:
    build: ./kanban-backend
    container_name: kanban-app
    depends_on:
      kanban-postgres:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://kanban-postgres:5432/kanban
      SPRING_DATASOURCE_USERNAME: kanban
      SPRING_DATASOURCE_PASSWORD: kanban
    ports:
      - "8081:8080"   # API -> http://localhost:8081

  kanban-ui:
    build: ./kanban-frontend
    container_name: kanban-ui
    depends_on: [ kanban-app ]
    ports:
      - "8080:80"     # Front -> http://localhost:8080

volumes:
  pgdata:
```

> **Почему без `deploy.replicas`:** этот ключ работает в Swarm; в обычном Docker Compose сделаем 2 фронта в отдельном файле (ниже).

---

## ⚖️ (Опционально) 2 фронта + балансировщик — `docker-compose.lb.yml` и `lb.conf`
`docker-compose.lb.yml`:
```yaml
version: "3.9"
services:
  kanban-postgres:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: kanban
      POSTGRES_USER: kanban
      POSTGRES_PASSWORD: kanban
    volumes: [ "pgdata:/var/lib/postgresql/data" ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kanban -d kanban"]
      interval: 10s
      timeout: 5s
      retries: 5

  kanban-app:
    build: ./kanban-backend
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://kanban-postgres:5432/kanban
      SPRING_DATASOURCE_USERNAME: kanban
      SPRING_DATASOURCE_PASSWORD: kanban
    ports: [ "8081:8080" ]
    depends_on:
      kanban-postgres:
        condition: service_healthy

  kanban-ui-1:
    build: ./kanban-frontend
    depends_on: [ kanban-app ]

  kanban-ui-2:
    build: ./kanban-frontend
    depends_on: [ kanban-app ]

  kanban-lb:
    image: nginx:1.25-alpine
    volumes:
      - ./lb.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: [ kanban-ui-1, kanban-ui-2, kanban-app ]
    ports: [ "8080:80" ]

volumes:
  pgdata:
```

`lb.conf`:
```nginx
upstream ui_upstream {
    server kanban-ui-1:80;
    server kanban-ui-2:80;
}
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://ui_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /api/ {
        proxy_pass http://kanban-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # proxy_set_header Origin "";
    }
}
```

---

## ▶️ Запуск

**База:**
```bash
cd task6
docker compose up -d --build
# Front: http://localhost:8080
# API:   http://localhost:8081
```

**Со звёздочкой (LB + 2 UI):**
```bash
docker compose -f docker-compose.lb.yml up -d --build
# Открывать http://localhost:8080
```

Остановить и удалить данные БД (если нужно):
```bash
docker compose down
# docker compose down -v && docker volume rm task6_pgdata
```

---

## ✅ Проверка

API жив?
```bash
curl -s http://localhost:8081/api/actuator/health
```

Создаём доску и задачу:
```bash
curl -i -H "Content-Type: application/json"   -d '{"title":"BOARD1"}' http://localhost:8081/api/kanbans/

curl -i -H "Content-Type: application/json"   -d '{"title":"Demo","description":"from curl","color":"#ff0","status":"TODO"}'   http://localhost:8081/api/kanbans/1/tasks/
```

Списки:
```bash
curl -i http://localhost:8081/api/kanbans/
curl -i http://localhost:8081/api/tasks/   # со слэшем!
```

---

## 🧯 Частые проблемы и быстрые решения
- **Сборка фронта падает на `npm ci` / `node-sass`.**  
  Используйте `node:12` и `npm install --legacy-peer-deps --unsafe-perm` (как в Dockerfile).
- **В UI `POST/PUT` → 403.**  
  Это CORS. Добавьте `@CrossOrigin(origins = "*")` (или свой Origin) в контроллерах **или** `proxy_set_header Origin ""` в Nginx для `/api/` и пересоберите.
- **`kanban-app` не стартует из-за `service_healthy`.**  
  Удалите том с БД и поднимите заново:  
  `docker compose down -v && docker volume rm task6_pgdata && docker compose up -d --build`.
- **`GET /api/tasks` без слэша возвращает 400.**  
  Это «поиск по title». Для списка используйте **`/api/tasks/`** со слэшем.

Удачной сдачи! 🚀
