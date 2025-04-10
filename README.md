# Початкове безпечне налаштування сервера Ubuntu 24.04

## 1. Оновлення системи

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
sudo apt autoclean
```

---

## 2. Створення користувача з правами sudo

```bash
adduser alex
usermod -aG sudo alex
```

---

## 3. Налаштування SSH

### Генерація ключів на локальній машині

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub alex@your_server_ip
```
### Виставляємо правильні права доступу
## Чому це важливо?

SSH при підключенні перевіряє права доступу до файлів. Якщо вони занадто відкриті — SSH просто не дозволить підключення.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### Налаштування /etc/ssh/sshd_config

```conf
PasswordAuthentication no
PermitRootLogin no
AllowUsers alex
Port 2222  # опційно
```

### Перезапуск SSH

```bash
sudo systemctl restart sshd
```

---

## 4. Налаштування UFW (фаєрвол)

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp  # ваш SSH порт
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
sudo ufw status verbose
```

---

## 5. Встановлення Fail2Ban

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

### Основні параметри [DEFAULT]

```conf
ignoreip = 127.0.0.1/8 ::1 your_ip
bantime = 1h
findtime = 10m
maxretry = 5
banaction = ufw
```

### Налаштування для SSH

```conf
[sshd]
enabled = true
port = 2222
logpath = %(sshd_log)s
maxretry = 5
findtime = 10m
bantime = 1h
```

### Перезапуск Fail2Ban

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## 6. Автоматичні оновлення безпеки

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 7. AppArmor

### Перевірка статусу

```bash
sudo aa-status
```

---

## 8. Резервне копіювання

- Зберігати копії налаштованих конфігів:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.orig
cp /etc/fail2ban/jail.local /etc/fail2ban/jail.local.orig
```

- Використовувати rsync, duplicity, borg для бекапів даних та конфігів.

---

## Рекомендації

- Слідкувати за логами:

```bash
sudo journalctl -u ssh -p WARN
sudo zgrep "Ban" /var/log/fail2ban.log*
```

- Робити регулярні бекапи
- Постійно оновлювати систему
- Моніторити нові сервіси та налаштовувати захист

---

### Обов’язково мати emergency plan
Що робити якщо трапилась одна з цих проблем:
- Впав сервер.
- DDoS атака чи хакери.
- Пропали бекапи.
- Згорів хард.

> Безпечний сервер = регулярне оновлення + багаторівневий захист + резервне копіювання

---

