# Конфигурация виртуальной машины devops-vm

## 1. Параметры машины

- ОС: Ubuntu 26.04.1 LTS
- Оперативная память: 2 ГиБ (по конфигурации VirtualBox; система видит около 1,6 ГиБ)
- Процессоры: 2
- Виртуальный диск: 25 ГБ
- Корневой логический том: около 11,5 ГБ

## 2. Сетевые интерфейсы

- `enp0s3`: NAT, IPv4 `10.0.2.15/24`. Используется для доступа к внешней сети.
- `enp0s8`: Host-only, IPv4 `192.168.56.101/24`. Используется для связи хоста с виртуальной машиной.
- Имя хоста: `devops-vm`
- Полное доменное имя: `devops-vm.devops.local`
- На хосте настроено разрешение имени `devops.local` в адрес `192.168.56.101`.

## 3. Правило проброса портов

- Адрес хоста: `127.0.0.1`
- Порт хоста: `2222`
- Порт гостевой системы: `2222`
- Протокол: TCP
- Назначение: SSH-подключение к виртуальной машине через NAT.

## 4. Учётные записи

- `devops` — основной пользователь для администрирования.
- Группы пользователя: `devops`, `sudo`, `users`.
- Аутентификация SSH: по открытому ключу.
- Пользователь `root` не может подключаться по SSH.
- Пользователь `student` также присутствует в системе (UID 1000).

## 5. Служба SSH

- Порт: `2222`
- Файл дополнительных настроек: `/etc/ssh/sshd_config.d/99-hardening.conf`
- `PermitRootLogin no`
- `PasswordAuthentication no`
- `PubkeyAuthentication yes`
- `PermitEmptyPasswords no`
- `MaxAuthTries 3`
- `LoginGraceTime 30`
- `AllowUsers devops`
- `X11Forwarding no`
- `ClientAliveInterval 300`
- `ClientAliveCountMax 2`
- Служба `ssh.service` включена; `ssh.socket` отключён.
- Конфигурация проверена командой `sudo sshd -t`.

## 6. Правила межсетевого экрана

- UFW активен.
- Политика входящих соединений по умолчанию: `deny`.
- Политика исходящих соединений по умолчанию: `allow`.
- Политика маршрутизируемых соединений: `disabled`.
- Уровень журналирования: `medium`.
- `80/tcp` — `ALLOW IN` (HTTP).
- `443/tcp` — `ALLOW IN` (HTTPS).
- `2222/tcp` — `LIMIT IN` (SSH с ограничением частоты подключений).
- Правила применяются для IPv4 и IPv6.

## 7. Снимки состояния

- `01-clean-install` — исходное состояние после установки ОС.
- `02-keys-configured` — состояние после настройки ключей SSH.
- `03-ssh-hardened` — состояние после усиления безопасности SSH.

## 8. Веб-сервер

- **Пакет:** `nginx` (устанавливается командой `sudo apt install -y nginx`, версия из репозитория Ubuntu 1.28.3).
- **Конфигурация ресурса:** `/etc/nginx/sites-available/devops-site`, активирована ссылкой в `/etc/nginx/sites-enabled/`; стандартный ресурс отключён удалением ссылки `/etc/nginx/sites-enabled/default`.
- **Описание ресурса:** блок на порту 80 с `server_name devops.local` и перенаправлением `return 301 https://$host$request_uri`; блок на порту 443 (`listen 443 ssl`, `ssl_protocols TLSv1.2 TLSv1.3`) с `root /var/www/devops-site`, `error_page 404 /404.html` и журналами `/var/log/nginx/devops-site.access.log` и `devops-site.error.log`.
- **Каталог ресурса:** `/var/www/devops-site`, владелец `devops:devops`, права 755 для каталогов и 644 для файлов (приводятся командой `rsync --chmod=D755,F644`).
- **Сертификат:** `/etc/ssl/certs/devops.crt`, владелец root, права 644.
- **Закрытый ключ:** `/etc/ssl/private/devops.key`, владелец root, права 600.
- **Команда формирования сертификата** (самоподписанный, срок действия 365 дней, имя в поле SAN):

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/devops.key \
      -out /etc/ssl/certs/devops.crt \
      -subj "/CN=devops.local" \
      -addext "subjectAltName=DNS:devops.local"


- **Применение конфигурации:** `sudo nginx -t && sudo systemctl reload nginx`.
- **Доставка содержимого:** `scripts/deploy.sh` (rsync по SSH, проверка `https://devops.local`).
- **Межсетевой экран:** правила для портов 80 и 443 созданы в ПР № 5.

## 9. Проверка конфигурации

Скрипт `scripts/audit.sh` проверяет запрет SSH-входа для root, отключение парольной аутентификации, порт SSH, ограничение попыток входа, активность UFW и политику входящих соединений.

Последний запуск: 6 проверок пройдено, 0 ошибок.
