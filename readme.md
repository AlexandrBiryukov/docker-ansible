
# 🛠 Домашнее задание: Логгер в Linux

## 📌 Цель

Создать Bash-скрипт, который:
- Создаёт папку `/opt/app`, если её нет
- Создаёт файл `/opt/app/log.txt`, если его нет
- Каждые 17 секунд записывает в файл случайную строку длиной от 1 до 20 символов
- (*Опционально*) Добавляется в автозагрузку через `systemd`

---

## 📁 1. Создание скрипта

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
       mkdir /opt/app
   fi

   if test -f "/opt/app/log.txt"; then
       echo "Файл существует. Пропускаем создание .."
   else
       touch /opt/app/log.txt
   fi

   while true; do
       number=$(shuf -i 1-20 -n 1)
       tr -dc '[:graph:]' </dev/urandom | fold -w $number | head -n 1 >> /opt/app/log.txt
       sleep 17
   done
   ```

4. Сделайте скрипт исполняемым:
   ```bash
   chmod +x /home/alex/task3.sh
   ```

---

## ⚙️ 2. Настройка автозагрузки через `systemd`

1. Создайте unit-файл:
   ```bash
   sudo nano /etc/systemd/system/task3.service
   ```

2. Вставьте содержимое:

   ```ini
   [Unit]
   Description=Запуск скрипта логгера при старте системы
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

## 📎 Полезные команды

| Команда                                  | Описание                          |
|------------------------------------------|-----------------------------------|
| `sudo systemctl start task3.service`     | Запуск сервиса                    |
| `sudo systemctl stop task3.service`      | Остановка сервиса                 |
| `sudo systemctl restart task3.service`   | Перезапуск сервиса                |
| `sudo systemctl status task3.service`    | Проверка статуса                  |
| `sudo journalctl -u task3.service`       | Просмотр логов сервиса            |

---

## 🧪 Проверка результата

После запуска сервиса проверьте содержимое файла `/opt/app/log.txt`. В нём каждые 17 секунд должна появляться случайная строка.

```bash
tail -f /opt/app/log.txt
```

---

✅ **Готово!** Теперь ваш скрипт работает в фоне и запускается при старте системы.
