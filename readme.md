# 🛠 Домашнее задание: Скрипт для генерации строк

## 📌 Цель

Создать Bash-скрипт, который:
- Создаёт папку `/opt/app`, если её нет
- Создаёт файл `/opt/app/log.txt`, если его нет
- Каждые 17 секунд записывает в файл случайную строку длиной от 1 до 20 символов
(*Опционально*) Добавить в автозагрузку
(*Опционально*) Настроить ротацию log-файла с помощью logrotate

---

##  1. Создание скрипта

1. Перейдите в домашнюю директорию:
   ```bash
   cd /home/alex
   ```

2. Создайте файл `task3.sh`:
   ```bash
   nano task3.sh
   ```

3. Вставьте следующий код:

   ```bash
   #!/bin/bash

   if test -d "/opt/app"; then
       echo "Папка существует. Пропускаем создание .."
   else
       echo "Создание папки app"
       mkdir /opt/app
       echo "Папка создана."
   fi

   if test -f "/opt/app/log.txt"; then
       echo "Файл существует. Пропускаем создание .."
   else
       echo "Создание файла log.txt"
       touch /opt/app/log.txt
       echo "Файл создан."
   fi

   while true; do
        number=$(shuf -i 1-20 -n 1)
        string=$(tr -dc '[:graph:]' </dev/urandom | head -c $number)
        echo "$string" >> /opt/app/log.txt
        echo "Добавлена строка: $string"
        sleep 17
   done
   ```

4. Сделайте скрипт исполняемым:
   ```bash
   chmod +x /home/alex/task3.sh
   ```

---

##  2. Настройка автозагрузки через `systemd`

1. Создайте unit-файл:
   ```bash
   sudo nano /etc/systemd/system/start_script_task3.service
   ```

2. Вставьте содержимое:

   ```ini
   [Unit]
   Description=Запуск скрипта при старте системы
   After=network.target

   [Service]
   ExecStart=/home/alex/task3.sh
   Restart=on-failure
   Type=simple

   [Install]
   WantedBy=multi-user.target
   ```

3. Обновите конфигурацию systemd:
   ```bash
   sudo systemctl daemon-reexec
   ```

4. Включите автозагрузку:
   ```bash
   sudo systemctl enable task3.service
   ```

5. Запустите вручную (для проверки):
   ```bash
   sudo systemctl start task3.service
   ```

6. Посмотрите статус:
   ```bash
   sudo systemctl status task3.service
   ```

---

## 3. Настройка ротации логов с помощью logrotate

1. Убедитесь, что `logrotate` установлен:
   ```bash
   sudo apt install logrotate
   ```

2. Создайте конфиг для логов:
   ```bash
   sudo nano /etc/logrotate.d/task3
   ```

3. Вставьте следующее содержимое(без комментариев, ругается на них):

   ```conf
   /opt/app/log.txt {
       daily              # ротация каждый день
       rotate 7           # хранить 7 файлов: log.txt.1 ... log.txt.7
       compress           # сжимать старые файлы (gzip)
       missingok          # не ругаться, если файла нет
       notifempty         # не обрабатывать, если файл пустой
       copytruncate       # обрезать текущий файл, не прерывая запись (важно для твоего скрипта!)
   }
   ```

4. Протестируйте вручную:
   ```bash
   sudo logrotate -f /etc/logrotate.d/task3
   ls -lh /opt/app/log.txt*
   ```
### Почему copytruncate важно?
Твой скрипт вечно пишет в log.txt. Без этой опции logrotate переименует файл, а скрипт будет писать в старый, больше неотслеживаемый файл. copytruncate решает это: он копирует содержимое и обнуляет текущий файл, не трогая файловый дескриптор.
---

## Полезные команды

| Команда                                  | Описание                          |
|------------------------------------------|-----------------------------------|
| `sudo systemctl start start_script_task3.service`     | Запуск сервиса                    |
| `sudo systemctl stop start_script_task3.service`      | Остановка сервиса                 |
| `sudo systemctl restart start_script_task3.service`   | Перезапуск сервиса                |
| `sudo systemctl status start_script_task3.service`    | Проверка статуса                  |
| `sudo journalctl -u start_script_task3.service`       | Просмотр логов сервиса            |
| `tail -f /opt/app/log.txt`                            | Наблюдение за логом в реальном времени |

---

## Проверка результата

После запуска сервиса проверьте содержимое файла `/opt/app/log.txt`. В нём каждую секунду должна появляться случайная строка.

```bash
tail -f /opt/app/log.txt
```

Также проверьте, создаются ли архивы:
```bash
ls -lh /opt/app/log.txt*
```

---
