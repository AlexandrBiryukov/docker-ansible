# Задание 7 (task7): HTTPS, балансировщик, централизованные логи и метрики для Kanban

## Цель
Доработать проект **task6**:
1) Вынести фронтенд/бэкенд за **Nginx HTTPS reverse‑proxy** (или **LB** с двумя фронтами).  
2) Включить **централизованные логи** (Loki+Promtail).  
3) Поднять **метрики** (Prometheus + cAdvisor + Nginx Exporter) и **Grafana** с готовым дашбордом.

---

## Что добавлено в репозиторий
- `docker-compose.proxy.yml` — Nginx как HTTPS reverse‑proxy (`app.local`).
- `nginx/app-https.conf` — HTTPS-конфиг; редирект 80→443; `/stub_status` на :81 для метрик.
- `nginx/make-certs.sh` — генерация self‑signed сертификатов для `app.local`.
- `docker-compose.observability.yml` — стек наблюдаемости (Prometheus, cAdvisor, Loki, Promtail, Grafana, Nginx Exporter).
- `observability/prometheus.yml` — скрейп Prometheus.
- `observability/loki-config.yml` — конфиг Loki.
- `observability/promtail-config.yml` — конфиг Promtail (Docker discovery; путь к json‑логам из метаданных).
- `observability/grafana/provisioning/datasources/datasources.yml` — датасорсы Grafana (Prometheus, Loki).
- `observability/grafana/provisioning/dashboards/app-observability.json` — дашборд Grafana:  
  Nginx, контейнеры (CPU/Memory), логи (Loki), переменные `project/log_service` с поддержкой **All**.

> Для режима **LB** используется ваш `docker-compose.lb.yml` + `lb.conf` (там добавлен HTTP `listen 81` со `stub_status`).

---

## Подготовка (общая)
**hosts** (Windows: `C:\Windows\System32\drivers\etc\hosts`, Linux/Mac: `/etc/hosts`):
```
127.0.0.1 app.local
```
**TLS‑сертификаты**:
```bash
bash ./nginx/make-certs.sh
```

---

## Запуск: Вариант A — HTTPS reverse‑proxy (рекомендуется для одиночного фронта)
```bash
docker compose -f docker-compose.yml -f docker-compose.proxy.yml -f docker-compose.observability.yml up -d --build
```

Проверки (одной строкой):
```bash
curl -kI https://app.local/ && curl -ks https://app.local/api/tasks/ | head && curl -s http://localhost:9113/metrics | head && curl -s http://localhost:9090/api/v1/targets | grep -E '"health":"up"' || true
```

---

## Запуск: Вариант B — LB с двумя фронтендами
**LB (`lb.conf`) должен содержать HTTP‑сервер для метрик:**
```nginx
server {
    listen 81;
    server_name _;
    location = /stub_status { stub_status on; access_log off; }
}
```
Запуск:
```bash
docker compose -f docker-compose.yml -f docker-compose.lb.yml -f docker-compose.observability.yml up -d --build
```
Если порт `8080` занят одиночным `kanban-ui`:
```bash
docker compose -f docker-compose.yml -f docker-compose.lb.yml -f docker-compose.observability.yml stop kanban-ui && docker compose -f docker-compose.yml -f docker-compose.lb.yml -f docker-compose.observability.yml rm -f kanban-ui
```
Проверки (одной строкой):
```bash
curl -sI http://localhost:8080/ && curl -s http://localhost:8080/api/tasks/ | head && curl -s http://localhost:9113/metrics | head && curl -s http://localhost:9090/api/v1/targets | grep -E '"health":"up"' || true
```

---

## Grafana / Prometheus / Loki
- **Grafana**: `http://localhost:3000` (admin / admin) → папка **App Monitoring** → дашборд **Task6: Nginx + Containers + Logs**.  
  Вверху поставьте переменные: `project = All`, `log_service = All`.
- **Prometheus**: `http://localhost:9090/targets` — все цели должны быть **UP**.  
- **Loki**: `http://localhost:3100/metrics` или API‑запрос:
```bash
curl -sG 'http://localhost:3100/loki/api/v1/query' --data-urlencode 'query={compose_project=~".+"}' | head
```

### Быстрые запросы
**PromQL (через Grafana/Prometheus):**
```text
nginx_up
rate(nginx_http_requests_total[5m])
nginx_connections_active
sum by (container_label_com_docker_compose_service) (rate(container_cpu_usage_seconds_total{container!=""}[2m]))
max by (container_label_com_docker_compose_service) (container_memory_usage_bytes{container!=""})
```
**LogQL (через Grafana Explore / API):**
```text
{compose_project=~".+"}
{compose_project=~".+", service=~"kanban-.*"}
```

---

## Частые ошибки и решения ( one‑liner fix )
1) **Смешали proxy и LB файлы** → «depends on undefined service».  
   ➤ Выберите один режим и запускайте **всегда** с одинаковым набором `-f`:
```bash
# proxy:   -f docker-compose.yml -f docker-compose.proxy.yml -f docker-compose.observability.yml
# LB:      -f docker-compose.yml -f docker-compose.lb.yml -f docker-compose.observability.yml
```
2) **nginx‑exporter падает из‑за HTTPS‑редиректа**.  
   ➤ Скрейпим внутренний HTTP `:81` без TLS:
```yaml
# docker-compose.observability.yml
nginx-exporter:
  command: ["-nginx.scrape-uri=http://kanban-lb:81/stub_status"]  # или http://nginx:81/… в proxy‑режиме
```
3) **Loki 400: `=~".*"` запрещён**.  
   ➤ Используйте `=~".+"` или точное `=`; в дашборде `AllValue` = `.+`.
4) **Promtail не видит логи**.  
   ➤ Проверьте маунты и путь:
```bash
# volumes у promtail
- /var/run/docker.sock:/var/run/docker.sock:ro
- /var/lib/docker/containers:/var/lib/docker/containers:ro
```
   ➤ В `promtail-config.yml` путь из метаданных:
```yaml
- source_labels: ["__meta_docker_container_log_path"]
  target_label: "__path__"
```
5) **В лог‑панели Grafana «no data» / переменная пустая**.  
   ➤ Переменным включить **includeAll** + **allValue=".+"**; использовать в выражении `${var:regex}`.  
   ➤ Добавить не пустой матчер, напр. `container=~".+"` в запрос панели.
6) **Порт 8080 занят**.  
   ➤ Остановить одиночный `kanban-ui` **или** поменять порт LB:
```bash
sed -i 's/8080:80/8082:80/' docker-compose.lb.yml
```

---

## Приёмка (что приложить при сдаче)
1) Скриншоты:
   - Grafana → дашборд **Task6: Nginx + Containers + Logs** с данными.
   - Prometheus → `/targets` (все цели `UP`).
   - `https://app.local/` и `…/api/tasks/` (proxy‑режим) **или** `http://localhost:8080/` (LB‑режим).
2) Логи/метрики доступны локально:  
   - `http://localhost:9113/metrics` (nginx exporter),  
   - `http://localhost:3100/metrics` (Loki),  
   - `http://localhost:9090` (Prometheus).
3) Команды запуска (скопируйте из разделов «Запуск», **одной строкой**).

---

## Дополнительно (по желанию)
- Метрики приложения (Spring Boot): добавить зависимости `spring-boot-starter-actuator`, `micrometer-registry-prometheus`, включить `/actuator/prometheus`, добавить джоб в `prometheus.yml`.
- Алерты Prometheus (5xx, nginx_up==0, OOM, high CPU).  
- Разделение логов по уровням (pipeline в promtail) и ретеншн Loki.
