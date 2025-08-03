# ⚙️ Домашнее задание: Ansible + Nginx + VM

## 📋 Цель
Автоматизировать развёртывание Nginx на виртуальной машине с помощью Ansible.

---

## 🧱 Шаг 1: Установка Ansible

```bash
sudo apt update
sudo apt install ansible -y
```

Проверка:
```bash
ansible --version
```

---

## 💻 Шаг 2: Установка ПО для виртуальных машин

Установите VirtualBox или другое средство виртуализации:

```bash
sudo apt install virtualbox -y
```

Создайте новую виртуальную машину (например, Ubuntu Server). Убедитесь, что она доступна по SSH.

Пример настройки IP:
- IP: `192.168.56.10`
- Пользователь: `vagrant`
- Пароль или ключ: `vagrant`

---

## 📁 Шаг 3: Структура ansible-проекта

Создайте папку `task5` и добавьте туда следующие файлы:

```text
task5/
├── inventory.ini
├── playbook.yml
└── files/
    └── index.html
```

---

## 📄 inventory.ini

```ini
[web]
192.168.56.10 ansible_user=vagrant ansible_password=vagrant ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## 📄 playbook.yml

```yaml
- name: Установка и настройка Nginx
  hosts: web
  become: true

  tasks:
    - name: Обновление системных пакетов
      apt:
        update_cache: yes

    - name: Установка Nginx
      apt:
        name: nginx
        state: present

    - name: Копирование кастомной index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
```

---

## 📝 index.html

Создайте простой HTML-файл `task5/files/index.html`:
```html
<!DOCTYPE html>
<html>
<head><title>Успешно!</title></head>
<body><h1>Установка через Ansible прошла успешно 🚀</h1></body>
</html>
```

---

## ▶️ Запуск плейбука

```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## ✅ Проверка

Откройте в браузере: [http://192.168.56.10](http://192.168.56.10) — вы должны увидеть кастомную страницу.

---

**Готовый плейбук должен быть сохранён в директории `task5/`.**