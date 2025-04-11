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
adduser [YOUR_USERNAME]
usermod -aG sudo [YOUR_USERNAME]
```

---

## 3. Налаштування SSH

### Генерація ключів на локальній машині

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub [YOUR_EMAIL_ADDRESS]
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
AllowUsers [YOUR_USERNAME]
Port 2222  # опційно
```

### Перезапуск SSH

```bash
sudo systemctl restart sshd
```

### 2FA/MFA для SSH

#### Навіщо це потрібно

Навіть якщо SSH є доволі хорошим захистом для входу на сервер, він все одно залишається "дверима", які видно зловмисникам. Вони можуть намагатися підібрати пароль (brute-force атаки). Навіть при використанні Fail2ban, додатковий рівень безпеки ніколи не буде зайвим.

Двофакторна аутентифікація (2FA/MFA) додає ще один рівень безпеки, вимагаючи дві умови для входу:

1. Пароль користувача
2. 6-значний код, який змінюється кожні 30 секунд

Без обох елементів аутентифікації доступ до системи буде неможливим.

#### Можливі недоліки

Деяким користувачам може здатися, що це незручно. Для входу необхідно мати під рукою додаток для генерації кодів (authenticator app).

#### Як це працює

У Linux за аутентифікацію відповідає PAM (Pluggable Authentication Modules). Ми налаштуємо PAM для SSH так, щоб під час входу сервер вимагав і пароль, і 6-значний код.

Використовуватимемо модуль від Google: [libpam-google-authenticator](https://github.com/google/google-authenticator-libpam), який реалізує перевірку TOTP (Time-based One-Time Password).

#### Цілі

- Налаштувати 2FA/MFA для всіх підключень по SSH.

#### Зауваження

- Необхідно мати базове розуміння роботи 2FA/MFA та встановлений authenticator app на телефоні.
- За замовчуванням, якщо використовується вхід по SSH ключу — 2FA/MFA вводити не потрібно. Це можна змінити у конфігурації.

#### Посилання

- https://github.com/google/google-authenticator-libpam
- https://en.wikipedia.org/wiki/Linux_PAM
- https://en.wikipedia.org/wiki/Time-based_One-time_Password_algorithm

#### Кроки налаштування

1. Встановити libpam-google-authenticator:

```bash
sudo apt install libpam-google-authenticator
```

2. Виконати команду `google-authenticator` під користувачем, для якого налаштовується 2FA:

```bash
google-authenticator
```

Підтверджуйте всі запитання відповіді "y". Збережіть резервні коди (scratch codes).

3. Зробити резервну копію конфігурації PAM для SSH:

```bash
sudo cp --archive /etc/pam.d/sshd /etc/pam.d/sshd-COPY-$(date +"%Y%m%d%H%M%S")
```

4. Додати в кінець файлу `/etc/pam.d/sshd`:

```
auth       required     pam_google_authenticator.so nullok
```

5. У файлі `/etc/ssh/sshd_config` змінити або додати:

```
ChallengeResponseAuthentication yes
```

6. Перезапустити SSH:

```bash
sudo service sshd restart
```

---

Тепер при підключенні по SSH користувач вводитиме спочатку пароль, а потім 6-значний код з мобільного додатку.

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
ignoreip = 127.0.0.1/8 ::1 [YOUR_IP]
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
Має вивести результат такого типу:

```
apparmor module is loaded.
X profiles are loaded.
Y profiles are in enforce mode.
Z profiles are in complain mode.
```

Перевірити логи:
```bash
sudo journalctl -k | grep apparmor
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

