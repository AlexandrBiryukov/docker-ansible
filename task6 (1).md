# Task 6 — Docker + Compose (Kanban)

## Цель
Поднять демо‑приложение (Angular фронтенд + Spring Boot бэкенд + PostgreSQL) в Docker.  
Фронт должен открываться на **http://localhost:8080**, бэк — на **http://localhost:8081**.  
(Опционально) — два фронта за балансировщиком Nginx.

---

## Требования
- Docker Engine / Docker Desktop и Docker Compose v2.
- Распакованный архив проекта:  
  ```
  task6/
   ├─ kanban-backend    (Spring Boot)
   ├─ kanban-frontend   (Angular 7)
   └─ docker-compose*.yml, lb.conf, README.md (наши файлы)
  ```

---

## Финальные конфигурации

### 1) Backend — `kanban-backend/Dockerfile`
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

> Бэкенд слушает порт **8080** внутри контейнера и доступен с хоста по **http://localhost:8081** (см. compose).  
> В контроллерах разрешён CORS для UI (см. ниже).

**CORS (важно!)** — в `KanbanController` и `TaskController` используем:
```java
@CrossOrigin(origins = "*")
```
или:
```java
@CrossOrigin(origins = {"http://localhost:8080", "http://127.0.0.1:8080"})
```
(если фронт открывается по IP — добавьте его, например `http://192.168.1.31:8080`).

---

### 2) Frontend — `kanban-frontend/Dockerfile`
```dockerfile
# ------------ Build stage ------------
FROM node:12-bullseye AS build
WORKDIR /app

# инструменты для сборки старого node-sass
RUN apt-get update && apt-get install -y python3 make g++ && rm -rf /var/lib/apt/lists/*

COPY package*.json ./
# важно: используем npm install, а не npm ci
RUN npm install --legacy-peer-deps --unsafe-perm --no-audit --no-fund

COPY . .
RUN npm run build

# ------------ Runtime stage ------------
FROM nginx:1.25-alpine
COPY --from=build /app/dist/kanban-ui /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

`.dockerignore` в той же папке:
```
node_modules
dist
.git
```

**Окружение Angular**  
Убедитесь, что и dev, и prod указывают на `/api`:
`src/environments/environment.ts`:
```ts
export const environment = {
  production: false,
  kanbanAppUrl: '/api'
};
```
`src/environments/environment.prod.ts` уже содержит:
```ts
export const environment = {
  production: true,
  kanbanAppUrl: '/api'
};
```

**Nginx фронта** — `kanban-frontend/default.conf`:
```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Весь API идёт на backend (context-path /api уже есть на бэке)
    location /api/ {
        proxy_pass http://kanban-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # Если не хотите править CORS в Java — можно "спрятать" Origin:
        # proxy_set_header Origin "";
    }

    # SPA роутинг
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

### 3) Compose — базовый запуск (`task6/docker-compose.yml`)
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
      - "8081:8080"   # backend -> http://localhost:8081

  kanban-ui:
    build: ./kanban-frontend
    container_name: kanban-ui
    depends_on:
      - kanban-app
    ports:
      - "8080:80"     # frontend -> http://localhost:8080

volumes:
  pgdata:
```

---

### 4) (Опционально) Два фронта + балансировщик (`task6/docker-compose.lb.yml`)
```yaml
version: "3.9"

services:
  kanban-postgres:
    image: postgres:13-alpine
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
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://kanban-postgres:5432/kanban
      SPRING_DATASOURCE_USERNAME: kanban
      SPRING_DATASOURCE_PASSWORD: kanban
    ports:
      - "8081:8080"
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
    depends_on:
      - kanban-ui-1
      - kanban-ui-2
      - kanban-app
    ports:
      - "8080:80"

volumes:
  pgdata:
```

`task6/lb.conf`:
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

## Запуск

### Базовый вариант
```bash
cd task6
docker compose up -d --build
# Фронт:  http://localhost:8080
# Бэк:    http://localhost:8081
```

### Вариант со звёздочкой (LB + 2 UI)
```bash
cd task6
docker compose -f docker-compose.lb.yml up -d --build
# Открывать http://localhost:8080
```

Остановить:
```bash
docker compose down              # или -f docker-compose.lb.yml down
# Удалить данные БД (если нужно):
# docker volume rm task6_pgdata
```

---

## Проверка

Бэкенд:
```bash
curl -i http://localhost:8081/api/actuator/health           # {"status":"UP"}
curl -i http://localhost:8081/api/kanbans/                  # список досок
curl -i http://localhost:8081/api/tasks/                    # список задач
```

Создать доску и задачу:
```bash
curl -i -H "Content-Type: application/json" \
  -d '{"title":"BOARD1"}' http://localhost:8081/api/kanbans/

curl -i -H "Content-Type: application/json" \
  -d '{"title":"Demo","description":"from curl","color":"#ff0","status":"TODO"}' \
  http://localhost:8081/api/kanbans/1/tasks/
```

Фронт через LB:
```bash
curl -i http://localhost:8080/api/kanbans/
curl -i http://localhost:8080/api/tasks/
```

---

## Типичные проблемы и решения

- **`npm ci` падает / `node-sass` не поддерживает версию Node.**  
  Используйте `node:12-bullseye` и `npm install --legacy-peer-deps --unsafe-perm`.

- **`POST/PUT` дают `403` через UI.**  
  Это CORS. Расширьте `@CrossOrigin` в контроллерах или в Nginx для `/api/` добавьте `proxy_set_header Origin "";` и пересоберите фронт.

- **`kanban-app` не стартует, зависимость `service_healthy`.**  
  Удалите старый volume и поднимите заново:
  ```bash
  docker compose down -v
  docker volume rm task6_pgdata 2>/dev/null || true
  docker compose up -d --build
  ```

- **`GET /api/tasks/` «думает».**  
  Проверьте, что обращаетесь на правильный IP/порт. `GET /api/tasks` без слэша — это поиск по `title` и вернёт 400, нужен **слэш**: `/api/tasks/`.

- **Страница не отправляет запрос при Create Task.**  
  Убедитесь, что открыта страница конкретной доски `/kanban/1`, заполнены поля в диалоге, и в `Network` виден `POST /api/kanbans/1/tasks/` (201).

---

## Краткий чек‑лист для сдачи
- [x] `kanban-backend/Dockerfile` и `kanban-frontend/Dockerfile` созданы и собираются.
- [x] `docker-compose.yml` поднимает 3 сервиса, фронт доступен на `8080`, бэк — на `8081`.
- [x] Через фронт можно создать задачу (Create Task) — в логах видно `POST /api/kanbans/:id/tasks/` → 201.
- [x] (Опционально) `docker-compose.lb.yml` + `lb.conf` запускают 2 UI и балансировщик на `8080`.

Удачи! 🚀
